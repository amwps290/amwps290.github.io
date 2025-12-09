---
title: RisingLight 源码分析-4
author: amwps290
date: 2025-12-08 00:00
categories: [RISINGIGHT,源码分析]
tags: [risinglight,db]
render_with_liquid: false
---

# RisingLight 学习：DataType 的实现方法分析

在上一节中，我们了解了 `DataType` 枚举的定义，它构成了 RisingLight 数据类型系统的基础。本节我们将深入分析 `DataType` 的实现方法，这些方法为类型系统提供了丰富的功能支持。

## 方法实现分析

以下是 `DataType` 中定义的关键方法：

```rust
impl DataType {
    pub const fn is_null(&self) -> bool {
        matches!(self, Self::Null)
    }

    pub const fn is_number(&self) -> bool {
        matches!(
            self,
            Self::Int16 | Self::Int32 | Self::Int64 | Self::Float64 | Self::Decimal(_, _)
        )
    }

    pub const fn is_parametric_decimal(&self) -> bool {
        matches!(self, Self::Decimal(Some(_), _) | Self::Decimal(_, Some(_)))
    }

    /// Returns the inner types of the struct.
    pub fn as_struct(&self) -> &[DataType] {
        let Self::Struct(types) = self else {
            panic!("not a struct: {self}")
        };
        types
    }

    /// Returns the minimum compatible type of 2 types.
    pub fn union(&self, other: &Self) -> Option<Self> {
        use DataType::*;
        let (a, b) = if self <= other {
            (self, other)
        } else {
            (other, self)
        }; // a <= b
        match (a, b) {
            (Null, _) => Some(b.clone()),
            (Bool, Bool | Int32 | Int64 | Float64 | Decimal(_, _) | String) => Some(b.clone()),
            (Int32, Int32 | Int64 | Float64 | Decimal(_, _) | String) => Some(b.clone()),
            (Int64, Int64 | Float64 | Decimal(_, _) | String) => Some(b.clone()),
            (Float64, Float64 | Decimal(_, _) | String) => Some(b.clone()),
            (Decimal(_, _), Decimal(_, _) | String) => Some(b.clone()),
            (Date, Date | String) => Some(b.clone()),
            (Interval, Interval | String) => Some(b.clone()),
            (String, String | Blob) => Some(b.clone()),
            (Blob, Blob) => Some(b.clone()),
            (Struct(a), Struct(b)) => {
                if a.len() != b.len() {
                    return None;
                }
                let c = (a.iter().zip(b.iter()))
                    .map(|(a, b)| a.union(b))
                    .try_collect()?;
                Some(Struct(c))
            }
            _ => None,
        }
    }
}
```

## 方法详解

### `is_null()` - 空值检测
- **功能**：判断当前类型是否为 `Null` 类型
- **实现**：使用 `matches!` 宏进行模式匹配，简洁高效
- **特点**：标记为 `const fn`，可在编译期求值

### `is_number()` - 数值类型检测
- **功能**：判断是否为数值类型
- **覆盖类型**：`Int16`、`Int32`、`Int64`、`Float64` 以及任意精度的 `Decimal`
- **设计考虑**：`Decimal` 无论是否指定精度和范围，都被视为数值类型

### `is_parametric_decimal()` - 参数化 Decimal 检测
- **功能**：检测 `Decimal` 类型是否指定了精度或范围参数
- **参数说明**：`Decimal(Option<u8>, Option<u8>)` 的第一个参数是精度（precision），第二个是范围（scale）
- **使用场景**：区分完全通用的 `Decimal(None, None)` 与具有特定精度/范围的 Decimal

### `as_struct()` - 结构体类型解构
- **功能**：安全地获取结构体类型的内部类型列表
- **错误处理**：如果类型不是 `Struct`，则通过 `panic!` 提供清晰的错误信息
- **返回类型**：返回 `&[DataType]` 切片，避免所有权转移

### `union()` - 类型兼容性合并
- **核心功能**：计算两个数据类型的最小兼容类型
- **算法步骤**：
  1. **排序保证**：利用 `DataType` 实现的 `PartialOrd` trait，确保 `a <= b`，简化模式匹配
  2. **兼容性规则**：基于类型系统的隐式转换规则：
     - `Null` 可转换为任何类型
     - 数值类型存在向上转换链：`Bool → Int32 → Int64 → Float64 → Decimal → String`
     - 时间/日期类型可转换为 `String`
     - `String` 可转换为 `Blob`
  3. **结构体处理**：递归地对齐字段并合并，要求字段数量相同
- **返回值**：返回 `Option<DataType>`，`None` 表示类型不兼容

## 设计亮点

1. **常量函数优化**：`is_null`、`is_number`、`is_parametric_decimal` 均为 `const fn`，支持编译期计算
2. **模式匹配的优雅使用**：Rust 的 `matches!` 宏和模式匹配使类型检测代码简洁清晰
3. **错误信息友好**：`as_struct` 在 panic 时包含具体类型信息，便于调试
4. **递归算法设计**：`union` 方法对 `Struct` 类型采用递归处理，体现了复合类型的统一处理策略
5. **迭代器链式调用**：`try_collect()` 与 `zip()`、`map()` 的组合展示了 Rust 迭代器的强大表达能力

## 总结

`DataType` 的实现方法展示了 RisingLight 类型系统的核心能力：类型检测、安全访问和类型兼容性计算。这些方法为查询优化、类型推导和运行时检查提供了坚实基础。下一节我们将探讨这些类型在物理存储层面的具体实现。