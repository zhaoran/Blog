本文会介绍ES6规范中 `instanceof` 操作符的实现，以及自定义 `instanceof` 操作符行为的几个方法。

文中涉及的规范相关的代码皆为伪代码，为了便于理解，其中可能会省略一些参数判断逻辑或者使用ES语法来代替规范内置的方法，如果发现纰漏，欢迎随时指出。

### `instanceof` 操作符的实现

#### `InstanceofOperator(O, C)`

`O instanceof C` 会被编译为方法调用 -- `InstanceofOperator(O, C)`，其实现如下：

```javascript
InstanceofOperator(O, C){

    if(typeof C !== 'object'){
        throw TypeError;
    }
    
    let instOfHandler = C[Symbol.hasInstance];
    
    if(typeof instOfHandler !== 'undefined'){
        return !!instOfHandler.call(C, O);
    }
    
    if(typeof C !== 'function'){
        throw TypeError;
    }
    
    return OrdinaryHasInstance(C, O);
}

```

该方法首先判断了 `C[Symbol.hasInstance]` 方法是否存在，如果存在，就调用；如果不存在，就调用 `OrdinaryHasInstance(C, O)`方法。

#### `Function.prototype[Symbol.hasInstance](V)`

对于 ES 内置构造器如 `Function()`, `Array()`，其本身是没有 `[Symbol.hasInstance]` 属性的，都继承自 `Function.prototype`，这个方法是预定义的，不可修改：

```javascript
Reflect.getOwnPropertyDescriptor(Function.prototype, Symbol.hasInstance)
=>
Object {writable: false, enumerable: false, configurable: false, value: function}
```

其实现如下：

```javascript
Function.prototype[Symbol.hasInstance](V){

    let F = this;
    
    return OrdinaryHasInstance(F, V);
}
```
比较简单明了，直接调用 `OrdinaryHasInstance(F, V)` 方法。


#### `OrdinaryHasInstance(C, O)`

上述两个方法都最终调用到了 `OrdinaryHasInstance(C, O)` ，其实现如下：

```javascript
OrdinaryHasInstance(C, O){
    
    if(typeof C !== 'function'){
        return false;
    }
    
    if(typeof O !== 'object'){
        return false;
    }

    let P = C.prototype;
    
    while(true){
    
        let O = Object.getPrototypeOf(O);
        
        if(O === null){
            return false;
        }
        
        if(SameValue(P, O)){
            return true;
        }
    }
}
```

这个方法是判断 `C.prototype` 是否在 `O` 的原型链上。

知道了 `instanceof` 操作符的实现原理，可以发现有3个地方可以自定义操作符行为。

### 自定义 `instanceof` 操作符行为的几个方法

1. `InstanceofOperator(O, C)` 方法中的 `let instOfHandler = C[Symbol.hasInstance]`

这是对操作符右侧变量做修改

普通的对象，默认是没有 `[Symbol.hasInstance]` 属性的，也继承不到内置的 `Function.prototype[Symbol.hasInstance]()` 方法：

```javascript
let o = {};
let a = new Array();

console.log(a instanceof Array) // true
console.log(a instanceof o) // Uncaught TypeError: Right-hand side of 'instanceof' is not callable
```

如果要避免报错，可以让 `o` 继承系统内置方法：

```javascript
Reflect.setPrototypeOf(o, Function.prototype);

console.log(a instanceof o) // false
```

也可以直接给其添加 `[Symbol.hasInstance]` 属性：
```javascript
Reflect.defineProperty(o, Symbol.hasInstance, {
    value(instance){
        return Array.isArray(instance);
    }
});

console.log(a instanceof o) // true
```

一种更常规的自定义方法是：
```javascript
class C {
    static [Symbol.hasInstance](instance){
        
        return false;
    }
}

let c = new C();

console.log(c instanceof C) // false
    
```
注意，这里定义的是静态方法，是直接挂在 `C` 上的方法，而不是实例方法：

```javascript
Reflect.getOwnPropertyDescriptor(C, Symbol.hasInstance);
=>
Object {writable: true, enumerable: false, configurable: true, value: function}
```

使用传统的模拟构造函数法：

```javascript
function F(){}

Reflect.defineProperty(F, Symbol.hasInstance, {
    value(instance){
        return false;
    }
});

let f = new F();

console.log(f instanceof F) // false

```

内置构造器也是可以添加`Symbol.hasInstance`方法的：
```javascript
Reflect.defineProperty(Array, Symbol.hasInstance, {
    value(instance){ return typeof instance === 'function';}
})
console.log(Array[Symbol.hasInstance]) // function value(instance){ return typeof instance === 'function';}
console.log([] instanceof Array) // false
console.log(function(){} instanceof Array) // true
```

注意，如果不使用 `defineProperty` 方法，而是用 `[]` 的方法来设置属性的话，是不生效的：
```javascript
Array[Symbol.hasInstance] = function(){ return typeof instance === 'function';}
console.log(Array[Symbol.hasInstance]) // function [Symbol.hasInstance]() { [native code] }
```

2. `OrdinaryHasInstance(C, O)` 方法中的 `let P = C.prototype`;

也是对操作符右侧变量做修改

```javascript
function F(){}

F.prototype = {};

let f = new F();

console.log(f instanceof F) // true

F.prototype = {};

console.log(f instanceof F) // false
```
在实例化之后，再重新设置构造函数的 `prototype` 属性，会导致修改之前创建的实例做 `instanceof` 操作时不再符合预期。

3. `OrdinaryHasInstance(C, O)` 方法中的 `let O = Object.getPrototypeOf(O)`

这是对操作符左侧变量做修改：

```javascript
var a = new Array();

console.log(a instanceof Array) // true

Object.setPrototypeOf(a, Function.prototype);

console.log(a instanceof Array) // false
console.log(a instanceof Function) // true
```
对 `a` 的原型链上的任何环节做修改，都可以改变 `instanceof` 操作符的行为。

