title: "javascript pattern modules"
date: 2015-05-07 21:16:03
categories: 
- Tech
tags:
- javascript
- pattern
---

**A reading notes of &lt;&lt;Learning javascript design patterns&gt;&gt;**<!--more-->


##Why modules?
* It's natural to divide an application to modules.  
* To avoid name conflicts  
* It provides encapsulation, cohesion inside a module, decouping between modules (may use IOC & DI)

##How?
Two ways here:

* module pattern  
* object literal  

####A module defined using object literal syntax:
{% codeblock lang:javascript %}
  var myModule = {
    myProperty: 'someValue',
    //here another object is defined for configuration purposes:
    myConfig: {
  	useCaching: true,
  	language: 'en'
    },
    //a very basic method
    myMethod: function() {
  	console.log('I can haz functionality?');
    },
    //output a value based on current configuration
    myMethod2: function() {
  	console.log('Caching is: ' + (this.myConfig.useCaching) ? 'enabled' : 'disabled');
    },
    //override the current configuration
    myMethod3: function(newConfig) {
  	if (typeof newConfig == 'object'){
  	  this.myConfig = newConfig;
  	  console.log(this.myConfig.language);
  	}
    }
  };
{% endcodeblock %}
Usage:
{% codeblock lang:javascript %}
	myModule.myMethod();
	myModule.myMethod2();
	myModule.myMethod3({
	  language: 'fr',
	  useCaching: false
	});
{% endcodeblock %}

####The module pattern
The module pattern is used to further *emulate* the concept of classes in such a way that we're able to include both public/private methods and variables inside a single object, thus shielding particular parts from the global scope. What this results in is a reduction in the likelihood of your function names conflicting with other functions defined in additional scripts on the page.

The key to encapsulates "privacy" is to use **closures**.

Within the module pattern, variables or methods declared are only available inside the module itself thanks to closure.

A simple example:

{% codeblock lang:javascript %}
  var testModule = (function(){
    //private variable
    var counter = 0;
    return {
      incrementCounter: function() {
        return counter++;
      },
      resetCounter: function() {
        console.log('counter value prior to reset: ' + counter);
        counter = 0;
      }
    };
  })();

  //test
  testModule.incrementCounter();
  testModule.resetCounter();

  {% endcodeblock %}

We may view the 'testModule' as a 'prefix'.

##Disadvantage

* You can't access private members in methods that are added to the object at a later point.
* Can't creat automated unit test for private members and additional complexity when bugs require hot fixes.  
* It's not easy to extend privates.

##Module in js lib
We may imagine a module function first:

{% codeblock lang:javascript %}

var module = function(name){
  console.log(name);
  return function(name){
    var moduleName = name;
    return {
      returnName: function(){
        return moduleName;
      }
    };
  }(name);
};
var koly = module('koly');

console.log(koly);
console.log(koly.returnName());
koly.gender = 'male';
console.log(koly.gender);

{% endcodeblock %}

####angular.module in AngularJS
In AngularJS, there is a `angular.module`, which can be used to create a module.

We can use it by:

{% codeblock %}
	var car = angular.module('car');
{% endcodeblock %}

Dive into the source:

{% codeblock lang:javascript %}

function setupModuleLoader(window) {

  function ensure(obj, name, factory) {
	 return obj[name] || (obj[name] = factory());
  }

  return ensure(ensure(window, 'angular', Object), 'module', function(){
	var modules = {};
	return function module(name, requires, configFn) {
	  if (requires && modules.hasOwnProperty(name)) {
	     modules[name] = null;
	  }
	  return ensure(modules, name, function() {
	    if (!requires) {
	       throw Error('No module: ' + name);
	    }

	    var invokeQueue = [];
	    var runBlocks = [];
	    var config = invokeLater('$injector', 'invoke');
	    var moduleInstance = {

	      _invokeQueue: invokeQueue,
	      _runBlocks: runBlocks,
	      requires: requires,
	      name: name,
	      provider: invokeLater('$provider', 'provider'),
	      factory: invokeLater('$provider', 'factory'),
	      service: invokeLater('$provider', 'service'),
	      value: invokeLater('$provider', 'value'),
	      constant: invokeLater('$provider', 'constant', 'unshift'),
	      animation: invokeLater('$animationProvider', 'register'),
	      filter: invokeLater('$filterProvider', 'register'),
	      controller: invokeLater('$controllerProvider', 'register'),
	      directive: invokeLater('$compileProvider', 'directive'),
	      config: config,
	      run: function(block) {
	        runBlocks.push(block);
	        return this;
	      }
	     };

	     if (configFn) {
	        config(configFn);
	     }

	     return moduleInstance;

	     function invokeLater(provider, method, insertMethod) {
	        return function() {
	          invokeQueue[insertMethod || 'push']([]provider, method, arguments]);
	          return moduleInstance;
	        };
	      }
	     });
	    };
	  });
	}

  {% endcodeblock %}
Usage of setupModuleLoader:

{% codeblock lang:javascript %}

	angularModule = setupModuleLoader(window);
	try {
	  angularModule('ngLocale');
	} catch (e) {
	  angularModule('ngLocale', []).provider('$locale', $LocalProvider);
	}

  {% endcodeblock %}

When the `setupModuleLoader()` function is called, it will return a object, which is `window[angular][module]`, e.g. `window.angular.module`

`window.angular.module` is a function itself, the function signiture is : `module(name, requires, configFn)`

when you call `angularModule`, you acturally are calling `window.angular.module`, like `module(name, requires, configFn)`. This function will return `modules[name] = moduleInstance`

##The Revealing Module Pattern
First see an example:

{% codeblock lang:javascript %}

  var myRevealingModule = (function(){
    var name = 'John Smith';
    var age = 40;

    function updatePerson(){
      name = 'John Smith Updated';
    }
    function setPerson(){
      name = 'John Smith Set';
    }
    function getPerson(){
      return name;
    }
    return {
      set: setPerson,
      get: getPerson
    };
  }());

  {% endcodeblock %}

##Discussion
For better using module patter, you need to think more about 'What is a module?'.

One clue is to think about cohesion and decoupling, and code management and abstract and encapsulation. These are why we want to use the 'module'.
