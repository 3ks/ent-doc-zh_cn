---
id: schema-edges
title: Edges
---

## 简介

边，是实体间的关系（或者是关联）。例如，用户的宠物，群组的用户（成员）。

![er-group-users](https://entgo.io/assets/er_user_pets_groups.png)

在上面的例子中，你可以看到两个使用边声明的关系。让我们来实现他们。

1\. `pets` / `owner` 边; 用户的宠物和宠物的主人： 

`ent/schema/user.go`
```go
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/edge"
)

// User schema.
type User struct {
	ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		// ...
	}
}

// Edges of the user.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("pets", Pet.Type),
	}
}
```


`ent/schema/pet.go`
```go
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/edge"
)

// User schema.
type Pet struct {
	ent.Schema
}

// Fields of the user.
func (Pet) Fields() []ent.Field {
	return []ent.Field{
		// ...
	}
}

// Edges of the user.
func (Pet) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("owner", User.Type).
			Ref("pets").
			Unique(),
	}
}
```

如你所见，一个 `User` 实体可以拥有 **多个** `Pet`，但是一个 `Pet` 只能被 **一个** `User` 拥有。
在关系定义时，`pets` 这条边是一个 *02M*（一对多）关系，`owner` 这条边是一个 *M20*（多对一）关系。

`User` 模式拥有 `pets/owner` 关系，因为它使用了 `edge.To`，而 `Pet` 模式只是通过 `edge.From` 和 `Ref` 方法反向引用了 `User`.

因为从一个模式到另一个模式可以有多个引用，所以用 `Ref` 方法指明 `Pet` 想要引用 `User` 中的哪一条边，

可以使用 `Unique` 方法控制边（关系）的类型，下面会有更多的说明。

2\. `users` / `groups` 边; 群组包含的用户和用户所属的群组。 

`ent/schema/group.go`
```go
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/edge"
)

// Group schema.
type Group struct {
	ent.Schema
}

// Fields of the group.
func (Group) Fields() []ent.Field {
	return []ent.Field{
		// ...
	}
}

// Edges of the group.
func (Group) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("users", User.Type),
	}
}
```

`ent/schema/user.go`
```go
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/edge"
)

// User schema.
type User struct {
	ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		// ...
	}
}

// Edges of the user.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("groups", Group.Type).
			Ref("users"),
		// "pets" declared in the example above.
		edge.To("pets", Pet.Type),
	}
}
```

如你所见，一个群组实体可以有 **多个** 用户，并且一个用户也可以属于 **多个** 群组。
在关系定义时，`users` 这条边是一个 *M2M*（多对多）关系，`groups` 这条件也是一个 *M2M* 的关系。 

## To 和 From

`edge.To` 和 `edge.From` 是两个用于创建边（关系）的构建器.

在一个关系中，使用 `edge.To` 定义边的模式，拥有该关系；而使用 `edge.From` 定义边的模式；只是反向引用了该关系。

继续看一些例子，这些例子示范了如何使用边定义不同的关系。

## Relationship

- [O2O Two Types](#o2o-two-types)
- [O2O Same Type](#o2o-same-type)
- [O2O Bidirectional](#o2o-bidirectional)
- [O2M Two Types](#o2m-two-types)
- [O2M Same Type](#o2m-same-type)
- [M2M Two Types](#m2m-two-types)
- [M2M Same Type](#m2m-same-type)
- [M2M Bidirectional](#m2m-bidirectional)

## 两种类型的一对一关系

![er-user-card](https://entgo.io/assets/er_user_card.png)

在这个例子中，一个用户只有 **一张** 信用卡，一张信用卡只能有 **一个** 户主。 

在 `User` 模式中使用 `edge.To` 为信用卡定义一条名为 `card` 的边。并在 `Card` 模式中使用 `edge.From` 为用户定义一条名为 `owner` 的逆边。

`ent/schema/user.go`
```go
// Edges of the user.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("card", Card.Type).
			Unique(),
	}
}
```

`ent/schema/card.go`
```go
// Edges of the user.
func (Card) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("owner", User.Type).
			Ref("card").
			Unique().
            // 我们在构建器中添加 `Required` 方法。
            // 使得创建实体时也必须满足这条边，即：信用卡在创建时，不能没有户主。
			Required(),
	}
}
```

下面是一些与边交互的 API：
```go
func Do(ctx context.Context, client *ent.Client) error {
	a8m, err := client.User.
		Create().
		SetAge(30).
		SetName("Mashraki").
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating user: %v", err)
	}
	log.Println("user:", a8m)
	card1, err := client.Card.
		Create().
		SetOwner(a8m).
		SetNumber("1020").
		SetExpired(time.Now().Add(time.Minute)).
		Save(ctx)
	if err != nil {
    	return fmt.Errorf("creating card: %v", err)
    }
	log.Println("card:", card1)
	// 返回满足条件的用户的信用卡，并且期望的数量是 1.
	card2, err := a8m.QueryCard().Only(ctx)
	if err != nil {
		return fmt.Errorf("querying card: %v", err)
    }
	log.Println("card:", card2)
    // 在信用卡实体中，可以通过反向引用查询其户主。
	owner, err := card2.QueryOwner().Only(ctx)
	if err != nil {
		return fmt.Errorf("querying owner: %v", err)
    }
	log.Println("owner:", owner)
	return nil
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2o2types).

## 同类型的一对一关系

![er-linked-list](https://entgo.io/assets/er_linked_list.png)

在这个链表例子中，我们有一个名为 `next`/`prev` 的递归关系。链表中的每个节点都只有 **一个** `next`和 `prev` 节点。
如果，节点 A 可以通过 `next` 指向 节点 B，则节点 B 也可以通过 `prev`(反向引用) 指向节点 A。

`ent/schema/node.go`
```go
// Edges of the Node.
func (Node) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("next", Node.Type).
			Unique().
			From("prev").
			Unique(),
	}
}
```

As you can see, in cases of relations of the same type, you can declare the edge and its
reference in the same builder.


```go
func (Node) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("next", Node.Type).
			Unique().
			From("prev").
			Unique(),
	}
}
```

The API for interacting with these edges is as follows:

```go
func Do(ctx context.Context, client *ent.Client) error {
	head, err := client.Node.
		Create().
		SetValue(1).
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating the head: %v", err)
	}
	curr := head
	// Generate the following linked-list: 1<->2<->3<->4<->5.
	for i := 0; i < 4; i++ {
		curr, err = client.Node.
			Create().
			SetValue(curr.Value + 1).
			SetPrev(curr).
			Save(ctx)
		if err != nil {
			return err
		}
	}

	// Loop over the list and print it. `FirstX` panics if an error occur.
	for curr = head; curr != nil; curr = curr.QueryNext().FirstX(ctx) {
		fmt.Printf("%d ", curr.Value)
	}
	// Output: 1 2 3 4 5

	// Make the linked-list circular:
	// The tail of the list, has no "next".
	tail, err := client.Node.
		Query().
		Where(node.Not(node.HasNext())).
		Only(ctx)
	if err != nil {
		return fmt.Errorf("getting the tail of the list: %v", tail)
	}
	tail, err = tail.Update().SetNext(head).Save(ctx)
	if err != nil {
		return err
	}
	// Check that the change actually applied:
	prev, err := head.QueryPrev().Only(ctx)
	if err != nil {
		return fmt.Errorf("getting head's prev: %v", err)
	}
	fmt.Printf("\n%v", prev.Value == tail.Value)
	// Output: true
	return nil
}
```

The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2o2recur).

## O2O Bidirectional

![er-user-spouse](https://entgo.io/assets/er_user_spouse.png)

In this user-spouse example, we have a **symmetric O2O relation** named `spouse`. Each user can **have only one** spouse.
If user A sets its spouse (using `spouse`) to B, B can get its spouse using the `spouse` edge.

Note that there are no owner/inverse terms in cases of bidirectional edges.

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("spouse", User.Type).
			Unique(),
	}
}
```

The API for interacting with this edge is as follows:

```go
func Do(ctx context.Context, client *ent.Client) error {
	a8m, err := client.User.
		Create().
		SetAge(30).
		SetName("a8m").
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating user: %v", err)
	}
	nati, err := client.User.
		Create().
		SetAge(28).
		SetName("nati").
		SetSpouse(a8m).
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating user: %v", err)
	}

	// Query the spouse edge.
	// Unlike `Only`, `OnlyX` panics if an error occurs.
	spouse := nati.QuerySpouse().OnlyX(ctx)
	fmt.Println(spouse.Name)
	// Output: a8m

	spouse = a8m.QuerySpouse().OnlyX(ctx)
	fmt.Println(spouse.Name)
	// Output: nati

	// Query how many users have a spouse.
	// Unlike `Count`, `CountX` panics if an error occurs.
	count := client.User.
		Query().
		Where(user.HasSpouse()).
		CountX(ctx)
	fmt.Println(count)
	// Output: 2

	// Get the user, that has a spouse with name="a8m".
	spouse = client.User.
		Query().
		Where(user.HasSpouseWith(user.Name("a8m"))).
		OnlyX(ctx)
	fmt.Println(spouse.Name)
	// Output: nati
	return nil
}
```

The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2obidi).

## O2M Two Types

![er-user-pets](https://entgo.io/assets/er_user_pets.png)

In this user-pets example, we have a O2M relation between user and its pets.
Each user **has many** pets, and a pet **has one** owner.
If user A adds a pet B using the `pets` edge, B can get its owner using the `owner` edge (the back-reference edge).

Note that this relation is also a M2O (many-to-one) from the point of view of the `Pet` schema. 

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("pets", Pet.Type),
	}
}
```

`ent/schema/pet.go`
```go
// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("owner", User.Type).
			Ref("pets").
			Unique(),
	}
}
```

The API for interacting with these edges is as follows:

```go
func Do(ctx context.Context, client *ent.Client) error {
	// Create the 2 pets.
	pedro, err := client.Pet.
		Create().
		SetName("pedro").
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating pet: %v", err)
	}
	lola, err := client.Pet.
		Create().
		SetName("lola").
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating pet: %v", err)
	}
	// Create the user, and add its pets on the creation.
	a8m, err := client.User.
		Create().
		SetAge(30).
		SetName("a8m").
		AddPets(pedro, lola).
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating user: %v", err)
	}
	fmt.Println("User created:", a8m)
	// Output: User(id=1, age=30, name=a8m)

	// Query the owner. Unlike `Only`, `OnlyX` panics if an error occurs.
	owner := pedro.QueryOwner().OnlyX(ctx)
	fmt.Println(owner.Name)
	// Output: a8m

	// Traverse the sub-graph. Unlike `Count`, `CountX` panics if an error occurs.
	count := pedro.
		QueryOwner(). // a8m
		QueryPets().  // pedro, lola
		CountX(ctx)   // count
	fmt.Println(count)
	// Output: 2
	return nil
}
```
The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2m2types).

## O2M Same Type

![er-tree](https://entgo.io/assets/er_tree.png)

In this example, we have a recursive O2M relation between tree's nodes and their children (or their parent).  
Each node in the tree **has many** children, and **has one** parent. If node A adds B to its children,
B can get its owner using the `owner` edge.


`ent/schema/node.go`
```go
// Edges of the Node.
func (Node) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("children", Node.Type).
			From("parent").
			Unique(),
	}
}
```

As you can see, in cases of relations of the same type, you can declare the edge and its
reference in the same builder.

```go
func (Node) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("children", Node.Type).
			From("parent").
			Unique(),
	}
}
```

The API for interacting with these edges is as follows:

```go
func Do(ctx context.Context, client *ent.Client) error {
	root, err := client.Node.
		Create().
		SetValue(2).
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating the root: %v", err)
	}
	// Add additional nodes to the tree:
	//
	//       2
	//     /   \
	//    1     4
	//        /   \
	//       3     5
	//
	// Unlike `Create`, `CreateX` panics if an error occurs.
	n1 := client.Node.
		Create().
		SetValue(1).
		SetParent(root).
		SaveX(ctx)
	n4 := client.Node.
		Create().
		SetValue(4).
		SetParent(root).
		SaveX(ctx)
	n3 := client.Node.
		Create().
		SetValue(3).
		SetParent(n4).
		SaveX(ctx)
	n5 := client.Node.
		Create().
		SetValue(5).
		SetParent(n4).
		SaveX(ctx)

	fmt.Println("Tree leafs", []int{n1.Value, n3.Value, n5.Value})
	// Output: Tree leafs [1 3 5]

	// Get all leafs (nodes without children).
	// Unlike `Int`, `IntX` panics if an error occurs.
	ints := client.Node.
		Query().                             // All nodes.
		Where(node.Not(node.HasChildren())). // Only leafs.
		Order(ent.Asc(node.FieldValue)).     // Order by their `value` field.
		GroupBy(node.FieldValue).            // Extract only the `value` field.
		IntsX(ctx)
	fmt.Println(ints)
	// Output: [1 3 5]

	// Get orphan nodes (nodes without parent).
	// Unlike `Only`, `OnlyX` panics if an error occurs.
	orphan := client.Node.
		Query().
		Where(node.Not(node.HasParent())).
		OnlyX(ctx)
	fmt.Println(orphan)
	// Output: Node(id=1, value=2)

	return nil
}
```

The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2mrecur).

## M2M Two Types

![er-user-groups](https://entgo.io/assets/er_user_groups.png)

In this groups-users example, we have a M2M relation between groups and their users.
Each group **has many** users, and each user can be joined to **many** groups.

`ent/schema/group.go`
```go
// Edges of the Group.
func (Group) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("users", User.Type),
	}
}
```

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("groups", Group.Type).
			Ref("users"),
	}
}
```

The API for interacting with these edges is as follows:

```go
func Do(ctx context.Context, client *ent.Client) error {
	// Unlike `Save`, `SaveX` panics if an error occurs.
	hub := client.Group.
		Create().
		SetName("GitHub").
		SaveX(ctx)
	lab := client.Group.
		Create().
		SetName("GitLab").
		SaveX(ctx)
	a8m := client.User.
		Create().
		SetAge(30).
		SetName("a8m").
		AddGroups(hub, lab).
		SaveX(ctx)
	nati := client.User.
		Create().
		SetAge(28).
		SetName("nati").
		AddGroups(hub).
		SaveX(ctx)

	// Query the edges.
	groups, err := a8m.
		QueryGroups().
		All(ctx)
	if err != nil {
		return fmt.Errorf("querying a8m groups: %v", err)
	}
	fmt.Println(groups)
	// Output: [Group(id=1, name=GitHub) Group(id=2, name=GitLab)]

	groups, err = nati.
		QueryGroups().
		All(ctx)
	if err != nil {
		return fmt.Errorf("querying nati groups: %v", err)
	}
	fmt.Println(groups)
	// Output: [Group(id=1, name=GitHub)]

	// Traverse the graph.
	users, err := a8m.
		QueryGroups().                                           // [hub, lab]
		Where(group.Not(group.HasUsersWith(user.Name("nati")))). // [lab]
		QueryUsers().                                            // [a8m]
		QueryGroups().                                           // [hub, lab]
		QueryUsers().                                            // [a8m, nati]
		All(ctx)
	if err != nil {
		return fmt.Errorf("traversing the graph: %v", err)
	}
	fmt.Println(users)
	// Output: [User(id=1, age=30, name=a8m) User(id=2, age=28, name=nati)]
	return nil
}
```

The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/m2m2types).

## M2M Same Type

![er-following-followers](https://entgo.io/assets/er_following_followers.png)

In this following-followers example, we have a M2M relation between users to their followers. Each user 
can follow **many** users, and can have **many** followers.

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("following", User.Type).
			From("followers"),
	}
}
```


As you can see, in cases of relations of the same type, you can declare the edge and its
reference in the same builder.

```go
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("following", User.Type).
			From("followers"),
	}
}
```

The API for interacting with these edges is as follows:

```go
func Do(ctx context.Context, client *ent.Client) error {
	// Unlike `Save`, `SaveX` panics if an error occurs.
	a8m := client.User.
		Create().
		SetAge(30).
		SetName("a8m").
		SaveX(ctx)
	nati := client.User.
		Create().
		SetAge(28).
		SetName("nati").
		AddFollowers(a8m).
		SaveX(ctx)

	// Query following/followers:

	flw := a8m.QueryFollowing().AllX(ctx)
	fmt.Println(flw)
	// Output: [User(id=2, age=28, name=nati)]

	flr := a8m.QueryFollowers().AllX(ctx)
	fmt.Println(flr)
	// Output: []

	flw = nati.QueryFollowing().AllX(ctx)
	fmt.Println(flw)
	// Output: []

	flr = nati.QueryFollowers().AllX(ctx)
	fmt.Println(flr)
	// Output: [User(id=1, age=30, name=a8m)]

	// Traverse the graph:

	ages := nati.
		QueryFollowers().       // [a8m]
		QueryFollowing().       // [nati]
		GroupBy(user.FieldAge). // [28]
		IntsX(ctx)
	fmt.Println(ages)
	// Output: [28]

	names := client.User.
		Query().
		Where(user.Not(user.HasFollowers())).
		GroupBy(user.FieldName).
		StringsX(ctx)
	fmt.Println(names)
	// Output: [a8m]
	return nil
}
```

The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/m2mrecur).


## M2M Bidirectional

![er-user-friends](https://entgo.io/assets/er_user_friends.png)

In this user-friends example, we have a **symmetric M2M relation** named `friends`.
Each user can **have many** friends. If user A becomes a friend of B, B is also a friend of A.

Note that there are no owner/inverse terms in cases of bidirectional edges.

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("friends", User.Type),
	}
}
```

The API for interacting with these edges is as follows:

```go
func Do(ctx context.Context, client *ent.Client) error {
	// Unlike `Save`, `SaveX` panics if an error occurs.
	a8m := client.User.
		Create().
		SetAge(30).
		SetName("a8m").
		SaveX(ctx)
	nati := client.User.
		Create().
		SetAge(28).
		SetName("nati").
		AddFriends(a8m).
		SaveX(ctx)

	// Query friends. Unlike `All`, `AllX` panics if an error occurs.
	friends := nati.
		QueryFriends().
		AllX(ctx)
	fmt.Println(friends)
	// Output: [User(id=1, age=30, name=a8m)]

	friends = a8m.
		QueryFriends().
		AllX(ctx)
	fmt.Println(friends)
	// Output: [User(id=2, age=28, name=nati)]

	// Query the graph:
	friends = client.User.
		Query().
		Where(user.HasFriends()).
		AllX(ctx)
	fmt.Println(friends)
	// Output: [User(id=1, age=30, name=a8m) User(id=2, age=28, name=nati)]
	return nil
}
```

The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/m2mbidi).


## Required

Edges can be defined as required in the entity creation using the `Required` method on the builder.

```go
// Edges of the user.
func (Card) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("owner", User.Type).
			Ref("card").
			Unique().
			Required(),
	}
}
```

If the example above, a card entity cannot be created without its owner. 

## Indexes

Indexes can be defined on multi fields and some types of edges as well.
However, you should note, that this is currently an SQL-only feature.

Read more about this in the [Indexes](schema-indexes.md) section.