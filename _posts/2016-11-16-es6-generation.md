---
layout: post
title: ES6中的异步编程：Generators函数（一）
date: 2016-11-16 16:00:24.000000000 +08:00
---

[访问原文地址](http://njfeng.com/2016/11/es6-generation/)

对ES6的generators的介绍分为3个部分

-	第一部分base介绍及使用
- 第二部分基于generators来实现es7的async函数
- 第三部分基于async的Observable数据流开发

## 概述

Generator函数是协程在ES6的实现，用来做异步流程的封装，最大特点就是可以交出函数的执行权（即暂停执行）。十分的奇葩，光看语法，简直认不出这也是JavaScript了。由于可以使用yield语句来暂停异步操作，这让generators异步编程的代码，很像同步数据流方法一样。因为从语法角度来看，generators函数是一个状态机，封装了多个内部状态，通过iterator来分步调用。

## 基本语法

### 2个关键字搞定generators语法

-	function与函数名直接的星号：*
- 函数体内yield语句

```javascript
function* testGenerator() {
	yield 'first yield';
	yield 'second yield';
	return 'last';
}

var gen = testGenerator();

console.log(gen.next().value);// first yield 
// { value: 'first yield', done: false }
console.log(gen.next().value);// second yield
// { value: 'second yield', done: false }
console.log(gen.next().value);// last 
// { value: 'last', done: true }

console.log(gen.next().value);// undefined

```

### for...of遍历

for...of循环可以自动遍历generators函数的iterator对象，且不再需要调用next方法。for...of需要检查iterator对象的done属性，如果为true，则结束循环，因此return语句不能被遍历到

```javascript
for (let i of testGenerator) {
	console.log(i);
}
// first yield
// second yield
```

### next方法的参数

yield句本身没有返回值，或者说总是返回undefined。next方法可以带一个参数，该参数就会被当作上一个yield语句的返回值。

```javascript
function *gen(){
  let arr = [];
  while(true){
    arr.push(yield arr);
  }
}

var name = gen();

console.log(name.next('first').value);//[]
console.log(name.next('second').value);//["second"]
console.log(name.next('thrid').value);//["second","thrid"]
```

需要注意的是，第一次执行next设置参数没有效果。

## generators实践

### 实现Fibonacci数列

递归实现：

```javascript
function* fib (n, current = 0, next = 1) {
  if (n === 0) {
    return 0;
  }

  yield current;
  yield* fib(n - 1, next, current + next);
}

for (let n of fibonacci()) {
  if (n > 1000) break;
  console.log(n);
}
```

注：如果存储计算结果再过运算，这样的实现比递归方法效率高3倍

```javascript
function* fibonacci() {
  let [prev, curr] = [0, 1];
  for (;;) {
    [prev, curr] = [curr, prev + curr];
    yield curr;
  }
}

for (let n of fibonacci()) {
  if (n > 1000) break;
  console.log(n);
}
```

### 利用for...of循环，遍历任意对象（object）的方法 

原生的JavaScript对象没有遍历接口，无法使用for...of循环，通过Generator函数为它加上这个接口，就可以用了。

```javascript
function* objectEntries(obj) {
  let propKeys = Reflect.ownKeys(obj);

  for (let propKey of propKeys) {
    yield [propKey, obj[propKey]];
  }
}

let jane = { first: 'Jane', last: 'Doe' };

for (let [key, value] of objectEntries(jane)) {
  console.log(`${key}: ${value}`);
}
// first: Jane
// last: Doe
```

### ES6中iterator遍历接口汇总

-	for...of循环
- 扩展运算符（...）
- 解构赋值
- Array.from方法内部调用的

它们都可以将Generator函数返回的Iterator对象，作为参数来使用。

```javascript
function* numbers () {
  yield 1
  yield 2
  return 3
}

// 扩展运算符
[...numbers()] // [1, 2]

// Array.from 方法
Array.from(numbers()) // [1, 2]

// 解构赋值
let [x, y] = numbers();
x // 1
y // 2

// for...of 循环
for (let n of numbers()) {
  console.log(n)
}
// 1
// 2
```

### generators与同步

generators一个特点就是代码看上去非常像同步编程的效果

```javascript
function* test() {
    yield( "1st" );
    yield( "2nd" );
    yield( "3rd" );
    yield( "4th" );
}
var iterator = test();

console.log( "== Start of Line ==" );
console.log( iterator.next().value );
console.log( iterator.next().value );
for ( var line of iterator ) {
    console.log( line );
}
console.log( "== End of Line ==" );

```

看下输出，浓浓的同步执行风格。

```javascript
== Start of Line ==
1st
2nd
3rd
4th
== End of Line ==
```


## callback、Promises、Generators比较

举例说一个场景，查询一篇新闻文章的作者信息，流程是：请求最新文章列表->请求某文章相关id->作者id信息

### callback实现

```javascript
getArticleList(function(articles){
    getArticle(articles[0].id, function(article){
        getAuthor(article.authorId, function(author){
            alert(author.email);
        })
    })
})

function getAuthor(id, callback){
    $.ajax(url,{
        author: id
    }).done(function(result){
        callback(result);
    })
}

function getArticle(id, callback){
    $.ajax(url,{
        id: id
    }).done(function(result){
        callback(result);
    })
}

function getArticleList(callback){
    $.ajax(url)
    .done(function(result){
        callback(result);
    });
}
```

### 用Promise来做

```javascript
getArticleList()
.then(articles => getArticle(articles[0].id))
.then(article => getAuthor(article.authorId))
.then(author => {
    alert(author.email);
});

function getAuthor(id){
    return new Promise(function(resolve, reject){
        $.ajax({
            url: id+'author.json',
            success: function(data) {
              resolve(data);
          }
        })
    });
}

function getArticle(id){
    return new Promise(function(resolve, reject){
        $.ajax({
            url: id+'.json',
            success: function(data) {
              resolve(data);
          }
        })
    });
}

function getArticleList(){
    return new Promise(function(resolve, reject){
       $.ajax({
           url: 'all.json',
           success: function(data) {
             resolve(data);
         }
       }) 
    });
}
```

### Gererator来实现

```javascript
function* run(){
  var articles = yield getArticleList();
  var article = yield getArticle(articles[0].id);
  var author = yield getAuthor(article.authorId);
  alert(author.email);  
}

var gen = run();
gen.next().value.then(function(r1){
  gen.next(r1).value.then(function(r2){
      gen.next(r2).value.then(function(r3){
        gen.next(r3);
        console.log("done");
      })
  })
});
```




## 参考
- 	[\[Javascript\] Promise, generator, async與ES6](http://huli.logdown.com/posts/292655-javascript-promise-generator-async-es6)
- [Generator 函数](http://es6.ruanyifeng.com/#docs/generator)
