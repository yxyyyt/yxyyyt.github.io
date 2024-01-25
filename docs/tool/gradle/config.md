# 安装 Gradle

下载 gradle-5.2 `https://gradle.org/releases/`，之后解压到相应目录



# 配置环境变量

设置 | 搜索 ”编辑系统环境变量“

- 系统变量
  - GRADLE_HOME：D:\install\gradle-5.2
  - Path：%GRADLE_HOME%\bin
- 用户变量
  - GRADLE_USER_HOME：D:\data\.gradle



# 验证

```shell
gradle -v
```



# 配置镜像仓库

在 D:\install\gradle-5.2\init.d 目录新建 init.gradle 文件

```groovy
// allprojects的repositories为多项目提供仓库和依赖包
// buildScript的repositories为Gradle脚本提供仓库和依赖包
// 根级别的repositories为当前项目提供仓库和依赖包

allprojects {
    repositories {
        mavenLocal()
        maven { 
			url "http://192.168.8.100:8081/repository/custom_group/" 
			credentials {
				username = 'admin'
				password = 'admin'
			}
		}
    }
 
    buildscript { 
        repositories {
            mavenLocal()
            maven { 
                url "http://192.168.8.100:8081/repository/custom_group/" 
                credentials {
                    username = 'admin'
                    password = 'admin'
                }
            }
        }
    }
}
```

