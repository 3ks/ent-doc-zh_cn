---
id: schema-def
title: Introduction
---

## 简介

模式（Schema）描述了图中一个实体类型，比如 `User` 或 `Group`，并且可以包含以下配置：
- 字段（或属性），例如：`User` 的姓名或年龄。
- 边（或关系），例如：`User` 的群组或 `User` 的朋友。
- 数据库特定的选项，例如：索引和唯一索引。

<br/>
这里有一个模式的例子：

```go
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/field"
	"github.com/facebookincubator/ent/schema/edge"
	"github.com/facebookincubator/ent/schema/index"
)

type User struct {
	ent.Schema
}

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("name"),
		field.String("nickname").
			Unique(),
	}
}

func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("groups", Group.Type),
		edge.To("friends", User.Type),
	}
}

func (User) Index() []ent.Index {
	return []ent.Index{
		index.Fields("age", "name").
			Unique(),
	}
}
```

实体的模式通常存储在你项目根目录下的 `ent/schema` 目录下，可以使用 `entc` ，通过下面的命令生成：

```console
entc init User Group
```

## ent 只是又一个 ORM

如果您习惯通过边定义关系，这很好。建模是相同的。您可以像使用其他传统 ORM 一样使用 `ent` 进行建模。
这里有很多示例，可以帮助您从 [边](schema-edges.md) 部分开始。
