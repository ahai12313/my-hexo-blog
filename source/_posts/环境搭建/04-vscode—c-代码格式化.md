---
title: 04_vscode—c++代码格式化
date: 2025-09-21 15:08:39
tags:
categories: 环境搭建
---
1. 项目根目录执行
`sudo apt install clang-format`
`clang-format -style=llvm -dump-config > .clang-format`
2. 修改.clang-format
```yaml
BasedOnStyle: LLVM
IndentWidth: 4
TabWidth: 4
UseTab: Never
```