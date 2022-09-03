---
title: 'C++的未定义,未指定,引用定义等概念是什么'
abbrlink: 55f19523
date: 2022-09-03 09:57:10
categories:
  - 细说CPP标准和stl源码
tags:
  - C++
---

摘录自C++draft第三章

<!-- more -->

## 3.14 条件支持（conditionally-supported）

标准不做出要求的，不一定所有编译器都支持的行为。

（每个编译期的实现都会有一个文档列出他们支持了哪些）

## 3.16 默认行为（default behavior）

在要求的行为的范围内，具体实现提供的明确行为。

## 3.25 非良构程序（ill-formed program）

任意一个不是良构（见下文）的程序。

## 3.26 实现定义行为（implementation-defined behavior）

在一个良构的程序和正确的数据中的，依赖于具体编译期实现和文档的行为。

## 3.30 本地指定行为（locale-specific behavior）

依赖于国家，文化，和语言的行为（什么鬼？？）

## 3.65 未定义行为（undefined behavior）

在本文档中没有做出任何要求的行为。

## 3.66 未指定行为（unspecified behavior）

在一个良构的程序和正确的数据中的，依赖于具体编译期实现的行为。

## 3.68 良构程序（well-formed program）

结构遵守了语法规则和语义规则的`C++`程序。