# 源码使用

- `git clone git@github.com:yxyyyt/jdk.git`

- `cd D:\repository\contribute\jdk`

- `git checkout -b jdk8-b120.yxyyyt jdk8-b120` 

  根据某一tag创建branch

- `git push -u origin jdk8-b120.yxyyyt`

  提交本地分支到远程

- 个人项目引用开源源码

  - dev（个人项目）| open Module Settings | SDKs | 1.8 | Sourcepath
    - 加载源码目录：D:\repository\contribute\jdk\jdk\src\share\classes
    - 移除：D:\install\Java\jdk1.8.0_231下的src.zip和javafx-src.zip



# 源码分析

参看 `https://github.com/yxyyyt/jdk/tree/jdk8-b120.yxyyyt` 注释





# 配置环境变量

设置|关于|高级系统设置|环境变量

- CLASSPATH

  - .;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar

- JAVA_HOME

  - D:\install\Java\jdk1.8.0_231

- Path

  注意使用绝对路径

  - D:\install\Java\jdk1.8.0_231\bin
  - D:\install\Java\jdk1.8.0_231\jre\bin

