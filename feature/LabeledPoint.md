LabeledPoint
--

- 作用

**LabeledPoint**是ml里的数据结构，主要包括label(double)和feature(Vector)。

- 使用

对于二元分类，它的标签数据为0和1，而对于多类分类，它的标签数据为0，1，2，…。它同本地向量一样，同时具有Sparse和Dense两种实现方式