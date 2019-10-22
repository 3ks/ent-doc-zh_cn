# 译者的废话

本文是 facebook 开源库 `ent` 文档的中文翻译，由于译者水平有限，错误的地方，欢迎各位批评指出。

官方仓库地址：[ent](https://github.com/facebookincubator/ent)

官方文档地址：[文档](https://entgo.io/docs/getting-started/)

译文仓库地址为：[ent-doc-zh_cn](https://github.com/gorda/ent-doc-zh_cn)

译文 gitbook 在线阅读地址为：[ent-zh_cn](https://gordaaa.gitbook.io/ent-zh_cn/)


# 一个奇葩 ORM
 
`ent` 的目标是开发一个图数据库 ORM ，最吸引译者的是其 `代码生成` 功能，[最初](https://entgo.io/blog/2019/10/03/introducing-ent/) 支持 `Gremlin` (AWS Neptune，一种图数据库) ，后面开始支持 MySql 等关系型数据库，后续还会支持 `PostgreSql`，详细路线图可以查看官方的 [RoadMap](https://github.com/facebookincubator/ent/issues/46).

但是现在作者建议使用 `Mysql` 等关系型数据库作为存储实现，而不是 Gremlin。对此，作者的 [解释](https://github.com/facebookincubator/ent/issues/81) 是，他们发现在 Mysql 下可以获得更好的性能（只比较了 AWS Neptune，没有比较 neo4j, janus 等），支持事务，人们更熟悉 SQL 等。(图数据库的深度遍历比关系型数据库慢是很不可思议的的， 鉴于 facebook 已经将 ent 用于生产环境，这里不清楚是 AWS Neptune 的性能太差，还是 ent 存在什么黑魔法)

虽然现在 `ent` 将关系统数据库作为主要存储实现，但是代码和文档中使用了很多图数据库的 `概念名词` ，对此作者的 [解释](https://github.com/facebookincubator/ent/issues/81#issuecomment-540634087) 是，这是有意为之的，他们希望用户只需要将 ent 当作一个图数据库的 ORM 使用，通过 ORM 完成数据交互，不关心底层实现，所以：

> `ent` 是一个基于关系型数据库的图数据库 ORM.

# 一个菜鸡译者

上面提到的本人水平有限，可能存在错误的地方，这不是一句客套话。我都没有使用过图数据库，也不理解其很多概念，感觉 `ent` 的文档也有点混乱，Schema,Entity 等与传统数据库所表示的含义也不同。`ent` 最吸引我的地方是其代码生成功能，所以我决定学习它并尝试翻译文档。