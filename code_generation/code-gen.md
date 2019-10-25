---
id: code-gen
title: Introduction
---

## 安装

`ent` 有一个代码生成工具—— `entc`. 可以通过下面的命令安装 `entc`:

```bash
go get github.com/facebookincubator/ent/cmd/entc
``` 

## 初始化一个新的模式

想要生成一个或多个模式的模板，可以像下面这样运行 `entc init`:

```bash
entc init User Pet
```

`init` 会在 `ent/schema` 目录下生成两个模式(`user.go` 和 `pet.go`).
一般地，`ent` 目录位于项目根目录下，如果 `ent` 目录不存在，`entc` 会自动创建该目录。

## 生成代码

在添加好 [字段](../schema/schema-fields.md) 和 [边](../schema/schema-edges.md) 以后，你可以通过下面的命令生成可以使用的代码。

```bash
entc generate ./ent/schema
```

`generate` 命令生成的代码包含以下内容：

- 用于和图（数据库）交互的 `Client` 和 `Tx`. 
- 每个模式的增查改删（CRUD）构建器，查看 [增查改删API](crud.md) 详情。
- 每个模式的实体 (Go struct).
- 包含用于和构建器交互的常量和条件查询方法。
- 为 SQL 提供的 `migrate` 包. 查看 [数据库迁移](../migration/migrate.md) 详情。

## `entc` 和 `ent` 的版本兼容性

在一个项目中，你需要确保 `entc` 和 `ent` 的版本是 **完全相同的**。一个可选的办法是，运行 `entc` 时，要求 `go generate` 使用 `go.mod` 文件中中使用的版本。

如果你的项目还没有使用 [Go modules](https://github.com/golang/go/wiki/Modules#quick-start), 你应该做的第一步是:

```console
go mod init <project>
```

然后，再运行下面的命令将 `ent` 添加至 `go.mod` 文件：

```console
go get github.com/facebookincubator/ent/cmd/entc
```

添加一个 `generate.go` 文件到 `<project>/ent` 目录下：

```go
package ent

//go:generate go run github.com/facebookincubator/ent/cmd/entc generate ./schema
```

最后，你可以在项目根目录下运行 `go generate ./ent` ，然后就可以运行 `entc` 工具为项目生成代码了。

## 代码生成选项

有关代码生成的更多选项，请运行 `entc generate -h`:

```console
generate go code for the schema directory

Usage:
  entc generate [flags] path

Examples:
  entc generate ./ent/schema
  entc generate github.com/a8m/x

Flags:
      --header string         override codegen header
  -h, --help                  help for generate
      --idtype [int string]   type of the id field (default int)
      --storage strings       list of storage drivers to support (default [sql])
      --target string         target directory for codegen
      --template strings      external templates to execute
```

## 储存选项

`entc` 可以为 SQL 和 Gremlin 生成代码，默认是为 SQL 生成代码。

## 外部模板

`entc` 可以接受运行 Go 外部模板。如果外部模板名与 `entc` 的某个模板名重复，那么外部模板会覆盖 `entc` 的模板。否则，`entc` 会运行并输出该模板的内容到一个与该模板同名的文件。

这里有一个 GraphQL API 自定义模板的例子- [Github](https://github.com/facebookincubator/ent/blob/master/entc/integration/template/ent/template/node.tmpl).

## 通过包调用 `entc`

另一种运行 `entc` 的方法是像下面的代码一样通过包调用：

```go
package main

import (
	"log"

	"github.com/facebookincubator/ent/entc"
	"github.com/facebookincubator/ent/entc/gen"
	"github.com/facebookincubator/ent/schema/field"
)

func main() {
	err := entc.Generate("./schema", &gen.Config{
		Header: "// Your Custom Header",
		IDType: &field.TypeInfo{Type: field.TypeInt},
	})
	if err != nil {
		log.Fatal("running ent codegen:", err)
	}
}
```

完整的实例请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/entcpkg).


## 模式的描述

想要获取模式的描述，请运行：

```bash
entc describe ./ent/schema
```

输出实例如下：

```console
Pet:
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| Field |  Type   | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| id    | int     | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
	| name  | string  | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	+-------+------+---------+---------+----------+--------+----------+
	| Edge  | Type | Inverse | BackRef | Relation | Unique | Optional |
	+-------+------+---------+---------+----------+--------+----------+
	| owner | User | true    | pets    | M2O      | true   | true     |
	+-------+------+---------+---------+----------+--------+----------+
	
User:
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| Field |  Type   | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| id    | int     | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
	| age   | int     | false  | false    | false    | false   | false         | false     | json:"age,omitempty"  |          0 |
	| name  | string  | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	+------+------+---------+---------+----------+--------+----------+
	| Edge | Type | Inverse | BackRef | Relation | Unique | Optional |
	+------+------+---------+---------+----------+--------+----------+
	| pets | Pet  | false   |         | O2M      | false  | true     |
	+------+------+---------+---------+----------+--------+----------+
```