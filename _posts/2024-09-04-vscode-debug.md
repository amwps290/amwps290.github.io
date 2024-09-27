---
title: VScode 调试命令记录
author: amwps290
date: 2024-09-04 00:10:00 +0800
categories: [Vscode]
tags: [vscode,debug]
render_with_liquid: false
---


## LLDB

### 打印数组

在 vscode 中使用 lldb 调试 C/C++ 时，我们想打印数组的时候，可以在 DEBUG 命令行输入 

`parray` 即可输出数组，也可以在 WATCH 窗口中使用 `var,[N]` 的语法形式,注意逗号后不要有空格。


