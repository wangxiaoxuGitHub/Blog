---
title: 漫谈Guava之Multiset
date: 2020-03-15 17:40:24
tags:
- Guava
- Collection
- 集合
categories: Java
---



### Guava源码分析之Multiset

> Guava provides a new collection type, [`Multiset`](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/collect/Multiset.html), which supports adding multiples of elements. Wikipedia defines a multiset, in mathematics, as “a generalization of the notion of set in which members are allowed to appear more than once...In multisets, as in sets and in contrast to tuples, the order of elements is irrelevant: The multisets {a, a, b} and {a, b, a} are equal.”

![20200427093254](https://img.yjll.art//img/20200427093254.png)

