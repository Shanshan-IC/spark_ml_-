Binarizer
--

- 作用

**Binarizer**是将数值特征转换为二值特征的过程。


- 方法

值大于阈值的特征二值化为1,否则二值化为0。

- 参数

threshold / double: 阈值


- 主要代码

仅支持double和vector数据类型
```scala
override def transform(dataset: Dataset[_]): DataFrame = {
	val outputSchema = transformSchema(dataset.schema, logging = true)
	val schema = dataset.schema
	val inputType = schema($(inputCol)).dataType
	val td = $(threshold)
	// 对double类型，判断是否大于阈值，是设为1，否则为0
	val binarizerDouble = udf { in: Double => if (in > td) 1.0 else 0.0 }
	// 对于vector类型，返回列表（大于阈值的存储index和1）
	val binarizerVector = udf { (data: Vector) =>
	  val indices = ArrayBuilder.make[Int]
	  val values = ArrayBuilder.make[Double]
	
	  data.foreachActive { (index, value) =>
	    if (value > td) {
	      indices += index
	      values +=  1.0
	    }
	  }

  Vectors.sparse(data.size, indices.result(), values.result()).compressed
}

val metadata = outputSchema($(outputCol)).metadata

inputType match {
  case DoubleType =>
    dataset.select(col("*"), binarizerDouble(col($(inputCol))).as($(outputCol), metadata))
  case _: VectorUDT =>
    dataset.select(col("*"), binarizerVector(col($(inputCol))).as($(outputCol), metadata))
}
}
```





