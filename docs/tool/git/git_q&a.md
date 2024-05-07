- 解决命令行下中文乱码问题

  - 修改 Gitconfig

    ```shell
    # 解决git status中文件路径的编码问题
    git config --global core.quotepath false
    # 图形Git GUI界面编码
    git config --global gui.encoding utf-8
    # 提交信息编码
    git config --global i18n.commit.encoding utf-8
    # 输出 log 编码
    git config --global i18n.logoutputencoding utf-8
    ```

  - 修改环境变量

    新建系统变量 LESSCHARSET=utf-8

- 无法访问Github的22端口

  在~/.ssh目录下新建config文件，内容填入

  ```config
  Host github.com
  Hostname ssh.github.com
  Port 443
  ```

  