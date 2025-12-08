---
title: RisingLight 源码分析-2
author: amwps290
date: 2025-12-07 00:00
categories: [RISINGIGHT,源码分析]
tags: [risinglight,db]
render_with_liquid: false
---


在上一节中，我们探讨了 RisingLight 如何利用 sqlparser 实现 SQL 解析功能。现在，为了更好地组织代码，我们将此功能封装为一个独立的子模块，并将其命名为 parser。此命名旨在明确其作为数据库执行 SQL 语句首个阶段（解析）的核心作用。

```rust
// Copyright 2024 RisingLight Project Authors. Licensed under Apache-2.0.

//! The parser module directly uses the [`sqlparser`] crate
//! and re-exports its AST types.

pub use sqlparser::ast::*;
use sqlparser::dialect::PostgreSqlDialect;
use sqlparser::parser::Parser;
pub use sqlparser::parser::ParserError;

/// Parse the SQL string into a list of ASTs.
pub fn parse(sql: &str) -> Result<Vec<Statement>, ParserError> {
    let dialect = PostgreSqlDialect {};
    Parser::parse_sql(&dialect, sql)
}

```

从上述代码可以看出，parse 函数直接封装了 sqlparser 提供的 SQL 解析功能。同时，为了便于其他模块使用，我们通过 pub use 语句重新导出了 sqlparser 中的抽象语法树（AST）相关类型和 ParserError 类型。接下来，我们将深入探讨数据库中的“类型”概念，它作为数据库的基础要素，决定了可存储数据的种类及其可执行的操作。

这里我们将 `main.rs` 中的代码修改位使用 `parser` 模块中的内容,这里还使用了 anyhow 来进行错误处理：

```rust
mod parser;
use crate::parser::parse;
use anyhow::Result;

fn main() -> Result<()> {
    let sql = "SELECT * from test_table";
    let ast = parse(sql)?;
    println!("{:?}", ast);
    Ok(())
}

```