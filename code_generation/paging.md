---
id: paging
title: Paging And Ordering
---

## Limit

`Limit` 将限制查询结果的实体数量为 `n`.

```go
users, err := client.User.
	Query().
	Limit(n).
	All(ctx)
```


## Offset

`Offset` 设置查询结果第一个顶点的位置。 

```go
users, err := client.User.
	Query().
	Offset(10).
	All(ctx)
```

## Ordering

`Order` 返回按照一个或多个字段的值排序的结果。

```go
users, err := client.User.Query().
	Order(ent.Asc(user.FieldName)).
	All(ctx)
```