# 开发环境

## JDK环境

- 项目右键 |  open module settings
  - sdks | + jdk 指定 jdk home path
  - project
    - 指定sdk
    - Language level 指定 8
  - modules
    - Sources | language level: 8
- Settings | Build, Execution, Deployment | compiler | java compiler

  1. module 设置为8
  2. project bytecode version 设置为8



## 配置

- Settings | Editor | Inspections | JVM languages | Serializable class without 'serialVersionUID' 

  实现 `Serializable` 接口的类需要提供 `serialVersionUID` 属性并可自动生成

- Settings | Editor | Inlay Hints | Code vision

  取消 Usages、Code author、Inheritors。取消代码中的内嵌提示。

- Settings | Editor | File Encodings 

  将 Global Encoding 、 Project Encoding 、Default encoding for properties files 都设置成统一的编码 UTF-8

- Settings | Editor | File and Code Templates

  - 选中 files | class

  - 选中 includes | file header 在最右边的输入框输入注释模板

    ```java
    /**
      * Created by ${USER} on ${DATE}<br>
      * All Rights Reserved(C) 2017 - ${YEAR} SCIATTA <br> <p/>
      * TODO
    */
    ```



# 插件

- jclasslib Bytecode viewer

  反编译插件

- Scala

  Scala 插件

- CFR Decompile

  Scala 反编译插件

- Git Commit Message Helper

  Git 格式化提交消息插件
  
- Gitee

  Gitee插件