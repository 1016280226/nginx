# 笔记二 Nginx + lvs + keepalived + docker 双机主从热备

[TOC]

## 1.lvs

### 1.1 lvs 是什么

> LVS（Linux Virtual Server） 
>
> - 是linux 虚拟服务器。
> - 是一个开源的软件。



### 1.2 lvs 有什么作用

> - 可以**实现传输四层负载均衡**。
>
> - 目前有**三种IP负载均衡技术**（VS/NAT、VS/TUN和VS/DR）。
> - **八种调度算法**（rr,wrr,lc,wlc,lblc,lblcr,dh,sh）。



## 2.Keepalived

### 2.1 keepalived 是什么

> - 是一个类似于Layer2,4,7**交换机制**的软件。
> - 是Linux集群管理中保证**集群高可用**的一个服务软件



### 2.2 keepalived 有什么作用

> - 用来**防止单点故障**。
> - keepalive 软件可以进行**健康检查**，而且能同时实现 **LVS 的高可用性**，解决 LVS 单点故障的问题，其实 keepalive 就是为 LVS 而生的。(LVS可以实现负载均衡，但是不能够进行健康检查，比如一个rs出现故障，LVS 仍然会把请求转发给故障的rs服务器，这样就会导致请求的无效性。)



## 3.lvs + Nginx + keepalived

### 3.1 lvs + Nginx + keepalived 有什么作用

![1554610591645](C:\Users\Calvin\AppData\Roaming\Typora\typora-user-images\1554610591645.png)

> 1. 系统软件的发布。
> 2. 单点故障的转移，实现高可用性。



### 3.2 lvs + Nginx + keepalived + docker 搭建和配置

#### 3.2.1 下载安装docker 

> 教程: 笔记一 docker :<https://github.com/1016280226/docker/blob/master/%E7%AC%94%E8%AE%B0%E4%B8%80%20Docker.md>



#### 3.2.2 下载安装keeplived

```bash
cd /tmp
wget http://www.keepalived.org/software/keepalived-1.2.18.tar.gz    #1.下载keepalived
tar -zxvf keepalived-1.2.18.tar.gz -C /usr/local/                   #2.解压安装
yum install -y openssl openssl-devel（需要安装一个软件包）              #3.下载插件openssl
cd /usr/local
cd keepalived-1.2.18/ && ./configure --prefix=/usr/local/keepalived #4.开始编译keepalived
make                                                                
cd /usr/local/keepalived-1.2.18/genhash
make install                                                        #5.安装
```

> *注意：*
>
> 报错: eepalived执行./configure --prefix=/usr/local/keepalived时报错：configure: error: Popt libraries is required
>
> 出现此错误的原因：未安装popt的开发包
>
> 解决方法：
>
> yum install popt-devel
>
> 安装好popt的开发包。重新./configure 即可。



#### 3.2.3 下载安装keepalived

将keepalived安装成Linux系统服务，因为没有使用keepalived的默认安装路径（默认路径：/usr/local）,安装完成之后，需要做一些修改工作：
首先创建文件夹，将keepalived配置文件进行复制：

```shell
#1.创建keepalived 文件夹
mkdir /etc/keepalived                                        
#2.将keepalived 配置文件 -> 复制 -> 到新创建的keeplived 文件夹中。
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
#3.然后复制keepalived脚本文件：
cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
ln -s /usr/local/sbin/keepalived /usr/sbin/
ln -s /usr/local/keepalived/sbin/keepalived /sbin/
#4.设置开机启动：，到此我们安装完毕!
chkconfig keepalived on 
#5.启动 keepalived
service keepalived start
```

> *注意：*
>
> 启动报错Starting keepalived (via systemctl):  Job for keepalived.service failed. See 'systemctl status keepalived.service' and 'journalctl -xn' for details.  
>
> 解决办法
>
> [root@edu-proxy-01 sbin]# cd /usr/sbin/  
>
> [root@edu-proxy-01 sbin]# rm -f keepalived   
>
> [root@edu-proxy-01 sbin]# cp /usr/local/keepalived/sbin/keepalived  /usr/sbin/  



#### 3.2.4  使用keepalived虚拟(lvs)VIP

1. vi /etc/keepalived/keepalived.conf

   ```
   vrrp_script chk_nginx {
       #运行脚本，脚本内容下面有，就是起到一个nginx宕机以后，自动开启服务
       script "/etc/keepalived/nginx_check.sh"
       #检测时间间隔
       interval 2 
       #如果条件成立的话，则权重 -20
       weight -20 
   }
   
   vrrp_instance VI_1 {
       state MASTER           	     # 来决定主从 (MASTER、BACKUP)
       interface ens33   			 # 绑定虚拟 IP 的网络接口，根据自己的机器填写
       virtual_router_id 121 		 # 虚拟路由的 ID 号， 两个节点设置必须一样
       mcast_src_ip 192.168.212.140 # 填写本机ip
       priority 100 				 # 节点优先级,主要比从节点优先级高
       nopreempt 					 # 优先级高的设置 nopreempt 解决异常恢复后再次抢占的问题
       advert_int 1                 # 组播信息发送间隔，两个节点设置必须一样，默认 1s
       authentication {
           auth_type PASS
           auth_pass 1111
       }
       
                                    # 将 track_script 块加入 instance 配置块
       track_script {
           chk_nginx                #执行 Nginx 监控的服务
       }
   
       virtual_ipaddress {
           192.168.110.110          # 虚拟ip,也就是解决写死程序的ip怎么能切换的ip,也可扩展，用途广泛。可配置多个。
       }
   }
   ```

2. 添加ip 规则

   ```shell
   # 1.关闭防火墙
   systemctl stop firewalld  
   systemctl mask firewalld
   # 2.开放443端口(HTTPS)
   iptables -A INPUT -p tcp --dport 443 -j ACCEPT
   # 3.保存上述规则
   service iptables save
   # 4.开启服务
   systemctl restart iptables.service
   
   ```

   > *注意：如果没有安装iiptables.service*
   >
   > #安装
   > sudo yum install iptables-services 
   > #开启iptables
   > sudo systemctl enable iptables 
   > sudo systemctl enable ip6tables 
   > #启动服务
   > sudo systemctl start iptables 



#### 3.2.5 nginx+keepalived简单双机主从热备

可以两台机子互为热备，平时各自负责各自的服务。在做上线更新的时候，关闭一台服务器的tomcat后，nginx自动把流量切换到另外一台服务的后备机子上，从而实现无痛更新，保持服务的持续性，提高服务的可靠性，从而保证服务器7*24小时运行。

##### 例子一：Nginx Upstream 实现简单双机主从热备

```
upstream testproxy {
      server 127.0.0.1:8080;
      server 127.0.0.1:8081 backup;
}

server {
      listen       80;
      server_name  localhost;
      location / {
          proxy_pass   http://testproxy;
          index  index.html index.htm;
      }
	  #nginx与上游服务器(真实访问的服务器)超时时间 后端服务器连接的超时时间_发起握手等候响应超时时间
	  proxy_connect_timeout 1s;
		#nginx发送给上游服务器(真实访问的服务器)超时时间
      proxy_send_timeout 1s;
	  #nginx接受上游服务器(真实访问的服务器)超时时间
      proxy_read_timeout 1s;
}
```

只要在希望成为后备的服务器 ip 后面多添加一个 backup 参数，这台服务器就会成为备份服务器。

在平时不使用，nginx 不会给它转发任何请求。只有当其他节点全部无法连接的时候，nginx 才会启用这个节点。

一旦有可用的节点恢复服务，该节点则不再使用，又进入后备状态。



##### 例子二：Nginx Upstream 实现简单双机主从热备

每个服务虚拟安装keepalived 虚拟一个VIP ，配置主从关系，当主挂了,直接走备机。

Keepalived虚拟VIP 地址 192.168.212.110

A 服务器 192.168.212.142

B 服务器 192.168.212.143

###### 1. 修改：主keepalived信息

修改主Nginx服务器keepalived文件,  /etc/keepalived/keepalived.conf 

State 为MASTER

```
! Configuration File for keepalived

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh" #运行脚本，脚本内容下面有，就是起到一个nginx宕机以后，自动开启服务
    interval 2 #检测时间间隔
    weight -20 #如果条件成立的话，则权重 -20
}
# 定义虚拟路由，VI_1 为虚拟路由的标示符，自己定义名称
vrrp_instance VI_1 {
    state MASTER #来决定主从
    interface ens33 # 绑定虚拟 IP 的网络接口，根据自己的机器填写
    virtual_router_id 121 # 虚拟路由的 ID 号， 两个节点设置必须一样
    mcast_src_ip 192.168.212.141 #填写本机ip
    priority 100 # 节点优先级,主要比从节点优先级高
    nopreempt # 优先级高的设置 nopreempt 解决异常恢复后再次抢占的问题
    advert_int 1 # 组播信息发送间隔，两个节点设置必须一样，默认 1s
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 将 track_script 块加入 instance 配置块
    track_script {
        chk_nginx #执行 Nginx 监控的服务
    }

    virtual_ipaddress {
        192.168.212.110 # 虚拟ip,也就是解决写死程序的ip怎么能切换的ip,也可扩展，用途广泛。可配置多个。
    }
}
```

 

###### 2.修改：主keepalived信息

修改主Nginx服务器keepalived文件,  /etc/keepalived/keepalived.conf 

State 为BACKUP

```
! Configuration File for keepalived

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh" #运行脚本，脚本内容下面有，就是起到一个nginx宕机以后，自动开启服务
    interval 2 #检测时间间隔
    weight -20 #如果条件成立的话，则权重 -20
}
# 定义虚拟路由，VI_1 为虚拟路由的标示符，自己定义名称
vrrp_instance VI_1 {
    state BACKUP #来决定主从
interface ens33 # 绑定虚拟 IP 的网络接口，根据自己的机器填写
    virtual_router_id 121 # 虚拟路由的 ID 号， 两个节点设置必须一样
    mcast_src_ip 192.168.212.141 #填写本机ip
    priority 100 # 节点优先级,主要比从节点优先级高
    nopreempt # 优先级高的设置 nopreempt 解决异常恢复后再次抢占的问题
    advert_int 1 # 组播信息发送间隔，两个节点设置必须一样，默认 1s
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 将 track_script 块加入 instance 配置块
    track_script {
        chk_nginx #执行 Nginx 监控的服务
    }

    virtual_ipaddress {
        192.168.212.110 # 虚拟ip,也就是解决写死程序的ip怎么能切换的ip,也可扩展，用途广泛。可配置多个。
    }
}
```



###### 3.nginx+keepalived实现高可用

写入nginx_check.sh脚本 /etc/keepalived/nginx_check.sh

```shell
#!/bin/bash
A=`ps -C nginx --no-header |wc -l`
if [ $A -eq 0 ];then
    echo "重启docker"
    systemctl restart docker && docker start nginx
    sleep 2
    if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
        echo "关闭 keepalived"
        killall keepalived
    fi
fi

```

> *注意该脚本一定要授权: chmod 777 nginx_check.sh。*



###### 4.先让主机keepalived 添加虚拟VIP，然后再让备机keepalived添加虚拟VIP

1. 主机keepalived，添加虚拟vip

```shell
service keeplived start && tail -f /var/log/messages
```

   upstream   testproxy {         server 127.0.0.1:8080;         server 127.0.0.1:8081 backup;     }           server {           listen       80;           server_name  localhost;                          location / {               proxy_pass   http://testproxy;               index  index.html index.htm;           }                      ###nginx与上游服务器(真实访问的服务器)超时时间 后端服务器连接的超时时间_发起握手等候响应超时时间                       proxy_connect_timeout 1s;                                ###nginx发送给上游服务器(真实访问的服务器)超时时间            proxy_send_timeout 1s;                       ### nginx接受上游服务器(真实访问的服务器)超时时间            proxy_read_timeout 1s;           }   

然后，关闭掉主机keepalived 。

```shell
service keeplived stop 
```

2. 启动备机keepalived ,让从机添加虚拟vip

```
service keeplived start && tail -f /var/log/messages
```

![1554613932430](C:\Users\Calvin\AppData\Roaming\Typora\typora-user-images\1554613932430.png)

然后，关闭掉备机keepalived 。

```shell
service keeplived stop 
```

3. 启动主机keepalived.

```shell
service keeplived start && tail -f /var/log/messages
```

![1554613932430](C:\Users\Calvin\AppData\Roaming\Typora\typora-user-images\1554613932430.png)

4. 启动备份keepalived，可以看到如下图跳转的Mast主机。

```shell
service keeplived start && tail -f /var/log/messages
```

![1554614501389](C:\Users\Calvin\AppData\Roaming\Typora\typora-user-images\1554614501389.png)

![1554614537502](C:\Users\Calvin\AppData\Roaming\Typora\typora-user-images\1554614537502.png)

5. 关闭主机keepalived后，可以看到如下跳转到backup备机。

![1554614594847](C:\Users\Calvin\AppData\Roaming\Typora\typora-user-images\1554614594847.png)



## 4.lvs与Nginx区别

### 4.1 lvs

1. LVS的负载能力强，因为其工作方式逻辑非常简单，仅进行请求分发，而且工作在网络的第4层，没有流量，所以其效率不需要有过多的忧虑。

2. LVS基本能**支持所有应用**，因为工作在**第4层**，所以LVS可以对几乎所有应用**进行负载均衡**，包括**Web、数据库等。**

   > *注意：LVS并不能完全判别节点故障，比如在WLC规则下，如果集群里有一个节点没有配置VIP，将会导致整个集群不能使用。还有一些其他问题，目前尚需进一步测试。*



### 4.2 nginx

1. Nginx工作在**网路第7层**，所以可以**对HTTP应用实施分流策略，比如域名、结构**等。相比之下，LVS并不具备这样的功能，所以Nginx可使用的场合远多于LVS。并且Nginx对网络的依赖比较小，理论上只要Ping得通，网页访问正常就能连通。LVS比较依赖网络环境。只有使用DR模式且服务器在同一网段内分流，效果才能得到保证。

2. Nginx可以通过服务器**处理网页返回的状态吗**、**超时等来检测服务器内部的故障**，并会把**返回错误的请求重新发送到另一个节点**。目前**LVS和LDirectd 也支持对服务器内部情况的监控，但不能重新发送请求**。

3. 比如用户正在上传一个文件，而处理该上传信息的节点刚好出现故障，则Nginx会把上传请求重新发送到另一台服务器，而LVS在这种情况下会直接断掉。Nginx还能支持HTTP和Email（Email功能很少有人使用），LVS所支持的应用在这个电商比Nginx更多。
4. Nginx同样能承受很高负载并且能稳定运行，由于**处理流量受限于机器I/O等配置**，所以**负载能力相对较差**。



###  4.3 lvs 和nginx 优缺点

 nginx 优点：

> 1. Nginx 安装、配置及测试相对来说比较简单，因为有相应的错误日志进行提示。

nginx 缺点：

> 2. Nginx本身没有现成的热备方案，所以在单机上运行风险较大，建议KeepAlived配合使用。

lvs 优点：

> 3. LVS的负载能力强，

lvs 缺点：

> 4. LVS的安装、配置及测试所花的时间比较长，因为LVS对网络依赖比较大，很多时候有可能因为网络问题而配置不能成功，出现问题时，解决的难度也相对较大。
> 5. 建议KeepAlived配合使用。另外，Nginx可以作为LVS的节点机器使用，充分利用Nginx的功能和性能。当然这种情况也可以直接使用Squid等其他具备分发功能的软件。



### 4.4  具体应用具体分析

> - 如果是比较小型的网站（每日PV小于100万），用户Nginx就完全可以应对。
> - 如果机器也不少，可以用DNS轮询。
>
> - LVS后用的机器较多，在构建大型网站或者提供重要服务且机器较多时，可多加考虑利用LVS。



## 5. 注意阿里云默认不支持虚拟VIP技术

阿里云默认不支持虚拟VIP技术，

https://yq.aliyun.com/ask/61502





 



### 



