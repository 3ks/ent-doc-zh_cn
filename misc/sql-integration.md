---
id: sql-integration
title: sql.DB Integration
---

下面的示例将说明如何将自定义 `sql.DB`对象传递给 `ent.Client`.

## 配置 `sql.DB`

方法一：

```go
package main

import (
    "time"

    "<your_project>/ent"
    "github.com/facebookincubator/ent/dialect/sql"
)

func Open() (*ent.Client, error) {
    drv, err := sql.Open("mysql", "<mysql-dsn>")
    if err != nil {
    	return nil, err
    }
    // 获取驱动的底层 sql.DB 对象。
    db := drv.DB()
    db.SetMaxIdleConns(10)
    db.SetMaxOpenConns(100)
    db.SetConnMaxLifetime(time.Hour)
    return ent.NewClient(ent.Driver(drv)), nil
}
```

方法二：

```go
package main

import (
    "database/sql"
    "time"

    "<your_project>/ent"
    entsql "github.com/facebookincubator/ent/dialect/sql"
)

func Open() (*ent.Client, error) {
    db, err := sql.Open("mysql", "<mysql-dsn>")
    if err != nil {
    	return nil, err
    }
    db.SetMaxIdleConns(10)
    db.SetMaxOpenConns(100)
    db.SetConnMaxLifetime(time.Hour)
    // Create an ent.Driver from `db`.
    // 从 `db` 创建一个 ent.Driver.
    drv := entsql.OpenDB("mysql", db)
    return ent.NewClient(ent.Driver(drv)), nil
}
```

## 在 MySQL 中使用 Opencensus

> 译者注：OpenCensus 是 Google 开源的一个用来收集和追踪应用程序指标的第三方库。目前 Opencensus 已经与 OpenTracing [合并](https://www.cncf.io/blog/2019/05/21/a-brief-history-of-opentelemetry-so-far/)

```go
package main

import (
	"context"
	"database/sql"
	"database/sql/driver"

	"<project>/ent"
	
	"contrib.go.opencensus.io/integrations/ocsql"
	"github.com/go-sql-driver/mysql"
	entsql "github.com/facebookincubator/ent/dialect/sql"
)

type connector struct {
	dsn string
}

func (c connector) Connect(context.Context) (driver.Conn, error) {
	return c.Driver().Open(c.dsn)
}

func (connector) Driver() driver.Driver {
	return ocsql.Wrap(
		mysql.MySQLDriver{},
		ocsql.WithAllTraceOptions(),
		ocsql.WithRowsClose(false),
		ocsql.WithRowsNext(false),
		ocsql.WithDisableErrSkip(true),
	)
}

// 打开新的连接并启动统计记录器。
func Open(dsn string) *ent.Client {
	db := sql.OpenDB(connector{dsn})
	// 从 `db` 创建一个 ent.Driver.
    drv := entsql.OpenDB("mysql", db)
    return ent.NewClient(ent.Driver(drv))
}
```
