ElementwiseProduct
--

- 作用

**ElementwiseProduct**是将对每一个输入向量乘以一个给定的“权重”向量的过程。


- 方法

公式如下：

\begin{pmatrix} v_1 \\ \vdots\\ v_n \\ \end{pmatrix}
\begin{pmatrix} w_1 \\ \vdots \\ w_n \\ \end{pmatrix} 
\begin{pmatrix} v_1w_1 \\ \vdots \\ v_nw_n \\ \end{pmatrix}

- 参数

scalingVec / Vector: 要相乘的vector


- 主要代码

实际调用的是mllib里的ElementwiseProduct类的transform函数
```scala
  override def transform(vector: Vector): Vector = {
    require(vector.size == scalingVec.size,
      s"vector sizes do not match: Expected ${scalingVec.size} but found ${vector.size}")
    vector match {
    // 遍历每个相同位置的元素相乘
      case dv: DenseVector =>
        val values: Array[Double] = dv.values.clone()
        val dim = scalingVec.size
        var i = 0
        while (i < dim) {
          values(i) *= scalingVec(i)
          i += 1
        }
        Vectors.dense(values)
      case SparseVector(size, indices, vs) =>
        val values = vs.clone()
        val dim = values.length
        var i = 0
        while (i < dim) {
          values(i) *= scalingVec(indices(i))
          i += 1
        }
        Vectors.sparse(size, indices, values)
      case v => throw new IllegalArgumentException("Does not support vector type " + v.getClass)
    }
  }
```





