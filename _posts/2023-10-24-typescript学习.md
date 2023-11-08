---
title: typescript 学习
author: setKing
date: 2023-10-24 11:33:00 +0800
categories: [学习]
tags: [学习]
pin: true
math: true
mermaid: true
---

## typescript 策略行为

```{
  "compilerOptions": {
    // 项目的编译选项
  },
  "include": [
    // 项目包含哪些文件
  ],
  "exclude": [
    // 在 include 包含的文件夹中需要排除哪些文件
  ],
  "references": [
    //通过 references 可以引用其他 project
  ]
}
```

`include` 与 `exclude` 字段通过 `glob` 语法进行文件匹配
