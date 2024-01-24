- VMware Workstation 无法恢复错误: (vcpu-0) Exception 0xc0000005 (access violation) has occurred.

  ```shell
  # VMware Workstation的虚拟化与window10自带的Hyper-V服务发生冲突
  # 管理员运行Terminal执行命令后，重启系统
  bcdedit /set hypervisorlaunchtype off
  ```

- SSH远程连接虚拟机，发生错误 “Add correct host key in C:\\Users\\Rain/.ssh/known_hosts to get rid of this message”

  ```shell
  ssh-keygen -R 192.168.8.100
  ```

  

