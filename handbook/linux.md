# <center>个人Linux手册（问题总结）</center>

## 1.问题总结

### 1.宝塔软件相关问题

1. 宝塔安装的软件如`nginx`会自动添加环境变量不用自己额外配置,直接可用各种命令，使用which nginx 可以看到 `/usr/bin/nginx`的结果。
2. 宝塔安装的软件都在`/www/server`里面。
3. 通过宝塔创建的网站（nginx服务器）配置文件位于`/www/server/panel/vhost/nginx`目录中

### 2.linux碎片问题

#### 1. 查看yum默认安装目录的方法如下：

1. 首先，找到软件包

​        命令如下：

​        `rpm -qa` 例： `rpm -qa|grep httpd`

2. 然后，根据包名查看具体安装位置即可

   命令如下：

   `rpm -ql XXXXX` 例：

   ` rpm -ql httpd-2.4.6-90.el7.centos.x86_64`

#### 2.配置固定ip

编辑 **vi**  /etc/sysconfig/network-scripts/ifcfg-ens33

   要求：将 ip 地址配置的静态的，比如: ip 地址为 	192.168.200.130



```
ifcfg-ens33 文件说明

DEVICE=eth0	#接口名（设备,网卡）

HWADDR=00:0C:2x:6x:0x:xx	#MAC 地址

TYPE=Ethernet	#网络类型（通常是 Ethemet）

UUID=926a57ba-92c6-4231-bacb-f27e5e6a9f44	#随机 id

\#系统启动的时候网络接口是否有效（yes/no）

ONBOOT=yes

\# IP 的配置方法[none|static|bootp|dhcp]（引导时不使用协议|静态分配 IP|BOOTP 协议|DHCP 协议）

BOOTPROTO="static"

#IP地址
IPADDR=192.168.200.130
#网关
GATEWAY=192.168.200.2
#域名解析器
DNS1=192.168.200.2
```

==将bootproto改为static,然后加上ip地址、网关、域名解析器==

重启网络服务或者重启系统生效

service	network restart	、reboot

## 2.linux常用命令

**[Linux命令大全](https://www.linuxcool.com/)**

