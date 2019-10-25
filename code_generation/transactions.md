---
id: transactions
title: Transactions
---

## 开始事务

```go
// GenTx 在一个事务中创建多个群组的实体及其用户
func GenTx(ctx context.Context, client *ent.Client) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return fmt.Errorf("starting a transaction: %v", err)
	}
	hub, err := tx.Group.
		Create().
		SetName("Github").
		Save(ctx)
	if err != nil {
		return rollback(tx, fmt.Errorf("failed creating the group: %v", err))
	}
	// 为群组添加一个用户
	dan, err := tx.User.
		Create().
		SetAge(29).
		SetName("Dan").
		AddManage(hub).
		Save(ctx)
	if err != nil {
		return rollback(tx, err)
	}
	// 创建一个用户 "Ariel".
	a8m, err := tx.User.
		Create().
		SetAge(30).
		SetName("Ariel").
		AddGroups(hub).
		AddFriends(dan).
		Save(ctx)
	if err != nil {
		return rollback(tx, err)
	}
	fmt.Println(a8m)
	// Output:
	// User(id=2, age=30, name=Ariel)
	
	// 提交事务
	return tx.Commit()
}

// rollback 会调用 tx.Rollback，如果此时发生错误，rollback 会将该错误也包装进去。
func rollback(tx *ent.Tx, err error) error {
	if rerr := tx.Rollback(); rerr != nil {
		err = fmt.Errorf("%v: %v", err, rerr)
	}
	return err
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/traversal).

## 事务客户端

Transactional Client，事务客户端。
有时候，你的现有代码已经使用了 `*ent.Client`，但是你想将他修改（或包装）为使用事务客户端实现。
对于这种情况：你可以从现有的事务客户端获取一个 `*ent.Client`： 

```go
// WrapGen 函数将现有的 "Gen" 函数包装成事务
func WrapGen(ctx context.Context, client *ent.Client) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return err
	}
	txClient := tx.Client()
	//下面会调用 "Gen"，但是传输一个事务客户端给它；这样就不用修改 "Gen" 函数的代码。
	if err := Gen(ctx, txClient); err != nil {
		return rollback(tx, err)
	}
	return tx.Commit()
}

// Gen 函数用于创建一个群组实体。
func Gen(ctx context.Context, client *ent.Client) error {
	// ...
	return nil
}
```

完整的例子请查看 [GitHub](https://github.com/facebookincubator/ent/tree/master/examples/traversal).

## 最佳实践

在事务中运行回调函数：

```go
func WithTx(ctx context.Context, client *ent.Client, fn func(tx *ent.Tx) error) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return err
	}
	defer func() {
		if v := recover(); v != nil {
			tx.Rollback()
			panic(v)
		}
	}()
	if err := fn(tx); err != nil {
		if rerr := tx.Rollback(); rerr != nil {
			err = errors.Wrapf(err, "rolling back transaction: %v", rerr)
		}
		return err
	}
	if err := tx.Commit(); err != nil {
		return errors.Wrapf(err, "committing transaction: %v", err)
	}
	return nil
}
```

用法：

```go
func Do(ctx context.Context, client *ent.Client) {
	// WithTx helper.
	if err := WithTx(ctx, client, func(tx *ent.Tx) error {
		return Gen(ctx, tx.Client())
	}); err != nil {
		log.Fatal(err)
	}
}
```
