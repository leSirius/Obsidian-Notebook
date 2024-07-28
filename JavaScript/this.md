>>**This:** an implicit _context object_ accessible from a code of an execution context — in order to apply the same code for multiple objects. ([source](http://dmitrysoshnikov.com/ecmascript/javascript-the-core-2nd-edition/#this))

this是一个关键字。对除箭头函数外的函数的调用，this的值是调用上下文（invocation context）。（注意“定义”与“调用”的用词）
（函数或者全局）执行上下文中有一个thisBinding值。箭头函数没有自己的this值或许指这个，然后从outer指向的执行上下文拿，所以行为上更像是变量。下文伪代码 [来源](https://blog.bitsrc.io/understanding-execution-context-and-execution-stack-in-javascript-1c9ea8642dd0)
``` 
FunctionExectionContext = {
  ThisBinding: <Global Object>,
  LexicalEnvironment: {         // let、const 和函数
    EnvironmentRecord: {   
      Type: "Declarative",
      // 在这里绑定标识符
      Arguments: {0: 20, 1: 30, length: 2},   // arguments对象
    },
    outer: <GlobalLexicalEnvironment>
  },
  VariableEnvironment: {          // var 变量
     EnvironmentRecord: {
       Type: "Declarative",
       // 在这里绑定标识符
     },
     outer: <GlobalLexicalEnvironment>
  }
}
// 函数存疑，有说法函数跟var在一起。此外函数也是函数作用域，这跟var更像
```

**函数四种定义方式**
- 函数定义和函数表达式
- 箭头函数
- 函数构造函数
- 生成器函数

**函数有五种调用方式**
- 作为函数
- 作为方法
- 作为构造函数
- 通过call()或者apply()间接调用
- 通过Javascript语言特性隐式调用

### 函数定义和函数表达式

函数作为函数调用时，this的值为全局对象或者undefined（严格模式），作为方法调用时，this的值为调用对象。方法调用指JS函数作为对象的属性调用。
```
let calculator = {
    operand1: 1,
    operand2: 1,
    add(){
        this.result = this.operand1 + this.operand2;
    }
};
calculator.add();
console.log(calculator.result);  // 2
let add = calculator.add;
add();  //该处调用并不是作为方法调用，this是全局变量或者undefined，且严格模式报错
```
除了箭头函数，this关键字不具有变量那样的作用域机制，嵌套函数不会继承包含函数的this值。嵌套函数的this值仍取决于调用方式。
函数声明会被提升，函数表达式不会。let变量定义会提升，但没做初始化（暂时性死区），var变量会初始化为`undefined`。
```
let o = {
    m: function(){
        let self = this;
        console.log(this===o);        // true

        f();
        function f(){           
            console.log(this===o);    // false
            console.log(self===o);    // true; closure
        }
        //let f = function() { ... }  // 报错
        //let f = ()=>console.log(this===o);   // true
        //f();                          
    }
}
o.m();
```

### 箭头函数
箭头函数从**定义**自己的环境继承this关键字的值，而不是像其他函数那样有自己的**调用**上下文。（另一种表述，没有自己单独的this值，其this与声明所在的上下文相同）。`bind`,`apply`,`call`都不对箭头函数起作用。

箭头函数没有prototype和arguments属性，不能作为新类的构造函数，不应当作为方法，不能使用yield，从而不能做生成器函数（[link](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/Arrow_functions)）。

直接作为方法定义的箭头函数拿不到this值，不过在构造函数中作为this的方法定义的箭头函数可以，在方法中定义的箭头函数也可以。

`{ method(args){...}... }`这种简写形式的函数也没有`prototype`和`arguments`属性，但有自己的`this`指向自己的调用上下文。

### `bind()`,`apply()`,`call()`方法
bind()方法，对上例可以有：
```
const f = (function(){
    console.log(this===o)      // true
}).bind(this);
```
bind()方法对箭头函数不起作用，它的最常见目的就是让非箭头函数变得像箭头函数。
```
function f(y) { return this.x+y; }
let o = {x: 1};
let g = f.bind(o);
console.log(g(2));       // 3
let p = {x:10, g};
p.g(2)                   // 仍是3，g仍然绑定到o，而非p
```
`bind()`可以做柯里化，执行“部分应用”(partial application)，即第一个参数之后传给bind()的参数也会被一起绑定
```
let sum = (x,y) =>x+y;
let succ = sum.bind(null, 1);
console.log(succ(2));          //3

function(y, z) {return this.x + y + z;}
let g = f.bind({x: 1}, 2);
g(3)                           //6
```
`call()`,`apply()`允许函数的间接调用。第一个参数要调用这个函数的对象，即调用上下文。
```
function trace(o, m){
    let original = o[m]
    o[m] = function(...args){
        console.log(new Date(), "Entering:", m);
        let result = original.apply(this, args);
        console.log(new Date(), "Existing:", m);
        return result;
    }
}
```

### 构造函数
`new` 关键字以构造函数的方式调用函数，先构建一个空对象，函数的this值会指向这个空对象，最后隐式返回这个对象。构造函数如果有自己的显示返回值：返回值不是对象会被忽略；是对象会被返回，原本的构造对象不会返回。
```
function construct() {
    this.val = 0
    return 1;   // construct { val: 0 }
    return {};  // {}
}
console.log(new construct());
```
 The `new` operator calls the internal \[\[Construct]] method of the `A` function which, in turn, after object creation, calls the internal \[\[Call]] method, all the same function `A`, having provided as `this` value newly created object. ([Source](http://dmitrysoshnikov.com/ecmascript/chapter-3-this/))
```
 function A() {
	console.log(this);    // newly created object, below - "a" object
	this.x = 10;
}
var a = new A();

```

In case some [obstacle things](http://dmitrysoshnikov.com/ecmascript/chapter-3-this/) are needed.