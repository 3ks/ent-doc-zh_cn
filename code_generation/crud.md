---
id: crud
title: CRUD API
---

如 [代码生成](code-gen.md) 部分描述的那样，定义好模式后，运行 `entc` 可以生成以下内容

- 用于和图（数据库）交互的 `Client` 和 `Tx`. 
- 每个模式的增查改删（CRUD）构建器，查看 [增查改删API](crud.md) 详情。
- 每个模式的实体 (Go struct).
- 包含用于和构建器交互的常量和条件查询方法。
- 为 SQL 提供的 `migrate` 包. 查看 [数据库迁移](../migration/migrate.md) 详情。

## 创建一个客户端

**MySQL**

```go
package main

import (
	"log"

	"<project>/ent"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	client, err := ent.Open("mysql", "<user>:<pass>@tcp(<host>:<port>)/<database>?parseTime=True")
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()
}
```

**SQLite**

```go
package main

import (
	"log"

	"<project>/ent"

	_ "github.com/mattn/go-sqlite3"
)

func main() {
	client, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()
}
```


**Gremlin (AWS Neptune)**

```go
package main

import (
	"log"

	"<project>/ent"
)

func main() {
	client, err := ent.Open("gremlin", "http://localhost:8182")
	if err != nil {
		log.Fatal(err)
	}
}
```

## 创建

创建一个实体。

**Save** 创建一个用户.

```go
a8m, err := client.User.	// UserClient.
	Create().				// User create builder.
	SetName("a8m").			// Set field value.
	SetNillableAge(age).	// Avoid nil checks.
	AddGroups(g1, g2).		// Add many edges.
	SetSpouse(nati).		// Set unique edge.
	Save(ctx)				// Create and return.
```

**SaveX** 创建一个用户；不同于 **Save**, **SaveX** 遇到错误时会引起 panics.

```go
pedro := client.Pet.	// PetClient.
	Create().			// User create builder.
	SetName("pedro").	// Set field value.
	SetOwner(a8m).		// Set owner (unique edge).
	SaveX(ctx)			// Create and return.
```

## 更新

更新一个数据库返回的实体。

```go
a8m, err = a8m.Update().	// User update builder.
	RemoveGroup(g2).		// Remove specific edge.
	ClearCard().			// Clear unique edge.
	SetAge(30).				// Set field value
	Save(ctx)				// Save and return.
```


## 根据 ID 更新

根据 ID 更新一个实体。

```go
pedro, err := client.Pet.	// PetClient.
	UpdateOneID(id).		// Pet update builder.
	SetName("pedro").		// Set field name.
	SetOwnerID(owner).		// Set unique edge, using id.
	Save(ctx)				// Save and return.
```

## 更新多个

根据条件过滤更新多个实体。

```go
n, err := client.User.			// UserClient.
	Update().					// Pet update builder.
	Where(						//
		user.Or(				// (age >= 30 OR name = "bar") 
			user.AgeEQ(30), 	//
			user.Name("bar"),	// AND
		),						//  
		user.HasFollowers(),	// UserHasFollowers()  
	).							//
	SetName("foo").				// Set field name.
	Save(ctx)					// exec and return.
```

Query edge-predicates.

```go
n, err := client.User.			// UserClient.
	Update().					// Pet update builder.
	Where(						// 
		user.HasFriendsWith(	// UserHasFriendsWith (
			user.Or(			//   age = 20
				user.Age(20),	//      OR
				user.Age(30),	//   age = 30
			)					// )
		), 						//
	).							//
	SetName("a8m").				// Set field name.
	Save(ctx)					// exec and return.
```

## 查询

Query The Graph. 查询有粉丝的用户。
```go
users, err := client.User.		// UserClient.
	Query().					// User query builder.
	Where(user.HasFollowers()).	// filter only users with followers.
	All(ctx)					// query and return.
```

获取某个用户的粉丝列表；从图中的某个节点开始图遍历。
```go
users, err := a8m.
	QueryFollowers().
	All(ctx)
```

获取某个用户的所有粉丝的所有宠物。
```go
users, err := a8m.
	QueryFollowers().
	QueryPets().
	All(ctx)
```

获取所有宠物的名字。

```go
names, err := client.Pet.
	Query().
	Select(pet.FieldName).
	Strings(ctx)
```

获取所有宠物的名字和年龄。

```go
var v []struct {
	Age  int    `json:"age"`
	Name string `json:"name"`
}
err := client.Pet.
	Query().
	Select(pet.FieldAge, pet.FieldName).
	Scan(ctx, &v)
if err != nil {
	log.Fatal(err)
}
```

你可以在 [遍历](traversals.md) 找到更多遍历的高级用法。

## 删除

删除一个实体。

```go
err := client.User.
	DeleteOne(a8m).
	Exec(ctx)
```

Delete by ID.

```go
err := client.User.
	DeleteOneID(id).
	Exec(ctx)
```

## 删除多个

根据条件过滤删除多个实体。

```go
err := client.File.
	Delete().
	Where(file.UpdatedAtLT(date))
	Exec(ctx)
```