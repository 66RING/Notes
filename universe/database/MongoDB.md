---
title: mongoDB usage
date: 2019-2-29
tags: mongoDB
---

### 结构:

-db(database)
-collections
-document

### 基本指令:

-show db
-use <数据库名>
	-进入指定数据库
-db
	-显示当前数据库
-show collections
    -显示数据库里所有的集合

### 数据库CRUD(增删改查)操作：

-向数据库中写入文档
		db.<collection>.insert(doc)

-查询集合中的文档
		db.<collection>.find() #括号里写条件{add:'asf'}等
		db.<collection>.find().count()#查询数量

