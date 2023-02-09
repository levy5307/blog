---
layout: post
title: Doris查询原理 
date: 2023-02-01
Author: levy5307
tags: []
comments: true
toc: true
---

同大多数数据库一样，Doris的查询主要分为查询接收、Parse（词法分析和语法分析）、Analyze（语义分析）、Rewrite（查询重写）、逻辑计划生成（单机执行计划）、物理计划生成（分布式执行计划）、Schedule、查询计划执行、查询结果返回等步骤。其中查询计划执行由BE负责，其他均由FE负责。

## 查询接收

在Doris中，`AcceptListener`负责监听mysql连接。当一个新连接到来时`AcceptListener`会创建一个`ConnectProcessor`类型对象。`ConnectProcessor`周期性的获取该连接上的请求，针对不同的command执行不同的操作，包括use database操作（`COM_INIT_DB`）、查询操作（`COM_QUERY`）、quit（`COM_QUIT`）等

## Parse

`ConnectProcessor::handleQuery`主要处理查询操作，随后使用[Java CUP Parser](http://www2.cs.tum.edu/projects/cup/)对输入的字符串进行词法和语法分析。词法分析主要将输入的字符串解析成一系列token（例如select、from等），而语法分析则根据词法分析生成的token，生成一颗抽象语法树（Abstract Syntax Tree），在Doris中用类`StatementBase`表示。`StatementBase`是一个虚类，它有多个不同的子类，例如`SelectStmt`、`InsertStmt`等，分别用于表示查询请求、写入请求等等。

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

当通过Parse阶段得到AST后，随后会根据`parsedStmt`创建一个`StmtExecutor`。该`StmtExecutor`首先为该查询分配一个queryId，并通过`analyzeVariablesInStmt`函数获取select查询中的[Optimizer Hints](https://github.com/apache/doris/pull/4504)。随后便通过`Analyzer`来进行语义分析。注意这里的`Analyzer`类其实并不是语义解析器，正确的名字应该叫AnalyzerContext，其保存的是语义解析所需要的各种上下文及状态信息。真正的语义解析是在具体的StatementBase子类里的，最终通过多态来获取不同执行语句的不同语义分析实现。

对于SelectStmt，语义分析阶段所做的主要工作有：

- 检查cluster name是否为空，若是则抛出AnalysisException异常

- 对于含有limit的查询：

  - 当offset大于0且不含有order by时报错。

  - 当limit = 0时，设置`hasEmptyResultSet`为true，该变量表示该查询结果一定返回空

- 对于含有with的查询，依次analyze该查询中的所有的view

- 对于analyze from从句：

  - 检查从句中的所有表，如果该表没有指定database，则指定为默认database。

  - Analyze the join clause，该操作只有left表analyze之后才会执行

  - analyze sort hint

  - analyze hint，当前仅支持PREAGGOPEN

- 对于select list：

  - 如果是select *，则将*扩展成所有列

  -

## Rewrite

## 逻辑计划生成

## 物理计划生成

## Schedule

## 查询计划执行

查询计划是由BE负责执行的，其执行引擎采用Batch模式的Volcano模型，相对于Tuple模式的Volcano，执行效率更高。

## 查询结果返回
