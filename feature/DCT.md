DCT
--

- 作用

**DCT** (Discrete Cosine Transform)是将一个在时间域(time domain)内长度为N的实值序列转换为另外一个 在频率域(frequency domain)内的长度为N的实值序列的过程。


- 方法

值大于阈值的特征二值化为1,否则二值化为0。

- 参数

inverse / boolean: 向前(false)或向后DCT(true)


- 主要代码


```scala
  override protected def createTransformFunc: Vector => Vector = { vec =>
    val result = vec.toArray
    // 1D Discrete Cosine Transform (DCT) of double precision data
    val jTransformer = new DoubleDCT_1D(result.length)
    if ($(inverse)) jTransformer.inverse(result, true) else jTransformer.forward(result, true)
    Vectors.dense(result)
  }
```





