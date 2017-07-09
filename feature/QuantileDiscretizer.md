QuantileDiscretizer
--

- 作用

**QuantileDiscretizer** 分位数离散化，是将输入连续的特征列,输出分箱的类别特征的过程。

- 方法

bin（分级）的数量由numBuckets 参数设置。buckets（区间数）有可能小于这个值，例如,如果输入的不同值太少,就无法创建足够的不同的quantiles（分位数）。

- 参数

numBuckets  / Int: 桶的数量，默认是2。

relativeError / Double: 近似的精度。箱的范围是通过使用近似算法([见approxQuantile](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.DataFrameStatFunctions)
)来得到的。 

handleInvalid / String: 处理无效值的方式，可以是'skip'跳过，'error'抛出异常，或者是'keep'保留异常数据在某个桶里

- 主要代码

计算近似分位数approxQuantile的方法：如果dataframe里有N个元素，以概率probabilities请求分位数，直到错误relativeError，该算法返回一个x，以x的exact接近（probabilities*N）,具体详见[Greenwald-Khanna algorithm]("http://dx.doi.org/10.1145/375663.375670)

```scala
  def approxQuantile(
      col: String,
      probabilities: Array[Double],
      relativeError: Double): Array[Double] = {
    approxQuantile(Array(col), probabilities, relativeError).head
  }
```

```scala
  override def fit(dataset: Dataset[_]): Bucketizer = {
    transformSchema(dataset.schema, logging = true)
    // 调用的近似算法
    val splits = dataset.stat.approxQuantile($(inputCol),
      (0.0 to 1.0 by 1.0/$(numBuckets)).toArray, $(relativeError))
    splits(0) = Double.NegativeInfinity
    splits(splits.length - 1) = Double.PositiveInfinity

    val distinctSplits = splits.distinct
    if (splits.length != distinctSplits.length) {
      log.warn(s"Some quantiles were identical. Bucketing to ${distinctSplits.length - 1}" +
        s" buckets as a result.")
    }
    val bucketizer = new Bucketizer(uid)
      .setSplits(distinctSplits.sorted)
      .setHandleInvalid($(handleInvalid))
    copyValues(bucketizer.setParent(this))
  }
```

