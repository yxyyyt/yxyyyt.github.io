# Dockerfile

- Docker daemon运行Dockerfile的每一条指令
- 每一条指令的结果提交到一个新的镜像。每一条指令独立运行，并且都会创建一个新的镜像
- Docker 在构建过程中会重用intermediate镜像，在构建过程中会显示`Using cache`
- 指令大小写不明感，但建议大写用以区分参数



## .dockerignore

Docker CLI 发送上下文到 docker daemon 之前，会在上下文根目录查找 `.dockerignore` 。如果存在，则会排除此文件匹配样式的文件和目录。如果确实需要，则使用 `ADD` or `COPY` 命令来增加到镜像。

- `# comment` 注释，被忽略
- `*/temp*` 以temp开头的文件或目录，在根目录的直接子目录内
- `*/*/temp*` 以temp开头的文件或目录，在根目录的二级子目录内
- `temp?` 根目录内以temp开头，后跟一个字符
- `**/*.go` 任何目录下以.go作为后缀
- `!README.md` 排除例外



## 指令

### Parser directives

- 可选，影响后续指令
- 构建时不会增加层
- 一个指令只允许被使用一次
- 一旦 `comment` ，空行， 构建指令被处理，docker不会再寻找Parser directives
- 大小写不明感，建议小写
- 在Parser directives之后，包含一个空行
- 不支持行连续字符 `\`
- `# directive=value`
  - `# escape=\` （默认）



### ENV

设置环境变量。

```shell
ENV FOO=/bar
WORKDIR ${FOO}   # WORKDIR /bar
```



### ARG

- `ARG` 处于构建之外，因此不可在 `FROM` 之后声明。但可以 `ARG VERSION` （不赋值）使用其默认值
- `ARG` 指令定义的变量，用户可以在构建阶段，通过 `docker build` 指令使用 `--build-arg <varname>=<value>` 覆盖已定义的变量



### FROM

- 必须以 `FROM` 开头
- `Parser directives` , `ARG` 和 `comment` 可以先于FROM指令

```shell
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
```



### RUN

构建期间当前层<font color=red>执行命令</font>，<font color=red>提交结果</font>被用于Dockerfile的下一步。

- `RUN <command>` (shell form)
  - 底层会调用/bin/sh -c来执行命令，可以解析变量
- `RUN ["executable", "param1", "param2"]` (exec form)
  - 不会调用shell



### CMD

在Dockerfile中只能有一个 `CMD` 指令。如果有多个，则最后一个起作用。 `CMD` 指令的主要目的是<font color=red>为运行容器提供默认行为，构建期间不会执行</font>。`docker run` 指定的参数会覆盖 `CMD` 指令的默认行为。

- `CMD ["executable","param1","param2"]` (exec form, this is the preferred form)
  - 不需要调用/bin/sh执行命令
- `CMD ["param1","param2"]` (as default parameters to ENTRYPOINT; omit the executable, in which case you must specify an `ENTRYPOINT` instruction)
  - 结合 `ENTRYPOINT` 使用，`ENTRYPOINT` 为默认指令，`CMD` 为其后拼接的参数；`CMD` 的参数可以被 `docker run` 代替
- `CMD command param1 param2` (shell form)
  - PID为1的进程并不是在Dockerfile里面定义CMD命令，而是/bin/sh命令。如果从外部发送任何POSIX信号到docker容器, 由于/bin/sh命令不会转发消息给实际运行的CMD命令，则不能安全得关闭docker容器。



### LABEL

为镜像添加元数据。

- 可被继承，可被覆盖

```shell
LABEL "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```



### EXPOSE

通知docker，容器在运行时监听的网络端口。只是向使用者声称打算发布的端口，但实际上并没有发布端口。`docker run` 通过 `-p` 暴露端口映射；通过 `-P` 会自动映射主机的临时高阶端口。

```shell
EXPOSE 80/tcp
EXPOSE 80/udp
```



### ADD

向镜像的文件系统添加文件，目录和远程URL文件。源目录比必须存在于构建上下文中。

- `ADD [--chown=<user>:<group>] <src>... <dest>`
- `ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]` 可包含空格

```shell
ADD hom* /mydir/
```



### COPY

简版 `ADD` 指令，不支持URL。

- `COPY [--chown=<user>:<group>] <src>... <dest>`
- `COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]`

```shell
COPY hom* /mydir/
```



### ENTRYPOINT

<font color=red>容器启动后执行命令的入口</font>。和CMD类似，默认的ENTRYPOINT也在docker run时，也可以被覆盖。在运行时用--entrypoint覆盖默认的ENTRYPOINT。

- `ENTRYPOINT ["executable", "param1", "param2"]` (exec form)
- `ENTRYPOINT command param1 param2` (shell form)



### VOLUME

创建挂载点，无法指定主机上对应的目录，由docker自动生成。

```shell
VOLUME /myvol
```



### USER

指定用户运行镜像中的命令。

- `USER`指令设置运行镜像时要使用的用户名（或UID）以及可选的用户组（或GID），以及Dockerfile中跟随该镜像的所有`RUN`，`CMD`和`ENTRYPOINT`指令。
- 如果用户没有组，则该镜像（或接下来指令）将用root组运行。

```shell
USER patrick
```



### WORKDIR

设置指令的工作目录。

```shell
WORKDIR /path/to/workdir
```

