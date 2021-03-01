---
title: git操作
date: 2019-9-8
tags: 
- tools
- git
- operate
---

# Git 操作

git工作流程如下：

```
dir
  |
  | init
  v
working directory 工作目录
  |
  | add
  V
staging area 暂存区
  |
  | commit 提交仓库
  v
repository
```

一个目录初始化后(init)将称为工作目录，工作目录中通过`git add`添加要跟踪的文件，并将这些文件添加到 **暂存区** 。暂存区是一个索引文件，它记录下一次要提交的文件列表。最后通过`git commit`提交到**git仓库**。git仓库里保存了项目元数据和对象数据库，并且它是以快照的方式保存的。


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

- fork到自己仓库
- clone自己仓库到电脑
- 与源代码仓库建立连接
    * `git remote add upstream <url>`
    * 查看是否成功建立连接`git remote -v`
- 创建分支
    * `git switch -c <branch_name>`或`git checkout -b <branch>`
- 从其他仓库下载(先同步)
    * `git fetch upstream`
    * `git rebase upstream/<branch>`
    * 然后提交到自己仓库提pr`git push <origin> <branch>`
- 修改代码
    * 提交当前分支到自己仓库
        + `git push origin <branch_name>`
    * 提交pr


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

缺省前缀(如`<remote>/<branch>`)的操作，默认对本地的branch操作，所以`fetch upstream`后log还看不到本地commit的变化，因为还在upstream里用`git log <remote>/<bracn>`查看

- 查看分支信息和commit
    * `git log --graph --pretty=oneline --abbrev-commit`
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

- 查看分支信息
    * `git remote -v`
- 推送分支
    * `git push <origin> <branch_name>`

当小伙伴 **clone时，默认只能看到本地的master分支**。要在dev分支上开发就需要建立远程`origin`的`dev`分支到本地：

```shell
git checkout -b <local_branch> <origin>/<remote_branch>
# 不能缺少<local_branch>，不然就变成了创建分支
```

如果git提示`no tracking information`，则说明本地分支和远程分支的连接关系没有建立，使用命令`git branch --set-upstream-to <branch_name> origin/<branch_name>`远程的branchname建立连接到本地的branchname


#### Rebase

TODO


### 子模块

对于复杂的项目，主目录依赖子模块，但我们又不想把子模块合并到主项目中，想让他们相互独立。

我们可以将子模块加入`.gitignore`文件中，这么做虽然独立了，但这么做的前提是主项目的人需要在当前目录下放置某以版本的子模块代码。

Git提供了`submodule`功能，用于建立当前项目与子模块之间的依赖关系：子模块路径、子模块远程仓库、子模块版本号。即让你在一个git仓库中存在另一个git仓库。

- 添加子模块
    * `git submodule add <submodule_url> <local_path>`
    * 会在父仓库根目录下增加`.gitmodule`文件
        ```
        $ cat .gitmodule
        [submodule "XXX"]
        path = XXX
        url = XXX
        ```
    * 并在配置文件中加入submodule字段
        ```
        $ cat .git/config
        [submodule "sub"]
            url = ssh://git@10.2.237.56:23/dennis/sub.git
        ```
- checkout
    * clone一个包含子仓库的仓库并不会clone子仓库的文件，而是clone`.gitmodule`的描述文件用于构建子仓库
    * 使用
        ```shell
        // 初始化本地配置文件
        $ git submodule init
        // 检出父仓库列出的commit
        $ git submodule update

        或者

        $ git submodule update --init --recursive
        ```
    * 此时子仓库会在某个git提交本版，即可在子仓库中git命令(pull)进行版本控制
    * 主目录中会看到的子仓库的修改，是一个`commit id`
* 删除子仓库
    + `git submodule deinit <submodule>`
    + 会自动删除`.git/config`中的内容，但是`.submodule`和`.git/modules`还会保留，可以通过`.submodule`恢复
    + 使用`git rm <submodule_dir>`，移除子仓库文件夹，此时子模块信息基本移除(除了`.git/modules`)
        ```
        On branch main
        Your branch is up to date with 'origin/main'.

        Changes to be committed:
          (use "git restore --staged <file>..." to unstage)
                modified:   .gitmodules
                deleted:    <submodule_dir>
        ```


### 配置别名


## 搭建Git服务器

todo
