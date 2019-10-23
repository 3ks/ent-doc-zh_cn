---
id: schema-fields
title: Fields
---

## Quick Summary

Fields (or properties) in the schema are the attributes of the vertex. For example, a `User`
with 4 fields: `age`, `name`, `username` and `created_at`:
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
- `[]byte` (只支持 SQL 方言).
- `JSON` (只支持 SQL 方言) - **实验性**.
- `Enum` (只支持 SQL 方言).

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

字段验证器是类型的 `func(T) error` 函数，其使用模式中的 `Validate` 方法定义，并应用在创建或更新实体之前。

The supported types of field validators are `string` and all numeric types.

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

## Built-in Validators

The framework provides a few built-in validators for each type:

- Numeric types:
  - `Positive()`
  - `Negative()`
  - `Min(i)` - Validate that the given value is > i.
  - `Max(i)` - Validate that the given value is < i.
  - `Range(i, j)` - Validate that the given value is within the range [i, j].

- `string`
  - `MinLen(i)`
  - `MaxLen(i)`
  - `Match(regexp.Regexp)`

## Optional

Optional fields are fields that are not required in the entity creation, and
will be set to nullable fields in the database.
Unlike edges, **fields are required by default**, and setting them to
optional should be done explicitly using the `Optional` method.


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

## Nillable
Sometimes you want to be able to distinguish between the zero value of fields
and `nil`; for example if the database column contains `0` or `NULL`.
The `Nillable` option exists exactly for this.

If you have an `Optional` field of type `T`, setting it to `Nillable` will generate
a struct field with type `*T`. Hence, if the database returns `NULL` for this field,
the struct field will be `nil`. Otherwise, it will contains a pointer to the actual data.

For example, given this schema:
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

The generated struct for the `User` entity will be as follows:

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

## Immutable

Immutable fields are fields that can be set only in the creation of the entity.
i.e., no setters will be generated for the entity updater.

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

## Uniqueness
Fields can be defined as unique using the `Unique` method.
Note that unique fields cannot have default values.

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

## Storage Key

Custom storage name can be configured using the `StorageKey` method.  
It's mapped to a column name in SQL dialects and to property name in Gremlin.

```go
// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			StorageKey(`old_name"`),
	}
}
```

## Indexes
Indexes can be defined on multi fields and some types of edges as well.
However, you should note, that this is currently an SQL-only feature.

Read more about this in the [Indexes](schema-indexes.md) section.

## Struct Tags

Custom struct tags can be added to the generated entities using the `StructTag`
method. Note that if this option was not provided, or provided and did not
contain the `json` tag, the default `json` tag will be added with the field name.

```go
// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			StructTag(`gqlgen:"gql_name"`),
	}
}
```

## Additional Struct Fields

By default, `entc` generates the entity model with fields that are configured in the `schema.Fields` method.
For example, given this schema configuration:

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

The generated model will be as follows:

```go
// User is the model entity for the User schema.
type User struct {
	// Age holds the value of the "age" field.
	Age  *int	`json:"age,omitempty"`
	// Name holds the value of the "name" field.
	Name string `json:"name,omitempty" gqlgen:"gql_name"`
}
```

In order to add additional fields to the generated struct **that are not stored in the database**,
use [external templates](code-gen.md/#external-templates). For example:

```gotemplate
{{ define "model/fields/additional" }}
	{{- if eq $.Name "User" }}
		// StaticField defined by template.
		StaticField string `json:"static,omitempty"`
	{{- end }}
{{ end }}
```

The generated model will be as follows:

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

## Sensitive Fields

String fields can be defined as sensitive using the `Sensitive` method. Sensitive fields
won't be printed and they will be omitted when encoding.  

Note that sensitive fields cannot have struct tags.

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