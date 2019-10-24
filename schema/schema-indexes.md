---
id: schema-indexes
title: Indexes
---

## 符合索引

为了提高数据检索的速度或确保数据的唯一性，可以在一个或多个字段上配置索引。

```go
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/index"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

func (User) Indexes() []ent.Index {
	return []ent.Index{
        // non-unique index.
        index.Fields("field1", "field2"),
        // unique index.
        index.Fields("first_name", "last_name").
            Unique(),
	}
}
```

注意，要将单个字段设为唯一索引，请像下面的例子一样在构建器中使用 `Unique` 方法。

```go
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("phone").
			Unique(),
	}
}
```

## 边的索引

Indexes can be configured on composition of fields and edges. The main use-case
is setting uniqueness on fields under a specific relation. Let's take an example:
可以配置字段和边组成的索引，这主要用于确保某些关系下数据的唯一性。看一个例子：

![er-city-streets](https://entgo.io/assets/er_city_streets.png)

In the example above, we have a `City` with many `Street`s, and we want to set the
street name to be unique under each city.
上面的例子中，一个 `City` 有多条 `Street`, 我们想确保街道名在其所属的城市内是唯一的。

`ent/schema/city.go`
```go
// City holds the schema definition for the City entity.
type City struct {
	ent.Schema
}

// Fields of the City.
func (City) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the City.
func (City) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("streets", Street.Type),
	}
}
```

`ent/schema/street.go`
```go
// Street holds the schema definition for the Street entity.
type Street struct {
	ent.Schema
}

// Fields of the Street.
func (Street) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Street.
func (Street) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("city", City.Type).
			Ref("streets").
			Unique(),
	}
}

// Indexes of the Street.
func (Street) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("name").
			Edges("city").
			Unique(),
	}
}
```

`example.go`
```go
func Do(ctx context.Context, client *ent.Client) error {
	// 不同于 `Save`, `SaveX` 遇到错误是会引起 panics.
	tlv := client.City.
		Create().
		SetName("TLV").
		SaveX(ctx)
	nyc := client.City.
		Create().
		SetName("NYC").
		SaveX(ctx)
	// 将街道 "ST" 添加至城市 "TLV".
	client.Street.
		Create().
		SetName("ST").
		SetCity(tlv).
		SaveX(ctx)
    // 下面的操作会失败，因为城市 "TLV" 已经存在名为 "ST" 街道。
	_, err := client.Street.
		Create().
		SetName("ST").
		SetCity(tlv).
		Save(ctx)
	if err == nil {
		return fmt.Errorf("expecting creation to fail")
	}
	// 将街道 "ST" 添加至城市 "NYC".
	client.Street.
		Create().
		SetName("ST").
		SetCity(nyc).
		SaveX(ctx)
	return nil
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/edgeindex).

## 支持的方言

目前不支持 Gremlin 的索引，只支持 SQL 的索引。

