Interaction
--

未完

- 作用

**Interaction**是一个变换器，它采用向量或双值列，并生成一个单个向量列，其中包含来自每个输入列的一个值的所有组合的乘积的过程。

比如，输入

id1|vec1          |vec2         
 ---|--------------|--------------
 1  |[1.0,2.0,3.0] |[8.0,4.0,5.0]

将返回

id1|vec1          |vec2          |interactedCol                                        
 ---|--------------|--------------|------------------------------------------------------
 1  |[1.0,2.0,3.0] |[8.0,4.0,5.0] |[8.0,4.0,5.0,16.0,8.0,10.0,24.0,12.0,15.0]  
 
 vec1的每一项乘以vec2得到新的值放入interactedCol中
 
- 方法

遍历两个两两相乘


- 主要代码

输入只支持double或者vector数据类型

```scala
  override def transform(dataset: Dataset[_]): DataFrame = {
    transformSchema(dataset.schema, logging = true)
    val inputFeatures = $(inputCols).map(c => dataset.schema(c))
    val featureEncoders = getFeatureEncoders(inputFeatures)
    val featureAttrs = getFeatureAttrs(inputFeatures)

    def interactFunc = udf { row: Row =>
      var indices = ArrayBuilder.make[Int]
      var values = ArrayBuilder.make[Double]
      var size = 1
      indices += 0
      values += 1.0
      var featureIndex = row.length - 1
      while (featureIndex >= 0) {
        val prevIndices = indices.result()
        val prevValues = values.result()
        val prevSize = size
        val currentEncoder = featureEncoders(featureIndex)
        indices = ArrayBuilder.make[Int]
        values = ArrayBuilder.make[Double]
        size *= currentEncoder.outputSize
        currentEncoder.foreachNonzeroOutput(row(featureIndex), (i, a) => {
          var j = 0
          while (j < prevIndices.length) {
            indices += prevIndices(j) + i * prevSize
            values += prevValues(j) * a
            j += 1
          }
        })
        featureIndex -= 1
      }
      Vectors.sparse(size, indices.result(), values.result()).compressed
    }

    val featureCols = inputFeatures.map { f =>
      f.dataType match {
        case DoubleType => dataset(f.name)
        case _: VectorUDT => dataset(f.name)
        case _: NumericType | BooleanType => dataset(f.name).cast(DoubleType)
      }
    }
    dataset.select(
      col("*"),
      interactFunc(struct(featureCols: _*)).as($(outputCol), featureAttrs.toMetadata()))
  }

  /**
   * Creates a feature encoder for each input column, which supports efficient iteration over
   * one-hot encoded feature values. See also the class-level comment of [[FeatureEncoder]].
   *
   * @param features The input feature columns to create encoders for.
   */
  private def getFeatureEncoders(features: Seq[StructField]): Array[FeatureEncoder] = {
    def getNumFeatures(attr: Attribute): Int = {
      attr match {
        case nominal: NominalAttribute =>
          math.max(1, nominal.getNumValues.getOrElse(
            throw new SparkException("Nominal features must have attr numValues defined.")))
        case _ =>
          1  // numeric feature
      }
    }
    features.map { f =>
      val numFeatures = f.dataType match {
        case _: NumericType | BooleanType =>
          Array(getNumFeatures(Attribute.fromStructField(f)))
        case _: VectorUDT =>
          val attrs = AttributeGroup.fromStructField(f).attributes.getOrElse(
            throw new SparkException("Vector attributes must be defined for interaction."))
          attrs.map(getNumFeatures)
      }
      new FeatureEncoder(numFeatures)
    }.toArray
  }

```

