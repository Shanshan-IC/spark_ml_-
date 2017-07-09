VectorIndexer
--

- 作用

**VectorIndexer**将标签列的字符串编码为标签索引。索引类别特征允许诸如决策树和集合树等算法适当处理可分类化的特征,提高效率。

- 方法

1 对vector的输入列和参数maxCategories

2 根据有区别的值的数量,判断哪些特征可以类别化。拥有的不同值的数量至少要为maxCategories的特征才能判断可以类别化。

3 对每一个可以类别化的特征计算基于0的类别索引。

4 为类别特征建立索引,将原有的特征值转换为索引。

- 参数

maxCategories  / Int: 可分类的最大数量，默认是20

1. 'frequencyDesc': 按照出现频率的降序

2. 'frequencyAsc': 按照出现频率的升序  
3. 'alphabetDesc': 按照字母的降序  
4. 'alphabetAsc': 按照字母的升序 

handleInvalid / String: 处理无效值的方式，可以是'skip'跳过，'error'抛出异常，或者是'keep'保留异常数据在某个桶里

- 主要代码

fit函数返回排序
```scala
  override def fit(dataset: Dataset[_]): StringIndexerModel = {
    transformSchema(dataset.schema, logging = true)
    val values = dataset.na.drop(Array($(inputCol)))
      .select(col($(inputCol)).cast(StringType))
      .rdd.map(_.getString(0))
    val labels = $(stringOrderType) match {
    // 根据stringOrderType进行排序并返回，且不重复
      case StringIndexer.frequencyDesc => values.countByValue().toSeq.sortBy(-_._2)
        .map(_._1).toArray
      case StringIndexer.frequencyAsc => values.countByValue().toSeq.sortBy(_._2)
        .map(_._1).toArray
      case StringIndexer.alphabetDesc => values.distinct.collect.sortWith(_ > _)
      case StringIndexer.alphabetAsc => values.distinct.collect.sortWith(_ < _)
    }
    copyValues(new StringIndexerModel(uid, labels).setParent(this))
  }
```

生成该列的所有值，并按顺序放在map里

```scala
// label存储着所有列的值
  private val labelToIndex: OpenHashMap[String, Double] = {
    val n = labels.length
    val map = new OpenHashMap[String, Double](n)
    var i = 0
    while (i < n) {
      map.update(labels(i), i)
      i += 1
    }
    map
  }
```


```scala
  override def transform(dataset: Dataset[_]): DataFrame = {
    if (!dataset.schema.fieldNames.contains($(inputCol))) {
      logInfo(s"Input column ${$(inputCol)} does not exist during transformation. " +
        "Skip StringIndexerModel.")
      return dataset.toDF
    }
    transformSchema(dataset.schema, logging = true)

    val filteredLabels = getHandleInvalid match {
      case StringIndexer.KEEP_INVALID => labels :+ "__unknown"
      case _ => labels
    }

    val metadata = NominalAttribute.defaultAttr
      .withName($(outputCol)).withValues(filteredLabels).toMetadata()
    // 如果选择跳过空值，则先过滤掉这部分数据
    val (filteredDataset, keepInvalid) = getHandleInvalid match {
      case StringIndexer.SKIP_INVALID =>
        val filterer = udf { label: String =>
          labelToIndex.contains(label)
        }
        (dataset.na.drop(Array($(inputCol))).where(filterer(dataset($(inputCol)))), false)
      case _ => (dataset, getHandleInvalid == StringIndexer.KEEP_INVALID)
    }

    val indexer = udf { label: String =>
      if (label == null) {
      // 如果全是空值，若保持空值，则全部返回0，否则抛出异常
        if (keepInvalid) {
          labels.length
        } else {
          throw new SparkException("StringIndexer encountered NULL value. To handle or skip " +
            "NULLS, try setting StringIndexer.handleInvalid.")
        }
      } else {
      // 将值放进前面生成的map里查找index，找不到的话处理和上一步一样
        if (labelToIndex.contains(label)) {
          labelToIndex(label)
        } else if (keepInvalid) {
          labels.length
        } else {
          throw new SparkException(s"Unseen label: $label.  To handle unseen labels, " +
            s"set Param handleInvalid to ${StringIndexer.KEEP_INVALID}.")
        }
      }
    }

    filteredDataset.select(col("*"),
      indexer(dataset($(inputCol)).cast(StringType)).as($(outputCol), metadata))
  }
```

