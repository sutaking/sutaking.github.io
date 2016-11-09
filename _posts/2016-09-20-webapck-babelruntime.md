---
layout: post
title: webpack+babel-runtime编译代码后支持老版本浏览器
date: 2016-09-20 15:32:24.000000000 +09:00
---

同事弄来了一段代码，发现在chrome上跑的好好的，但是在别的一台也用webkit的浏览器上却无法完成初始化。看了一下，用了比较新的Promise和new Map()。

开始让他去找2个单独的实现去替换下，应该就可以了。

后来突然想起来，可以用babel-runtime来做compile，做一劳永逸的方案。

### 首先配置下npm的依赖和webpack环境，安装下babel的相关组件。

```javascript
npm install webpack --save-dev
npm install babel-core --save-dev
npm install babel-loader --save-dev
npm install babel-polyfill --save-dev
```

### npm script中配置webpack

```javascript
"scripts": {
    "build": "webpack --progress --color",
    "watch": "webpack --progress --color --watch"  
},
```

### 配置webpack.config.js,引入babel-loader，webpack的输出路径

```javascript
var webpack = require('webpack');
var path = require('path');

module.exports = {
  entry: [
  	//引入babel-runtime的依赖组件
  	'babel-polyfill',
  	'./src/main.js'],
  output: {
    path: __dirname + '/build',
    filename: 'omnitone.js',
    libraryTarget: 'umd'
  },
  module:{ 
  	//增加一个loader组件，加载babel
  	loaders:[{
  		'loader':'babel-loader',
  		exclude:[
  		    //在node_modules的文件不被babel理会
  		    path.resolve(__dirname,'node_modules'),
  		],
  		include:[
  		    //指定app这个文件里面的采用babel
  		    path.resolve(__dirname,'src'),
  		]
  	}]
  }
};
```

### 最后在目标代码中加入polyfill

```javascript
require("babel-polyfill");
//如果是ES6的代码
import "babel-polyfill";

```

执行`npm run build`执行输出js成功，发现已经没有promise了。
