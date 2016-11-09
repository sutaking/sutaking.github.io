---
layout: post
title: NetFlix对React的评价
date: 2016-11-01 09:32:24.000000000 +09:00
---

[原文](http://techblog.netflix.com/2015/01/netflix-likes-react.html)

netflix对自身的前端开发提出了3大难题

- 启动速度
- 运行时性能
- 模块化

#### 启动速度
对于SPA应用，启动速度主要来自加载时间和UI的渲染时间2大块。特别是移动端要面对各种复杂的网络环境的时候，这两部分的cost会显得非常残。

SPA启动时需要做太多的事情了，加载scripts、css，数据请求，图片，初始化界面，等等。除了网速问题这个最大的瓶颈。还有就是根据发生请求后，拿到json数据之后再渲染界面。为了缩短这个过程。Netflix采取了一种ismorphic的方式。由服务端渲染UI界面的首屏，之后的DOM操作再由浏览器端来渲染。这样做，由于是一组静态页面，因此根本不需要考虑数据请求的时间了。

#### 运行性能
复杂应用的主要消耗来源于UI组件渲染与操作。对一些性能较弱的移动设备，这部门显得由为的明显。DOM的重绘和重新排列，对用户的体现显得最为重要。

#### 模块化
前端开发必须要支持多种`A/B test`，NetFlix自己就有9种a/b测试UI版本，这就意味着需要维护至少10个版本的界面代码。我们必须要能在这种复杂的测试环境下，修复所有的bug，并且保持代码的结构清晰。模块化显得尤为重要。

## React的优势

#### ismorphic JavaScript
React支持前后端渲染，先最一个后端渲染的静态页面发到浏览器，之后的UI操作由后续的JavaScript在客户端完成。

#### Virtual DOM
为了减少DOM操作，react引入了VDOM操作，核心是diff算法，尽可能的减少不必要的DOM操作。结合`WebKit’s Nitro with JIT compilation`会显得更为有效。而且react还可以使用Flux担心数据了，可以抛弃传统的数据绑定方法，对app的性能和数据管理复杂性有很好的帮助

#### React的组件化和Mixins
组件和Mixins APIs可以很好的帮我们开发可以复用的组件，以及把公用模块抽取出来合成一个Mixin功能。这对于A/B Test有很好的帮助。views可以分成多个React Component，逻辑部分则抽象成多个Mixin来完成。并且，可以通过ES6的Class语法来组织Component的结构。可以把那种经常要改动的模块抽象成父子继承关系，新的feature放在子类中，那些经过测试的部分放在基类中，提高改动代码后对整体框架的稳定性。







