---
title: golang操作 mgov2篇
date: 2019-8-27
tags: mongo
---

### 打开mongo

```go
import(
  "gopkg.in/mgo.v2"
)

	url:="mongodb://localhost:27017"  // 不登录mongo的
	url:="mongodb://root:password@localhost:27017/登录的用户名" // 登录mongo的
	session,err:=mgo.Dial(url)
	defer session.Close() // 最后记得关闭 减压
	ses:=session.DB("databaseName").C("collectionName")
	if err!=nil{
		fmt.Println(err)
	}
```

### Update更新数据库

```go
import (
	"gopkg.in/mgo.v2"
	"gopkg.in/mgo.v2/bson"
)

	// Update只能更新一条，UpdateAll批量更新
	selector := bson.M{"word":"god"}  // 选择器：按word:god找，不存在返回nor found
	//bson.M{key:value}就相当于python的字典
	data := bson.M{"$set":bson.M{"word":"good"}} // 找到后用$set修改
	ses.Update(selector,data)

	// 如果要更新集合里一个数组的
```



