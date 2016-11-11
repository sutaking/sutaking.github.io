---
layout: post
title: 基于ES6的代码重构要点
date: 2016-11-09 19:52:24.000000000 +09:00
---

## Why

- 对原本代码逻辑进行提炼，提高代码的阅读性。
- 增强代码的Object Oriented和Functional Programming特性。
- 再逻辑还不算太复杂的时候，早点修订一些Coding规范。

## 重构要点

1. Construct HTML using template strings.
2. Eliminate if/else blocks with hash maps.
3. Collapse multiple arguments with a config object.
4. Pre-bind arguments to make point-free functions.
5. Break complex conditionals with intention revealing .variables.
6. Inline IIFE for complex initialization.
7. Use hoisting/composed-method to focus on important code.

### 1.使用ES6的template string来替换不好用的html文本
	
```javascript
var html = '<div class="my-car-class-contain">' +
			'<div class="my-car-class-title">' + title + '</div>' + 
			'<img src="' + url + '">' +
			'</div>';

//使用template直接就可以像些angular的模板一样在js代码中插入html
let tmp_html = `<div class="my-car-class-contain">
				<div class="my-car-class-title">${title}</div>
				<img src="${url}" alt="">
				</div>`;

```

### 2.用hash-map来替换复杂的if/else判断条件

```javascript
function getSomething(type) {
	if (type === 'boy') {
		return '/img/bog.jpg'
	} else if (type === 'man') {
		return '/img/man.jpg'
	} else if (type === 'girl') {
		return '/img/girl.jpg'
	} else (type === 'woman') {
		return '/img/woman.jpg'
	}
}

//hash-map
function getSomething(type) {
	var typeMap = {
		boy: '/img/bog.jpg',
		man: '/img/man.jpg',
		girl: '/img/girl.jpg',
		woman: '/img/woman.jpg',
	}
	
	return typeMap[type] || '/img/default.jpg';
}

//当condition或者type条件很复杂的时候，可以把他们提取出来，以便维护
import typeMap from './conditions';

```

### 3.减少函数入参，用Object来代替。

```javascript
function formatNumber(value, decimals, asPercent, prefix, suffix) {
	var formattedNumber = value;
	
	//do something
	
	return formattedNumber;
}
formatNumber(10,2,false,'','%');

//利用ES6默认值特性，以及对象定义参数
function formatNumber({value=0, decimals=0, asPercent=false, prefix=‘’, suffix=‘’}){}
var num = 20;
formatNumber({value: num, suffix:'%'});
```

### 4.使用bind方法，将一个函数分离成多个更好理解的函数

```javascript
function doOperation(name='default', args=[]) {
	// 这是一个通过name来决定操作方法的函数
}
function findSome() {
	return Promise.resolve('LS460h');
}
function getCarList() {
	return Promise.resolve({size:4800, seats:5, output:6.0});
}
findSome().then(args => {
	doOperation('searchCar', args);
});
getCarList().then(args => {
	doOperation('buyLexus', args);
});

//修改过程
//先通过bind函数，将操作方法从参数导入的方式分离成2个函数
var play = doOperation.bind(null, 'searchCar');
var buy = doOperation.bind(null, 'buyLexus');

findSome().then(play);
getCarList().then(buy);
```

### 5.使用array的some方法来重构条件判断语句

```javascript
import _ from 'lodash';
var age,
	gamer = [],
	adNetworks = [],
	specialAchievements = [];

function showGame() {
	/* do something */
}

if((age > 0 && age <0) || 
	gamer.isFirstTime ||
	_.some(adNetworks, function(n) {
		return gamer.network === n;
	}) ||
	_.some(specialAchievements, function(ach) {
		return gamer.specialAchievement = ach;
	})) {
	showGame();	
}
```

把conditions提炼成Array,并使用箭头函数

```javascript
var conditions = [
	() => {
		return age > 0 && age <0;
	},
	() => gamer.isFirstTime,
	() => {
		return _.some(adNetworks, n => gamer.network === n);
	},
	() => {
		return _.some(specialAchievements, ach => gamer.specialAchievement === ach);
	},
	//加入新的条件，就变得很轻松
	() => specialAchievements.length > 0
];

let matches = _.some(conditions, c=>c());
if(matches) {
	showGame();	
}
```

### 6.使用IIFE(Immediately-Invoked Function Expression)做复杂初始化

```javascript
var settings = readSettings();
var apiUrl = `https://${settings.apiDomain}:${settings.apiPort}/api`;

var defaultConfig = {
	apiUrl:apiUrl,
	timezone: settings.timezone,
	account: settings.account,
	theme: settings.theme
}

// IIFE 
var defaultConfig = (() => {
	var settings = readSettings();
	var apiUrl = `https://${settings.apiDomain}:${settings.apiPort}/api`;

	var defaultConfig = {
		apiUrl:apiUrl,
		timezone: settings.timezone,
		account: settings.account,
		theme: settings.theme
	}
})();
```

### 7.把回调的逻辑抽取成函数，隐藏逻辑，让代码语义更清晰。
 
```javascript
_fetchData(url)
	.then(data => {
		//do something
	})
	.then(transactions => {
		//do something
	})

//after
import {processData, renderHtml} from './doSomething'
_fetchData(url)
	.then(processData)
	.then(renderHtml);

```

