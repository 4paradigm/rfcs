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



# Feasibility

> ZetaSQL defines a language (grammar, types, data model, and semantics) as well as a parser and analyzer. It is not itself a database or query engine. Instead it is intended to be used by multiple engines wanting to provide consistent behavior for all semantic analysis, name resolution, type checking, implicit casting, etc. 
>
> [ZetaSQL Language Guide](https://github.com/google/zetasql/blob/master/docs/README.md)
>
> [ZetaSQL Resolved AST](https://github.com/google/zetasql/blob/master/docs/resolved_ast.md)

## Quick Insight ZetaSQL Parser 

We do some survey on ZetaSQL. ZetaSQL provides many APIs and services. But we **only** use its parser module, since It can fullfill our requirements. 

### ParseStatement

`#include "zetasql/parser/parser.h"`

Parser module provides a set of APIs to parse `SQL` script, statement, expression etc.

```c++
absl::Status ParseStatement(absl::string_view statement_string,
                            const ParserOptions& parser_options_in,
                            std::unique_ptr<ParserOutput>* output);

absl::Status ParseScript(absl::string_view script_string,
                         const ParserOptions& parser_options_in,
                         ErrorMessageMode error_message_mode,
                         std::unique_ptr<ParserOutput>* output);
//...
```



## Build & test

In order to build and test in different sysmtem, it would be better to simplify build and test packages.  Check [Discussion link](https://github.com/4paradigm/HybridSE/discussions/42) for more details.

- Build

```shell
bazel build --features=-supports_dynamic_linker -- //... -zetasql/jdk/... -zetasql/local_service/... -java/... -javatests/...
```

- Test

```shell
bazel test --test_summary=detailed --features=-supports_dynamic_linker -- //... -zetasql/jdk/... -zetasql/local_service/... -java/... -javatests/...
```

Linux system compatibility can be shown as below table:

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
| Mac:10.15     | 4.2.1  |       | pass  | fail |      |

Linux building status: https://github.com/aceforeverd/zetasql/actions/runs/750588862

Mac system compatibility can be shown as below table:

| os        | Clang                                           | build | test |      |
| --------- | ----------------------------------------------- | ----- | ---- | ---- |
| Mac:10.15 | Apple clang version 12.0.0 (clang-1200.0.32.21) | pass  | Pass |      |

Mac building status: https://github.com/jingchen2222/zetasql/runs/2348401484?check_suite_focus=true

## Install (WIP)

We create a cmake sampel project integrated zetasql. Check [Discussion link](https://github.com/4paradigm/HybridSE/discussions/42) for more details.

https://github.com/aceforeverd/zetasql-sample

Check [Discussion link](https://github.com/4paradigm/HybridSE/discussions/42) for more details.



# Design [WIP]

## Use ZetaSQL as SQL analyzer

ZetaSQL supports standare ANSI SQL. Can fulfill our requirements.

**Pros**:

We keep `ExprNode`  of hybridse. This will introduce least changes into codegen module and plan module. Also, most APIs will change which would be friendly to users (e.g. fedb, sparkfe)

**Cons**:

We hava `ExprNode` and `ResolvedExpr` in hybridse at the same time. This is redundant. And we have to implement more `ExprNode` node to overlap all SQL expression syntax.

1. Use `Analyzer` APIs to generate `ResolvedAST` nodes. 
2. Convert `ResolvedExpr` (of zetasql) to `ExprNode` (of hybridse).
3. Convert `ResolvedStatement` to `LogicalPlan` (of hybridse).



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



## Do not use ZetaSQL validation and rewrite

ZetaSQL can validate SQL statement when we register catalog information to zetasql. We won't use it in this vesion untill we refactor our `Catalog`.



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



# Adoption strategy

If we implement this proposal, it might bring some changes of APIs which is used by SparkFE and FEDB.

We will try to avoid introducing such changes.



