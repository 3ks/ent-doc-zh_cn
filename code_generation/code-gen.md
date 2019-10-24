---
id: code-gen
title: Introduction
---

## Installation

`ent` 有一个代码生成工具—— `entc`. 可以通过下面的命令安装 `entc`:

```bash
go get github.com/facebookincubator/ent/cmd/entc
``` 

## 初始化一个新的模式

想要生成一个或多个模式模板，可以像下面这样运行 `entc init`:

```bash
entc init User Pet
```

`init` 会在 `ent/schema` 目录下生成两个模式(`user.go` and `pet.go`).
一般地，`ent` 目录位于项目根目录下，如果 `ent` 目录不存在，`entc` 会自动创建该目录。

## 生成代码

在添加好 [字段](../schema/schema-fields.md) 和 [边](../schema/schema-edges.md)，你可以通过下面的命令生成可以使用的代码。

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

在一个项目中，你需要确保 `entc` 和 `ent` 的版本是 **完全相同的**。

One of the options for achieving this is asking `go generate` to use the version
mentioned in the `go.mod` file when running `entc`. If your project does not use
[Go modules](https://github.com/golang/go/wiki/Modules#quick-start), setup one as follows:

```console
go mod init <project>
```

And then, re-run the following command in order to add `ent` to your `go.mod` file:

```console
go get github.com/facebookincubator/ent/cmd/entc
```

Add a `generate.go` file to your project under `<project>/ent`:

```go
package ent

//go:generate go run github.com/facebookincubator/ent/cmd/entc generate ./schema
```

Finally, you can run `go generate ./ent` from the root directory of your project
in order to run `entc` code generation on your project schemas.

## Code Generation Options

For more info about codegen options, run `entc generate -h`:

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

## Storage Options

`entc` can generate assets for both SQL and Gremlin dialect. The default dialect is SQL.

## External Templates

`entc` accepts external Go templates to execute. If the template name is already defined by
`entc`, it will override the existing one. Otherwise, it will write the execution output to
a file with the same name as the template.

Example of a custom template provides a `Node` API for GraphQL - 
[Github](https://github.com/facebookincubator/ent/blob/master/entc/integration/template/ent/template/node.tmpl).

## Use `entc` As A Package

Another option for running `entc` is to use it as a package as follows:

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

The full example exists in [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/entcpkg).


## Schema Description

In order to get a description of your graph schema, run:

```bash
entc describe ./ent/schema
```

An example for the output is as follows:

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