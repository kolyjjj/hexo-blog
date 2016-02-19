title: "ES6 generator探索"
date: 2015-07-25 12:02:05
categories: 
- Tech
tags: 
- javascript
---


Javascript语言在经历了这些年的发展之后，在2015年终于推出了新的语法规范。其中，有一个叫做generator，由于在koajs中大量出现，于是我想探索一番：<!--more-->

### \# 怎么写generator?
首先来看看Javascript中的函数：
```
function iAmFunction() {
  console.log('hi, I am a function.');
}
```
再来看看generator:
```
function *iAmGenerator() {
  console.log('I am a generator');
}
```
在关键字`function`后面加上'\*'即是声明一个generator。'\*'可以靠近函数名，也可以靠近function关键字，比如：
```
function* iAmGenerator() {
  console.log('I am a generator');
}
```
当然，也可以这样写：
```
var iAmGenerator = function* () {
  console.log('I am a generator');
};
```

那么，问题来了，调用`iAmFunction`会再控制台输出'hi, I am a function.'。但是调用`iAmGenerator`却没有输出'I am a generator'。怎么回事呢？如果不能调用，那正确的打开方式是怎么样的呢？

### \# generator是怎么玩的?
ES6除了有generator之外，还有`yield`，这两个要配合使用：

```
function *gen(x) {
  var y = yield (x + 1);
  var z = yield (y + 2);
  console.log('console logging', z);
  return z;
}
```
调用：
```
var it = gen(1); // 可以接受参数
it.next();       // 返回Object {value: 2, done: false}
it.next(3);      // 返回Object {value: 5, done: false}
it.next(20);     // 返回Object {value: 20, done: true}，同时打印出'console logging, 20'
```
1，首先调用`gen(1)`，此时将定义中的`x`换成`1`，并返回一个iterator，这个迭代器会负责对yield序列的迭代。
2，调用`it.next()`，此时的返回值是跟在跟在yield之后的表达式的值，即`(x + 1)`的值，由于上面x已被赋为1，所以此时的值是2。`next`函数的返回值是一个对象，其中`value`表示yield之后的表达式的值（称之为yield的输入），而`done`表示迭代是否结束。`false`表示没有结束。
3，再次调用`it.next(3)`，传入的参数3便是上一次yield的返回值（yield的输出），此时y的值变成了3。而`next`函数需要返回当前yield的输入，即(y + 2) => 3 + 2，即5.
调用`it.next(20)`，传入的参数变成上一次yield的返回值，即z为20，打印出log信息。而由于没有当前的yield，所以返回return之后的表达式的值，同时，done变为`true`，表示迭代结束。如果函数没有返回值，则返回的对象的value属性是`undefined`。

### \# 怎么使用generator?
首先，我们来看看generator有什么特点：
* 调用generator函数返回的是迭代器，函数不会执行。
* 使用yield，可以分段执行函数体。
* 通过yield，可以实现数据的传入和传出

这里先介绍一下ES6里面的for循环，for..of..：
```
for (one of [1,2,3]){
  console.log(one);
}
```
*解耦数据和循环*：
不用generator:
```

function increase(array) {
  var result = [];
  for (o of array) {
    result.push(o + 1);
  }
  return result;
}

increase([1,2,3]); // [2,3,4]

```
使用generator:
```
function add(a, b) {
  return a + b;
}

function addOne(input) {   //数据生成
  return add(input, 1);
}

function* gen(array) {     //初始数据，构造yield输出
  for (var i = 0; i < array.length; i++) {
    yield (addOne(array[i]));
  }
}

function increase(array) {  //数据组装
  var result = [];
  for (value of gen(array)) {
    result.push(value);
  }
  return result;
}

increase([1,2,3]); // [2,3,4]
```
将一个函数分成几个函数，将数据的生成和循环分离。PS，可以尝试用回调的方式来重构。

*生成‘无限数组’*：
```
function *infiniteArray(start) {
  yield start;
  while (true) {
    yield (start = start + 1);
  }
}

var it = infiniteArray(0);
console.log(it.next().value);  // 0
console.log(it.next().value);  // 1
console.log(it.next().value);  // 2
```
*异步*
promise:

```
function retrieveData() {
  return new Promise(function(resolve, reject){
      setTimeout(function(){
          resolve('Hello Promise');
        }, 3000);
  });
}

function test(prefix) {
  retrieveData().then(function(data) {
      console.log(prefix + data);
  })
}

test('wow,'); // print wow,Hello Promise after 3 seconds

```
generator:
```
function retrieveData() {
  return new Promise(function(resolve, reject){
      setTimeout(function(){
          resolve('Hello Promise');
        }, 3000);
  });
}

function *gen(prefix) {
  var data = yield retrieveData();
  return (prefix + data);
}

function test(prefix) {
  var it = gen(prefix);
  it.next().value.then(function(data){
    console.log(it.next(data).value);
  })
}

test('wow,'); // print wow,Hello Promise after 3 seconds

```
写到这里，好好想想，generator和yield带来的是什么，上面的代码好像都是然并卵（然而这一切并没有什么卵用）。**yield表示一种行为，它暂停执行，将函数内部的状态传递出去，然后接收外面的状态。这很像是回调函数，用参数传入内部状态，进行处理之后返回处理之后的结果。** 只不过一个的执行者是回调函数本身，另外一个的执行者是generator的调用者。换句话说，用generator，谁用谁知道（返回值和传入值），而用回调，则只是回调函数自己知道。感觉generator比回调函数要灵活，当然还可以更易读一些。

-------------
参考资料：
[1] [http://davidwalsh.name/es6-generators](http://davidwalsh.name/es6-generators)
[2] [ES6 In Depth: Generators](https://hacks.mozilla.org/2015/05/es6-in-depth-generators/)
[3] [http://www.ruanyifeng.com/blog/2015/04/generator.html](http://www.ruanyifeng.com/blog/2015/04/generator.html)
