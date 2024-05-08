# 向开源项目贡献代码

## 将开源项目fork到远程origin仓库

`https://github.com/yxyyyt/shardingsphere`



## 将远程origin仓库clone到本地仓库

- 将远程origin仓库clone到本地仓库 `git clone git@github.com:yxyyyt/shardingsphere.git`

- 通过 `git status` 查看当前所在分支 On branch master

- 通过 `git remote -v` 查看远程仓库映射关系

  ```shell
  origin	git@github.com:yxyyyt/shardingsphere.git (fetch)
  origin	git@github.com:yxyyyt/shardingsphere.git (push)
  ```



## 与上游upstream仓库建立映射关系

- 建立映射 `git remote add upstream https://github.com/apache/shardingsphere.git`

- 通过 `git remote -v` 查看远程仓库映射关系

  ```shell
  origin	git@github.com:yxyyyt/shardingsphere.git (fetch)
  origin	git@github.com:yxyyyt/shardingsphere.git (push)
  upstream	https://github.com/apache/shardingsphere.git (fetch)
  upstream	https://github.com/apache/shardingsphere.git (push)
  ```

  

## 创建本地仓库develop分支开发

- 创建develop分支 `git checkout -b develop`
- 在此分支上进行开发 `git add -A`
- 提交本地仓库 `git commit -m` 



## 提交远程origin仓库

- 本地仓库拉取远程upstream仓库的最新内容 `git fetch upstream`
- 切换到本地master分支 `git checkout master`
- 本地仓库和远程upstream仓库的master分支同步 `git rebase upstream/master`
  - <font color=red>rebase</font> 合并为一条时间轴。如在master中执行git rebase develop，找到master和develop的公共祖先，祖先先合并develop的新增提交，然后在后面追加master的新增提交，即**变更起始点**。**注意必须没有待提交的文件**
    - 优点：得到更简洁的项目历史，去掉了merge commit
    - 缺点：如果合并出现代码问题不容易定位，因为re-write了history
  - <font color=red>merge</font> 合并路径为分叉时间轴。如在master中执行git merge develop，找到master和develop的公共祖先，然后由公共祖先、master最新提交和develop最新提交，三方合并产生一个**新的提交**
    - 优点：记录了真实的commit情况，包括每个分支的详情
    - 缺点：因为每次merge会自动产生一个merge commit，所以在使用一些git 的GUI tools，特别是commit比较频繁时，看到分支很杂乱。
- 切换到本地develop分支 `git checkout develop` 不要在master分区rebase，这样会将最新修改放在最前面
- 合并最新master分支 `git rebase master` 
  - 远程master分支位置不变，其他人修改位置不变。但最新起始点已经变成develop的位置，本地master位置改变
  - 提交到远程，其他人更新前fetch最新版本再提交，不会冲突
- 将本地develop分支提交到origin仓库 `git push origin develop:develop`



## 提交 Pull Request

在远程origin仓库 `https://github.com/yxyyyt/shardingsphere` 提交 Pull request

- `upstream/main` <- `orgin/develop`



## 合并upstream最新版本

- 同步本地master
  - git fetch upstream
  - git checkout master
  - git rebase upstream/master
- 同步本地develop
  - git checkout develop
  - git rebase master
- 更新远程
  - git push origin master
  - git push origin develop

