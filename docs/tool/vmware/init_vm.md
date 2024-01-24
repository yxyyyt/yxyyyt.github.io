# 设置静态IP

```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens33

BOOTPROTO="static"
IPADDR=192.168.8.100
NETMASK=255.255.255.0
GATEWAY=192.168.8.2
DNS1=114.114.114.114

# 重启网卡使得配置生效
service network restart

# 虚拟机可以ping通网关
ping -c 3 192.168.8.2
# 虚拟机可以ping通主机
ping -c 3 192.168.2.3
# 虚拟机可以ping通外网
ping -c 3 www.baidu.com

# 主机可以ping通虚拟机
ping 192.168.8.100
```



# 关闭防火墙

root用户下执行

```shell
systemctl stop firewalld
systemctl disable firewalld
```



# 关闭selinux

root用户下执行

```shell
vi /etc/selinux/config

# 修改
SELINUX=disabled
```



# 更改主机名

```shell
vi /etc/hostname

test100
```



# 更改主机名与IP地址映射

```shell
vi /etc/hosts

192.168.8.100 test100
```



# 同步时间

定时同步阿里云服务器时间

```shell
crontab -e

*/1 * * * * /usr/sbin/ntpdate time1.aliyun.com
```



# 添加用户（可选）

```shell
# 添加用户
useradd test
passwd test

# 为普通用户添加sudo权限
visudo

test  ALL=(ALL)       ALL
```



# 定义统一目录

```bash
# 软件压缩包存放目录
mkdir -p /root/software
# 软件解压后存放目录
mkdir -p /root/install
# 数据存放目录
mkdir -p /root/data
```

