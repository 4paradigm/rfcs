- Start Date:  2021-04-19
- Target Major Version: (0.2.0)
- Reference Issues:  https://github.com/4paradigm/HybridSE/issues/51
- Implementation PR:

# Using ZetaSQL as front-end

# Summary

We are going to use ZetaSQL as our new SQL front-end. The document is trying to figure out:

- **Motivation**: Why we want to refactor the front-end
- **Feasibility**: Whether ZetaSQL is a good option or not
- **Design**: How to integrate ZetaSQL into HybridSE
- **Adoption strategy**: What breaking changes it will bring to FEDB and SparkFE



# Motivation

HybridSE is LLVM-based SQL engine written in C++. It aims at supporting statndare ANSI SQL and some new SQL traits at the same time. However, as a newly engine, HybridSE only supports a limited set of SQL syntaxs currently and its syntax error message is unfriendly. For example, the syntax error message can't tell the location of wrong expression.

ZetaSQL is a SQL Analyzer Framework from Google. We are considering using it as HybridSE's SQL parser and analyzer for following reasons:

- ZetaSQL has been used for BigQuery, Spanner and DataflowSQL by Google
- ZetaSQL is a C++ SQL parser, which can be easily integrate into HybridSE
- ZetaSQL supports standare ANSI SQL and provides clearly error messge. 



# Design

> ZetaSQL defines a language (grammar, types, data model, and semantics) as well as a parser and analyzer. It is not itself a database or query engine. Instead it is intended to be used by multiple engines wanting to provide consistent behavior for all semantic analysis, name resolution, type checking, implicit casting, etc. 
>
> [ZetaSQL Language Guide](https://github.com/google/zetasql/blob/master/docs/README.md)
>
> [ZetaSQL Resolved AST](https://github.com/google/zetasql/blob/master/docs/resolved_ast.md)

## Quick Insight ZetaSQL Parser 

We do some survey on ZetaSQL. ZetaSQL provides many APIs and services. 

![zetasql analyzer workflow](./images/image-zetasql-workflow.png)

The figure above demonstrate the zetasql's analyze workflow. But we **only** use its parser module, since It can fullfill our requirements.

## Do not use ZetaSQL analyzer

We won't use zetasql analyer in this vesion for the following reasons:

- HybridSE already has its statement validation module. We use `SchemaContext` and `ExprPass` to validate column, expression and functions. ZetaSQL might not meet our need, since it would be difficult to validate our udf and udaf. A lot works will be done with few benefits.
- Analyzer outputs AnalyzerOutput which mainly contains ResolvedStatement. It would be much harder to convert `ResolvedExpression` comparing to convert `ASTExpression`. For instance:
  - `ExprNode` and `ASTExpression` both use `BinaryExpression` to represent operator like `+`, `-`, `*`, `/`, `MOD`, but `ResolvedExpression` use `FunctionCallNode` instead. 
  - `ResolvedExpression` do not have `CastNode`, `CaseWhenNode`
- ZetaSQL can validate SQL statement when we register catalog information to zetasql.  It would be better if we use analyzer after refactor hyrbidse `Catalog`.

## Use ZetaSQL parser

ZetaSQL supports standare ANSI SQL.  We are going to integrate zetasql's parser by following steps:

1. Use `ParseStatement` APIs to generate `ASTStatement` . 
2. Transform `ASTStatement` into `LogicalPlan` .
3. Convert `ASTExpression`  to `ExprNode` 

**Pros**:

We will keep using `ExprNode`  in hybridse. This will introduce least changes into codegen module and physical plan module. Also, keeping the APIs stable would be friendly to out users (e.g. fedb, sparkfe)

**Cons**:

We hava `ExprNode` and `ASTExpression` in hybridse at the same time. This is redundant. 

### Convert `ASTExpression` to `ExprNode`

| ASTExpression                                                | `ExprNode`       | New traits |
| ------------------------------------------------------------ | ---------------- | ---------- |
| ResolvedParameter                                            |                  |            |
| `ASTStringLiteral` <br/>ASTFloatLiteral <br/>ASTNullLiteral <br/>ASTNumericLiteral <br/>ASTIntLiteral <br/>ASTBooleanLiteral | ConstNode        |            |
| ResolvedFunctionCallBase                                     |                  |            |
| ASTCastExpression                                            | CastExprNode     |            |
| ASTCaseValueExpression                                       | CaseWhenExprNode |            |
| ASTFunctionCall                                              | CallExprNode     |            |
| ASTBinaryExpression <br/>ASTAndExpr <br/>                    | BinaryExpr       |            |
| ASTUnaryExpression                                           | UnaryExpr        |            |
| ASTIdentifier                                                | ExprIdNode       |            |
| ASTPathExpression                                            | ColumnRefNode    |            |
| ASTBetweenExpression                                         | BetweenExpr      |            |
| ASTOrderBy                                                   | OrderByNode      |            |
| ASTTablePathExpression                                       | TableRefNode     |            |
|                                                              |                  |            |
|                                                              |                  |            |

 #### Convert `ParserOutput` to LogicalPlan  [WIP]

| ParserOutput / ASTNode                                       | SQLNode         | LogicalPlan                                                  |
| ------------------------------------------------------------ | --------------- | ------------------------------------------------------------ |
| ASTSelect  <br/>ASTSelectList <br/>ASTSelectColumn <br/>ASTFromClause <br/>ASTWhereClause | SelectQueryNode | `bool Planner::CreateSelectQueryPlan(const ASTSelect *root,                                     PlanNode **plan_tree, Status &status) ` |
| ASTInsertStatement                                           | InsertStmt      | `bool Planner::CreateInsertPlan(const node::ASTInsertStatement*,  *root,                                node::PlanNode **output,                                Status &status) {  // NOLINT (runtime/references)` |
| ASTCreateStatement                                           | CreateStmt      |                                                              |
| ASTFunctionDeclaration                                       | FnNodeFnDef     |                                                              |



## Add new syntax into ZetaSQL (WPI)

ZetaSQL also use bison and flex to parse SQL. It will easy for us to imigrate our New SQL syntax, e.g., `LAST JOIN`, `WINDOW ROWS_RANGE`.

### Last Join

**Solution**:

the `ORDER BY` option can be added into `opt_on_or_using_clause_list`  and add code to deal with it.

### Window ROWS_RANGE

- Make sure HybridSE `udf` and `udaf` works with zetasql - todo



### Create SQL (WIP)

`CREATE` statement in hybridse isn't standard. 

The `Index` and `distribution` options.



## Design Error Tips

### Syntax Error Tips

#### Use ZetaSQL syntax error tips format

- SELECT list empty

```SQL
select from t
--
ERROR: Syntax error: SELECT list must not be empty [at 1:9]
select  from t
        ^
```

```SQL
select DISTINCT from t
--
ERROR: Syntax error: SELECT list must not be empty [at 1:17]
select DISTINCT from t
                ^
```

- Join 

```SQL
# Error checking: invalid JOIN (not enough ON clause) is detected
select * from a join b join c join d on cond1 on cond2
--
ERROR: Syntax error: The number of join conditions is 2 but the number of joins that require a join condition is 3. INNER JOIN must have an ON or USING clause [at 1:17]
select * from a join b join c join d on cond1 on cond2
                ^
==
```

```SQL
# Error checking: too many ON clauses
select * from a join b on cond1 on cond2
--
ERROR: Syntax error: The number of join conditions is 2 but the number of joins that require a join condition is only 1. Unexpected keyword ON [at 1:33]
select * from a join b on cond1 on cond2
                                ^
==
```

```SQL

select * from a join b on cond1 join c on cond2 on cond3
--
ERROR: Syntax error: The number of join conditions is 2 but the number of joins that require a join condition is only 1. Unexpected keyword ON [at 1:49]
select * from a join b on cond1 join c on cond2 on cond3
                                                ^
==
```

- Create table

```SQL
# Column names must be identifiers.
create table t1 (a.b.c int64);
--
ERROR: Syntax error: Unexpected "." [at 1:19]
create table t1 (a.b.c int64);
                  ^
==
```

```SQL
# Missing type.
create table t1 (a, b string);
--
ERROR: Syntax error: Unexpected "," [at 1:19]
create table t1 (a, b string);
                  ^
==
```

- Drop table

```SQL
drop table;
--
ERROR: Syntax error: Unexpected ";" [at 1:11]
drop table;
          ^
==
```



#### All tips must be include help links

```SQL
select from t
--
ERROR: Syntax error: SELECT list must not be empty [at 1:9]
select  from t
        ^
If the tips above don't help, you can get more help from [slack channel](https://hybridsql-ws.slack.com/archives/C01R7LAF6AY)
```

If the tips above don't help, you can get more help from [slack channel](https://hybridsql-ws.slack.com/archives/C01R7LAF6AY)



### Plan Error Tips

We will discuss plan error tips in the furture works.

## Build & test

In order to build and test in different sysmtem,  we can simplify build and test packages.  Check [Discussion link](https://github.com/4paradigm/HybridSE/discussions/42) for more details.

- Build

```shell
bazel build --features=-supports_dynamic_linker -- //... -zetasql/jdk/... -zetasql/local_service/... -java/... -javatests/...
```

- Test

```shell
bazel test --test_summary=detailed --features=-supports_dynamic_linker -- //... -zetasql/jdk/... -zetasql/local_service/... -java/... -javatests/...
```

**Linux system compatibility can be shown as below table:**

| os            | gcc    | glibc | build | test |      |
| ------------- | ------ | ----- | ----- | ---- | ---- |
| centos:7      | 7.3.1  | 2.17  | pass  | fail |      |
| centos:7      | 8.3.1  | 2.17  | pass  | fail |      |
| centos:7      | 9.3.1  | 2.17  | pass  | fail |      |
| centos:8      | 8.3.1  | 2.28  | pass  | pass |      |
| ubuntu:bionic | 8.4.0  | 2.27  | pass  | fail |      |
| ubuntu:focal  | 9.3.0  | 2.31  | pass  | pass |      |
| Debian:buster | 8.31   | 2.28  | pass  | pass |      |
| ArchLinux     | 10.2.0 | 2.33  | pass  | pass |      |

Linux building status: https://github.com/aceforeverd/zetasql/actions/runs/750588862

**Mac system compatibility can be shown as below table:**

| os        | Clang                                           | build | test |      |
| --------- | ----------------------------------------------- | ----- | ---- | ---- |
| Mac:10.15 | Apple clang version 12.0.0 (clang-1200.0.32.21) | pass  | Pass |      |

Mac building status: https://github.com/jingchen2222/zetasql/runs/2348401484?check_suite_focus=true

## Install (WIP)

We create a cmake sampel project integrated zetasql. Check [Discussion link](https://github.com/4paradigm/HybridSE/discussions/42) for more details.

https://github.com/aceforeverd/zetasql-sample

Check [Discussion link](https://github.com/4paradigm/HybridSE/discussions/42) for more details.

# Adoption strategy

If we implement this proposal, it might bring some changes of APIs which is used by SparkFE and FEDB.

We will try to avoid introducing such changes, but there are some changes might be:

- We have to follow standare SQL `CREATE TABLE` Syntax, especially `Index` and `Partition` option configure. We couldn't confirm the `CREATE` syntax would 



