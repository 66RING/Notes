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

