SQLTransformer
--

- 作用

**SQLTransformer**是通过SQL语句来对数据进行转换的过程。

- 例子

比如，输入是

```
id  |  v1 |  v2
----|-----|-----
 0  | 1.0 | 3.0
 2  | 2.0 | 5.0
```

命令是

```scala
SELECT *, (v1 + v2) AS v3, (v1 * v2) AS v4 FROM __THIS__
```

返回

```scala
id |  v1 |  v2 |  v3 |  v4
----|-----|-----|-----|-----
 0  | 1.0 | 3.0 | 4.0 | 3.0
 2  | 2.0 | 5.0 | 7.0 |10.0
```

- 参数

statement  / String: SQL语句。


- 主要代码

```scala
  override def transform(dataset: Dataset[_]): DataFrame = {
    transformSchema(dataset.schema, logging = true)
    val tableName = Identifiable.randomUID(uid)
    dataset.createOrReplaceTempView(tableName)
    val realStatement = $(statement).replace(tableIdentifier, tableName)
    val result = dataset.sparkSession.sql(realStatement)
    dataset.sparkSession.catalog.dropTempView(tableName)
    result
  }
```