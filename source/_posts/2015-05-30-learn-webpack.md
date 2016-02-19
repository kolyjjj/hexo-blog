title: "学习webpack"
date: 2015-05-30 11:41:33
categories: 
- Tech
tags: 
- FrontEnd
---

随着后端[NodeJS](https://nodejs.org/)的流行，模块化这个词已经在JavaScript的世界里占据了一席之地。NodeJS具有自己的[模块管理机制](https://nodejs.org/api/modules.html)。通过require，可以将具体的模块引入到文件中使用。但是这仅限于NodeJS运行环境。那么在编写一些前端代码时，就不能用这种require的模块管理，因为代码需要在前端浏览器运行，但是浏览器并不知道require这个函数。<!--more-->

除了希望能够在前端代码编写的时候，能够使用模块化管理之外，我们还需要对js文件的打包，压缩等。什么工具可以帮助我们呢？我选择了[webpack](http://webpack.github.io/)。

有一些问题：

* webpack是什么？
* webpack能够做什么？
* 怎么使用webpack完成模块管理？
* 怎么使用webpack来打包？

### 1，webpack是什么？
简单来说，webpack是一个module bundler。

{% blockquote @webpack  http://webpack.github.io/%}
webpack is a module bundler
This means webpack takes modules with dependencies, and emits static assets representing those modules.
{% endblockquote %}

两个概念：module(模块) 和 bundler(打包机)。
####  module
module，简单来说就是一块代码。这里可以看看[CommonJS](http://www.commonjs.org/)对module的规定。但是在此之前，我们先看看什么是CommonJS。

{% blockquote @CommonJS  http://www.commonjs.org/ %}
JavaScript is a powerful object oriented language with some of the fastest dynamic language interpreters around. The official JavaScript specification defines APIs for some objects that are useful for building browser-based applications. However, the spec does not define a standard library that is useful for building a broader range of applications.
The CommonJS API will fill that gap by defining APIs that handle many common application needs, ultimately providing a standard library as rich as those of Python, Ruby and Java.
{% endblockquote %}

如果上面的文字太多了，你不想看，没关系。翻译一下，就是：
CommonJS是一套JavaScript的API规范(specification)。它定义了一些API，规定了这些API的具体接口，使用方式等。比如参数，返回值，使用场景。就像Java, Ruby和Python里面的各种库。

CommonJS定义了[module](http://wiki.commonjs.org/wiki/Modules/1.1.1)。NodeJS的module模块实现了CommonJS定义的这个规范。在NodeJS里面，定义模块和使用模块可以这么写：

```
// add.js
module.exports = function(x, y) {
  return x + y;
}
// main.js
var add = require('./add.js');
console.log(add(2, 3));
```
两点：
* 通过require来引入模块
* 把一个对象赋给module.exports，以将这个对象“导出”。在使用require的时候，实际引用的就是这个对象。

现在，我们大概知道什么是一个模块了。就是a chunk of codes。使用的时候需要通过require函数引入。

####  bundler
bundler就是打包机。bundle是捆的意思，bundler就是把一堆东西捆在一块儿的工具。
在这里的意思就是，比如有多个javascript文件，使用bundler可以把多个javascript文件“捆”成一个文件。

#### module bundler
webpack是一个module bundler。把module和bundler连起来，是什么意思呢？
* webpack会把多个文件“捆”成一个或者几个文件；
* module是指webpack“认识”文件中的模块及模块require。在“捆”的时候会引入对应的模块。

来看个例子，在NodeJS环境里，我们可以：

```
// add.js
module.exports = function(x, y) {
  return x + y;
}

//add3.js
var add = require('./add.js');
module.exports = function(x, y, z){
  return add(add(x, y), z);
};

// main.js
var add = require('./add3.js');
console.log(add(2, 3, 4));
```
在使用webpack处理这些文件的时候，webpack会找到对应的代码，并将这些代码拼起来，保证能在浏览器环境上运行。比如我们使用webpack来打包上面三个文件，生成一个js文件：

```
webpack main.js bundle.js
```
webpack是一个控制台的命令，[这里](http://webpack.github.io/docs/cli.html)有安装文档。
main.js是一个entry。就是入口文件。webpack会根据这个文件来逐步查找对应的modules。比如main.js里面有一个require函数，那么webpack就会根据这个require函数去寻找对应的javascritp代码。然后将这些代码给bundle进来。
这里是最终bundle出的代码：
```
/******/ (function(modules) { // webpackBootstrap
/******/ 	// The module cache
/******/ 	var installedModules = {};

/******/ 	// The require function
/******/ 	function __webpack_require__(moduleId) {

/******/ 		// Check if module is in cache
/******/ 		if(installedModules[moduleId])
/******/ 			return installedModules[moduleId].exports;

/******/ 		// Create a new module (and put it into the cache)
/******/ 		var module = installedModules[moduleId] = {
/******/ 			exports: {},
/******/ 			id: moduleId,
/******/ 			loaded: false
/******/ 		};

/******/ 		// Execute the module function
/******/ 		modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);

/******/ 		// Flag the module as loaded
/******/ 		module.loaded = true;

/******/ 		// Return the exports of the module
/******/ 		return module.exports;
/******/ 	}


/******/ 	// expose the modules object (__webpack_modules__)
/******/ 	__webpack_require__.m = modules;

/******/ 	// expose the module cache
/******/ 	__webpack_require__.c = installedModules;

/******/ 	// __webpack_public_path__
/******/ 	__webpack_require__.p = "";

/******/ 	// Load entry module and return exports
/******/ 	return __webpack_require__(0);
/******/ })
/************************************************************************/
/******/ ([
/* 0 */
/***/ function(module, exports, __webpack_require__) {

	var add = __webpack_require__(1);

	console.log(add(2, 3, 4));


/***/ },
/* 1 */
/***/ function(module, exports, __webpack_require__) {

	var add = __webpack_require__(2);

	module.exports = function(x, y, z){
	  return add(add(x, y), z);
	};


/***/ },
/* 2 */
/***/ function(module, exports, __webpack_require__) {

	module.exports = function(x, y) {
	  return x + y;
	}


/***/ }
/******/ ]);
```
**所以，module bundler的意思就是，第一找到对应的Module，第二打包。**

### 2，webpack能够做什么？
通过上一节，这个问题就知道答案了：
* 分析module依赖
* 打包


### 3，怎么使用webpack完成模块管理？
webpack有两种使用方式：
* [cli](http://webpack.github.io/docs/cli.html)，就是一个控制台命令
* [node API](http://webpack.github.io/docs/node.js-api.html)

cli的安装：
```
npm install webpack -g
```
使用：
```
webpack main.js bundle.js
```

Node API的使用：
```
// webpack.js
var webpack = require('webpack');

webpack({
 context: "./",
 entry: "./main",
 output: {
     path: "./",
     filename: "bundle.js"
 }
}, function(err, stats) {
});

// command line
node webpack.js
```

cli和Node API都需要一个configuration。cli默认寻找的是一个`webpack.config.js`。Node API是传入webpack()函数的对象。
最后，webpack并不是一个模块管理的工具，所以并不能用它来完成模块管理。

### 4，怎么使用webpack来打包？
第三节已经包含了打包。

### 5，使用require来引入css

```
// webpack.config.js
module.exports = {
    entry: "./entry.js",
    output: {
        path: __dirname,
        filename: "bundle.js"
    },
    module: {
        loaders: [
            { test: /\.css$/, loader: "style!css" }
        ]
    }
};
```
使用webpack的Loader，可以对文件进行处理：

{% blockquote @Webpack  http://webpack.github.io/docs/using-loaders.html %}
Loaders are transformations that are applied on a resource file of your app. They are functions (running in node.js) that take the source of a resource file as the parameter and return the new source.
{% endblockquote %}

`style!css`使用了两个loader: style和css.

{% blockquote @Webpack  http://webpack.github.io/docs/list-of-loaders.html#styling %}
style | Add exports of a module as style to DOM
css | Loads css file with resolved imports and returns css code
{% endblockquote %}

这样，你就可以在代码里面使用：
```
require('./one.css')
```
引入css文件了。这样css文件也被“模块化管理”了。

------
总的来说，使用webpack能够对module文件进行打包，包括js和css。它提供的loader还可以支持更多语言，比如less， coffeescript。
