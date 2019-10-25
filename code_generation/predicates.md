---
id: predicates
title: Predicates
---

## 字段条件

- **布尔**:
  - =, !=
- **数值**:
  - =, !=, >, <, >=, <=,
  - IN, NOT IN
- **时间**:
  - =, !=, >, <, >=, <=
  - IN, NOT IN
- **字符**:
  - =, !=, >, <, >=, <=
  - IN, NOT IN
  - Contains, HasPrefix, HasSuffix
  - ContainsFold, EqualFold (**SQL** specific)
- **可选** 字段:
  - IsNil, NotNil

## 边条件

- **HasEdge**. 满足边的实体，例如：`Pet` 类型定义了 `owner` 边，要查询满足该边（有主人的宠物）的实体：

  ```go
   client.Pet.
		Query().
		Where(user.HasOwner()).
		All(ctx)
  ```

- **HasEdgeWith**. 满足边及其条件的实体列表，例如，在满足上一个例子的情况下，还要求主人的姓名为 `a8m`：
  ```go
   client.Pet.
		Query().
		Where(user.HasOwnerWith(user.Name("a8m"))).
		All(ctx)
  ```


## 非 (NOT)

```go
client.Pet.
	Query().
	Where(user.Not(user.NameHasPrefix("Ari"))).
	All(ctx)
```

## 或 (OR)

```go
client.Pet.
	Query().
	Where(
		user.Or(
			user.HasOwner(),
			user.Not(user.HasFriends()),
		)
	).
	All(ctx)
```

## 与 (AND)

```go
client.Pet.
	Query().
	Where(
		user.And(
			user.HasOwner(),
			user.Not(user.HasFriends()),
		)
	).
	All(ctx)
```

## 自定义条件

自定义条件，可以让你自行书写满足所使用方言的查询条件。

```go
pets := client.Pet.
	Query().
	Where(predicate.Pet(func(s *sql.Selector) {
		s.Where(sql.InInts(pet.OwnerColumn, 1, 2, 3))
	})).
	AllX(ctx)
```