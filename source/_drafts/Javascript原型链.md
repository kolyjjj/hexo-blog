title: Javascript原型链
categories:
- Tech
tags:
- Javascript
---

Javascript是一门通过原型链来实现继承的面向对象的动态语言。其原型链机制本身并不复杂，但是理解起来有些绕。本文试图理清原型链的相关知识，并尝试归并以便于理解记忆。<!-- more -->
本文的所有代码均是在chrome的console中执行。
## 简单先行
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

相同之处，两个都是primitive values。不同之处undefined是global property，而null是Javascript literal。关于两者的区别，对于没有声明的对象，其值为undefined。一个对象的值为null则表示该值已被声明，且被赋值为null。比如koly.one是undefined，而执行`koly.one=null`，才能将koly.one赋值为null。所以null表示的其实是一个值，是一个基本的literal。跟‘one’这个string literal是一样的。
所以undefined是原始状态，而null是具有人为处理的状态。
说完了undefined和null，我们回到`prototype`和`__proto__`，前者输出了undefined，表示koly这个对象上并没有prototype这个属性。后者输出了一个非undefined，表示koly上面是有这个东西的。但是我们在定义koly这个对象的时候，并没有添加`__proto__`这个属性啊，它从哪儿来的呢？

>The __proto__ property of Object.prototype is an accessor property (a getter function and a setter function) that exposes the internal [[Prototype]] (either an object or null) of the object through which it is accessed.
>--- from [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)

上面的大意就是说通过`__proto__`可以访问对象内部的`[[Prototype]]`，这下好了，`[[Prototype]]`又是什么呢？

>Following the ECMAScript standard, the notation someObject.[[Prototype]] is used to designate the prototype of someObject. This is equivalent to the JavaScript property __proto__ (now deprecated). Since ECMAScript 6, the [[Prototype]] is accessed using the accessors Object.getPrototypeOf() and Object.setPrototypeOf().
>--- [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)


