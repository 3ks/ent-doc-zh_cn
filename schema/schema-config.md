---
id: schema-config
title: Config
---

## 自定义表名

可以像下面的例子一样，使用 `Table` 选项自定义表名。

```go
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/field"
)

type User struct {
	ent.Schema
}

func (User) Config() ent.Config {
	return ent.Config{
		Table: "Users",
	}
}

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("name"),
	}
}
```  
