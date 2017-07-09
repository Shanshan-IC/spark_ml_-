ChiSqSelector
--

- 作用

**ChiSqSelector** 卡方选择器。


- 方法

根据独立的卡方测试对特征进行排序，然后选择排序最高的特征。针对类别特征。

- 参数

numTopFeatures / Int: 选取排名前多少个特征向量，默认值为50

percentile / Double: 和numTopFeatures概念一样，不过是百分比，默认0.1

fpr / Double: 选择p值低于阈值的特征，默认是0.05

fdr / Double: 错误率的上限，默认是0.05。（参考[Benjamini-Hochberg procedure](https://en.wikipedia.org/wiki/False_discovery_rate#Benjamini.E2.80.93Hochberg_procedure)
)

fwe / Double: 多重比较谬误的上限，默认是0.05

selectorType / String: 评判依据，包括"numTopFeatures" (default), "percentile", "fpr", "fdr", "fwe"

- 主要代码
fit函数调用的是mllib里feature的ChiSqSelector类
```scala
  override def fit(dataset: Dataset[_]): ChiSqSelectorModel = {
    transformSchema(dataset.schema, logging = true)
    val input: RDD[OldLabeledPoint] =
      dataset.select(col($(labelCol)).cast(DoubleType), col($(featuresCol))).rdd.map {
        case Row(label: Double, features: Vector) =>
          OldLabeledPoint(label, OldVectors.fromML(features))
      }
    val selector = new feature.ChiSqSelector()
      .setSelectorType($(selectorType))
      .setNumTopFeatures($(numTopFeatures))
      .setPercentile($(percentile))
      .setFpr($(fpr))
      .setFdr($(fdr))
      .setFwe($(fwe))
    val model = selector.fit(input)
    copyValues(new ChiSqSelectorModel(uid, model).setParent(this))
  }
```

mllib里feature的ChiSqSelector类

fit函数

```scala
  def fit(data: RDD[LabeledPoint]): ChiSqSelectorModel = {
    val chiSqTestResult = Statistics.chiSqTest(data).zipWithIndex
    val features = selectorType match {
      case ChiSqSelector.NumTopFeatures =>
      // 判断如何选取top
        chiSqTestResult
          .sortBy { case (res, _) => res.pValue }
          .take(numTopFeatures) // 取前几个
      case ChiSqSelector.Percentile =>
        chiSqTestResult
          .sortBy { case (res, _) => res.pValue }
          .take((chiSqTestResult.length * percentile).toInt) // 取百分值多少
      case ChiSqSelector.FPR =>
        chiSqTestResult
          .filter { case (res, _) => res.pValue < fpr } // 取pValue小于fpr的
      case ChiSqSelector.FDR =>
        // This uses the Benjamini-Hochberg procedure.
        // https://en.wikipedia.org/wiki/False_discovery_rate#Benjamini.E2.80.93Hochberg_procedure
        val tempRes = chiSqTestResult
          .sortBy { case (res, _) => res.pValue }
        val maxIndex = tempRes
          .zipWithIndex
          .filter { case ((res, _), index) =>
            res.pValue <= fdr * (index + 1) / chiSqTestResult.length }
          .map { case (_, index) => index }
          .max
        tempRes.take(maxIndex + 1)
      case ChiSqSelector.FWE =>
        chiSqTestResult
          .filter { case (res, _) => res.pValue < fwe / chiSqTestResult.length } // 取pValue小于fwe/length
      case errorType =>
        throw new IllegalStateException(s"Unknown ChiSqSelector Type: $errorType")
    }
    val indices = features.map { case (_, index) => index }
    new ChiSqSelectorModel(indices)
  }
```  

chiSqTestResult调用的stat里的**ChiSqTest**

如果输入是Vector，则根据预期分布进行Pearson的观察数据的拟合优度拟合测试，或者按照预期的频率为1 / len（观察）再次进行均匀分布（默认情况下）。

如果输入是矩阵，则执行Pearson独立性测试，该矩阵不能包含总和为0的列或行。

如果输入是是LabeledPoint的RDD，则针对输入RDD上的标签对每个特征进行Pearson独立性测试。 对于每个特征，（特征，标签）对被转换为计算卡方统计量的偶然矩阵。 所有标签和特征值必须是分类的。