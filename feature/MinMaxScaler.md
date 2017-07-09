MinMaxScaler
--

- 作用

**MinMaxScaler**是将每个特征,将每个特征调整到一个特定的范围(通常是[0,1])。


- 方法

对于单个特征值，
$$ Rescaled(e_i) = \frac{e_i - E_{min}}{E_{max} - E_{min}} * (max - min) + min $$

如果$$E_{max} == E_{min}$$则

$$Rescaled(e_i) = 0.5 * (max + min)$$

仅支持double类型

- 参数

min: 默认是0。转换的下界,被所有的特征共享。

max: 默认是1。转换的上界,被所有的特征共享。

- 主要代码

计算数据集上的统计数据
```scala
  override def fit(dataset: Dataset[_]): MinMaxScalerModel = {
    transformSchema(dataset.schema, logging = true)
    val input: RDD[OldVector] = dataset.select($(inputCol)).rdd.map {
      case Row(v: Vector) => OldVectors.fromML(v)
    }
    val summary = Statistics.colStats(input)
    copyValues(new MinMaxScalerModel(uid, summary.min, summary.max).setParent(this))
  }
```
用fit函数生成的统计数据，转换原数据到范围[0,1]
```scala
  override def transform(dataset: Dataset[_]): DataFrame = {
    transformSchema(dataset.schema, logging = true)
    // vector to array
    val originalRange = (originalMax.asBreeze - originalMin.asBreeze).toArray
    val minArray = originalMin.toArray

    val reScale = udf { (vector: Vector) =>
      val scale = $(max) - $(min)

      // 0 in sparse vector will probably be rescaled to non-zero
      val values = vector.toArray
      val size = values.length
      var i = 0
      // 根据上面的公式
      while (i < size) {
        if (!values(i).isNaN) {
          val raw = if (originalRange(i) != 0) (values(i) - minArray(i)) / originalRange(i) else 0.5
          values(i) = raw * scale + $(min)
        }
        i += 1
      }
      Vectors.dense(values)
    }

    dataset.withColumn($(outputCol), reScale(col($(inputCol))))
  }
```