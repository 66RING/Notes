---
Title:git操作
Date:2019-9-8
Tag:git
---

# Git 操作

### 初始化

``` 
git init
```

### 查看状态

```
git status
```
###查看修改

```
git diff BRANCHNAME
```

###查看提交记录

```
git log
```
### 添加修改并追踪

```
git add FILE
git add .  // 添加所有
```

### 退回

```
git reset
git reset --hard HEAD^ 退回上一个版本
git reset --hard sjaieral 退回某个版本
```

### 添加用户

```
git config --global user.name "NAME"
git config --global user.email "EMAIL"
...等等
```

### 拉取项目

```
1.获取工程
git clone URL

2.更新
git pull
```

### 添加远程仓库

```
1.gibhub上创建一个仓库

2.添加远程仓库
git remote add origin ULR

3.推到远程仓库
git push --set-upstream origin master
// 然后登录

4.记住密码
git config credential.helper store
// 再次登录就可以记住了
```

### 提交

```
git commit // 会打开文件让你添加描述
git commit -m "描述" // 快捷添加描述提交
```

### 不让git管理指定文件

```
1.新建一个 .gitignore 文件

2.写入不需要git管理等文件名
// 但是git一旦追踪某个文件那就会一只追踪
// 停止追踪 git rm --cached FILE
```

### git分支

```
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

