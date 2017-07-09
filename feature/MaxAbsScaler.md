 MaxAbsScaler
--

- 作用

**MaxAbsScaler**是将每个特征调整到[-1,1]的范围,它通过每个特征内的最大绝对值来划分的过程。


- 方法

特征除以该特征中最大的特征值


- 主要代码

计算数据集上的统计数据
```scala
  override def fit(dataset: Dataset[_]): MaxAbsScalerModel = {
    transformSchema(dataset.schema, logging = true)
    val input: RDD[OldVector] = dataset.select($(inputCol)).rdd.map {
      // 把linalg类型转换成mllib类型
      case Row(v: Vector) => OldVectors.fromML(v)
    }
    // 获得统计特征
    val summary = Statistics.colStats(input)
    val minVals = summary.min.toArray
    val maxVals = summary.max.toArray
    val n = minVals.length
    // 生成n的特征序列，存储绝对值最大的值
    val maxAbs = Array.tabulate(n) { i => math.max(math.abs(minVals(i)), math.abs(maxVals(i))) }

    copyValues(new MaxAbsScalerModel(uid, Vectors.dense(maxAbs)).setParent(this))
  }
}
```
用fit函数生成的统计数据，转换原数据到范围[-1,1]
```scala
  override def transform(dataset: Dataset[_]): DataFrame = {
    transformSchema(dataset.schema, logging = true)
    // TODO: this looks hack, we may have to handle sparse and dense vectors separately.
    val maxAbsUnzero = Vectors.dense(maxAbs.toArray.map(x => if (x == 0) 1 else x))
    val reScale = udf { (vector: Vector) =>
    	// asBreeze转换成breeze vector，breeze库里包含较多的数学计算函数
      val brz = vector.asBreeze / maxAbsUnzero.asBreeze
      Vectors.fromBreeze(brz)
    }
    dataset.withColumn($(outputCol), reScale(col($(inputCol))))
  }
```