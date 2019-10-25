---
id: aggregate
title: Aggregation
---

## Group By

所有用户按照 `name` 和 `age` 分组（Group by），并计算他们的年龄之和。

```go
package main

import (
	"context"
	
	"<project>/ent"
	"<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
	var v []struct {
		Name  string `json:"name"`
		Age   int    `json:"age"`
		Sum   int    `json:"sum"`
		Count int    `json:"count"`
	}
	err := client.User.Query().
		GroupBy(user.FieldName, user.FieldAge).
		Aggregate(ent.Count(), ent.Sum(user.FieldAge)).
		Scan(ctx, &v)
}
```

根据单个字段分组。

```go
package main

import (
	"context"
	
	"<project>/ent"
	"<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
	names, err := client.User.
		Query().
		GroupBy(user.FieldName).
		Strings(ctx)
}
```