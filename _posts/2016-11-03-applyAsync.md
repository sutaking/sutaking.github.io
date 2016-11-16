---
layout: post
title: $applyAsync() 到底是什么
date: 2016-11-03 09:32:24.000000000 +09:00
---

[访问原文地址](http://njfeng.com/2016/11/applyAsync/)

## 概述

applyAsync是随着1.3发布的API，1.3的版本主要目的是在于提高angular的整体性能。可以说这是一个高性能的接口。

文章主要介绍了，在处理多个$http请求的时，如果响应时间发生一致时，`$rootScope`会提供`$applyAsync()`来执行一次`$digest`来解决这个问题。


## why&when 我们需要执行`$digest cysle`

angular为了实现双向绑定，做了一个`event loop`来自动管理和更新app的model和DOM，也就是`$digest`方法。

但是这不是完全自动的，有几个情况还是需要我们去手动激活`$digest`

-	User interaction through events: 发生来自用户的UI操作，比如点击button或者输入input框之类，改变了app数据的state。
- http请求：XMLHttpRequests，或者说AJAX。
- Timeouts：通过异步操作计时器来改变app数据的state。

一旦发生上面的操作，都是需要去手动调用一下`$digest`，这样看起来感觉用起来很麻烦，对不对。

但事实上，angular开发中不要担心这些，因为angular已经提供了配置好`$digest`的组件，`ng-click`、`ng-input`、`$http`，这些组件会自动的去调用trigger。


## 该`$applyAsync()`出场了

就像名字显示的那样，`$applyAsync()`是`$apply()`的异步方法，用来提示性能的。

当我们发起一个`$apply`时会执行一次loop-event，也就是说，当发起多次http请求，哪怕是同时响应回调，angular仍然会执行多次loop-event。这样会影响到我们app的性能。如果能让同时发生的数据请求操作，减少执行loop-event的次数，很显然可以提高app的速度。

`$applyAsync()`就是做这件事的。他可以很好的控制多次请求后的`$apply`操作发生在一次更新中。

- 他把这次请求的promise放在一个queue里
- 通过让浏览器请求数据的时候执行setTimeout()，他不执行结束不安排下一次计时器。
- 当timeout执行，queue中的所有请求都完成后，执行一次`$apply`就可以了
- 如果浏览器的请求间隔不高10毫秒，setTimeout会默认为0延迟，这样，多次请求返回在同一时间，等于执行一次就可以了。

### 引用
[http://blog.thoughtram.io/angularjs/2015/01/14/exploring-angular-1.3-speed-up-with-applyAsync.html](http://blog.thoughtram.io/angularjs/2015/01/14/exploring-angular-1.3-speed-up-with-applyAsync.html)



