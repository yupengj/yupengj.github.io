---
layout: post
title:  "hello jekyll!"
date:   2018-11-07 15:14:54
categories: jekyll
tags: jekyll
excerpt: 当年创建 jekyll 时默认的一篇文章，没什么意义，我也一直没删除，留个纪念吧。
mathjax: true
---

响应式编程（Reactive Programming）可以理解为一种处理数据项（Data Item）的异步流，即在数据项产生的时候，接收者就对其进行响应。在响应式编程中，会有一个数据发布者（Publisher）和数据订阅者（Subscriber），后者用于异步接收发布者发布的数据。在该模式中，还引入了一个更高级的特性：数据处理器（Processor），它用于将数据发布者发布的数据进行某些转换操作，然后再发布给数据订阅者。

总之，响应式编程是异步非阻塞编程，能够提升程序性能，可以解决传统编程模型遇到的困境。基于这个模型实现的有Java 9 Flow API、RxJava和Reactor等，这里主要介绍的是Java 9 Flow API的使用。

