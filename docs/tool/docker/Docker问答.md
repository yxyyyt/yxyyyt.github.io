- 镜像启动失败

  ```shell
  # 执行命令获取错误日志（LogPath）分析
  docker container inspect mysql
  ```

- 解决macOS无法访问docker容器服务的问题
  
  - 原因
  
    - 对于docker网络是bridge的情况，宿主机是Linux会创建docker0虚拟网卡，通讯通过虚拟网卡，宿主机和容器通讯完全支持；而宿主机是Mac，bridge桥接网络是运行在docker创建的一个Linux虚拟机中，因此，宿主机和容器无法通讯
    - Mac不支持host主机网络驱动程序，其只适用于Linux
  
  - 目标
  
    - **Mac**使用bridge网络，主机无法ping通容器IP，通过**tunnelblick**解决
    - 主机访问容器内开放端口提供的服务，只需要通过 `--expose` 暴露容器的端口，直接通过容器IP和端口号即可访问容器提供的服务
  
  - 解决
  
    解决问题的方案是 github 上的 docker-for-mac `https://github.com/wojas/docker-mac-network` 项目，主要方法是使用OpenVpn 来访问 docker。
  
    - 安装 `brew install tunnelblick`
  
    - 克隆 `git clone https://github.com/wojas/docker-mac-network.git`
  
    - **修改配置**
  
      - `cd /Users/yangxiaoyu/work/develop/star/docker-mac-network/helpers`
  
      - `vi run.sh`
  
        ```shell
        # 修改为实际的容器IP段和子网掩码
        route 172.82.0.0 255.255.255.0
        ```
  
    - 启动tunnelblick
  
      - `cd /Users/yangxiaoyu/work/develop/star/docker-mac-network`
  
      - <font color=red>必须保持启动</font> `docker-compose up -d` 
  
      - 修改新生成的文件 `vi docker-for-mac.ovpn`
  
        ```shell
        </tls-auth>
        comp-lzo yes	# 添加一行
        
        route 172.82.0.0 255.255.255.0
        ```
  
      - 双击 `docker-for-mac.ovpn` 将配置导入到tunnelblick
  
      - 打开tunnelblick客户端 | 菜单 | VPN 详情 | 连接
  
      - 验证
  
        ```shell
        ping 172.82.0.100
        telnet 172.82.0.100 6379
        ```
  
    - 重新生成
  
      - 删除 `/Users/yangxiaoyu/work/develop/star/docker-mac-network` 目录下文件 `config/*` 和 `docker-for-mac.ovpn`
      - 修改配置
      - 启动tunnelblick，删除tunnelblick原来的配置
  
  