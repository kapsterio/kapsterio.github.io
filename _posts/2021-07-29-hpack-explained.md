---
layout: post
title: "HPACK notes"
description: ""
category: 
tags: []
---

## Header Compression for HTTP/2

### 为什么需要Hpack
- http headers越来越大，浪费带宽，导致延迟
- 使用常规DEFLATE算法压缩header存在安全问题，[CRIME](https://zh.wikipedia.org/zh-hans/CRIME)

### 