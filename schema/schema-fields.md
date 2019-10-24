---
id: schema-fields
title: Fields
---

## 简介

模式中的字段相当于顶点的属性。
例如，`User` 有 4 个字段： `age`, `name`, `username` 和 `created_at`.

![re-fields-properties](https://entgo.io/assets/er_fields_properties.png)

模式的 `Fields` 方法可以返回字段列表。例如:

```go
package schema

import (
	"time"

	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/field"
)

// User schema.
type User struct {
	ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("name"),
		field.String("username").
			Unique(),
		field.Time("created_at").
			Default(time.Now),
	}
}
```

默认情况下，所有字段都是必填的，但是可以使用 `Optional` 方法将其设置为可选的。

## 类型

`ent` 目前支持以下类型：

- go 的全部数值类型。如： `int`, `uint8`, `float64` 等等.
- `bool`
- `string`
- `time.Time`
- `[]byte` (只支持 SQL).
- `JSON` (只支持 SQL) - **实验性**.
- `Enum` (只支持 SQL).

<br/>

```go
package schema

import (
	"time"
	"net/url"

	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/field"
)

// User schema.
type User struct {
	ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age").
			Positive(),
		field.Float("rank").
			Optional(),
		field.Bool("active").
			Default(false),
		field.String("name").
			Unique(),
		field.Time("created_at").
			Default(time.Now),
		field.JSON("url", &url.URL{}).
			Optional(),
		field.JSON("strings", []string{}).
			Optional(),
		field.Enum("state").
			Values("on", "off").
			Optional(),
	}
}
```

要了解有关每种类型如何映射到其数据库类型的更多细节，请转到 [Migration](../migration/migrate.md) 部分。

## 默认值

**非唯一** 字段支持通过 `.Default` 或 `.UpdateDefault` 方法设置默认值.

```go
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Time("created_at").
			Default(time.Now),
		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now),
	}
}
```

## 验证器

字段验证器是类型为 `func(T) error` 的函数，其使用模式中的 `Validate` 方法定义，并应用在创建或更新实体之前。

字段验证器支持 `string` 和所有数值类型。

```go
package schema

import (
	"errors"
	"regexp"
	"strings"
	"time"

	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/field"
)


// Group schema.
type Group struct {
	ent.Schema
}

// Fields of the group.
func (Group) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			Match(regexp.MustCompile("[a-zA-Z_]+$")).
			Validate(func(s string) error {
				if strings.ToLower(s) == s {
					return errors.New("group name must begin with uppercase")
				}
				return nil
			}),
	}
}
```

## 内建验证器

`ent` 为支持的每种类型提供了一些内建验证器

- 数值类型:
  - `Positive()`
  - `Negative()`
  - `Min(i)` - 验证字段值是否大于 i.
  - `Max(i)` - 验证字段值是否小于 i.
  - `Range(i, j)` - 验证字段值是否在 [i, j] 区间。

- `string`
  - `MinLen(i)`
  - `MaxLen(i)`
  - `Match(regexp.Regexp)`

## 可选字段

可选字段是实体创建时是非必填的字段，在数据库中将其设置为可空字段。
不同于边，**字段默认是必填的**，但是可以使用 `Optional` 方法将其设为可选字段。


```go
// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("required_name"),
		field.String("optional_name").
			Optional(),
	}
}
```

## 区分零值

`Nillable`, 有时候你想区分字段的零值和 `nil`；例如，你想数据库中某个字段的值，既能为 `0` 又能为 `NULL`。那么 `Nillable` 就是为此而生的。

如果你有一个可选（`Optional`）的类型 `T` ，将其设为 `Nillable` 会在生成的结构体中得到一个 `*T`.
因此，如果数据库中的该字段返回 `NULL`，这个类型值为 `nil`. 否则，他将包含一个指向实际数据的指针。

例如，有如下模式：
```go
// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("required_name"),
		field.String("optional_name").
			Optional(),
		field.String("nillable_name").
			Optional().
			Nillable(),
	}
}
```

`User` 生成的结构体如下：

```go
// ent/user.go
package ent

// User entity.
type User struct {
	RequiredName string `json:"required_name,omitempty"`
	OptionalName string `json:"optional_name,omitempty"`
	NillableName *string `json:"nillable_name,omitempty"`
}
```

## 不可变字段

`Immutable`, 不可变字段是指其值只能在创建实体时赋值的字段。即，不会为该字段生成更新方法。

```go
// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
		field.Time("created_at").
			Default(time.Now),
			Immutable(),
	}
}
```

## 唯一性

可以通过 `Unique` 方法将字段定义为唯一的（唯一索引）。需要注意的是唯一字段不能有默认值。

```go
// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
		field.String("nickname").
			Unique(),
	}
}
```

## 存储键

可以通过 `StorageKey` 方法自定义存储名。
在 SQL 中，会将字段名映射为列名；在 Gremlin 中，会将字段名映射为属性名。

```go
// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			StorageKey(`old_name"`),
	}
}
```

## 索引

索引可以被定义在多个字段中，以及某些类型的边。
然而，你需要注意，目前只有 SQL 支持索引特性。

更多关于索引的内容，可以查阅 [索引](schema-indexes.md) 部分。

## 标签

可以使用 `StructTag` 方法为生成的实体添加标签。
注意，如果没有提供或者提供的标签不包含 `json`, 那么 `entc` 在生成代码时，会添加默认的 `json` 标签。

```go
// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			StructTag(`gqlgen:"gql_name"`),
	}
}
```

## 其它结构体字段

默认情况下，`entc` 会根据 `schema.Fields` 方法中配置的字段生成实体模型。
例如，给定如下模式配置：

```go
// User schema.
type User struct {
	ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age").
			Optional().
			Nillable(),
		field.String("name").
			StructTag(`gqlgen:"gql_name"`),
	}
}
```

生成的实体如下：

```go
// User is the model entity for the User schema.
type User struct {
	// Age holds the value of the "age" field.
	Age  *int	`json:"age,omitempty"`
	// Name holds the value of the "name" field.
	Name string `json:"name,omitempty" gqlgen:"gql_name"`
}
```

为了将 **不存储在数据库中的字段** 生成到实体中，请使用 [外部模板](code-gen.md/#external-templates). 例如：

```gotemplate
{{ define "model/fields/additional" }}
	{{- if eq $.Name "User" }}
		// StaticField defined by template.
		StaticField string `json:"static,omitempty"`
	{{- end }}
{{ end }}
```

生成的实体如下：

```go
// User is the model entity for the User schema.
type User struct {
	// Age holds the value of the "age" field.
	Age  *int	`json:"age,omitempty"`
	// Name holds the value of the "name" field.
	Name string `json:"name,omitempty" gqlgen:"gql_name"` 
	// StaticField defined by template.
	StaticField string `json:"static,omitempty"`
}
```

## 脱敏

可以使用 `Sensitive` 方法将 String 字段设置为敏感字段。敏感字段不会被打印，在编码时也会被忽略。

注意：敏感字段不能有 struct 标签（StructTag 方法）。

```go
// User schema.
type User struct {
	ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("password").
			Sensitive(),
	}
}
```