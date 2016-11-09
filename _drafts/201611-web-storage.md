#Web Storage相关

##概述

DOM存储的机制是通过存储字符串类型的键/值对,来提供一种安全的存取方式.这个附加功能的目标是提供一个全面的,可以用来创建交互式应用程序的方法(包括那些高级功能,例如可以离线工作一段时间).

DOM存储很有用，因为在浏览器端没有好的方法来持久保存大量数据。浏览器cookie的能力有限，而且不支持组织持久数据，其他方法（如flash本地存储）需要外部插件支持。

-	readonly attribute unsigned long length;
- DOMString key(in unsigned long index);
- DOMString getItem(in DOMString key);
- void setItem(in DOMString key, in DOMString data);
- void removeItem(in DOMString key);
- void clear();

## sessionStorage（刷新后还在）

sessionStorage 是个全局对象，它维护着在页面会话(page session)期间有效的存储空间。只要浏览器开着，页面会话周期就会一直持续。当页面重新载入(reload)或者被恢复(restores)时，页面会话也是一直存在的。每在新标签或者新窗口中打开一个新页面，都会初始化一个新的会话。

自动保存一个文本域中的内容，如果浏览器被意外刷新，则恢复该文本域中的内容，所以不会丢失任何输入的数据。

````javascript
// 获取到我们要循环保存的文本域
 var field = document.getElementById("field");
 
 // 查看是否有一个自动保存的值
 // (只在浏览器被意外刷新时)
 if ( sessionStorage.getItem("autosave")) {
    // 恢复文本域中的内容
    field.value = sessionStorage.getItem("autosave");
 }
 
 // 每隔一秒检查文本域中的内容
 setInterval(function(){
    // 并将文本域的值保存到session storage对象中
    sessionStorage.setItem("autosave", field.value);
 }, 1000);
````

## localStorage(一直都在)
localStorage 属性允许你访问一个 local Storage 对象。localStorage 与 sessionStorage 相似。不同之处在于，存储在 localStorage 里面的数据没有过期时间（expiration time），而存储在 sessionStorage 里面的数据会在浏览器会话（browsing session）结束时被清除，即浏览器关闭时。

**当浏览器进入私人模式(private browsing mode，Google Chrome 上对应的应该是叫隐身模式)的时候，会创建一个新的、临时的、空的数据库，用以存储本地数据(local storage data)。当浏览器关闭时，里面的所有数据都将被丢弃。**

#### 测试本地存储是否已被填充
````javascript
if(!localStorage.getItem('bgcolor')) {
  populateStorage();
} else {
  setStyles();
}
````
Storage.getItem() 方法用来从存储中获取一个数据项。该例中，我们测试 bgcolor 数据项是否存在。如果不存在，执行 populateStorage() 来将存在的自定义值添加到存储中。如果有值存在，则执行 setStyles() 来使用存储的值更新页面的样式。

备注：你还可以使用 Storage.length 来测试存储对象是否为空。
#### 从存储中获取值
正如上面提到的，使用 Storage.getItem() 可以从存储中获取一个数据项。该方法接受数据项的键作为参数，并返回数据值。

````javascript
function setStyles() {
  var currentColor = localStorage.getItem('bgcolor');
  var currentFont = localStorage.getItem('font');
  var currentImage = localStorage.getItem('image');

  document.getElementById('bgcolor').value = currentColor;
  document.getElementById('font').value = currentFont;
  document.getElementById('image').value = currentImage;

  htmlElem.style.backgroundColor = '#' + currentColor;
  pElem.style.fontFamily = currentFont;
  imgElem.setAttribute('src', currentImage);
}
````
首先，前三行代码从本地中获取值。接着，将值显示到表单元素中，这样在重新加载页面时与自定义设置保持同步。最后，更新页面的样式和图片，这样重新加载页面后，你的自定义设置重新起作用了。

#### 在存储中设置值
Storage.setItem() 方法可被用来创建新数据项和更新已存在的值。该方法接受两个参数——要创建/修改的数据项的键，和对应的值。

````javascript
function populateStorage() {
  localStorage.setItem('bgcolor', document.getElementById('bgcolor').value);
  localStorage.setItem('font', document.getElementById('font').value);
  localStorage.setItem('image', document.getElementById('image').value);

  setStyles();
}
````

populateStorage() 方法在本地存储中设置三项数据 — 背景颜色、字体和图片路径。然后执行 setStyles() 方法来更新页面的样式。

同时，我们为每个表单元素绑定了一个 onchange 监听器，这样，一个表单值改变时，存储的数据和页面样式会更新。

````javascript
bgcolorForm.onchange = populateStorage;
fontForm.onchange = populateStorage;
imageForm.onchange = populateStorage;
````

#### 通过 StorageEvent 响应存储的变化
无论何时，Storage 对象发生变化时（即创建/更新/删除数据项时，重复设置相同的键值不会触发该事件，Storage.clear() 方法至多触发一次该事件），StorageEvent 事件会触发。在同一个页面内发生的改变不会起作用——在相同域名下的其他页面（如一个新标签或 iframe）发生的改变才会起作用。在其他域名下的页面不能访问相同的 Storage 对象。

````javascript
window.addEventListener('storage', function(e) {  
  document.querySelector('.my-key').textContent = e.key;
  document.querySelector('.my-old').textContent = e.oldValue;
  document.querySelector('.my-new').textContent = e.newValue;
  document.querySelector('.my-url').textContent = e.url;
  document.querySelector('.my-storage').textContent = e.storageArea;
});
````
这里，我们为 window 对象添加了一个事件监听器，在当前域名相关的 Storage 对象发生改变时该事件监听器会触发。正如你在上面看到的，此事件相关的事件对象有多个属性包含了有用的信息——改变的数据项的键，改变前的旧值，改变后的新值，改变的存储对象所在的文档的 URL，以及存储对象本身。

#### 删除数据记录
Web Storage 提供了一对简单的方法用于移除数据。我们没用在我们的 demo 中使用这些方法，但是添加到你自己的项目中很简单：

````javascript
//接受一个参数——你想要移除的数据项的键，然后会将对应的数据项从域名对应的存储对象中移除。
Storage.removeItem() 

//不接受参数，只是简单地清空域名对应的整个存储对象。
Storage.clear() 
````
## Cookie 和 localStorage 有啥差别
#### Cookie
Cookie 是小甜饼的意思。顾名思义，cookie 确实非常小，它的大小限制为4KB左右，是网景公司的前雇员 Lou Montulli 在1993年3月的发明。它的主要用途有保存登录信息，比如你登录某个网站市场可以看到“记住密码”，这通常就是通过在 Cookie 中存入一段辨别用户身份的数据来实现的。

####三者比较

|特性|Cookie|localStorage|sessionStorage
|---|---|---|---
|数据的生命期|一般由服务器生成，可设置失效时间。如果在浏览器端生成Cookie，默认是关闭浏览器后失效| 除非被清除，否则永久保存	|仅在当前会话下有效，关闭页面或浏览器后被清除
存放数据大小|4K左右|一般为5MB|
|与服务器端通信|每次都会携带在HTTP头中，如果使用cookie保存过多数据会带来性能问题|仅在客户端（即浏览器）中保存，不参与和服务器的通信||
|易用性|需要程序员自己封装，源生的Cookie接口不友好|源生接口可以接受，亦可再次封装来对Object和Array有更好的支持||

#### 应用场景
因为考虑到每个 HTTP 请求都会带着 Cookie 的信息，所以 Cookie 当然是能精简就精简啦，比较常用的一个应用场景就是判断用户是否登录。针对登录过的用户，服务器端会在他登录时往 Cookie 中插入一段加密过的唯一辨识单一用户的辨识码，下次只要读取这个值就可以判断当前用户是否登录啦。曾经还使用 Cookie 来保存用户在电商网站的购物车信息，如今有了 localStorage，似乎在这个方面也可以给 Cookie 放个假了~

而另一方面 localStorage 接替了 Cookie 管理购物车的工作，同时也能胜任其他一些工作。比如HTML5游戏通常会产生一些本地数据，localStorage 也是非常适用的。如果遇到一些内容特别多的表单，为了优化用户体验，我们可能要把表单页面拆分成多个子页面，然后按步骤引导用户填写。这时候 sessionStorage 的作用就发挥出来了。

####安全性的考虑
需要注意的是，不是什么数据都适合放在 Cookie、localStorage 和 sessionStorage 中的。使用它们的时候，需要时刻注意是否有代码存在 XSS 注入的风险。因为只要打开控制台，你就随意修改它们的值，也就是说如果你的网站中有 XSS 的风险，它们就能对你的 localStorage 肆意妄为。所以千万不要用它们存储你系统中的敏感数据。

## 浏览器中不同Tab页之间的通信
-	Use Cookie
- LocalStorage
- 通过window.open(...)方式打开的tab，可以操作Windows对象来通信
- postMessage API

````javascript
//In w1
var w2 = window.open("abc.do");
window.addEventListener("message", function(event){
    console.log(event.data);
});

//In w2(abc.do)
window.opener.postMessage("Hi! I'm w2", "*");
````
