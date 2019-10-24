---
id: schema-mixin
title: Mixin
---
 
`Mixin` 允许你创建可重复使用的 `ent.Schema` 的代码。

`ent.Mixin` 接口定义如下：

```go
type Mixin interface {
	Fields() []ent.Field
}
```

## 例子

一个 `Mixin` 的常见用法是，将通用字段与你的模式混合在一起。

```go
// -------------------------------------------------
// Mixin 定义部分

// TimeMixin 需要实现 ent.Mixin 接口。
// 时间字段及其定义。
type TimeMixin struct{}

func (TimeMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Time("created_at").
			Immutable().
			Default(time.Now),
		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now),
	}
}

// DetailsMixin 需要实现 ent.Mixin 接口。
// 详情字段及其定义。
type DetailsMixin struct{}

func (DetailsMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age").
			Positive(),
		field.String("name").
			NotEmpty(),
	}
}

// -------------------------------------------------
// Schema 定义部分

// User 混入了 TimeMixin 和 DetailsMixin 字段。
// 因此 User 有 5 个字段： `created_at`, `updated_at`, `age`, `name` and `nickname`.
type User struct {
	ent.Schema
}

func (User) Mixin() []ent.Mixin {
	return []ent.Mixin{
		TimeMixin{},
		DetailsMixin{},
	}
}

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("nickname").
			Unique(),
	}
}

// Pet 混入了 DetailsMixin 字段。
// 因此 Pet 有 3 个字段： `age`, `name` and `weight`.
type Pet struct {
	ent.Schema
}

func (Pet) Mixin() []ent.Mixin {
	return []ent.Mixin{
		DetailsMixin{},
	}
}

func (Pet) Fields() []ent.Field {
	return []ent.Field{
		field.Float("weight"),
	}
}
```
