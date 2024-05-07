# 安装

## Windows

安装 Git-2.34.1-64-bit.exe



# 配置

## 配置全局用户

```shell
# 影响commit时的用户
git config --global user.name "Rain"
git config --global user.email yxyyyt@126.com
```



## 生成SSH key

```shell
# 查看是否已生成key，若生成全部删除
cd ~/.ssh
ls

# 生成key
ssh-keygen -t rsa -C "yxyyyt@126.com"
```



## 查询SSH key

```shell
cd ~\.ssh
cat id_rsa.pub
```



# 配置Github环境

## 将SSH key添加到GitHub

Settings | SSH and GPG keys | new SSH key



## 验证

```shell
ssh -T git@github.com
```



# 配置Gitee环境

## 将SSH key添加到Gitee

设置 | SSH公钥 | 添加公钥



## 验证

```shell
ssh -T git@gitee.com
```