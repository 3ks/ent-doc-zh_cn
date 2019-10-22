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

To get started, create a new `ent.Client`. For this example, we will use SQLite3.
开始, 创建一个新的 `ent.Client`. 在这个例子中，我们将使用 SQLite3.

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


## Add Your First Edge (Relation)
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

Let's add the `"cars"` edge to the `User` schema, and run `entc generate ./ent/schema`:
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

We continue our example by creating 2 cars and adding them to a user.
我们继续我们的实例： 创建两辆车并将它们添加给一个用户
```go
func CreateCars(ctx context.Context, client *ent.Client) (*ent.User, error) {
	// creating new car with model "Tesla".
	tesla, err := client.Car.
		Create().
		SetModel("Tesla").
		SetRegisteredAt(time.Now()).
		Save(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed creating car: %v", err)
	}

	// creating new car with model "Ford".
	ford, err := client.Car.
		Create().
		SetModel("Ford").
		SetRegisteredAt(time.Now()).
		Save(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed creating car: %v", err)
	}
	log.Println("car was created: ", ford)

	// create a new user, and add it the 2 cars.
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
But what about querying the `cars` edge (relation)? Here's how we do it:
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

	// what about filtering specific cars.
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

## Add Your First Inverse Edge (BackRef)
Assume we have a `Car` object and we want to get its owner; the user that this car belongs to.
For this, we have another type of edge called "inverse edge" that is defined using the `edge.From`
function.

![er-cars-owner](https://entgo.io/assets/re_cars_owner.png)

The new edge created in the diagram above is translucent, to emphasize that we don't create another
edge in the database. It's just a back-reference to the real edge (relation).

Let's add an inverse edge named `owner` to the `Car` schema, reference it to the `cars` edge
in the `User` schema, and run `entc generate ./ent/schema`.

```go
import (
	"log"

	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/edge"
)

// Edges of the Car.
func (Car) Edges() []ent.Edge {
	return []ent.Edge{
		// create an inverse-edge called "owner" of type `User`
	 	// and reference it to the "cars" edge (in User schema)
	 	// explicitly using the `Ref` method.
	 	edge.From("owner", User.Type).
	 		Ref("cars").
			// setting the edge to unique, ensure
			// that a car can have only one owner.
			Unique(),
	}
}
```
We'll continue the user/cars example above by querying the inverse edge.

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
	// query the inverse edge.
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

## Create Your Second Edge

We'll continue our example by creating a M2M (many-to-many) relationship between users and groups.

![er-group-users](https://entgo.io/assets/re_group_users.png)

As you can see, each group entity can **have many** users, and a user can **be connected to many** groups;
a simple "many-to-many" relationship. In the above illustration, the `Group` schema is the owner
of the `users` edge (relation), and the `User` entity has a back-reference/inverse edge to this
relationship named `groups`. Let's define this relationship in our schemas:

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
		 	// create an inverse-edge called "groups" of type `Group`
		 	// and reference it to the "users" edge (in Group schema)
		 	// explicitly using the `Ref` method.
			edge.From("groups", Group.Type).
				Ref("users"),
	 	}
	 }
	```

We run `entc` on the schema directory to re-generate the assets.
```console
entc generate ./ent/schema
```

## Run Your First Graph Traversal

In order to run our first graph traversal, we need to generate some data (nodes and edges, or in other words, 
entities and relations). Let's create the following graph using the framework:

![re-graph](https://entgo.io/assets/re_graph_getting_started.png)


```go

func CreateGraph(ctx context.Context, client *ent.Client) error {
	// first, create the users.
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
	// then, create the cars, and attach them to the users in the creation.
	_, err = client.Car.
		Create().
		SetModel("Tesla").
		SetRegisteredAt(time.Now()). // ignore the time in the graph.
		SetOwner(a8m).               // attach this graph to Ariel.
		Save(ctx)
	if err != nil {
		return err
	}
	_, err = client.Car.
		Create().
		SetModel("Mazda").
		SetRegisteredAt(time.Now()). // ignore the time in the graph.
		SetOwner(a8m).               // attach this graph to Ariel.
		Save(ctx)
	if err != nil {
		return err
	}
	_, err = client.Car.
		Create().
		SetModel("Ford").
		SetRegisteredAt(time.Now()). // ignore the time in the graph.
		SetOwner(neta).              // attach this graph to Neta.
		Save(ctx)
	if err != nil {
		return err
	}
	// create the groups, and add their users in the creation.
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

Now when we have a graph with data, we can run a few queries on it:

1. Get all user's cars within the group named "GitHub":

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

2. Change the query above, so that the source of the traversal is the user *Ariel*:

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
		cars, err := a8m. 						// Get the groups, that a8m is connected to:
				QueryGroups(). 					// (Group(Name=GitHub), Group(Name=GitLab),)
				QueryUsers().  					// (User(Name=Ariel, Age=30), User(Name=Neta, Age=28),)
				QueryCars().   					//
				Where(         					//
					car.Not( 					//	Get Neta and Ariel cars, but filter out
						car.ModelEQ("Mazda"),	//	those who named "Mazda"
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

3. Get all groups that have users (query with a look-aside predicate):

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

The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/start).
