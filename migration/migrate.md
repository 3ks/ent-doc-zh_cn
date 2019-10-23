---
id: migrate
title: Database Migration
---

`ent` 提供了迁移选项，迁移工具可以保持你项目根目录下的 `ent/migrate/schema.go` 内定义的 对象模式 与 数据库模式 保持一致。 

## 自动迁移

在应用初始化时运行自动迁移的用法:

```go
if err := client.Schema.Create(ctx); err != nil {
	log.Fatalf("failed creating schema resources: %v", err)
}
```

`Create` 创建 `ent` 项目需要的所有的数据库资源。默认情况下，`Create` 工作在 *"append-only"* 模式下；这意味着，它仅创建新的表和索引、将列追加到表或扩展列类型。例如，更改 `int` 为 `bigint`.

怎么删除列或索引呢？

## 删除资源

`WithDropIndex` 和 `WithDropColumn` 是删除表的列和索引的两个选项。

```go
package main

import (
	"context"
	"log"
	
	"<project>/ent"
	"<project>/ent/migrate"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// 运行自动迁移.
	err = client.Schema.Create(
		ctx, 
		migrate.WithDropIndex(true),
		migrate.WithDropColumn(true), 
	)
	if err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
}
```

如果想以调试模式运行迁移（打印出所有 SQL 语句），请运行：

```go
err := client.Debug().Schema.Create(
	ctx, 
	migrate.WithDropIndex(true),
	migrate.WithDropColumn(true),
)
if err != nil {
	log.Fatalf("failed creating schema resources: %v", err)
}
```

## 通用标识符

默认情况下，每个表的 SQL 主键从 1 开始；这意味着多个不同的实体可以拥有相同的 ID。这不同于 AWS Neptune，它使用的是 UUID。

如果你使用了 [GraphQL](https://graphql.org/learn/schema/#scalar-types)，`ent` 将不能很好的工作，`GraphQL` 要求对象的 ID 是唯一的。

要为你的项目启用 通用ID 支持，需要将 `WithGlobalUniqueID` 选项传递给迁移工具。 

```go
package main

import (
	"context"
	"log"
	
	"<project>/ent"
	"<project>/ent/migrate"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Run migration.
	if err := client.Schema.Create(ctx, migrate.WithGlobalUniqueID(true)); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
}
```

**它是如何实现的？** `ent` 迁移会为每个实体（表）的 ID 分配 1<<32 的范围，并将此信息存储在名为 `ent_types` 的表中。
例如，type `A` 的 ID 范围 [1,4294967296) 为，type `B` 的 ID 范围将为[4294967296,8589934592)，以此类推。

需要注意的是，如果启用该选项，表的最大可能数量为 **65535**.

## 离线模式

离线模式允许你在模式修改之前将修改写入至一个 `io.Writer`.
这对于想在 SQL 语句运行之前对其进行验证，或想获取 SQL 脚本以手动运行是很有用。

**Print changes**
```go
package main

import (
	"context"
	"log"
	"os"
	
	"<project>/ent"
	"<project>/ent/migrate"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// 将迁移更改转储至标准输出
	if err := client.Schema.WriteTo(ctx, os.Stdout); err != nil {
		log.Fatalf("failed printing schema changes: %v", err)
	}
}
```

**将更改写到文件**
```go
package main

import (
	"context"
	"log"
	"os"
	
	"<project>/ent"
	"<project>/ent/migrate"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// 将迁移更改转储至 SQL 脚本文件。
	f, err := os.Create("migrate.sql")
	if err != nil {
		log.Fatalf("create migrate file: %v", err)
	}
	defer f.Close()
	if err := client.Schema.WriteTo(ctx, f); err != nil {
		log.Fatalf("failed printing schema changes: %v", err)
	}
}
```