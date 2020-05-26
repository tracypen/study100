#### 查看宿主机的网卡信息

`ipconfig`

记录下 子网掩码和网关的值

~~~properties
cd /etc/sysconfig/network-scripts
vim ifcfg-enp0s3

~~~

#### 配置静态ip

~~~properties
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
IPADDR=192.168.0.110
NETMASK=255.255.255.0
GETWAY=192.168.0.1
NAME=enp0s3
UUID=a7f0148a-f67a-4aa9-a3ba-7c0c4cd24e93
DEVICE=enp0s3
ONBOOT=yes
~~~

**注意**：

​	`BOOTPROTO`改为static表示静态ip；`ONBOOT=yes`表示开机启动网卡配置

​	其中`IPADDR`为宿主机同一网段下的ip；`NETMASK`子网掩码和`GETWAY`网管需要和宿主机一样

#### 重启网卡

~~~
systemctl restart network
~~~

