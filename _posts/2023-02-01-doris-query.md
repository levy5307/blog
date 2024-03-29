---
layout: post
title: Doris查询原理 
date: 2023-02-01
Author: levy5307
tags: []
comments: true
toc: true
---

同大多数数据库一样，Doris的查询主要分为查询接收、Parse（词法分析和语法分析）、Analyze（语义分析）、Rewrite（查询重写）、逻辑计划生成（单机执行计划）、分布式执行计划生成、Schedule、查询计划执行等步骤。其中查询计划执行由BE负责，其他均由FE负责。

## 查询接收

在Doris中，`AcceptListener`负责监听mysql连接。当一个新连接到来时`AcceptListener`会创建一个`ConnectProcessor`类型对象。`ConnectProcessor`周期性的获取该连接上的请求，针对不同的command执行不同的操作，包括use database操作（`COM_INIT_DB`）、查询操作（`COM_QUERY`）、quit（`COM_QUIT`）等

## Parse

`ConnectProcessor::handleQuery`主要处理查询操作，随后使用[Java CUP Parser](http://www2.cs.tum.edu/projects/cup/)对输入的字符串进行词法和语法分析。词法分析主要将输入的字符串解析成一系列token（例如select、from等），而语法分析则根据词法分析生成的token，生成一颗抽象语法树（Abstract Syntax Tree）。在Doris中抽象语法树用类`StatementBase`表示，`StatementBase`是一个虚类，它有多个不同的子类，例如`SelectStmt`、`InsertStmt`等，分别用于表示查询请求、写入请求等等。

其中，`SelectStmt`类中包含`selectList`、`fromClause`、`groupByClause`、`havingClause`等等，具体如下所示：

```
public class SelectStmt extends QueryStmt {
    protected SelectList selectList;
    private final ArrayList<String> colLabels; // lower case column labels
    protected final FromClause fromClause;
    protected GroupByClause groupByClause;
    private List<Expr> originalExpr;
    private Expr havingClause;  // original having clause
    protected Expr whereClause;
    // havingClause with aliases and agg output resolved
    private Expr havingPred;

    // set if we have any kind of aggregation operation, include SELECT DISTINCT
    private AggregateInfo aggInfo;
    // set if we have analytic function
    private AnalyticInfo analyticInfo;
    // substitutes all exprs in this select block to reference base tables
    // directly
    private ExprSubstitutionMap baseTblSmap = new ExprSubstitutionMap();

    private ValueList valueList;

    // if we have grouping extensions like cube or rollup or grouping sets
    private GroupingInfo groupingInfo;

    // having clause which has been analyzed
    // For example: select k1, sum(k2) a from t group by k1 having a>1;
    // this parameter: sum(t.k2) > 1
    private Expr havingClauseAfterAnaylzed;

    // SQL string of this SelectStmt before inline-view expression substitution.
    // Set in analyze().
    protected String sqlString;

    // Table alias generator used during query rewriting.
    private TableAliasGenerator tableAliasGenerator = null;

    // Members that need to be reset to origin
    private SelectList originSelectList;
}
```

## Analyze

当通过Parse阶段得到AST后，随后会根据`parsedStmt`创建一个`StmtExecutor`。该`StmtExecutor`首先为该查询分配一个16位的随机queryId，并通过`analyzeVariablesInStmt`函数获取select查询中的[Optimizer Hints](https://github.com/apache/doris/pull/4504)。

随后通过`Analyzer`来进行语义分析。注意这里的`Analyzer`类其实并不是语义解析器，正确的名字应该叫`AnalyzerContext`，其保存的是语义解析所需要的各种上下文及状态信息。真正的语义解析是在具体的`StatementBase`子类里的，最终通过多态来获取不同执行语句的不同语义分析实现。

对于`SelectStmt`，语义分析阶段所做的主要工作有：

- 检查cluster name是否为空，若是则抛出`AnalysisException`异常

- 对于含有order by但是order by字段为空的查询，将order by移除掉。

- 对于含有limit的查询：

  - 当offset大于0且不含有order by时报错。

  - 当limit = 0时，设置`hasEmptyResultSet`为true，该变量表示该查询结果一定返回空

  - ...

- 对于含有with的查询，依次analyze该查询中的所有的view（即子查询）

- 对于analyze from从句，依次analyze从句中的所有表（包括BaseTable、[LateralView](https://www.bookstack.cn/read/doris-1.0-zh/737eaa4bfec68762.md)、LocalView）。对于BaseTable，如果该表没有指定database，则指定为默认database。

- 对于select list：

  - 如果是select `*`，则将`*`扩展成所有列

  - 判断select从句中是否包含子查询，如果是则抛出异常

  - 依次解析select list中的所有expr及其child expr（如果有）

  - ...

- 对于where从句：

  - 查看where从句中是否有group操作，如果有则抛出异常

  - 查看where从句中是否有聚合操作，如果有则抛出异常

  - 检查where从句是否返回bool类型或者null，否则抛异常

  - 依次解析where从句中的所有child expr（如果有）

  - ...

## Rewrite

当执行完词法分析后，要根据具体的`ExprRewriteRule`进行查询重写，例如：

- `FoldConstantsRule`，通过对expr求值并进行替换：`1 + 1 + 1` --> `3`，`toupper('abc')` --> `'ABC'`、`cast('2016-11-09' as timestamp)` --> `TIMESTAMP '2016-11-09 00:00:00'`。

- `NormalizeBinaryPredicatesRule`，规范化二进制谓词，使`slot`位于左侧，例如：`5 > id` --> `id < 5`

- `BetweenPredicates`，将`between`谓词转换成`conjunctive`/`disjunctive`谓词，例如： `BETWEEN X AND Y` --> `A >= X AND A <= Y`、 `NOT BETWEEN X AND Y` --> `A < X OR A > Y`

另外对于存在子查询的查询，会依次进行from从句子查询重写、where从句子查询重写（`rewriteWhereClauseSubqueries`）以及have从句中子查询重写（`rewriteHavingClauseSubqueries`）。例如，对于have从句子查询的重写，会先将having子句用where进行等价重写，再将where子句等价重写（例如使用in重写）

当某些重写发生时，需要重新执行语义分析。

## 单节点执行计划

逻辑计划生成和物理计划生成都是由`Planner`来完成的。在`Planner`中，`SingleNodePlanner`通过AST生成单节点执行计划（`PlanNode`构成的算子树）。例如：

`
select A.a, B.b
from A join B
where A.a = B.b
`
生成的算子树如下：

![](../images/single-node-planner.jpg)

在生成单节点执行计划时，主要做了以下优化：

- Slot物化。slot物化是指标记出哪些列在表达式中涉及到，即哪些列需要被读取以及计算。

- 谓词下推。将谓词下推到Scan节点。

- 当没有开启排序落盘时（[enable_spilling](https://cloud.tencent.com/document/product/1387/70771)=true），将sort+limit修改成top n。

- 投影下推。保证在Scan时只读取必须的列。

- Join reorder。对于 Inner Join, Doris 会根据行数调整表的顺序，将大表放在前面。

- 分区，分桶裁剪：根据过滤条件中的信息，确定需要扫描哪些分区，哪些桶的tablet。（在OlapScanNode::init时做分区裁剪，OlapScanNode::finalize做分桶裁剪）

- MaterializedView选择：会根据查询需要的列，过滤，排序和Join的列，行数，列数等因素选择最佳的物化视图。

## 分布式计划生成

在`Planner`中，通过`DistributedPlanner`执行生成分布式执行计划。这里需要根据分布式环境，将单机的PlanNode树拆分成分布式`PlanFragment`树。该步骤的主要目标是最大化并行度和数据本地化。主要方法是将能够并行执行的节点拆分出去单独建立一个`PlanFragment`，用`ExchangeNode`代替被拆分出去的节点，用来接收数据。拆分出去的节点增加一个`DataSinkNode`，用来将计算之后的数据传送到`ExchangeNode`中，做进一步的处理。

![](../images/distribute-planner.jpg)

这一步主要采用自底向上的方法来遍历整个PlanNode树（通过`createPlanFragments`函数递归实现）。如上述例子中：

- 对于`ScanNode`，则直接创建一个对应的`PlanFragment`。上述例子中一共有两个`ScanNode`，则创建两个`PlanFragment`。

- 对于`HashJoinNode`，需要决定Hash Join的分布式执行策略，即Shuffle Join，Broadcast Join，Colocate Join: 

   - 如果使用colocate join，由于join操作都在本地，就不需要拆分。设置`HashJoinNode`与leftFragment共用一个PlanFragment，并删除掉rightFragment。

   - 如果使用bucket shuffle join，需要将右表的数据发送给左表。则和left child组成同一个`PlanFragment`（不单独创建新的，只是和上一条中为left child创建的`PlanFragment`组成同一个），在该`PlanFragment`中，对right child使用`ExchangeNode`来代替。同时对于right child，创建一个`DataSinkNode`，将数据sink到前述`ExchangeNode`。

   - 如果使用broadcast join，需要将右表的数据发送给左表。具体逻辑同上。

   - 如果使用hash partition join，左表和右边的数据都要切分，需要将左右节点都拆分出去，分别创建left ExchangeNode, right ExchangeNode，`HashJoinNode`指定左右节点为left ExchangeNode和right ExchangeNode。需要为`HashJoinNode`单独创建一个`PlanFragment`，指定`RootPlanNode`为这个`HashJoinNode`。最后指定leftFragment, rightFragment的数据发送目的地为left ExchangeNode, right ExchangeNode。

- 对于`SelectNode`，同样创建一个对应的`PlanFragment`

此三个`PlanFragment`则对应上图中的三个虚线框中的Fragment。

## Schedule

这一步是根据分布式逻辑计划，创建分布式物理计划。主要解决以下问题：

- 哪个BE执行哪个`PlanFragment`

- 每个Tablet选择哪个副本去查询

- 如何进行多实例并发

Doris根据`ScanNode`所对应的表，经过分区分桶裁剪之后，可以得到需要访问的Tablet列表。对于Tablet列表，Doris会过滤掉unrecoverable的（该replica随后会被删除）、缺少version的、状态不正常的、以及过滤掉compaction过慢的副本。然后通过Round-Robin的方式在be节点中选择各个tablet的副本，以保证BE之间的负载均衡。

另外，需要处理实例的并发问题。当`leftMostNode`是`ScanNode`时，则需要根据fragment的并发度来设置并发（`parallelExecNum`）。假如需要scan 10个tablet，并行度设置为5的话，那么Scan所在的 PlanFragment，每个BE上可以生成5个执行实例，每个执行实例会分别Scan 2个tablet。当该节点不是`ScanNode`时，则其肯定是`ExchangeNode`，则需要根据`exchangeInstanceParallel`设置并发度。

`FInstanceExecParam`创建完成后，会分配一个唯一的instancdID，方便追踪信息。如果`FragmentExecParams`中包含有`ExchangeNode`，需要计算有多少senders，以便知道需要接受多少个发送方的数据。最后`FragmentExecParams`确定destinations，并把目的地址填充上去。

最后，获取第一个fragment，由执行该fragment的节点作为Receiver，接收查询结果。并将这些fragment发送给BE执行。

## 查询计划执行

查询计划由BE负责执行，其执行引擎采用Batch模式的Volcano模型，相对于Tuple模式的Volcano，执行效率更高。

be中的`FragmentMgr`提供了rpc接口，用于处理这些fragment（`FragmentMgr::exec_plan_fragment`），在接收到请求后，会先通过查询计划生成对应的算子树（`PlanFragmentExecutor::_plan`）并对所有算子执行init操作（`ExecNode::create_tree`）及prepare操作（`_plan->prepare`）。

随后，创建一个线程用户执行查询计划(`FragmentMgr::_exec_actual`)。该线程中，先对算子树执行open操作，然后通过`PlanFragmentExecutor::get_next_internal`驱动整个算子树的执行。该方法自顶向下调用每个算子的`get_next`方法。最终数据会从`ScanNode`节点产生，向上层节点传递，每个节点都会按照自己的逻辑处理RowBatch。`PlanFragmentExecutor`在拿到每个RowBatch后，如果是中间结果，就会将数据传输给其他BE节点，如果是最终结果，就会将数据传输给FE节点（通过`DataSink::send`多态实现）

## 参考文档

https://doris.apache.org/zh-CN/blog/principle-of-Doris-SQL-parsing/#23-%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92

https://blog.bcmeng.com/post/apache-doris-query.html#doris-query-%E6%89%A7%E8%A1%8C

https://www.oomspot.com/post/apachedorischaxunyuanlijiexi

