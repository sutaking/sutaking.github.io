---
layout: post
title: Explain event delegation
date: 2016-10-17 15:32:24.000000000 +09:00
---


### keywords
-	父节点与子节点
- 	事件响应
-  	事件冒泡

## 概述

当事件由一个元素触发，这个event会被分配到起对应的eventTarget上，然后任何监听该事件的listeners都会触发。

事件冒泡机制，让这些**Event listeners**的触发顺序，从child一直传到最上面的父节点，最大的父节点是Document。

事件代理event delegtaion则是通过事件冒泡机制，让事件处理可以通过绑定在父节点上，让其每一个子元素都可以响应该事件的方法。如：

```javascript
<ul onclick="alert(event.type + '!')">
	<li>One</li>
	<li>Two</li>
	<li>Three</li>
</ul>
```

上面的代码中，每一个`<li>`标签都可以响应click事件。

## what's the benefit?
### 减少事件绑定操作,代码逻辑简单了
如果我们时把event事件绑定在ui中的`<li>`元素上，当子元素频发的进行添加和删除操作，我们需要每次都调用一个`addEventListener`方法来添加事件处理函数。

使用event delegation, 比如为父节点添加一个click事件，当子节点被点击时，click事件会从子节点向上冒泡。父节点捕获到该事件之后，通过`e.target.nodeName`找到对应的子节点，作出相应的操作。

```javascript
// 获取父节点，并为它添加一个click事件
document.getElementById("parent-list").addEventListener("click",function(e) {
  // 检查事件源e.targe是否为Li
  if(e.target && e.target.nodeName.toUpperCase == "LI") {
    // 真正的处理过程在这里
    console.log("List item ",e.target.id.replace("post-")," was clicked!");
  }
});
```

### 内存减少
使用事件代理会降低用于事件监听的总内存消耗，由于事件绑定的个数减少了。

这方面，对于那些`long-lived applications`会比较明显。而且在事件绑定的操作中经常会忘记解绑，从而造成一些内存泄漏，事件代理在这种情况下，相对容易操作的多。


## jQuery中的delegate函数

使用方法

```javascript
$("#link-list").delegate("a", "click", function(){
  // "$(this)" is the node that was clicked
  console.log("you clicked a link!",$(this));
})
```

jQuery的delegate的方法需要三个参数，一个选择器，一个Event名称，和事件处理函数。




