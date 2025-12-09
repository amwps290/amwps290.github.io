---
title: RisingLight 源码分析-4
author: amwps290
date: 2025-12-09 00:00
categories: [RISINGIGHT,源码分析]
tags: [risinglight,db]
render_with_liquid: false
---

### `DataType` 的转换与实现

今天我们继续分析 `types/mod.rs` 文件，重点关注 `DataType` 与其他类型之间的转换逻辑，以及相关的错误处理。

#### 1. 从 `sqlparser::DataType` 到自定义 `DataType`

为了将 SQL 解析器（`sqlparser`）的 AST 节点无缝转换为 RisingLight 内部的 `DataType`，我们为 `DataType` 实现 `From<&crate::parser::DataType>` Trait。这样，在代码中我们就可以使用 `.into()` 进行隐式类型转换，提升了代码的简洁性。

```rust
impl From<&crate::parser::DataType> for DataType {
    fn from(kind: &crate::parser::DataType) -> Self {
        use sqlparser::ast::ExactNumberInfo;

        use crate::parser::DataType::*;
        match kind {
            Char(_) | Varchar(_) | String(_) | Text => Self::String,
            Bytea | Binary(_) | Varbinary(_) | Blob(_) => Self::Blob,
            // Real => Self::Float32,
            Float(_) | Double => Self::Float64,
            SmallInt(_) => Self::Int16,
            Int(_) | Integer(_) => Self::Int32,
            BigInt(_) => Self::Int64,
            Boolean => Self::Bool,
            Decimal(info) => match info {
                ExactNumberInfo::None => Self::Decimal(None, None),
                ExactNumberInfo::Precision(p) => Self::Decimal(Some(*p as u8), None),
                ExactNumberInfo::PrecisionAndScale(p, s) => {
                    Self::Decimal(Some(*p as u8), Some(*s as u8))
                }
            },
            Date => Self::Date,
            Timestamp(_, TimezoneInfo::None) => Self::Timestamp,
            Timestamp(_, TimezoneInfo::Tz) => Self::TimestampTz,
            Interval => Self::Interval,
            Custom(name, items) => {
                if name.to_string().to_lowercase() == "vector" {
                    if items.len() != 1 {
                        panic!("must specify length for vector");
                    }
                    Self::Vector(items[0].parse().unwrap())
                } else {
                    todo!("not supported type: {:?}", kind)
                }
            }
            _ => todo!("not supported type: {:?}", kind),
        }
    }
}
```

转换逻辑清晰地将多种 SQL 标准类型映射到我们内部定义的 `DataType`：

*   **字符串类型**: `Char`, `Varchar`, `String`, `Text` -> `DataType::String`
*   **二进制类型**: `Bytea`, `Binary`, `Varbinary`, `Blob` -> `DataType::Blob`
*   **浮点数类型**: `Float`, `Double` -> `DataType::Float64`
*   **整数类型**: `SmallInt`, `Int`, `BigInt` -> `DataType::Int16`, `DataType::Int32`, `DataType::Int64`
*   **其他类型**: `Boolean`, `Decimal`, `Date` 等也都有对应的映射。

#### 2. `DataType` 的显示格式

为了方便地打印和调试 `DataType`，我们为其实现了 `Display` Trait，使其能够被格式化为 SQL 类型字符串。

```rust
impl std::fmt::Display for DataType {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Null => write!(f, "NULL"),
            Self::Int16 => write!(f, "SMALLINT"),
            Self::Int32 => write!(f, "INT"),
            Self::Int64 => write!(f, "BIGINT"),
            // Self::Float32 => write!(f, "REAL"),
            Self::Float64 => write!(f, "DOUBLE"),
            Self::String => write!(f, "STRING"),
            Self::Blob => write!(f, "BLOB"),
            Self::Bool => write!(f, "BOOLEAN"),
            Self::Decimal(p, s) => match (p, s) {
                (None, None) => write!(f, "DECIMAL"),
                (Some(p), None) => write!(f, "DECIMAL({p})"),
                (Some(p), Some(s)) => write!(f, "DECIMAL({p},{s})"),
                (None, Some(_)) => panic!("invalid decimal"),
            },
            Self::Date => write!(f, "DATE"),
            Self::Timestamp => write!(f, "TIMESTAMP"),
            Self::TimestampTz => write!(f, "TIMESTAMP WITH TIME ZONE"),
            Self::Interval => write!(f, "INTERVAL"),
            Self::Struct(types) => {
                write!(f, "STRUCT(")?;
                for t in types.iter().take(1) {
                    write!(f, "{}", t)?;
                }
                for t in types.iter().skip(1) {
                    write!(f, ", {}", t)?;
                }
                write!(f, ")")
            }
            Self::Vector(length) => write!(f, "VECTOR({length})"),
        }
    }
}
```

#### 3. 从字符串解析 `DataType`

通过实现 `FromStr` Trait，我们可以将字符串直接解析为 `DataType`。这个功能在处理 schema 或者 catalog 信息时非常有用。

```rust
impl FromStr for DataType {
    type Err = ParseTypeError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        use DataType::*;
        Ok(match s {
            "INT" => Int32,
            "BIGINT" => Int64,
            // "REAL" => Float32,
            "DOUBLE" => Float64,
            "STRING" => String,
            "BLOB" => Blob,
            "BOOLEAN" => Bool,
            "DECIMAL" => Decimal(None, None),
            _ if s.starts_with("DECIMAL") => {
                let para = s
                    .strip_prefix("DECIMAL")
                    .unwrap()
                    .trim_matches(|c: char| c == '(' || c == ')' || c.is_ascii_whitespace());
                match para.split_once(',') {
                    Some((p, s)) => Decimal(Some(p.parse()?), Some(s.parse()?)),
                    None => Decimal(Some(para.parse()?), None),
                }
            }
            "DATE" => Date,
            "INTERVAL" => Interval,
            _ => return Err(ParseTypeError::Invalid(s.to_owned())),
        })
    }
}
```

解析失败时，会返回一个 `ParseTypeError`。

#### 4. 统一的错误处理

代码中定义了 `ParseTypeError` 和 `ConvertError` 两种错误类型，并借助 `thiserror` 这个 crate 来简化错误处理。

`thiserror` 提供了一个派生宏，可以让我们用结构化的方式定义错误类型，而无需手动实现 `Display` 和 `Error` Trait。例如 `ParseTypeError` 的定义：

```rust
#[derive(thiserror::Error, Debug, Clone, PartialEq, Eq)]
pub enum ParseTypeError {
    #[error("invalid number: {0}")]
    ParseIntError(#[from] ParseIntError),
    #[error("invalid type: {0}")]
    Invalid(String),
}
```

这里的 `#[error("...")]` 属性宏会自动生成 `Display` 的实现。当一个 `ParseIntError` 发生时，它会被自动转换为 `ParseTypeError::ParseIntError`，并打印出 "invalid number: ..." 的信息，代码非常清晰。

`ConvertError` 则更全面地定义了在数据类型转换过程中可能出现的各种错误。

```rust
#[derive(thiserror::Error, Debug, Clone, PartialEq)]
pub enum ConvertError {
    #[error("failed to convert string {0:?} to int: {1}")]
    ParseInt(String, #[source] std::num::ParseIntError),
    #[error("failed to convert string {0:?} to float: {1}")]
    ParseFloat(String, #[source] std::num::ParseFloatError),
    #[error("failed to convert string {0:?} to bool: {1}")]
    ParseBool(String, #[source] std::str::ParseBoolError),
    // ... 其他错误定义 ...
    #[error("no cast {0} -> {1}")]
    NoCast(&'static str, DataType),
}
```

这些详尽的错误类型为后续的执行器和类型转换逻辑提供了健壮的错误处理基础。
