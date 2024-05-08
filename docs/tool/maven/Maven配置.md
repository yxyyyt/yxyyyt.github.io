# 配置

settings.xml

```xml
<!-- 设置本地仓库位置 -->
<localRepository>D:\data\.m2\repository</localRepository>

<!-- 阿里云仓库，加快下载依赖 -->
<mirrors>
	<mirror>
	  <id>aliyunmaven</id>
	  <mirrorOf>*</mirrorOf>
	  <name>阿里云公共仓库</name>
	  <url>https://maven.aliyun.com/repository/public</url>
	</mirror>
</mirrors>
```



# 配置环境变量

设置|关于|高级系统设置|系统变量

- MAVEN_HOME    D:\install\apache-maven-3.6.3
- Path    %MAVEN_HOME%\bin