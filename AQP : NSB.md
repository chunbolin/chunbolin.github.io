# Approximate Query Processing：No Silver Bullet 笔记

## 综述

衡量一个AQP系统好坏的4个指标：

- 其支持的查询的数量（the generality of the query language it supports）
- 其误差模型和精确度保障的好坏（it's error model and accuracy guarantee）
- 在运行时所能节约的工作量（the amount of work saved at runtime）
- 假如需要维护样本或索引，则需考虑其所需要的额外资源（the amount of additional resources it requires in pre-computation）

------

**但是一个AQP系统很难在上述4个指标上都表现得很好，必须有所取舍。**因此作者提出了两个较好的方向：

- query-time-sampling：这种方法放弃了对于精度的控制，但是在另外三个指标上有较好的表现
- pre-computation-samples：这种方法主要支持OLAP class of queries，并且需要去预处理和维护样本，但是可以提供一个精度保证并可以显著地减少SQL处理时间



## QUERY-TIME SAMPLE

### Beyond Uniform Sampling

目前大部分数据库都支持query-time sample，但都是以uniform sample的方式实现的，问题在于uniform采样有很多不足。下面对uniform方法的不足进行介绍。

#### groups and predicates

Uniform Sample之所以会在分组和谓词条件下产生误差，是因为它可能会对某些分组采样过少或者完全没有采到（谓词可以理解为其中的一个或多个分组），对此论文中提到了一种采样方法*distinct sampler*

##### distinct sampler

distinct sampler与uniform sampler相同，每一个tuple都有p的概率被采到；但是distinct sampler有一个限制条件，就是对于一个列，获取其所有值的集合，对于其中的每个值，distinct sampler至少会采到f个tuple（given parameters C, f, p, the distinct sampler yields at least f rows( if they exist ) for each distinct value of column set C and all rows are passed with at least probability p）

#### join

用户通常希望可以将采样算子放到Query的最底层（如在join或者selection之前），因为这样可以获得很大的性能提升。但是sample-then-join与join-then-sample并不相同，因此作者提出了一些方法可以将采样放在join之前。

- 在多对一等值连接（many-to-one equijoins）的情况下，join操作只是扩张了fact表的列数（[查看fact和dimension表的说明](https://www.cnblogs.com/qlee/archive/2011/04/09/2010164.html)），因此使用distinct sampler在join之前对表进行抽样就可以了
- 在多对多等值连接（many-to-many equijoins）的情况下，uniform sampling在性能和精确度上的表现都不好，表现为以下几点
  - 当多张表进行join时，只对其中一张表进行抽样的话，只有很小的性能提升
  - 假如要在join结果上获得一个期望的抽样率，那么在每一张表上的抽样率会比结果上的抽样率高很多，因此并不能获得很好的性能提升
  - 如果我们期望在join结果上获得一个uniform的sample，那么每一个tuple的抽样概率会与它和其他表中的tuple的匹配率相关，但是这种有关联的抽样的研究仍不够深入，并不能应用到所有SQL上，如嵌套查询

因此作者提出了一个新的抽样方法，名为*universe sampler*

##### universe sampler

universe sampler同样是获取将要被join的列的所有值的集合C，然后对这个值的集合进行概率为p的抽样，最后得到的输出是包含抽样得到的值的所有tuple（Given parameters C, p, the universe sampler picks uniformly at random a p-fraction of the values of columnset C and outputs all rows that have those values）。其与distinct sampler的区别是distinct sampler会保留列中的所有值，是分层抽样的变种；但是universe sampler不会保留所有值，它是对值进行抽样。其与uniform sampler的区别是这种方法的方差会更大。

如果是用universe sampler，那么显然sample-then-join与join-then-sample的结果是相同的。并且由于其在每一张表上的抽样率和要在join结果上获得一个期望的抽样率是相同的，它会有更好的性能表现。

------

### Query Optimization over Samplers

很显然，我们希望能够将采样深入到执行计划中，因为这样可以带来更多的性能提升。因此，如果有一个抽样语句，我们是否可以自动生成一个执行计划（或者从原始的查询计划转换而来），在提升性能的同时保留精确度？

我们希望可以找到这样一个执行计划的转换规则，对于转换前后的执行计划，满足

- 每个结果中出现的组在两个执行计划中出现的概率是相同的（Each group in the query answer has the same likelihood of appearing in both plans）
- 每个组的聚合函数得到的结果的期望是相同的（ Estimates of aggregates for each group have the same expected value in both plans）
- 聚合函数得到的结果的方差不会比转换之前的差（The plan produced by the transformation rule has a no worse variance of the estimate of aggregate (compared to the plan before the transformation)）

我们称这样的转换规则是*accuracy preserving*的。

接下来介绍一种可行的方法来将采样深入到执行计划中。

- 假如只支持uniform sampler，那么显然将抽样放到selection、projection、foreign-key equijoins之下是可以保持原来的精确度的，如对于foreign-key equijoins，由于存在外键约束，可以很容易得到tuple的匹配关系，因此很容易实现抽样下推（push down）

- 若支持多种抽样方法，那么情况会变得复杂。

  对于如下SQL：

  ```sql
  SELECT COUNT(*) FROM
      (SELECT DISTINCT customer_sk
      FROM store_sales, store_returns
      WHERE ss_customer_sk = sr_customer_sk)
      SAMPLE UNIFORM 10 %
  ```

  如果按照正常的执行顺序，那么抽样会在join之后进行；而直接将uniform sampler应用到join之前的表上会产生很大的误差。因此我们将uniform sampler替换为universe sampler，并将其应用到join之前的表上，这样做与原来的SQL有相同的精确度，因为子查询中有DISTINCT。

在优化过程中，我们需要去选择代价最低的计划，如对于如下SQL：

```sql
SELECT i_color, SUM(ss_sales)
FROM (SELECT ∗ FROM store_sales, item
    WHERE ss_item_sk = i_item_sk)
         SAMPLE DISTINCT {{i_color}, 30, 10%}
```

在这句SQL中，ss_item_sk是外键，并且store_sales表远远大于item表。因此若下推distinct sampler，理论上是可以提高性能的。问题在于我们无法将对i_color的distinct sampler直接下推到store_sales表上，因为store_sales表并没有这个列；但是呢，由于store_sales和item表示多对一关系，因此i_color是直接依赖于ss_item_sk，所以我们可以将distinct sampler下推到ss_item_sk上。但是这样的下推究竟能不能提高性能呢，因为i_color的值集合很大可能是小于ss_item_sk的值集合，如果不下推，那么基本可以认为样本是store_sales的10%；但是假如进行了下推，那么在ss_item_sk的值集合很大的情况下，很有可能会产生远大于store_sales的10%的样本，这样性能反而会有损失。因此是否要将抽样下推是需要仔细考虑的。

从上文可以看到，除了uniform sampler以外的抽样方法提供了更多的执行计划转换规则，使用这些规则可以将抽样下推到执行计划的更深处，从而获得更大的性能提升。

------

### Online Aggregation

在线聚合是指数据库系统可以较快地返回一个查询的结果，但是是以置信区间的形式；它会渐进地处理数据，即不断增加样本的数量，直到满足用户的要求。

但是要将在线聚合集成到现有的数据库系统中是十分困难的，原因这里不再叙述。



## ACCURACY GUARANTEES USING PRE-COMPUTED SAMPLES

我们从原始表里建立样本并将作为一个表存储到数据库中，使用这个表来响应查询，这样显然可以减少响应时间。但是这种方法只能处理一部分SQL（A query in this class is a SQL query with an aggregation on one or more columns, along with a predicate, a group-by operator, and possibly foreign-key joins），不过这种*aggregation queries*是OLAP和智能商业应用中常见且重要的语句；并且目前的预处理方法（pre-computed）无法提供先验的误差保证。

------

### Choosing Samples in Pre-processing

如何选择样本，应该选择多大的样本，这两个问题与精确度和响应时间紧密相关。有可能我们选择的样本在经过谓词筛选或者分组以后，无法提供足够的tuple去估计答案；或者当我们处理SUM、AVG等函数时，如果原始表中的异常值被选入了样本，那显然会对答案的精确度造成很大的影响。目前有两种思路来解决这个问题：

- *workload-independent*：这种方法完全依赖表结构和表的统计数据来建立样本，如AQUA系统
- *workload-aware*：这种方法假设未来的查询负载会与以前的一样或者有相同的分布，如STRAT，它期望建立的样本会在过去的查询负载中会有最小的误差

下面分别举一个例子来解释上述两种思路。

##### small group sampling

这种抽样方法是workload-independent的，它会建立一个全局的uniform样本并且在每一个列上都建立一个有偏的样本（分层抽样）。uniform样本用于估计较大的分组，而有偏样本则用于估计较小的分组，这样可以有效地克服上文中提到的问题。当然这种方法也有缺点，那就是当小的分组由多个列交互形成的时候，这种方法就无法使用了。

##### BlinkDB

BlinkDB可以处理由多个列交互形成的小的分组，假如这些分组在过去的查询负载中很常见。首先它将查询负载抽象为*the set of columns that appear in a query*（称为QCS），对于每个QCS，都创建一个分层抽样的样本。但是为所有的QCS都创建样本的空间成本太高，因此BlinkDB会根据过去的查询负载，生成一个优化策略，来合理地分配空间以获得最小的误差。这种方法固有的缺点是当查询负载发生变化时，重新建立样本的成本是非常高的。

------

### Lack of A Priori Accuracy Guarantees

对于使用预处理样本的AQP系统来说，提供先验误差估计功能是十分重要的。目前有一些误差衡量方法，如置信区间、均方误差、Bootstrap方法等。但是不管使用哪种方法，目前已有的系统只有在处理了查询之后才能提供估计的误差，却不能再处理查询之前提供一个误差的上限保证。比如，对于置信区间方法，它的一个重要的缺点在于它是数据相关的（data-dependent），当一个列A上的数据有很大的标准差或者MAX(A) − MIN(A)非常大，那么对于SUM（A）的置信区间可以是任意大的。也就是说，当样本表建立之后，误差可能会很大也可能会很小，因此AQP系统的效率无法保障。

------

### AQP with Query-Independent Accuracy Guarantee

据我们所知，目前没有使用预处理样本的AQP系统提供先验误差估计功能。当然此类误差估计暂时也无法在所有聚合函数和误差衡量模型上实现，我们会针对SUM函数展开讨论，如对于如下SQL：

```sql
SELECT B, SUM(A)
FROM T
WHERE C = 10
GROUP BY B
```

- Better sampling strategy is needed：如果在表中有很多满足谓词“WHERE C = 10”的tuple，并且这些tuple的分布是不均匀的，那么前面所提到的small group sampling和BlinkDB的方法都可以估计先验误差，但是实际上这个误差是可以任意大的。对于这种情况，我们有一种新的方法。

  ##### sample+seek

  我想大家都会觉得以比异常值更小的概率去选取接近平均值的tuple对最终的结果，即SUM(A)的估计不会产生很大的影响。sample+seek方法是指tuple被抽样的概率与该tuple在列A上的值成比例。

- Help from indices needed：如果在表中只有少量满足谓词“WHERE C = 10”的tuple，那么一个普通的随机样本是无法适用的，这时我们可以利用索引来处理。如果样本中的tuple不足，那么我们可以使用索引找到所有满足条件的tuple并计算精确答案，我们称这样的索引为*Low-Frequency Index*。此外，我们也可以利用索引生成样本，我们利用索引找到满足谓词的tuple的ID（主键），tuple被采样的概率与其在A上的值成比例；若谓词与多个列有关，我们需要计算多个一维索引的交集，但是我们只需要索引交集的前缀，也就是说，建立这个索引时我们要求它的前缀给我提供我们期望的样本中所包含的IDs。只有被包含在这个样本中的tuple，我们才会通过索引去找其在A和B上的值，从而去计算SUM(A)在每个组上的估计值。

  上文中提到的small group sampling可以解决第一张情况，但是它无法在保证精度的情况下，处理只有少量满足谓词的tuple，且谓词在多个列上的情况，如“C = 10 AND D = 20”。BlinkDB可以解决前面提到的情况，但是由于存储空间的限制，它无法保证高效性。

- Accuracy guarantee can be provided： Sample + Seek中的样本及其大小使用索引进行了仔细校准，它们可以无缝地覆盖选择性和非选择性查询。因此我们可以为COUNT和SUM函数来提供一个查询无关的精度保证，即*distribution accuracy*

  ##### distribution accuracy

  我们将精确值和估计值进行归一化，然后计算*distribution error*，distribution error即为归一化后的两个值的欧拉距离。如精确值为SUM(A) = <69, 31>，即上述SQL中B有两个分组；估计值为SUM(A) = <70, 32>，因此其distribution error为$\sqrt{(\frac{69}{100}-\frac{70}{102})^2+(\frac{31}{100}-\frac{32}{102})^2}\approx0.005$。

  我们需要保证，任何支持的查询的distribution error以极大的可能小于预先设定的误差阈值。

我们对这一部分做一个总结，对于一个需要提供先验精度保障的系统，如果用户给定一个误差阈值$\epsilon$，与计算的样本和索引的总的大小和以下因素相关：

- 数据库中数据量的大小
- 支持的查询的数量
- 聚合和谓词中涉及到的列的集合
- 误差阈值$\epsilon$

也就是说，这样的AQP系统保证了对于任何支持的查询，该系统使用预计算的样本和索引，给出的答案的误差以极大的概率小于误差阈值$\epsilon$（any incoming query in the supported class can be answered using pre-computed samples and indices with an error at most  $\epsilon$, with high probability）。

### Comparison with Query-Time Sampling

我们来介绍一下上文提到的两种不同思路的优缺点。

##### pre-compute samples

优点：

- 对比query-time，这种方法可以更加显著降低响应时间
- 如果用户可以接受distribution accuracy方法，那么sample+seek方法可以提供查询无关的严格的先验误差估计

缺点：

- 生成和维护样本和索引的累积代价是非常大的
- 支持的SQL种类较少，只支持聚合语句的其中一部分语句

##### query-time sampling

优点

- 没有预处理或维护样本和索引的开销
- 支持更多的SQL语句
- 更易于在现有的数据库中实现

缺点

- 没有那么好的性能提升除非采样可以被下推到执行计划的底部并可以影响对磁盘的访问
- 无法提供精度保证

## 展望

- Integrate AQP with data platforms：直接将AQP集成到现有的数据库系统中而不是去独立开发一个系统，因为独立开发一个系统的要求过高。
- Approximate execution mode for querying：即我们可以同时进行近似查询和精确查询，一般来说近似查询会更快地结束，并且提供了一个快速但不是很精确的结果，此时用户可以根据这个结果来选择是否继续精确查询。这个与在线聚合有所不同，它只会提供两个答案，而不是像在线聚合那样，渐进地提供越来越精确的结果。
- Experiment with new scenarios：就是尝试在新的场景中使用AQP

