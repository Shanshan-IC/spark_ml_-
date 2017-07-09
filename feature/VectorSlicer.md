VectorSlicer
--

- 作用

**VectorSlicer**对特征向量输出一个新的特征向量与原始特征的子阵列的过程，当我们只需要一部分特征可实现。

- 参数

indices / Int: 指定向量的整数索引

names / String Array: 需要转换的列名向量

- 主要代码

```scala
  override def transform(dataset: Dataset[_]): DataFrame = {
    // Validity checks
    transformSchema(dataset.schema)
    val inputAttr = AttributeGroup.fromStructField(dataset.schema($(inputCol)))
    inputAttr.numAttributes.foreach { numFeatures =>
      val maxIndex = $(indices).max
      require(maxIndex < numFeatures,
        s"Selected feature index $maxIndex invalid for only $numFeatures input features.")
    }

    // Prepare output attributes
    val inds = getSelectedFeatureIndices(dataset.schema)
    val selectedAttrs: Option[Array[Attribute]] = inputAttr.attributes.map { attrs =>
      inds.map(index => attrs(index))
    }
    val outputAttr = selectedAttrs match {
      case Some(attrs) => new AttributeGroup($(outputCol), attrs)
      case None => new AttributeGroup($(outputCol), inds.length)
    }

    // Select features
    val slicer = udf { vec: Vector =>
      vec match {
        case features: DenseVector => Vectors.dense(inds.map(features.apply))
        case features: SparseVector => features.slice(inds)
      }
    }
    dataset.withColumn($(outputCol), slicer(dataset($(inputCol))), outputAttr.toMetadata())
  }
```

获得名字names对应的索引indices
```scala
  private def getSelectedFeatureIndices(schema: StructType): Array[Int] = {
    val nameFeatures = MetadataUtils.getFeatureIndicesFromNames(schema($(inputCol)), $(names))
    val indFeatures = $(indices)
    val numDistinctFeatures = (nameFeatures ++ indFeatures).distinct.length
    lazy val errMsg = "VectorSlicer requires indices and names to be disjoint" +
      s" sets of features, but they overlap." +
      s" indices: ${indFeatures.mkString("[", ",", "]")}." +
      s" names: " +
      nameFeatures.zip($(names)).map { case (i, n) => s"$i:$n" }.mkString("[", ",", "]")
    require(nameFeatures.length + indFeatures.length == numDistinctFeatures, errMsg)
    indFeatures ++ nameFeatures
  }
```





