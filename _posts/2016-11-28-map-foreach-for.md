---
layout: post
title: .map() vs .forEach() vs for() 如何选择？
date: 2016-11-28 18:56:24.000000000 +08:00
---

[访问原文地址](http://njfeng.com/2016/11/map-foreach-for/)

## .map() vs .forEach() vs for()

笔者说，自己基本没怎么用过`for()`来遍历，主要是用`.forEach()`。

但是总是会被很多朋友说，这些人认为`for()`的速度要比`.forEach()`快一点。（其实这根本没有根据，下面会讲）

速度当然是很重要的，但是我们也需要从其他方面考虑一下，特别是代码资源。

这里有一篇很棒的[文章](http://zsoltfabok.com/blog/2012/08/javascript-foreach/),很好的分析了`for()`遍历。它同时也针对for()`遍历和`.forEach()`做了比对[测试](https://jsperf.com/for-vs-foreach/37)。`for()`与`.forEach()`相比会消耗更多的内存。

这样，又回到了老问题，是用空间换速度，还是反之？

当然，都很重要。首先，这2个方面都不会成为你代码中的瓶颈问题。其次，那些小小的优化技巧也不会很好的平衡这2个问题，只会增加你的工作量而已。那我在来看下可读性、可控性、以及可维护性之间的对比呢。

让我们先来看个基本的sample

比如这个数组

```javascript
var arr = [1, 2, 3];
```

`.map()`:

```javascript
arr.map(fcuntion(i) {
	console.log(i);
})
```

43个字母

`.forEach()`:

```javascript
arr.forEach(function(i){
	console.log(i)
})
```

47个字母

`for()`

```javascript
for(var i=0,l=arr.lengrh;i<l; i++) {
	console.log(i);
}
```

70个字母

`.map()`和`.forEach()`明显要简短一些，并且他们的可读性更强，同时他们也创建了自己的`scope`，而`for()`在执行完遍历之后会把`i`和`l`这两个元素挂起来，这让我们需要手动增加一些代码去清除他们所占用的内存。

所以，这时候可以告诉你的朋友：

用`.forEach()`或者`.map()`。

## .map() vs .forEach()

那么接下来，我继续做分析，为什么更推荐用`.map()`，而不是`.forEach()`？

首先，`.map()`要比`.forEach()`执行速度更快。虽然我也说过执行速度不是我们需要考虑的主要因素，但是他们都比`for()`要更好用，那肯定要选更优化的一个。

第二，`.forEach()`的返回值并不是array。如果你想用函数式编程写个链式表达式来装个逼，`.map()`将会是你不二的选择。

来看下面这个例子：

```javascript
var arr = [1, 2, 3];

console.log(
	arr.map(function(i){
		return i+i;
	})
	//链式风格
	.sort()
);// [2,4,6]

console.log(
	arr.forEach(function(i){
		return i+i;
	})
	//接不起来，断了
	.sort()
);//TypeError: Cannot read property 'sort' of undefined
```

## 最后

根据上面的代码，大家应该了解到`.forEach()`跟`.map()`的局限。

最后，感谢大家耐心的阅读，排个序

`.map() > .forEach() > for()`



## 英文原文
[https://ryanpcmcquen.org/javascript/2015/10/25/map-vs-foreach-vs-for.html](https://ryanpcmcquen.org/javascript/2015/10/25/map-vs-foreach-vs-for.html)