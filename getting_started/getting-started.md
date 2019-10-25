---
id: getting-started
title: Quick Introduction
sidebar_label: Quick Introduction
---

`ent` 是一个基于 SQL/Gremlin 构建的易于使用但功能强大的 Go Entity 框架，其遵循以下原则：
- 轻松将你的数据建模为图结构。
- 使用代码定义模式。
- 基于代码生成静态类型。
- 精简的图遍历。

<br/>

![gopher-schema-as-code](https://entgo.io/assets/gopher-schema-as-code.png)

## 安装

```console
go get github.com/facebookincubator/ent/cmd/entc
```

完成 `entc` (为 `ent` 生成代码) 的安装后, 你应该将其放入 `PATH` 中。

## 创建第一个模式 (Schema)

进行你的项目根目录，并运行命令：

```console
entc init User
```
上面的命令将在 `<project>/ent/schema/` 目录下为 `User` 生成模式.

```go
// <project>/ent/schema/user.go

package schema

import "github.com/facebookincubator/ent"

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
    return nil
}

// Edges of the User.
func (User) Edges() []ent.Edge {
    return nil
}
```

为 `User` 模式添加两个字段：

```go
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/field"
)


// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age").
			Positive(),
		field.String("name").
			Default("unknown"),
	}
}
```

在项目根目录运行命令 `entc generate`:

```go
entc generate ./ent/schema
```

会生成以下文件：
```
ent
├── client.go
├── config.go
├── context.go
├── ent.go
├── example_test.go
├── migrate
│   ├── migrate.go
│   └── schema.go
├── predicate
│   └── predicate.go
├── schema
│   └── user.go
├── tx.go
├── user
│   ├── user.go
│   └── where.go
├── user.go
├── user_create.go
├── user_delete.go
├── user_query.go
└── user_update.go
```


## 创建第一个实体（Entity）

首先, 创建一个新的 `ent.Client`. 在这个例子中，我们将使用 SQLite3.

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
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	defer client.Close()
	// 运行自动迁移工具。
	if err := client.Schema.Create(context.Background()); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
}
```

现在，我们可以创建我们的用户了. 调用函数 `CreateUser` :
```go
func CreateUser(ctx context.Context, client *ent.Client) (*ent.User, error) {
	u, err := client.User.
		Create().
		SetAge(30).
		SetName("a8m").
		Save(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed creating user: %v", err)
	}
	log.Println("user was created: ", u)
	return u, nil
}
```

## 查询实体

`entc` 会为每个实体的模式生成到一个包内，并包含条件，默认值，验证器和存储相关的附加信息（列名，主键等）。

```go
package main

import (
	"log"

	"<project>/ent"
	"<project>/ent/user"
)

func QueryUser(ctx context.Context, client *ent.Client) (*ent.User, error) {
	u, err := client.User.
		Query().
		Where(user.NameEQ("a8m")).
        // `Only` 会查询失败，
        // 当未找到用户或找到多个（大于一个）用户时，
		Only(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed querying user: %v", err)
	}
	log.Println("user returned: ", u)
	return u, nil
}

```


## 添加第一个边 (关系)

在教程的这部分，我们要声明 `边` (关系) 到模式的另一个实体。
让我们创建另外两个名为 `Car` 和 `Group` 且有一些字段的实体。
我们使用 `entc` 去生成初始模式。

```console
entc init Car Group
```

然后我们手动添加剩下的字段：
```go
import (
	"regexp"

	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/field"
)

// Fields of the Car (car.go). 
func (Car) Fields() []ent.Field {
	return []ent.Field{
		field.String("model"),
		field.Time("registered_at"),
	}
}


// Fields of the Group (group.go).
func (Group) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			// 正则验证 group 名.
			Match(regexp.MustCompile("[a-zA-Z_]+$")),
	}
}
```

定义我们的第一个关系，定义一个从 `User` 到 `Car` 的边：
一个用户可以 **有一辆或多辆** 汽车，但是一辆汽车 **只有一个** 车主 （一对多关系）。

![er-user-cars](https://entgo.io/assets/re_user_cars.png)

让我们添加 `"cars"` 的边到 `User` 的模式中, 然后运行 `entc generate ./ent/schema`:

 ```go
 import (
 	"log"

 	"github.com/facebookincubator/ent"
 	"github.com/facebookincubator/ent/schema/edge"
 )

 // Edges of the User.
 func (User) Edges() []ent.Edge {
 	return []ent.Edge{
		edge.To("cars", Car.Type),
 	}
 }
 ```

下一个实例： 给一个用户添加两辆车。
```go
func CreateCars(ctx context.Context, client *ent.Client) (*ent.User, error) {
	// 买一辆 "Tesla".
	tesla, err := client.Car.
		Create().
		SetModel("Tesla").
		SetRegisteredAt(time.Now()).
		Save(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed creating car: %v", err)
	}

	// 买一辆 "Ford".
	ford, err := client.Car.
		Create().
		SetModel("Ford").
		SetRegisteredAt(time.Now()).
		Save(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed creating car: %v", err)
	}
	log.Println("car was created: ", ford)

	// 创建一个用户，并给他添加两辆车。
	a8m, err := client.User.
		Create().
		SetAge(30).
		SetName("a8m").
		AddCars(tesla, ford).
		Save(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed creating user: %v", err)
	}
	log.Println("user was created: ", a8m)
	return a8m, nil
}
```
怎么查询 `cars` 的边(关系)呢? 我们是这么做的:
```go
import (
	"log"

	"<project>/ent"
	"<project>/ent/car"
)

func QueryCars(ctx context.Context, a8m *ent.User) error {
	cars, err := a8m.QueryCars().All(ctx)
	if err != nil {
		return fmt.Errorf("failed querying user cars: %v", err)
	}
	log.Println("returned cars:", cars)

	// 筛选特定车型。
	ford, err := a8m.QueryCars().
		Where(car.ModelEQ("Ford")).
		Only(ctx)
	if err != nil {
		return fmt.Errorf("failed querying user cars: %v", err)
	}
	log.Println(ford)
	return nil
}
```

## 添加第一个逆边（反向引用）

假设我们有一个 `Car` 对象，并且我们想知道它的车主；即 `Car` 属于哪个 `User`.
对于这种情况，我们有另一种叫做 “逆边” 的边类型，他的定义函数是 `edge.From`.

![er-cars-owner](https://entgo.io/assets/re_cars_owner.png)

上图中半透明部分就是新的边，要强调的是，我们不会在数据库中创建这条边
它只是对上面的边的反向引用。

让我们为 `Car` 模式添加一个叫 `owner` 的逆边，将其引用至 `User` 模式中的 `cars` 边
然后运行 `entc generate ./ent/schema`.

```go
import (
	"log"

	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/edge"
)

// Edges of the Car.
func (Car) Edges() []ent.Edge {
	return []ent.Edge{
        // 创建一个类型为 `User` 名为 "owner" 的逆边
        // 并且使用 `Ref` 方法明确的将其引用至（User 模式中的） "cars" 边
	 	edge.From("owner", User.Type).
	 		Ref("cars").
            // 指定该边为唯一，确保一辆车只有一个车主。
			Unique(),
	}
}
```
接着上面的 user/cars 例子，我们来查询逆边。 

```go
import (
	"log"

	"<project>/ent"
)

func QueryCarUsers(ctx context.Context, a8m *ent.User) error {
	cars, err := a8m.QueryCars().All(ctx)
	if err != nil {
		return fmt.Errorf("failed querying user cars: %v", err)
	}
	// 查询逆边。
	for _, ca := range cars {
		owner, err := ca.QueryOwner().Only(ctx)
		if err != nil {
			return fmt.Errorf("failed querying car %q owner: %v", ca.Model, err)
		}
		log.Printf("car %q owner: %q\n", ca.Model, owner.Name)
	}
	return nil
}
```

## 创建第二个边

继续看例子，我们将在 users 和 groups 之间创建一个 M2M （多对多）的关系。

![er-group-users](https://entgo.io/assets/re_group_users.png)

如图所示，每个群组实体可以 **拥有多个** 用户，一个用户也可以 **被连接到多个** 群组，一个简单的 “多对多” 关系。
在上图中，`Group` 模式是 `users` 边（关系）的拥有者， `User` 实体有一个名为 `groups` 的反向引用/逆边。
开始定义这个多对多关系：

- `<project>/ent/schema/group.go`:

	```go
	 import (
		"log"
	
		"github.com/facebookincubator/ent"
		"github.com/facebookincubator/ent/schema/edge"
	 )
	
	 // Edges of the Group.
	 func (Group) Edges() []ent.Edge {
		return []ent.Edge{
			edge.To("users", User.Type),
		}
	 }
	```

- `<project>/ent/schema/user.go`:   
	```go
	 import (
	 	"log"
	
	 	"github.com/facebookincubator/ent"
	 	"github.com/facebookincubator/ent/schema/edge"
	 )
	
	 // Edges of the User.
	 func (User) Edges() []ent.Edge {
	 	return []ent.Edge{
			edge.To("cars", Car.Type),
		 	// 创建一个类型为 `Group` 名为 "groups" 的逆边
			edge.From("groups", Group.Type).
 	 	    //  并且使用 `Ref` 方法明确的将其引用至（Group 模式中的） "users" 边
				Ref("users"),
	 	}
	 }
	```

运行 `entc` 重新生成代码。 
```console
entc generate ./ent/schema
```

## 运行第一个图遍历

为了运行第一个图遍历，我们需要生成一些数据（节点和边，或者说实体和关系）。
让我们使用 ent 创建下面的图：

![re-graph](https://entgo.io/assets/re_graph_getting_started.png)


```go

func CreateGraph(ctx context.Context, client *ent.Client) error {
	// 首先创建一个用户
	a8m, err := client.User.
		Create().
		SetAge(30).
		SetName("Ariel").
		Save(ctx)
	if err != nil {
		return err
	}
	neta, err := client.User.
		Create().
		SetAge(28).
		SetName("Neta").
		Save(ctx)
	if err != nil {
		return err
	}
	// 然后，创建汽车，并指定其拥有者（车主）。
	_, err = client.Car.
		Create().
		SetModel("Tesla").
		SetRegisteredAt(time.Now()). // 忽略图中的时间
		SetOwner(a8m).               // 指定车主为 Ariel.
		Save(ctx)
	if err != nil {
		return err
	}
	_, err = client.Car.
		Create().
		SetModel("Mazda").
		SetRegisteredAt(time.Now()). // 忽略图中的时间
		SetOwner(a8m).               // 指定车主为 Ariel.
		Save(ctx)
	if err != nil {
		return err
	}
	_, err = client.Car.
		Create().
		SetModel("Ford").
		SetRegisteredAt(time.Now()). // 忽略图中的时间
		SetOwner(neta).              // 指定车主为 Neta.
		Save(ctx)
	if err != nil {
		return err
	}
	// 创建群组，并同时添加用户。
	_, err = client.Group.
		Create().
		SetName("GitLab").
		AddUsers(neta, a8m).
		Save(ctx)
	if err != nil {
		return err
	}
	_, err = client.Group.
		Create().
		SetName("GitHub").
		AddUsers(a8m).
		Save(ctx)
	if err != nil {
		return err
	}
	log.Println("The graph was created successfully")
	return nil
}
```

现在我们得到了一个有数据的图，我们可以运行一些查询：

1. 获取 "GitHub" 群组所有用户的全部汽车:

	```go
	import (
		"log"
		
		"<project>/ent"
		"<project>/ent/group"
	)

	func QueryGithub(ctx context.Context, client *ent.Client) error {
		cars, err := client.Group.
			Query().
			Where(group.Name("GitHub")). // (Group(Name=GitHub),)
			QueryUsers().                // (User(Name=Ariel, Age=30),)
			QueryCars().                 // (Car(Model=Tesla, RegisteredAt=<Time>), Car(Model=Mazda, RegisteredAt=<Time>),)
			All(ctx)
		if err != nil {
			return fmt.Errorf("failed getting cars: %v", err)
		}
		log.Println("cars returned:", cars)
		// Output: (Car(Model=Tesla, RegisteredAt=<Time>), Car(Model=Mazda, RegisteredAt=<Time>),)
		return nil
	}
	```

2. 修改上面的查询, 将遍历的起源修改为用户 *Ariel* （Ariel 所属群组的用户的汽车）:

	```go
	import (
		"log"
		
		"<project>/ent"
		"<project>/ent/car"
	)

	func QueryArielCars(ctx context.Context, client *ent.Client) error {
		// Get "Ariel" from previous steps.
		a8m := client.User.
			Query().
			Where(
				user.HasCars(),
				user.Name("Ariel"),
			).
			OnlyX(ctx)
		cars, err := a8m. 						// 首先获取群组，Ariel 所属的群主为:
				QueryGroups(). 					// (Group(Name=GitHub), Group(Name=GitLab),)
				QueryUsers().  					// (User(Name=Ariel, Age=30), User(Name=Neta, Age=28),)
				QueryCars().   					//
				Where(         					//
					car.Not( 					//	获取 Neta 和 Ariel 的汽车
						car.ModelEQ("Mazda"),	//	但是这里过滤掉了名为 "Mazda" 的汽车
					), 							//
				). 								//
				All(ctx)
		if err != nil {
			return fmt.Errorf("failed getting cars: %v", err)
		}
		log.Println("cars returned:", cars)
		// Output: (Car(Model=Tesla, RegisteredAt=<Time>), Car(Model=Ford, RegisteredAt=<Time>),)
		return nil
	}
	```

3. 获取有用户（非空）的群组 (使用自动生成的条件查询):

	```go
	import (
		"log"
		
		"<project>/ent"
		"<project>/ent/group"
	)

	func QueryGroupWithUsers(ctx context.Context, client *ent.Client) error {
    	groups, err := client.Group.
    		Query().
    		Where(group.HasUsers()).
    		All(ctx)
    	if err != nil {
    		return fmt.Errorf("failed getting groups: %v", err)
    	}
    	log.Println("groups returned:", groups)
    	// Output: (Group(Name=GitHub), Group(Name=GitLab),)
    	return nil
    }
    ```

完整的例子请参考： [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/start).
