- 创建来自github的project，提示目录已存在

  - 从github上克隆A项目
  - 不要用idea打开A项目目录，用其他项目，然后创建和A项目目录重合的project

- 删除module后，无法重复创建

  - 右键 | remove module

  - 右键 | delete module

  - 在父pom中删除module

  - Settings | Build, Execution, Deployment | Build Tools | Maven | Ignored Files

    反向勾选被选中的pom.xml

  - Reload project

- 解决 'Unable to resolve table' 问题

  Settings | Editor | Inspections | SQL | Unresolved reference 取消选中

- 无法进入JDK源码进行调试

  Preferences | Build, Execution, Deployment | Debugger | Stepping

  do not step into the classes；去掉 java.* 和 javax.*
