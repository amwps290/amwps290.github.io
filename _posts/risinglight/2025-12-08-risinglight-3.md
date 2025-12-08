---
title: RisingLight 源码分析-3
author: amwps290
date: 2025-12-06 00:00
categories: [RISINGIGHT,源码分析]
tags: [risinglight,db]
render_with_liquid: false
---

# RisingLight 学习：数据类型系统概览

在上一节中，我们对 RisingLight 的整体架构有了初步了解。从本节开始，我们将深入代码细节，首先从数据库的核心——数据类型系统入手。

所有与数据类型相关的实现都位于 `src/types` 目录下，其中包含了 `blob.rs`、`date.rs` 等多个模块，分别定义了各种具体的数据类型。

今天，我们首先分析 `src/types/mod.rs` 文件，它作为类型模块的入口，定义了整个数据类型系统的基础结构，并整合了所有数据类型子模块。

## `mod.rs` 文件结构分析

让我们看一下 `mod.rs` 的核心代码：

```rust
use std::hash::Hash;
use std::num::ParseIntError;
use std::str::FromStr;

use parse_display::Display;
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use sqlparser::ast::TimezoneInfo;

mod blob;
mod date;
mod interval;
mod native;
mod timestamp;
mod value;
mod vector;

pub use self::blob::*;
pub use self::date::*;
pub use self::interval::*;
pub use self::native::*;
pub use self::timestamp::*;
pub use self::value::*;
pub use self::vector::*;
```

这段代码主要完成了四项工作：

1.  **导入标准库**: 引入了 Rust 标准库中的 `Hash`、`FromStr` 等 Trait 和类型。
2.  **导入第三方库**: 引入了 `parse_display`、`rust_decimal`、`serde` 和 `sqlparser` 等外部库的关键组件。
3.  **声明子模块**: 通过 `mod` 关键字，声明了 `blob`、`date` 等数据类型的子模块。
4.  **重导出子模块内容**: 使用 `pub use` 语句，将所有子模块的公共接口导出，方便其他模块统一调用。

### 关键第三方库解析

-   **`parse-display`**: 这个库提供了一种便捷的方式，通过派生宏 `Display` 和 `FromStr` 来实现结构体与字符串之间的相互转换。例如：

    ```rust
    #[derive(Display, FromStr, PartialEq, Debug)]
    #[display("{a}-{b}")]
    struct X {
      a: u32,
      b: u32,
    }
    assert_eq!(X { a:10, b:20 }.to_string(), "10-20");
    assert_eq!("10-20".parse(), Ok(X { a:10, b:20 }));
    ```
    通过 `#[display("{a}-{b}")]` 宏，我们可以轻松地将结构体 `X` 格式化为 `10-20` 这样的字符串，反之亦然。

-   **`serde`**: 这是 Rust 生态中用于序列化和反序列化的标准库，在数据持久化、网络传输等场景中至关重要。

-   **`rust_decimal`**: 提供了一个用于精确金融计算的 `Decimal` 类型。与传统的浮点数（如 `f64`）相比，`Decimal` 避免了精度误差问题，这在数据库系统中是必不可少的。

    ```rust
    // 需要引入 `rust_decimal_macros::dec`
    use rust_decimal_macros::dec;

    let number = dec!(-1.23) + dec!(3.45);
    assert_eq!(number, dec!(2.22));
    assert_eq!(number.to_string(), "2.22");
    ```

-   **`sqlparser`**: `TimezoneInfo` 类型来自于这个库，用于表示和处理带时区信息的时间戳。我们将在后续分析时间类型时详细介绍它。

## `DataType` 枚举

接下来是 `mod.rs` 中最重要的定义——`DataType` 枚举。它定义了 RisingLight 中所有支持的逻辑数据类型。

```rust
/// Data type.
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash, Serialize, Deserialize)]
pub enum DataType {
    // NOTE: order matters
    Null,
    Bool,
    Int16,
    Int32,
    Int64,
    // Float32,
    Float64,
    // decimal (precision, scale)
    Decimal(Option<u8>, Option<u8>),
    Date,
    Timestamp,
    TimestampTz,
    Interval,
    String,
    Blob,
    Struct(Vec<DataType>),
    Vector(usize),
}
```

从定义中我们可以看到 RisingLight 支持以下数据类型：

-   **基础类型**: `Null`、`Bool`、`Int16`、`Int32`、`Int64`、`Float64`、`String`。
-   **高精度数值**: `Decimal`，支持指定精度和范围。
-   **时间与日期**: `Date`、`Timestamp` (无时区)、`TimestampTz` (带时区)、`Interval` (时间间隔)。
-   **二进制数据**: `Blob` (Binary Large Object)，用于存储二进制数据。
-   **复合类型**: `Struct` (结构体，由多个数据类型组成) 和 `Vector` (向量)。

## 总结

通过对 `src/types/mod.rs` 的分析，我们了解了 RisingLight 数据类型系统的整体设计和基础构成。`DataType` 枚举是理解整个系统的关键。

在下一节中，我们将逐一深入分析 `DataType` 中定义的每一种数据类型，并探究它们在物理层面的具体实现。
