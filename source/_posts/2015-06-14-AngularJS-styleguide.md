title: "AngularJS styleguide"
date: 2015-06-14 14:48:03
categories: 
- Tech
tags: 
- AngularJS
---

When developing a project, it's always good to have the same and consistent coding styles. Based on the [AngularJS styleguide](https://github.com/johnpapa/angular-styleguide). I wrote something that I think is practical. With both production and tests. It's a good example or guides to follow when writing "things" in AngularJS 1.4.x. If you want to know why, refer to [AngularJS styleguide](https://github.com/johnpapa/angular-styleguide).<!--more-->

## 1, Controllers
### \# naming
use camelCase for the js file name with "Controller" suffix: loginController.js

use camelCase with first letter capitalized for defining controller. E.g. LoginController
### \# Always use "controllerAs"
```
<div ng-controller="LoginController as login"></div>
```
### \# implementation
```
// loginController.js
// always use module getter
angular.module('app')
      .controller('LoginController', ['loginServie', LoginController]);

function LoginController(loginService){
  // use vm instead of this, vm = viewmodel
  var vm = this;

  // place bindable things up top
  vm.name = "name";
  vm.password = "password";
  vm.login = login;

  /////// use function declaration over function expression
  function login(credential) {
    loginService.login(credential);
  }
}

```
By assigning things to `this`, we do not need to inject `$scope`. `$scope` is introduced only when `$watch` or event is needed in controller.

### \# test
```
// decribe the name of the controller
describe('LoginController', function() {
    var scope, loginService = {
      login: function() {}
    };

    beforeEach(module('app'));

    beforeEach(inject(function ($rootScope, _$controller_) {
        scope = $rootScope.$new();
        _$controller_('LoginController', {
            '$scope': scope,
            loginService: loginService // use mocked service
        });
    }));

    it('should call login method on login service', function() {
      // mock the controller dependency
      spyOn(loginService, 'login');

      scope.login();

      expect(loginService.login).toHaveBeenCalled();
    });
});

```


## 2, Service
### \# naming
file name with "Service" suffix: camelcase, e.g. loginService.js
camelCase for name: loginService

### \# implementation:

```
// loginService.js
// always use factory instead of service
angular.module('app')
       .factory('loginService', loginService);

function loginService(){
  // things returned go first
  // first define a variable, then return it
  var service = {
    login: login
  };
  return service;

  /// function declaration
  function login(credentials) {
    console.log('credentials', credentials);
  }
}

```

### \# test

```
describe('login service', function(){
  var mock, notify;

  beforeEach(module('myServiceModule'));

  beforeEach(function() {
    mock = {alert: jasmine.createSpy()};

    module(function($provide) {
      // mock $window service
      $provide.value('$window', mock);
    });

    inject(function($injector) {
      // get service instance
      notify = $injector.get('loginService');
    });
  });

  it('should login', function() {
    login.log();

    expect(mock.alert).not.toHaveBeenCalled();
  });

});

```

## 3, directive
### \# naming
file: use camelcase, e.g. loginDirective.js
definition: use camelCase without any suffix. E.g. login, fancyBox.
### \# implementation
```
angular.module('app')
       .directive('login', login);

function login(){
  var directive = {
    restrict: 'E',
    scope: {},
    templateUrl: '/template/is/located/here.html',
    link: link
  };
  return directive;

  /// link function declaration
  function link(scope, element, attrs){

  }
}
```
### \# test

```
describe('login directive', function() {
  var compile,
      $rootScope;

  // Load the myApp module, which contains the directive
  beforeEach(module('app'));

  // Store references to $rootScope and $compile
  // so they are available to all tests in this describe block
  beforeEach(inject(function(_$compile_, _$rootScope_){
    // The injector unwraps the underscores (_) from around the parameter names when matching
    compile = _$compile_;
    rootScope = _$rootScope_;
  }));

  it('should show login htmls', function() {
    // Compile a piece of HTML containing the directive
    var element = compile("<login></login>")(rootScope);
    // fire all the watches, so the scope expression {{1 + 1}} will be evaluated
    $rootScope.$digest();
    // Check that the compiled element contains the templated content
    expect(element.html()).toContain("lidless, wreathed in flame, 2 times");
  });
});

```

## 4, retrieve data from backend
### \# always put request firing in a service
### \# use $resource for a restful resource
### \# implementation:
```
angular.module('app')
       .factory('productService', ['$resource', productService]);

function productService() {
  var products = $resource('api/products');
  var service = {
    get: products.get,
    save: products.save,
    delete: products.delete
  };
  return service;
}
```
### \# test: use $httpBackEnd to mock request and response
```
describe('MyController', function() {
   var $httpBackend, $rootScope, createController, authRequestHandler;

   // Set up the module
   beforeEach(module('MyApp'));

   beforeEach(inject(function($injector) {
     // Set up the mock http service responses
     $httpBackend = $injector.get('$httpBackend');
     // backend definition common for all tests
     authRequestHandler = $httpBackend.when('GET', '/auth.py')
                            .respond({userId: 'userX'}, {'A-Token': 'xxx'});

     // Get hold of a scope (i.e. the root scope)
     $rootScope = $injector.get('$rootScope');
     // The $controller service is used to create instances of controllers
     var $controller = $injector.get('$controller');

     createController = function() {
       return $controller('MyController', {'$scope' : $rootScope });
     };
   }));


   afterEach(function() {
     $httpBackend.verifyNoOutstandingExpectation();
     $httpBackend.verifyNoOutstandingRequest();
   });


   it('should fetch authentication token', function() {
     $httpBackend.expectGET('/auth.py');
     var controller = createController();
     $httpBackend.flush();
   });

```

## 5, filters
### \# naming
file: use camel case with filter suffix: orderByNameFilter.js
definition: camel case: orderByName
### \# implementation:
```
angular.module('app')
       .filter('orderByName', orderByNmae);
function orderByName(aList) {
  return 'orderedList';
}
```
### \# test
```
describe('orderByName filter', function() {
  var filter;

  beforeEach(module('app'));

  beforeEach(inject(function(_$filter_){
    filter= _$filter_;
  }));

  it('should order data', function() {
    var orderByName = filter('orderByName');
    expect(orderByName([])).toEqual([]);
  });
});

```
## 6, constant
### \# naming
file: use camel case with constant suffix: funConstant.js
definition: camel case: userRoles
### \# implementation:
```
angular.module('app')
       .constant('userRoles', {
           ADMIN: 'admin',
           USER: 'user'
         });

```
