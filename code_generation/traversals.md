---
id: traversals
title: Graph Traversal
---

为了举例，我们会生成一个下面这样的图：


![er-traversal-graph](https://entgo.io/assets/er_traversal_graph.png)

第一步，生成 3 个模式： `Pet`, `User`, `Group`.

```console
entc init Pet User Group
```

然后，为模式添加一些必要的字段和边：

`ent/schema/pet.go`

```go
// Pet holds the schema definition for the Pet entity.
type Pet struct {
	ent.Schema
}

// Fields of the Pet.
func (Pet) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("friends", Pet.Type),
		edge.From("owner", User.Type).
			Ref("pets").
			Unique(),
	}
}
``` 

`ent/schema/user.go`

```go
// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("name"),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("pets", Pet.Type),
		edge.To("friends", User.Type),
		edge.From("groups", Group.Type).
			Ref("users"),
	}
}
``` 

`ent/schema/group.go`

```go
// Group holds the schema definition for the Group entity.
type Group struct {
	ent.Schema
}

// Fields of the Group.
func (Group) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Group.
func (Group) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("users", User.Type),
		edge.To("admin", User.Type).
			Unique(),
	}
}
``` 

让我们开始编写用于填充图的顶点和边的代码：

```go
func Gen(ctx context.Context, client *ent.Client) error {
	hub, err := client.Group.
		Create().
		SetName("Github").
		Save(ctx)
	if err != nil {
		return fmt.Errorf("failed creating the group: %v", err)
	}
	// 为群组添加一个 admin.
	// 不同于 `Save`, `SaveX` 遇到错误时会引起 panics.
	dan := client.User.
		Create().
		SetAge(29).
		SetName("Dan").
		AddManage(hub).
		SaveX(ctx)

	// 创建 "Ariel" 用户和他的宠物
	a8m := client.User.
		Create().
		SetAge(30).
		SetName("Ariel").
		AddGroups(hub).
		AddFriends(dan).
		SaveX(ctx)
	pedro := client.Pet.
		Create().
		SetName("Pedro").
		SetOwner(a8m).
		SaveX(ctx)
	xabi := client.Pet.
		Create().
		SetName("Xabi").
		SetOwner(a8m).
		SaveX(ctx)

	// 创建 "Alex" 用户和他的宠物。
	alex := client.User.
		Create().
		SetAge(37).
		SetName("Alex").
		SaveX(ctx)
	coco := client.Pet.
		Create().
		SetName("Coco").
		SetOwner(alex).
		AddFriends(pedro).
		SaveX(ctx)

	fmt.Println("Pets created:", pedro, xabi, coco)
	// Output:
	// Pets created: Pet(id=1, name=Pedro) Pet(id=2, name=Xabi) Pet(id=3, name=Coco)
	return nil
}
```

再看一下我们要遍历的图及其代码：

![er-traversal-graph-gopher](https://entgo.io/assets/er_traversal_graph_gopher.png)

上面的遍历开始于一个 `Group` 实体：通过 `admin` 边、 `friends` 边、`pets` 边找到他们的宠物，然后再获取每个宠物的朋友（宠物的 `frieds` 边）的主人。

```go
func Traverse(ctx context.Context, client *ent.Client) error {
	owner, err := client.Group.			// GroupClient.
		Query().                     	// Query builder.
		Where(group.Name("Github")). 	// 要求群组名为 Github 
		QueryAdmin().                	// 找到 Dan.
		QueryFriends().              	// 找到 Dan 的朋友列表: [Ariel].
		QueryPets().                 	// 他们的宠物列表: [Pedro, Xabi].
		QueryFriends().              	// Pedro 的朋友: [Coco], Xabi 的朋友: [].
		QueryOwner().                	// Coco 的主人: Alex.
		Only(ctx)                    	// 本次遍历中期望只返回一个实体。
	if err != nil {
		return fmt.Errorf("failed querying the owner: %v", err)
	}
	fmt.Println(owner)
	// Output:
	// User(id=3, age=37, name=Alex)
	return nil
}
```

下面这个图又怎么遍历呢？

![er-traversal-graph-gopher-query](https://entgo.io/assets/er_traversal_graph_gopher_query.png)

我们想获取某个群组的 `admin` 的 `friend` 的全部宠物。

```go
func Traverse(ctx context.Context, client *ent.Client) error {
	pets, err := client.Pet.
		Query().
		Where(
			pet.HasOwnerWith(
				user.HasFriendsWith(
					user.HasManage(),
				),
			),
		).
		All(ctx)
	if err != nil {
		return fmt.Errorf("failed querying the pets: %v", err)
	}
	fmt.Println(pets)
	// Output:
	// [Pet(id=1, name=Pedro) Pet(id=2, name=Xabi)]
	return nil
}
```

完整的实例请参考 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/traversal).
