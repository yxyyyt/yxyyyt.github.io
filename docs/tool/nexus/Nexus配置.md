# 仓库类型

- hosted
  - 本地仓库，通常部署自己的构件到这一类型的仓库
- proxy
  - 代理仓库，代理远程的公共仓库，如maven中央仓库
- group
  - 组仓库，用来合并多个hosted/proxy仓库



# 仓库说明

- maven-central
  - maven中央库
  - 默认从 `https://repo1.maven.org/maven2/` 拉取
  - proxy
- maven-releases
  - 私库发行版本
  - `http://192.168.8.100:8081/repository/maven-releases/`
  - 初次安装将Deployment policy设置为Allow redeploy
  - hosted
- maven-snapshots
  - 私库快照版本
  - `http://192.168.8.100:8081/repository/maven-snapshots/`
  - hosted
- maven-public
  - 组仓库，把上面三个仓库组合在一起对外提供服务，在本地maven基础配置settings.xml或项目pom.xml中使用
  - `http://192.168.8.100:8081/repository/maven-public/`
  - group



# 创建仓库

## 创建Blob Stores

创建自定义存储路径。

Repository | Blob Stores

- Type

  File

- Name

  custom

- Path

  /nexus-data/blobs/custom

创建位置 /root/data/nexus/data/blobs/custom；默认 default。



## 创建代理仓库

指向远程代理仓库。

Repository | Repositories | Create repository，选择maven2(proxy)

- Name

  custom_proxy

- Remote storage

  `http://maven.aliyun.com/nexus/content/groups/public/`

- Blob store

  custom



## 创建本地仓库

存储第三方库。

Repository | Repositories | Create repository，选择maven2(hosted)

- Name

  custom_hosted

- Version policy

  Mixed

- Blob store

  custom

- Deployment policy

  Allow redeploy



## 创建快照仓库

存储本地快照版本。

Repository | Repositories | Create repository，选择maven2(hosted)

- Name

  custom_snapshots

- Version policy

  Snapshot

- Blob store

  custom

- Deployment policy

  Allow redeploy



## 创建发行仓库

存储本地发行版本。

Repository | Repositories | Create repository，选择maven2(hosted)

- Name

  custom_releases

- Version policy

  Release

- Blob store

  custom



## 创建组仓库

Repository | Repositories | Create repository，选择maven2(group)

- Name

  custom_group

- Blob store

  custom

- Member repositories

  - custom_hosted
  - custom_snapshots
  - custom_releases
  - custom_proxy
  - maven-central



# 配置文件

Maven下的setting.xml文件是全局设置，而项目中的pom.xml文件是局部设置。pom.xml文件对于项目来说，是优先使用的。而pom.xml文件中如果没有配置镜像地址的话，就按照settting.xml中定义的地址去查找。

## 全局setting.xml

```xml
<!-- 本地maven配置路径 -->
<localRepository>D:\data\.m2\repository</localRepository>

<!-- nexus服务器，id为组仓库name -->
<servers>
	<server>
		<id>custom_group</id>
    	<username>admin</username>
        <password>admin</password>
    </server>
    <server>  
    	<id>custom_snapshots</id>  
    	<username>admin</username>  
    	<password>admin</password>  
	</server>
	<server>  
    	<id>custom_releases</id>  
    	<username>admin</username>  
    	<password>admin</password>  
	</server>
</servers>

<!-- 组仓库的url地址，id和name为组仓库name，mirrorOf为central -->  
<mirrors>     
	<mirror>  
    	<id>custom_group</id>  
    	<name>custom_group</name>  
        <url>http://192.168.8.100:8081/repository/custom_group/</url>  
        <mirrorOf>central</mirrorOf>  
    </mirror>     
</mirrors>
```



## 项目pom.xml

```xml
<!-- pom.xml -->
<repositories>
	<repository>
    	<id>custom_group</id>
        <name>Nexus Repository</name>
        <url>http://192.168.8.100:8081/repository/custom_group/</url>
        <snapshots>
        	<enabled>true</enabled>
        </snapshots>
        <releases>
        	<enabled>true</enabled>
        </releases>
    </repository>
</repositories>
<pluginRepositories>
	<pluginRepository>
    	<id>custom_group</id>
        <name>Nexus Plugin Repository</name>
        <url>http://192.168.8.100:8081/repository/custom_group/</url>
        <snapshots>
        	<enabled>true</enabled>
        </snapshots>
        <releases>
        	<enabled>true</enabled>
        </releases>
    </pluginRepository>
</pluginRepositories>

<distributionManagement>
	<repository>
    	<id>custom_releases</id><!-- 此处id和settings.xml的id保持一致 -->
        <name>Nexus Release Repository</name>
        <url>http://192.168.8.100:8081/repository/custom_releases/</url>
    </repository>
	<snapshotRepository>
    	<id>custom_snapshots</id><!-- 此处id和settings.xml的id保持一致 -->
        <name>Nexus Snapshot Repository</name>
        <url>http://192.168.124.189:8081/repository/custom_snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```



## 编译

```shell
# -U,--update-snapshots，强制更新releases、snapshots类型的插件或依赖库;否则maven一天只会更新一次snapshot依赖
mvn clean compile -U
```



## 部署

```shell
mvn clean deploy -Dmaven.test.skip=true
```



# 第三方库上传

## 手动上传

Upload | custom_hosted

- File
- Group ID
- Artifact ID
- Version
- 选择 Generate a POM file with these coordinates
- Packing
  - jar



## 命令上传

```shell
# setting.xml
<server>  
    <id>custom_hosted</id>  
    <username>admin</username>  
    <password>admin</password>  
</server>

# 上传
# -DrepositoryId=custom_hosted，需要和setting.xml文件中server配置的ID一致
mvn deploy:deploy-file -DgroupId=x -DartifactId=y -Dversion=z -Dpackaging=jar -Dfile=local path -Durl=http://192.168.8.100:8081/repository/custom_hosted/ -DrepositoryId=custom_hosted
```

