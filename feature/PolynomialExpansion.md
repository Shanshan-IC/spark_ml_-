PolynomialExpansion
--

- 作用

**PolynomialExpansion**是一个将特征展开到多元空间的过程。


- 方法

它通过n-degree结合原始的维度来定义。比如设置degree为2就可以将(x, y)转化为(x, x x, y, x y, y y)。通过递归来实现的。

- 参数

degree  / Int: 扩展的维度，默认是2。


- 主要代码

分成dense和sparse两种vector进行expand实现。

下面代码是For dense vector，对于sparse vector,唯一的不同在于vector的index。

```scala
  private def expandDense(
      values: Array[Double],
      lastIdx: Int,
      degree: Int,
      multiplier: Double,
      polyValues: Array[Double],
      curPolyIdx: Int): Int = {
    if (multiplier == 0.0) {
      // do nothing
    } else if (degree == 0 || lastIdx < 0) {
      if (curPolyIdx >= 0) { // skip the very first 1
        polyValues(curPolyIdx) = multiplier
      }
    } else {
      val v = values(lastIdx)
      val lastIdx1 = lastIdx - 1
      var alpha = multiplier
      var i = 0
      var curStart = curPolyIdx
      while (i <= degree && alpha != 0.0) {
      // 递归更新multiplier 和curPolyIdx
        curStart = expandDense(values, lastIdx1, degree - i, alpha, polyValues, curStart)
        i += 1
        alpha *= v
      }
    }
    curPolyIdx + getPolySize(lastIdx + 1, degree)
  }
```