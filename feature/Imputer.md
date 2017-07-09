Imputer
--

- 作用

**Imputer**对空值的填充，包括使用均值和中位数两种方式。

- 参数

strategy / String: 填充的方式，包括'mean'和'median'

missingValue / Double: 填充的数字

- 主要代码


```scala
  override def fit(dataset: Dataset[_]): ImputerModel = {
    transformSchema(dataset.schema, logging = true)
    val spark = dataset.sparkSession
    import spark.implicits._
    val surrogates = $(inputCols).map { inputCol =>
      val ic = col(inputCol)
      // 过滤出该列的非空数据
      val filtered = dataset.select(ic.cast(DoubleType))
        .filter(ic.isNotNull && ic =!= $(missingValue) && !ic.isNaN)
      if(filtered.take(1).length == 0) {
        throw new SparkException(s"surrogate cannot be computed. " +
          s"All the values in $inputCol are Null, Nan or missingValue(${$(missingValue)})")
      }
      // 计算非空数据的均值和中位数
      val surrogate = $(strategy) match {
        case Imputer.mean => filtered.select(avg(inputCol)).as[Double].first()
        case Imputer.median => filtered.stat.approxQuantile(inputCol, Array(0.5), 0.001).head
      }
      surrogate
    }

    val rows = spark.sparkContext.parallelize(Seq(Row.fromSeq(surrogates)))
    val schema = StructType($(inputCols).map(col => StructField(col, DoubleType, nullable = false)))
    val surrogateDF = spark.createDataFrame(rows, schema)
    copyValues(new ImputerModel(uid, surrogateDF).setParent(this))
  }
```





