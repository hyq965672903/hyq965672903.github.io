---
title: JUC(七)-ThreadLocal详解
index_img: /img/default.png
banner_img: /img/default.png
abbrlink: 6f92e213
date: 2022-06-28 14:46:38
tags:
categories:
description:
---

学习并掌握ThreadLocal的使用，使用场景，原理。以及相关的强、软、弱、虚引用，ThreadLocal内存泄露问题剖析以及解决办法

<!-- more -->

## ThreadLocal简介

ThreadLocal又叫做线程局部变量，ThreadLocal为变量在每个线程中都创建了一个副本，那么每个线程可以访问自己内部的副本变量。

## 如何创建

## Thread、ThreadLocal和ThreadLocalMap三者之间的关系

## 源码分析 get和set

## 强引用、软引用、弱引用和虚引用是什么？ThreadLocal为什么采用弱引用

## ThreadLocal内存泄露原理分析，如何解决？

