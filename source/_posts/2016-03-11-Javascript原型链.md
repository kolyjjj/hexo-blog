title: Javascript原型链
categories:
  - Tech
tags:
  - Javascript
date: 2016-03-11 22:15:09
---


Javascript是一门通过原型链来实现继承的面向对象的动态语言。其原型链机制本身并不复杂，但是理解起来有些绕。本文试图理清原型链的相关知识，并尝试归并以便于理解记忆。<!-- more -->
本文的所有代码均是在chrome的console中执行。
## 从一个简单的例子开始
首先我们使用object literal来创建一个对象：
```javascript
var koly = {name:'koly'};
```
那么，什么叫object literal呢？先来看看literal的定义：

>literal: taking words in their usual or most basic sense without metaphor or exaggeration: dreadful in its literal sense, full of dread. 

literal就是指最简单最基本的语言表达，没有花哨，没有隐喻，没有夸张。所以object literal就是说最简单的没有花哨的创建object的形式。除了object literal外还有什么string literal，array literal等，这里就不描述了。
回到我们刚刚创建的对象`koly`，我们来执行几条代码：
```javascript
koly.prototype
//undefined
koly.__proto__
//Object {}
```
上面注释的部分表示chrome console里面的输出。`koly.prototype`的结果是`undefined`，那么`undefined`是什么意思呢？看看MDN上的定义，既然提到了`undefined`，那就顺便对比以下`null`好了：

>undefined: The global undefined property represents the primitive value undefined. It is one of JavaScript’s primitive types.
>null: The value null is a JavaScript literal representing null or an "empty" value, i.e. no object value is present. It is one of JavaScript's primitive values.
>--- from [MDN]((https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/undefined)

相同之处，两个都是primitive values。不同之处undefined是global property，而null是Javascript literal。关于两者的区别，对于没有声明的对象，其值为undefined;一个对象的值为null则表示该值已被声明，且被赋值为null，koly.one是undefined，而执行`koly.one=null`，才能将koly.one赋值为null。所以null表示的其实是一个值，是一个基本的literal。跟‘one’这个string literal是一样的。
说完了undefined和null，我们回到`prototype`和`__proto__`，前者输出了undefined，表示koly这个对象上并没有prototype这个属性。后者输出了一个非undefined，表示koly上面是有这个东西的。但是我们在定义koly这个对象的时候，并没有添加`__proto__`这个属性啊，它是什么呢？

>The `__proto__` property of Object.prototype is an accessor property (a getter function and a setter function) that exposes the internal [[Prototype]] (either an object or null) of the object through which it is accessed.
>--- from [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)

上面的大意就是说通过`__proto__`可以访问对象内部的`[[Prototype]]`，这下好了，`[[Prototype]]`又是什么呢？

>Following the ECMAScript standard, the notation someObject.[[Prototype]] is used to designate the prototype of someObject. This is equivalent to the JavaScript property __proto__ (now deprecated). Since ECMAScript 6, the [[Prototype]] is accessed using the accessors Object.getPrototypeOf() and Object.setPrototypeOf().
>--- from [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)

上面的意思就是`[[Prototype]]`是ECMAScript定义的东西，它跟之前的`__proto__`是一样的。`__proto__`被deprecated了，现在提倡用`[[Prototype]]`。那么，`[[Prototype]]`就代表了对象的prototype。
综合以上的信息，总结来说就是，**通过访问一个对象的`__proto__`可以访问这个对象的prototype。**那么，`koly.__proto__`就是koly这个对象的原型。
那么，对象的原型(prototype)是什么？

>Each object has an internal link to another object called its prototype. That prototype object has a prototype of its own, and so on until an object is reached with null as its prototype. null, by definition, has no prototype, and acts as the final link in this prototype chain.
>--- from [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)

说，每一个对象内部都有一个指向另一个对象的链接，这个链接就叫这个对象的prototype。这个链接也是一个对象，也有自己的prototype，这个prototype指向另一个对象，这个对象也有自己的prototype，指向另一个对象。对象及对象间的指向就叫做原型链，这个链的终点是null，null没有prototype。
以刚刚的koly为例子：
* koly这个对象的内部有一个prototype，我们可以使用`__proto__`来访问
* koly.prototype指向另一个对象，这个对象也有自己的prototype，可以使用`__proto__`来访问

那么koly.prototype指向的对象是谁呢？

>All objects in JavaScript are descended from Object; all objects inherit methods and properties from Object.prototype, although they may be overridden (except an Object with a null prototype, i.e. Object.create(null)).
>--- from [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/prototype) 

上面说所有的对象都由Object派生而来，所有对象都从Object.prototype继承方法。koly是一个对象，该对象是从Object派生而来，继承了Object.prototype的方法。我们来看看`koly.__proto__`跟`Object.prototype`是什么关系。
```javascript
koly.__proto__ === Object.prototype
// true
```
这里的结果true代表了什么呢？
我们得先搞清楚`===`意味着什么。Javascript里面对于Object，Array等类型是有引用存在的，比如`var koly = {name:'koly'}`中，koly就是一个引用，这个引用指向了`{name:'koly'}`，这个引用也有自己的值（可能是一个地址之类的）。当我们在evaluate koly这个引用的时候，得到的结果是`{name:'koly'}`这个对象，而不是koly本身的值。`===`比较的是引用本身的值，而不是引用指向的那个对象。
```javascript
var one = {name:'one'};
var anotherOne = {name:'one'};
one === anotherOne;
// false
anotherOne = one; // 这里将one本身的值赋给了anotherOne，可以想象为将one的地址赋给了anotherOne
one === anotherOne;
// true
```
所以`===`应用于两个对象，其返回为true的情况表示两个对象的引用其值一样。考虑到上面的`koly.__proto__ === Object.prototype`，那么我们可以想象一定有一步`koly.__proto__ = Object.prototype`。
那么`Object.prototype`是什么呢？Javascript里面全都是对象，那么它是Object的`__proto__`吗？
先回答第二个问题：
```javascript
Object.prototype === Object.__proto__;
// false
```
第一个问题，我们首先看看Object是什么？

> The Object constructor creates an object wrapper.
>--- from [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)

Object是一个"constructor"，那么"constructor"又是什么？举个例子：
```javascript
// This is a generic default constructor class Default
function Default() {
}
```
constructor就是一个函数。那么Object也是一个函数，通过这个函数可以创建对象，比如：
```javascript
// Called as a constructor
new Object([value])
```
当对一个函数使用`new`关键字的时候，我们就称这个函数为`constructor`，`new`出来的东西称为对象。
Javascript里面几乎所有东西都是对象，所以constructor也是对象，这个对象自然也可以有自己的属性，那么`prototype`就是Object这个constructor的一个属性。同时Object还有另外一个属性`[[Prototype]]`，这个属性可以通过`__proto__`在chrome里面访问到。

到这里，都有点晕了。我们试着来理一下：
* Javascript里面几乎所有东西都是对象
* 每一个对象都有一个`[[Prototype]]`，在chrome里面可以使用`__proto__`来访问
* constructor或者说函数都是对象，有`[[Prototype]]`，同时也有另外一个属性`prototype`
* `koly.__proto__`指向了`Object.prototype`
* `Object`是一个constructor，同时也是一个对象，就是说`Object`本身也有`__proto__`
* constructor是一个函数，只是在使用的时候不是直接调用，而是使用`new`，其结果是创建一个对象
* null没有`[[Prototype]]`

## 原型链
有了对上面这些概念的理解，我们再来看看由`koly`引出的原型链：
```javascript
koly.__proto__ === Object.prototype;
// true
Object.prototype.__proto__ === null
// true
```
上面就是`koly`的原型链，比较简单。我们再来看看`Object`的原型链：
````javascript
Object.__proto__ === Function.prototype;
// true
Function.prototype.__proto__ === Object.prototype;
// true
Object.prototype.__proto__ === null
//true
```
上面的`Object`和`Function`都是constructor，就是函数。**原型链的链条是由`__proto__`来实现的**。在跟原型链的时候只需聚焦`__proto__`就行了，constructor的`prototype`只是一个会出现在原型链条里面的特殊对象。
constructor的prototype会在使用constructor创建对象的时候赋给该对象的`__proto__`。由于koly是通过`Object`这个constructor创建的，所以`koly.__proto__`也就指向了`Object.prototype`。
```javascript
function Animal(){};
var aAnimal = new Animal();
aAnimal.__proto__ === Animal.prototype;
// true
Animal.prototype.__proto__ === Object.prototype;
Object.prototype.__proto__ === null;
```

## 原型链有什么用？
先来看一段文字：

>  When trying to access a property of an object, the property will not only be sought on the object but on the prototype of the object, the prototype of the prototype, and so on until either a property with a matching name is found or the end of the prototype chain is reached.
> --- from [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)

就是说在访问一个对象的属性的时候，会首先在对象上找，找不到就去对象的`[[Prototype]]`上找，找不着就去`[[Prototype]]`的`[[Prototype]]`上找，如果最后找到`Object.prototype`还没找到，就是undefined了，因为`Object.prototype.__proto__ === null`了。举个例子koly.toString()怎么找的呢？先在koly本身找，没有，就去`koly.__proto__`(`koly.__proto__ === Object.prototype`)上找，这时候就发现找到了，就返回了。

## 原型链与继承
先看一段代码：
```javascript
function Animal(){
    this.species = 'Animal';
  }
function Dog() {
    this.type = 'Dog';
  }
// Dog应该要继承Animal
Dog.prototype = new Animal();
Dog.prototype.constructor = Dog;
var aDog = new Dog();
console.log(aDog.species);
// Animal
```
说到继承，就得知道父与子。子继承了父的或者属性或者方法。换句话说，子可以使用父的方法。放在`[[Prototype]]`及原型链的情况下，就是说子没有的方法，可以在原型链上找到，即子的`[[Prototype]]`指向了父的某个对象的prototype，至于为什么不是直接指向父的prototype，参见阮一峰的[博客](http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html)。
上面的代码有一句`Dog.prototype.constructor=Dog;`，这个不设置constructor也没有太大问题，只要你不用，StackOverflow上有讨论：[这里](http://stackoverflow.com/questions/4012998/what-it-the-significance-of-the-javascript-constructor-property)。
顺便说说这个constructor对象是什么，这个东西保存了对象的构造函数，比如:
```javascript
var koly = {name:'koly'};
koly.constructor === Object;
// true
function Animal(){}
var a = new Animal();
a.constructor === Animal;
// true
```
## 更多原型链
来看一个图：
![javascript_prototype_chain](/images/javascript_prototype_chain.png)

箭头起点的对象的`__proto__`就是箭头终点的对象。

我觉得，理解原型链，一定要弄清楚的是函数对象和其他对象的区别，前者除了有`__proto__`之外，还有`prototype`，而这两个是不一样的东西。还有就是，函数对象的`prototype`会在创建对象的时候被赋给该对象的`__proto__`。
好了，就先到这里。
