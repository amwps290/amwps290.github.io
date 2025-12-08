---
title: RisingLight 源码分析-1
author: amwps290
date: 2025-12-03 00:00
categories: [RISINGIGHT,源码分析]
tags: [risinglight,db]
render_with_liquid: false
---

RisingLight 是一个专为教学目的设计的分析型数据库，采用 Rust 语言编写。

今天，我们将聚焦于其 parser（解析器）部分。在数据库处理一条 SQL 语句时，SQL 解析是执行流程中的第一个关键步骤。在 RisingLight 中，解析器的核心职责是将输入的 SQL 语句转化为抽象语法树（Abstract Syntax Tree, 简称 AST），为后续的查询优化和执行奠定基础。本项目中，我们采用了 sqlparser 这一流行的 Rust 库来完成 SQL 解析任务。

接下来，我们通过一个 sqlparser 的官方示例来具体了解其工作原理：

```rust
fn main() {
    use sqlparser::dialect::GenericDialect;
    use sqlparser::parser::Parser;

    let dialect = GenericDialect {}; // or AnsiDialect

    let sql = "SELECT * from test_table";

    let ast = Parser::parse_sql(&dialect, sql).unwrap();

    println!("AST: {:?}", ast);
}

```

```
Query(
    Query {
        with: None,
        body: Select(
            Select {
                select_token: TokenWithSpan {
                    token: Word(Word { value: "SELECT", ... }),
                    span: Span(Location(1,1)..Location(1,7))
                },
                distinct: None,
                top: None,
                top_before_distinct: false,
                projection: [
                    Wildcard(
                        WildcardAdditionalOptions {
                            wildcard_token: TokenWithSpan {
                                token: Mul, // The '*' character
                                span: Span(Location(1,8)..Location(1,9))
                            },
                            opt_ilike: None,
                            opt_exclude: None,
                            opt_except: None,
                            opt_replace: None,
                            opt_rename: None
                        }
                    )
                ],
                exclude: None,
                into: None,
                from: [
                    TableWithJoins {
                        relation: Table {
                            name: ObjectName([
                                Identifier(Ident {
                                    value: "test_table",
                                    quote_style: None,
                                    span: Span(Location(1,15)..Location(1,25))
                                })
                            ]),
                            alias: None,
                            args: None,
                            with_hints: [],
                            version: None,
                            with_ordinality: false,
                            partitions: [],
                            json_path: None,
                            sample: None,
                            index_hints: []
                        },
                        joins: []
                    }
                ],
                lateral_views: [],
                prewhere: None,
                selection: None,       // Corresponds to the WHERE clause
                group_by: Expressions([], []), // Corresponds to the GROUP BY clause
                cluster_by: [],
                distribute_by: [],
                sort_by: [],
                having: None,          // Corresponds to the HAVING clause
                named_window: [],
                qualify: None,
                window_before_qualify: false,
                value_table_mode: None,
                connect_by: None,
                flavor: Standard
            }
        ),
        order_by: None,        // Corresponds to the ORDER BY clause
        limit_clause: None,    // Corresponds to the LIMIT clause
        fetch: None,
        locks: [],
        for_clause: None,
        settings: None,
        format_clause: None,
        pipe_operators: []
    }
)

```

从这份详尽的抽象语法树输出中，我们可以清晰地提取出 SQL 语句的诸多关键信息：

SELECT 关键字的位置： 位于第 1 行的第 1 列至第 7 列。
投影方式： 采用通配符 * 查询所有字段。
查询目标表： test_table。
获取到这样的语法树之后，我们便能展开后续的处理工作。例如，我们可以根据 AST 中识别出的表名 test_table，到数据库的元数据信息中查询该表是否存在，并进一步验证其结构等。