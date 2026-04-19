---
layout: post
title:  "对pg进行扩展"
date:   2025-10-04 09:23:57 +0800
permalink: /postgres/extension.html
tags: [postgres]
---

本文还在施工中。

后续准备系统梳理一下 PG 的几类扩展入口，例如：
- SQL / C 函数与自定义类型
- hook 机制
- 自定义执行器节点（如 Custom Scan）
- 自定义索引 AM / Table AM
- FDW、background worker 等扩展点

等正文补全后，再把这些内容串起来看。
