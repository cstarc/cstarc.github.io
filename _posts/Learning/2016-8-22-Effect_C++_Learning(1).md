---
layout: post
title: Effect_C++_Learning 1
category: Learning
tags: c++, effect
keywords: c++, effect
description: item 1 and 2.
---

## *条款1 视c++为一个语言联邦*
随着C++的逐渐成熟，开始接受不同于C With Class的各种观念，特性和编程战略。
> 今天的C++已经是个多重泛型编程语言，一个同时支持过程形式,面向对象形式，函数形式，泛型形式，元编程形式的语言。


其主要次语言主要有四个，**每个次语言都有自己的规约**：
1. C
2. Object-Oriented C++
3. Template C++
4. STL

在使用不同的次语言时要使用符合这一种语言的策略，才能实现高效编程。

## *条款2 经量以const，enum，inline替换#define*
  处理#define的为预处理器
 1. 定义常量，若
  `#define MAX_NUM  1000`
  MAX_NUM可能并未进入符号表，那样当获得一个编译错误时，错误信息也许会提到1000而不是MAX_NUM，导致难以调试。
  ![](http://i.imgur.com/XKpIfXc.png)
  **可用const或enum替换**
 2. 若为形似函数的宏，由于其只是单纯的替换字符，可能会出现错误，在此书中就有个例子：
  ![](http://i.imgur.com/uND6VoK.png)
  **可用inline函数替换**

