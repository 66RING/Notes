---
title: git操作
date: 2019-9-8
tags: tools, git
---

# Git 操作

## 基本操作
### 初始化

``` shell
git init
```


### 查看状态

```shell
git status
```


### 查看修改

```shell
git diff BRANCHNAME
```


### 查看提交记录

```shell
git log
```


### 添加修改并追踪

```shell
git add FILE
git add .  // 添加所有
```


### 退回

```shell
git reset
git reset --hard HEAD^ 退回到上一个版本
git reset --hard HEAD~n 退回到上n个版本
git reset --hard sjaieral 退回某个版本
```


### 添加用户

```shell
git config --global user.name "NAME"
git config --global user.email "EMAIL"
...等等
```


### 拉取项目

```shell
1.获取工程
git clone URL

2.更新
git pull
```


### 添加远程仓库

```shell
1.gibhub上创建一个仓库

2.添加远程仓库
git remote add origin ULR

3.推到远程仓库
git push --set-upstream origin master
或者
git push -u origin master
// 然后登录
// push master分支到origin仓库

4.记住密码
git config credential.helper store
// 再次登录就可以记住了
```


### 提交

```shell
git commit // 会打开文件让你添加描述
git commit -m "描述" // 快捷添加描述提交
```


### PR

- 1. fork from rep
- 2. make some change on a new branch of the fork rep
- 3. commit and pr


### 不让git管理指定文件

```shell
1.新建一个 .gitignore 文件

2.写入不需要git管理等文件名
// 但是git一旦追踪某个文件那就会一只追踪
// 停止追踪 git rm --cached FILE
```


### git分支

```shell
1.创建分支
git branch NAME

2.切换分支
git checkout NAME

3.分支合并
git merge NAME // 会把NAME分支合并到当前分支

4.删除分支
git branch -d NAME // -D强制删除
// 主分支叫master
```


## 进阶

HEAD表示当前版本

### 时间胶囊

- `git log`查看提交历史，确定版本号
- `git reset --hard <commit_id>`退回到指定版本
    * 退回后`git log`将找不到新版本的信息，但是不用担心本版仓库还在，只要找到对应的id就能回去
- `git reflog`查看命令历史


### 增删改

- checkout
    * `git checkout -- <file>`，撤销修改(checkout文件到本版原始的状态)
    * `--`是必要的，否则就是分支的切换
- reset
    * `git reset HEAD <file>`，reset文件在暂存区(stage)的状态(清出暂存区)，也就是unstaged


### 删除

一个文件被删除后git会检测到修改(工作区和版本库不一样)，可以使用`git rm`删除版本库中的记录。 **git rm** 是一样的效果

如果版本库的文件没被删除，则可用`git checkout -- <file>`恢复，如果已经删除可以使用`git reset`的方法来unstage


### 分支管理

git把版本穿成一条时间线，每条时间线就是一个分支，每个版本就是一个节点。当创建新分支dev时相当于创建了一个dev的指针指向当前节点，再把HEAD指向dev，然后就可以以此为基础往后建立时间线。

- 建立分支
    * `git branch <branch_name>`
    * `git switch -c`
        + `-c`表示新建并切换
    * `git checkout -b <branch_name>`
        + `-b`表示新建并切换
        + 但是checkout命令又是撤销修改的命令，所以推荐使用switch
- 合并分支
    * `git merge <branch>`把指定分支合并到但前分支

如果两个分支只形成一条时间线，则git可以`fast forward` "快速合并"，即`git merge dev`把HEAD和master的指针指向dev，即可

```shell
|
* (master)
|
* dev
```

如果分支有各自的提交，则git无法执行"快速合并"

```shell
    |
    * 
    | \
dev *  * master
```


#### 分支管理策略

合并时，git会尽可能使用`Fast forward`模式，这模式下，删除分支后，会丢失部分分支信息。可以使用`--no-ff`参数来禁用`Fast forward`模式，以保留分支历史

```shell
初始情况
|
* master
|
* HEAD -> dev

Fast forward合并
|
*
|
* HEAD -> dev, master

no Fast forward合并
|
* 
| \
|  * dev
| /
*  HEAD -> master
```


#### 复制特定提交而不合并分支

如果一个bug早期就存在了，那每个分支上这个bug都存在。git提供了复制特定提交到当前分支的功能`cherry-pick <commit_id>`

这样一条命令就完成了修复bug，而不需要把分区合并


### 保存工作区

当你要修一个bug，但是你当前的工作没有完成(提交)时，使用`stash`功能可以把当前的工作现场保存起来。这样你转去修bug的时候就有一块干净的工作区了。

- `git stash`
    * 保存当前工作现场
- `git stash list`
    * 查看以保存工作现场
- 工作现场恢复
    * `git stash apply [stash_id]`
        + 这种恢复后不会删除stash内容，需要`git stash drop`删除
    * `git stash pop [stash_id]`
        + 恢复的同时删除
    * stash是栈的结构，所以apply和drop的时候默认都是操作栈顶的stash


### 标签管理

tag就是给commit起一个容易记住的名字，tag和commit是绑定在一起的

- `git tag <name> [commit_id]`
    * 给一个commit一个为名tag，默认是当前commit
- `git tag -a <name> -m <desc> [commit_id]`
    * 给`<commit_id>`(默认当前commit)创建一个带有说明信息`<desc>`的`<name>`标签
- `git show <tagname>`
    * 显示tag信息
- `git tag -d <tagname>`
    * 删除标签

 **注意** ：标签是按字母排序的，而不是commit是时间顺序

标签只会储存在本地，如果要推送到远程，使用`git push <origin> <tagname>`

一次性推送所有标签`git push <origin> --tags`

删除远程标签(前提是本地标签已经删除):`git push <origin> :refs/tags/<tagname>`


### 多人协作

TODO


### 配置别名


## 搭建Git服务器

todo
