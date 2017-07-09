Bucketizer
--

- 作用

**Bucketizer**是将连续性特征进行分桶的过程。


- 方法

根据用户给定的输入splits参数，按坐落在哪个范围进行分桶。

- 参数

splits / double array: 指定每个桶的范围，必须是严格递增的。


- 主要代码

仅支持double数据类型

判断有效，splits要严格递增，且长度大于2，且没有空
```scala
  private[feature] def checkSplits(splits: Array[Double]): Boolean = {
    if (splits.length < 3) {
      false
    } else {
      var i = 0
      val n = splits.length - 1
      while (i < n) {
        if (splits(i) >= splits(i + 1) || splits(i).isNaN) return false
        i += 1
      }
      !splits(n).isNaN
    }
  }
```

binary search位置

keepInvalid: Nan flag, true设置额外的桶给NaN，设置为false，当遇到NaN，报错。
```scala
  private[feature] def binarySearchForBuckets(
      splits: Array[Double],
      feature: Double,
      keepInvalid: Boolean): Double = {
    if (feature.isNaN) {
      if (keepInvalid) {
        splits.length - 1
      } else {
        throw new SparkException("Bucketizer encountered NaN value. To handle or skip NaNs," +
          " try setting Bucketizer.handleInvalid.")
      }
    } else if (feature == splits.last) {
      splits.length - 2
    } else {
      val idx = ju.Arrays.binarySearch(splits, feature) // binary search
      if (idx >= 0) {
        idx
      } else {
        val insertPos = -idx - 1
        // 判断是否有范围
        if (insertPos == 0 || insertPos == splits.length) {
          throw new SparkException(s"Feature value $feature out of Bucketizer bounds" +
            s" [${splits.head}, ${splits.last}].  Check your features, or loosen " +
            s"the lower/upper bound constraints.")
        } else {
          insertPos - 1
        }
      }
    }
```

需要注意的是，负无穷和正无穷必须明确的提供用来覆盖所有的双精度值,否则,超出splits的值将会被 认为是一个错误。如果你并不知道目标列的上界和下界,你应该添加Double.NegativeInfinity和Double.PositiveInfinity作为边界从而防止潜在的 超过边界的异常。下面是程序调用的例子。

