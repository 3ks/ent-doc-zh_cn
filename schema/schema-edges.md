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

## 关系

- [两种类型的一对一关系](#两种类型的一对一关系)
- [同类型的一对一关系](#同类型的一对一关系)
- [双向的一对一关系](#双向的一对一关系)
- [不同类型的一对多关系](#不同类型的一对多关系)
- [同类型的一对多关系](#同类型的一对多关系)
- [不同类型的多对多关系](#不同类型的多对多关系)
- [同类型的多对多关系](#同类型的多对多关系)
- [双向的多对多关系](#双向的多对多关系)

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

如你所见，在同类型关系的情况下，可以在一个构建器内声明边及其引用。


```diff
func (Node) Edges() []ent.Edge {
	return []ent.Edge{
+		edge.To("next", Node.Type).
+			Unique().
+			From("prev").
+			Unique(),

-      // 不必写两次。
-		edge.To("next", Node.Type).
-			Unique(),
-		edge.From("prev", Node.Type).
-			Ref("next).
-			Unique(),
	}
}
```

下面是一些与边交互的 API：

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
	// 下面的代码会生成链表： 1<->2<->3<->4<->5.
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

	// 遍历并打印列表. 如果遇到错误 `FirstX` 会 panics.
	for curr = head; curr != nil; curr = curr.QueryNext().FirstX(ctx) {
		fmt.Printf("%d ", curr.Value)
	}
	// Output: 1 2 3 4 5

	// 构建循环链表:
	// 链表的最后一个元素，没有 "next".
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
	// 检查修改是否生效：
	prev, err := head.QueryPrev().Only(ctx)
	if err != nil {
		return fmt.Errorf("getting head's prev: %v", err)
	}
	fmt.Printf("\n%v", prev.Value == tail.Value)
	// Output: true
	return nil
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2o2recur).

## 双向的一对一关系

![er-user-spouse](https://entgo.io/assets/er_user_spouse.png)

在这个用户-配偶例子中，我们有一个名为 `spouse` 的 **对称一对一** 关系。每个用户只能有 **一个配偶**。
如果用户 A 的配偶是（使用 `spouse` 关系） 是 B，那么也可以得知 B 的配偶（使用 `spouse` 关系）。 

注意，在双向关系中，不存在拥有/属于这种说法。

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

下面是一些与边交互的 API：

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

	// 查询名为 配偶 的边
    // 不同于 `Only`, `OnlyX`遇到错误会引起 panics. 
	spouse := nati.QuerySpouse().OnlyX(ctx)
	fmt.Println(spouse.Name)
	// Output: a8m

	spouse = a8m.QuerySpouse().OnlyX(ctx)
	fmt.Println(spouse.Name)
	// Output: nati

	// 查询有配偶用户的数量。
    // 不同于 `Count`, `CountX`遇到错误会引起 panics. 
	count := client.User.
		Query().
		Where(user.HasSpouse()).
		CountX(ctx)
	fmt.Println(count)
	// Output: 2

    // 获取有配偶，且其配偶姓名为 "a8m" 的用户。
	spouse = client.User.
		Query().
		Where(user.HasSpouseWith(user.Name("a8m"))).
		OnlyX(ctx)
	fmt.Println(spouse.Name)
	// Output: nati
	return nil
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2obidi).

## 不同类型的一对多关系

![er-user-pets](https://entgo.io/assets/er_user_pets.png)

在这个 用户-宠物 的例子中，用户和宠物之间存在一个 O2M （一对多）关系。
每个用户可以有 **多个** 宠物，但是一个宠物只有 **一个** 主人（用户）。如果用户 A 通过 `pets` 边添加了一个宠物 B，那么，宠物 B 可以通过 `owner` 边（反向引用边）找到他的主人。

注意，从 `Pet` （宠物）的角度来说，这就是多对一的关系（M20,many-to-one）。 

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

下面是一些与边交互的 API：

```go
func Do(ctx context.Context, client *ent.Client) error {
	// 创建两个宠物。
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
	// 创建用户的同时给他添加两个宠物。
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

    // 查询主人，不同于 `Only`, `OnlyX` 遇到错误时会引起 panics.
	owner := pedro.QueryOwner().OnlyX(ctx)
	fmt.Println(owner.Name)
	// Output: a8m

    // 遍历子图，不同于 `Count`, `CountX` 遇到错误时会引起 panics.
	count := pedro.
		QueryOwner(). // a8m
		QueryPets().  // pedro, lola
		CountX(ctx)   // count
	fmt.Println(count)
	// Output: 2
	return nil
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2m2types).

## 同类型的一对多关系

![er-tree](https://entgo.io/assets/er_tree.png)

In this example, we have a recursive O2M relation between tree's nodes and their children (or their parent).  
Each node in the tree **has many** children, and **has one** parent. If node A adds B to its children,
B can get its owner using the `owner` edge.
这个例子中，在树的节点及其子节点（或父节点）之间存在一对多（O2M）关系的关系。
树中的每个节点有 **多个** 子节点，但是它只有 **一个** 父节点。
如果节点 A 有一个子节点 B，那么节点 B 可以通过 `owner` 边找到节点 A。

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

如你所见，在同类型关系的情况下，可以在一个构建器内声明边及其引用。

```diff
func (Node) Edges() []ent.Edge {
	return []ent.Edge{
+		edge.To("children", Node.Type).
+			From("parent").
+			Unique(),

-      // 不必写两次。
-		edge.To("children", Node.Type),
-		edge.From("parent", Node.Type).
-			Ref("children").
-			Unique(),
	}
}
```

下面是一些与边交互的 API：

```go
func Do(ctx context.Context, client *ent.Client) error {
	root, err := client.Node.
		Create().
		SetValue(2).
		Save(ctx)
	if err != nil {
		return fmt.Errorf("creating the root: %v", err)
	}
	// 构建一颗这样的树：
	//
	//       2
	//     /   \
	//    1     4
	//        /   \
	//       3     5
	//
	// 不同于 `Create`, `CreateX` 遇到错误时会引起 panics.
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

    // 获取所有叶子节点（没有子节点的节点）。
	// Unlike `Int`, `IntX` panics if an error occurs.
	// 不同于 `Int`, `IntX` 遇到错误时会引起 panics.
	ints := client.Node.
		Query().                             // 全部节点.
		Where(node.Not(node.HasChildren())). // 叶子节点.
		Order(ent.Asc(node.FieldValue)).     // 根据 `value` 字段升序排序.
		GroupBy(node.FieldValue).            // 仅提取 `value` 字段。
		IntsX(ctx)
	fmt.Println(ints)
	// Output: [1 3 5]

	// 获取孤儿节点（没有父节点的节点）。
	// Unlike `Only`, `OnlyX` panics if an error occurs.
	// 不用于 `Only`, `OnlyX` 遇到错误时会引起 panics.
	orphan := client.Node.
		Query().
		Where(node.Not(node.HasParent())).
		OnlyX(ctx)
	fmt.Println(orphan)
	// Output: Node(id=1, value=2)

	return nil
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/o2mrecur).

## 不同类型的多对多关系

![er-user-groups](https://entgo.io/assets/er_user_groups.png)

在这个例子中，在群组和用户之间存在一个多对多（M2M）的关系。
每个群主可以有 **多个** 用户，每个用户也可以加入 **多个** 群组。

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

下面是一些与边交互的 API：

```go
func Do(ctx context.Context, client *ent.Client) error {
	// 不同于 `Save`, `SaveX` 遇到错误时会引起 panics.
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

	// 关系查询
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

	// 图遍历
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

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/m2m2types).

## 同类型的多对多关系

![er-following-followers](https://entgo.io/assets/er_following_followers.png)

下面这个 关注-粉丝 的例子，在用户及其粉丝之间存在一个多对多（M2M）的关系。
每个用户可以关注 **多个** 用户，也可以有 **多个** 粉丝。

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


如你所见，在同类型关系的情况下，可以在一个构建器内声明边及其引用。

```diff
func (User) Edges() []ent.Edge {
	return []ent.Edge{
+		edge.To("following", User.Type).
+			From("followers"),

-       // 不必写两次
-		edge.To("following", User.Type),
-		edge.From("followers", User.Type).
-			Ref("following"),
	}
}
```

下面是一些与边交互的 API：

```go
func Do(ctx context.Context, client *ent.Client) error {
	// 不同于 `Save`, `SaveX` 遇到错误时会引起 panics.
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

	// 查询关注/粉丝列表:

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

	// 图遍历:

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

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/m2mrecur).


## 双向的多对多关系

![er-user-friends](https://entgo.io/assets/er_user_friends.png)

In this user-friends example, we have a **symmetric M2M relation** named `friends`.
Each user can **have many** friends. If user A becomes a friend of B, B is also a friend of A.
在这个 用户-朋友 的例子中，存在一个名为 `freiends` 的双向多对多关系。
每个用户可以有 **多个** 朋友。如果用户 A 是用户 B 的朋友，那么用户 B 也肯定是用户 A 的朋友。

注意，在双向关系中，不存在拥有/属于这种说法。

`ent/schema/user.go`
```go
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("friends", User.Type),
	}
}
```

下面是一些与边交互的 API：

```go
func Do(ctx context.Context, client *ent.Client) error {
	// 不同于 `Save`, `SaveX` 遇到错误时会引起 panics.
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

	// 查询朋友列表。不同于 `All`, `AllX` 遇到错误是会引起 panics.
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

	// 图遍历：
	friends = client.User.
		Query().
		Where(user.HasFriends()).
		AllX(ctx)
	fmt.Println(friends)
	// Output: [User(id=1, age=30, name=a8m) User(id=2, age=28, name=nati)]
	return nil
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/m2mbidi).


## 必选项

Edges can be defined as required in the entity creation using the `Required` method on the builder.
可以使用构建器中的 `Required` 方法定义关系，使得实体创建时必须满足该关系。

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

比如说，无法创建一张没有户主的信用卡。 

## 索引

可以在多个字段或者某些边上添加索引。但是，需要注意的是，目前只有 SQL 支持索引特性。

更多关于索引的内容，可以查阅 [索引](schema-indexes.md) 部分。