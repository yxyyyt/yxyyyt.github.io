# 系统软件

## ifconfig

```shell
# ifconfig、netstat命令
sudo yum -y install net-tools
```



## ntpdate

```shell
# 时间同步
sudo yum -y install ntpdate
```



## wget

```shell
# wget
sudo yum -y install wget
```



## git

```shell
# git
sudo yum -y install git
```



## zip

```shell
yum install -y unzip zip
```





# 开发软件

## Docker

```shell
# yum-utils
yum install -y yum-utils device-mapper-persistent-data lvm2

# 为yum源添加docker仓库位置
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装docker
yum install docker-ce

# 启动docker
systemctl start docker

# 验证
docker info

# 安装上传下载软件，执行docker cp指令
# 将test.sql文件拷贝到mysql容器的/目录下 
# docker cp /root/data/test/test.sql mysql:/
yum -y install lrzsz
```

