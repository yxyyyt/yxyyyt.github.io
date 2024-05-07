# 安装 VMware

- 版本：VMware-workstation-full-16.2.1-18811642
- 序列号：ZF3R0-FHED2-M80TY-8QYGC-NPKYF
- 编辑 | 首选项
  - 工作区：虚拟机的默认位置
  - 内存：2048MB

- VMware Fusion 11 注册码 `7HYY8-Z8WWY-F1MAN-ECKNY-LUXYX` 
- VMware Fusion 12 注册码 `ZF3R0-FHED2-M80TY-8QYGC-NPKYF`



# 网络连接设置

- 安装完成后
  - **windows：查看网络连接**会新建VMnet1和VMnet8两个网络连接

- 编辑 | 虚拟网络编辑器 | 更改VMnet8设置
  - 子网IP：192.168.8.0
  - 子网掩码：255.255.255.0
  - NAT设置
    - 网关IP：192.168.8.2
  - DHCP设置
    - 起始：192.168.8.3
    - 结束：192.168.8.255



# 新建虚拟机

- 文件 | 新建虚拟机

  - 自定义（高级）

  - 稍后安装操作系统

  - Linux：CentOS 7 64 位
  - 虚拟机名称：test100

  - 处理器：2P2C
  - 内存：4096MB

  - 网络类型：NAT

  - 最大磁盘大小：100GB；将虚拟磁盘存储为单个文件



# 安装 CentOS

- 版本：CentOS-7-x86_64-DVD-1810
- 选择指定虚拟机 | 设置 
  - CD / DVD 使用ISO映像文件，选择：CentOS-7-x86_64-DVD-1810.iso
- 开启此虚拟机
  - install Centos 7
- 按提示向导安装
  - language：english
  - date & time：Asia/shanghai
  - Language support：添加中文，简体中文
  - software selection：Minimal install
  - installation destination：automatic configure  partitioning
  - network & hostname：ens33 | <font color=red>on</font>；Host name：test100
- 开始安装
  - root password：root（太短，双击确认即可）
- 重启

