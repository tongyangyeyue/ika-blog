---
title: go中的bson包中的E、D、M、A
date: 2021-01-19 16:18:26
tags: golang mongo
---
在mongodb官方提供的go语言驱动中，要操作数据库免不了要和bson包中的E、D、M、A这4个东西打交道。由于迷惑的命名，这里就简单说明一下他们。


### bson.E
bson.E其实是一个struct，他的定义是这样的
```
type E struct {
	Key   string
	Value interface{}
}
```
也就是说，他只能是一个指定Key/Value的元素，所以使用的时候也很简单，也一般不单独使用
```
orgCode := bson.E{
	Key:   "orgCode",
	Value: "general",
}
```

### bson.D
bson.D是一个bson.D的数组，也是我们比较常见的一种使用格式
```
type D []E
```
使用时，也就是数组的使用方式
```
fndOrg := bson.D{
	{Key: "orgCode", Value: "general"},
	{Key: "orgName", Value: "总部"},
}
// 或者
fndJob := bson.D{
	{"jobCode", "manager"},
	{"jobName", "经理"},
}

cur, err := collection.Find(ctx, bson.D{
	{"jobCode", "manager"},
})
```
### bson.M
bson.M其实是map类型，他的功能和bson.D类似，不过bson.M顺序和声明的顺序可能不同
```
type M map[string]interface{}
```
使用时
```
fndOrg := bson.M{
	"orgCode": "general",
	"orgName": "总部",
}

cur, err := collection.Find(ctx, fndOrg)
```

### bson.A
bson.A是一个数组，多数使用情况是对应bson中的数组，它的作用和bson.D不同
```
type A []interface{}
```
使用方式
```
result, err := collection.UpdateOne(ctx, bson.D{
	{"jobCode", "manager"},
}, bson.D{
	{"$set", bson.D{
		{"orgList", bson.A{
			"general", "east", "north", "south",
		}},
	}},
})
```