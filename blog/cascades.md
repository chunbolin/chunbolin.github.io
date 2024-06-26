## 为什么要做Cascades

起初我们的DB主要是通过C API进行增删改查等操作的，因此不涉及非常复杂的计划。但是随着业务的发展，我们逐步引入了一些复杂的查询语言，如SQL、Datalog等。由于复杂查询如果不经过优化直接运行，往往性能较差，这就对查询优化提出了要求。我们期望引入优化器能够为这些查询带来显著的性能提升。

## 为什么选择Cascades

我们也调研了很多其他的优化器实现方式。目前主流的优化器实现方式有如PostgreSQL利用启发式+自底向上的动态规划，也有自顶向下的实现方式，如Volcano和Cascades。

动态规划优化器的主要问题在于：（1）规则维护困难，PG优化器的规则是层层嵌套的并且嵌入在整个计划搜索逻辑里面的，比如说用if判断判断是否满足某个条件，如何去做某个转换，这就意味着如果规则很多，代码逻辑会非常复杂，造成维护困难 （2）自底向上自然意味着无法进行剪枝，自底向上进行计算时，当发现上层是个代价很高的计划时，底层其实已经做了无用的计算了 （3）无法尽快获取一个可行的计划，比如我们要求优化器在一定时间内给出一个可行计划，那么自底向上是没法做的，因为它必须迭代到最上层才能获得一个可行的计划。

Volcano和Cascades都是自顶向下的方式，可以很好地剪枝；并且它们的规则和搜索逻辑是分离的，规则的增加不会影响搜索逻辑，规则的定义相对简单。那么为什么选择Cascades而不是Volcano呢？它们的区别在于，Volcano采用广度优先算法，对一个算子会进行所有的逻辑变换，这点会增大搜索开销；此外Volcano区分了逻辑变换和物理变换，无法对这些规则统一处理。而Cascades则可以做到深度优先的搜索，这就意味这其可以尽快拿到一个可行的计划并用它进行剪枝；Cascade没有区分逻辑变换和物理变换，其搜索逻辑统一更易于开发。

## Cascades做了什么

前面也提到了，Cascades是个自顶向下的优化器框架。先来介绍下基本概念：

1. 查询计划：一个树状结构，由一系列关系代数的算子组成

2. 逻辑算子：表示一个逻辑的关系代数操作，如join，但不指定join的具体类型如hashjoin

3. 物理算子：比如hashjoin、nestloopjoin等能够被具体执行的算子

4. 逻辑计划：由逻辑算子组成的查询计划

5. 物理计划：由物理算子组成的查询计划

6. expr：查询计划的任一子树都是表达式，在cascades这边大部分情况可以认为是一个算子

7. rule：规则，规则可以分为两类：转换规则和实现规则，转换规则是逻辑到逻辑的转换，实现规则是逻辑到物理的转换，规则主要有下面几个成员组成：
   1. pattern，即可以应用规则的表达式的结构
   2. substitute，即应用该规则后生产的新的表达式
   3. promise，表示该规则的优先级，promise越大，规则被应用的优先级就越高
   
8. Group：表示逻辑等价的表达式（表达式的输出记录的集合相同）的集合，例如对于A join B，其group中可能包含A join B，B join A， A hashjoin B，A nestLoopJoin B，B hashjoin A，B nestLoopJoin A等，并记录group中最优的表达式及其代价；

9. mExpr：在表达式的基础上，将其输入抽象为group，这是cascades中很精妙的一点，由于将输入抽象成了group，该group的搜索始终会被记忆在memo中，从而避免重复搜索；

10. memo：memo一般是个哈希表，记录搜索过程中产生的所有group，避免重复搜索
    ![image-20240519211938751](https://github.com/chunbolin/chunbolin.github.io/blob/master/blog/cascades_images/image-20240519211938751.png)

介绍完这些基本概念后，我们以一个具体的例子再来理解下：

![image-20240519213412324](https://github.com/chunbolin/chunbolin.github.io/blob/master/blog/cascades_images/image-20240519213412324.png)![image-20240519213417890](https://github.com/chunbolin/chunbolin.github.io/blob/master/blog/cascades_images/image-20240519213417890.png)

比如对于上面的这个输入给优化器的计划，一开始会为每一个算子都生成一个group，随后自顶向下进行探索：

![image-20240519214745789](https://github.com/chunbolin/chunbolin.github.io/blob/master/blog/cascades_images/image-20240519214745789.png)

几个讲解点，enforcer、剪枝、

## 结果

复杂查询性能提升17%
