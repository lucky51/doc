# 你不知道的JavaScript上卷-第二部分-this和对象原型

## 第一章 关于this

this 关键字是JavaScript中最复杂的机制之一。它是一个很特别的关键字，被自动定义在所有函数的作用域中。但是即使是非常有经验的JavaScript开发者也很难说清楚它到底指向什么。

### 1.1 为什么要用this

`this` 提供了一种更优雅的方式来隐式 "传递" 一个对象引用，因此可以将API设计得更加简洁并且易于复用。
另外随着你的使用模式越来越复杂，显式传递上下文对象会让代码变得越来越混乱，使用this则不会这样。

### 1.2 误解

太拘泥于 "this" 的字面意思就会产生一些误解。有两种常见的对于this的解释，但是它们都是错误的。

#### 1.2.1 指向自身

如果要从函数对象内部引用它自身，那只使用`this`是不够的。一般来说你需要通过一个指向函数对象的词法标识符(变量) 来引用它。
思考一下下面的两个函数:

```js
function foo(){
    foo.count = 4; //foo指向它自身
}
setTimeout(function(){
    // 匿名(没有名字的) 函数无法指向自身
},10)
```

第一个函数被称为具名函数，在它内部可以使用foo来引用自身。
第二个匿名函数内部无法引用自身。
注意: 还有一种传统的但是现在已经被弃用和批判的用法，是使用arguments.callee来引用当前正在运行的函数对象。这是唯一一种可以从匿名函数对象内部引用自身的方法。然而，更好的方式是避免使用匿名函数，至少在需要自引用时使用具名函数(表达式)。arguments.callee 已经被弃用，不应该再使用它。
一种解决方法是 使用具名方法标识符代替this引用函数对象
另一种是使用`call` 或 `apply` 强制更改`this`的指向。

#### 1.2.2 它的作用域

需要明确的是，this在任何情况下都不指向函数的词法作用域。在JavaScript内部，作用域确实和对象类似，可见的标识符都是它的属性。但是作用域"对象" 无法通过JavaScript代码访问，它存于JavaScript引擎内部。
每当你想要把`this` 和词法作用域的查找混合使用时，一定要提醒自己，这是无法实现的。

### 1.3 this 到底是什么

`this` 是在运行时进行绑定，并不是在编写时绑定，它的上下文取决于函数调用时的各种条件。`this`的绑定和函数声明的位置没有任何关系，只取决于函数的调用方式。
当一个函数被调用时，会创建一个活动记录(有时候也称为执行上下文)。 这个记录会包含函数在哪里被调用(调用栈)、函数的调用方法、传入的参数等信息。this就是记录的其中一个属性，会在函数执行的过程中用到。

### 1.4 小结

学习`this`的第一步是明白`this`既不指向函数自身也不指向函数的词法作用域，你也许被这样的解释误导过，但其实它们都是错误的。
this实际上是在函数被调用时发生的绑定，它指向什么完全取决于函数在哪里被调用。

## 第二章 this全面解析

### 2.1 调用位置

理解调用位置:调用位置就是函数在代码中被调用的位置(而不是声明的位置)。
通常来说，寻找调用位置就是寻找"函数被调用的位置"， 但是做起来没有这么简单，因为某些编程模式可能会隐藏真正的调用位置。
最重要的是要分析调用栈(就是为了到达当前执行位置所调用的所有函数)。 我们关心的调用位置就是当前正在执行的函数的前一个调用中。

```js
function baz(){
    //当前调用栈是: baz
    //因此，当前调用位置是全局作用域
    console.log("baz");
    bar(); // <-- bar 的调用位置
}
function bar(){
    //当前调用栈 baz -> bar
    //因此，当前调用位置在baz中
    console.log("bar") ;
    foo(); // <- - foo 的调用位置
}
function foo(){
    //当前调用栈是 baz -> bar -> foo
    //因此，当前调用位置在 bar中
}
baz();  // < --baz的调用位置。
```

### 2.2 绑定规则

函数的执行过程中调用位置如何决定`this` 的绑定对象。

#### 2.2.1 默认绑定

在使用不带任何修饰的函数引用进行调用时，只能使用`默认绑定`，无法应用其他规则。
如果使用严格模式(strict mode) ,那么全局对象将无法使用默认绑定，因此`this`会绑定到 `undefined`,换句话说只有在非 strict mode下，默认绑定才能绑定到全局对象;严格模式下与方法的调用位置无关。

#### 2.2.2 隐式绑定

另一条需要考虑的规则是调用位置是否有上下文对象，或者说是否被某个对象拥有或者包含，不过这种说法可能会造成一些误导。

```js
function foo(){
    console.log(this.a);
}
var obj = {
    a:2,
    foo:foo
}
obj.foo(); //2
```

调用位置会使用obj上下文来引用函数，因此你可以说函数被调用时obj对象 "拥有" 或者 "包含" 它。
`隐式绑定` 规则则会把函数调用中的`this`绑定到这个上下文对象。

##### 隐式丢失

一个最常见的`this`绑定问题就是被隐式绑定的函数会丢失绑定对象，也就是说它会应用默认绑定，从而把`this`绑定到全局对象或者undefined上，取决于是否是严格模式。
回调函数丢失`this`绑定时非常常见的。除此之外，还有一种情况`this`的行为会出乎意料:调用回调函数的函数可能会修改`this`。在一些流行的JavaScript库中事件处理器常会把回调函数的`this`强制绑定到触发事件的DOM元素上。
这在一些情况下可能有用，但是有时它可能会让你感到非常郁闷。遗憾的是，这些工具通常无法选择是否启用这个行为。

#### 2.2.3 显示绑定

我们除了显示的利用对象属性的方式调用方法外，还可以使用call 和 apply强制调用函数来更改this或绑定this指向。这种方式便是 `显式绑定`。

1. 硬绑定
    思考下边的代码:

    ```js
    function foo(){
        console.log(this.a);
    };
    var obj = {
        a:2
    };
    bar(); //2
    setTimeout(bar ,100); //2
    //硬绑定的bar 不可能再修改它的this
    bar.call(window);  //2
    ```
    由于`硬绑定`是一种非常常用的模式，所以在ES5中提供了内置的方法`Function.prototype.bind`,bind(...)会返回一个`硬编码`的新函数，它会把参数设置为`this`的上下文并调用原始函数。
2. API调用的"上下文"
    第三方库的许多函数，以及JavaScript语言和宿主环境中许多新内置函数，都提供了一个可选参数，通常被称为"上下文" (context) , 其作用和bind(...)一样，确保你的回调函数使用指定的`this`。
    举例来说:

    ```js
    function foo(el){
        console.log(el, this.id);
    }
    var obj = {
        id: "awesome"
    };
    //调用 foo(...) 时 this 绑定到 obj
    [1,2,3].forEach(foo, obj);
    ```

这些函数实际上就是通过 call(...) 或者apply(...) 实现了显示绑定，这样你可以少写一些额外的代码。

#### 2.2.4 new绑定

这是第四条也是最后一条 `this` 的绑定规则
在使用`new`来调用函数，或者说发生构造函数调用时，会自动执行下面的操作。

1. 创建(或者说构造) 一个全新的对象。
2. 这个心对象会被执行[[原型]]连接。
3. 这个新对象会绑定到函数调用的 `this`
4. 如果函数没有返回其他对象，那么`new`表达式中的函数调用会自动返回这个对象.

new 是最后一种可以影响函数调用时 `this`绑定行为的方法，我们称之为`new绑定`。

### 2.3 优先级

隐式调用 小于 显式调用，内置的 bind方法中this 可以通过 new改变，这是由于内部实现

```js
this instanceof fNOP &&
oThis ? this : oThis
//...以及;
fNOP.prototype = this.prototype;
fBound.prototype = new fNOP();
```

这段代码会判断硬绑定函数是否是被 new调用，如果是的话就会使用新创建的 this来替换硬绑定的this。

#### 判断this

现在可以根据优先级来判断函数在某个调用位置应用的是哪条规则。可以按照下面的顺序进行判断。

1. 函数是否在 `new`中调用(new 绑定) ? 如果是的话 this绑定的是新创建的对象。
    `var bar =new foo()`
2. 函数是否通过 call、apply(显式绑定) 或者硬绑定调用? 如果是的话，`this`绑定的是指定的对象。
    `var bar = foo.call(obj2)`
3. 函数是否在某个上下文对象中调用 (隐式绑定) ? 如果是额话， `this` 绑定的是那个上下文对象。
    `var bar = obj1.foo()`
4. 如果都不是的话，使用默认绑定。如果在严格模式下，就绑定到undefined，否则绑定到全局对象。
    `var bar =foo()`

### 2.4 绑定例外

然而规则总有例外，在某些场景下`this`绑定行为会出乎意料，你认为应当应用其他绑定规则时，实际上应用的可能是默认绑定规则。

#### 2.4.1 被忽略的this

如果你把 `null`或者 `undefined`作为`this`的绑定对象传入`call`,`apply`或者`bind`,这些值在调用时会被忽略，实际应用的是默认绑定规则。
`foo.call(null)`

##### 更安全的this

一种 "更安全" 的做法是传入一个特殊的对象，把this绑定到这个对象不会对你的程序产生任何副作用。这使得函数变得更加"安全"， 而且可以提高代码的可读性，因为这个对象表示 "我希望this是空"，这比null的含义更清楚。

#### 2.4.2 间接引用

另一个需要注意的是，你有可能(有意或者无意的)创建一个函数的"间接引用"，在这种情况下，调用这个函数会应用默认绑定规则。
注意: 对于默认绑定来说，决定 this 绑定对象的并不是调用位置是否处于严格模式，而是函数体是否处于严格模式。如果函数体处于严格模式，this会绑定到undefined，否则this会绑定到全局对象。

#### 2.4.3 软绑定

之前看到的，硬绑定这种方式可以把`this`强制绑定到指定的对象(除了使用 new时)，防止函数调用应用默认绑定规则。问题在于硬绑定会大大降低函数的灵活性，使用硬绑定之后就无法使用隐式绑定或者显示绑定来修改`this`。
如果可以给默认绑定指定一个全局对象和 undefined以外的值，那就可以实现和硬绑定相同的效果，同时保留隐式绑定或者显式绑定修改this的能力。
通过一种被称之为`软绑定`的方法来实现我们想要的效果:

```js
if(!Function.prototype.softBind){
    Function.prototype.softBind = function(obj){
        var fn =this;
        //捕获所有 curried参数
        var curried = [].slice.call(arguments, 1);
        var bound = function(){
            return fn.apply(
                (!this || this === (window || global)) ?
                obj : this
                curried.concat.apply( curried, arguments)
            )
        }
        bound.prototype = Object.create(fn.prototype);
        return bound;
    }
}
```

除了软绑定之外，softBind(...) 的其他原理和ES5内置的bind(...)类似。它会对指定的函数进行封装，首先检查调用时的`this`,如果`this`绑定到全局对象或者undefined，那就把指定的默认对象obj绑定到this,否则不会修改this。此外，这段代码还支持可选的柯里化。

### 2.5 this 词法

箭头函数并不是使用 `function`关键字定义的，而是使用被称为 "胖箭头" 的操作符`=>`定义的。箭头函数不适用`this`的四种标准规则，而是根据外层(函数或者全局)作用域来决定`this`。
如果你经常编写this风格的代码，但是绝大部分时候都会使用self=this 或者箭头函数来否定this机制，那你或许应当:

1. 只使用词法作用域并完全抛弃错误`this`风格的代码；
2. 完全采用`this`风格，在必要时使用`bind(...)`,尽量避免使用 self =this 和箭头函数

### 2.6 小结

如果要判断一个运行中函数的`this`绑定，就需要找到这个函数的直接调用位置。找到之后就可以顺序应用下面这四条规则来判断`this`的绑定对象。

1. 由new调用? 绑定到新创建的对象。
2. 由call或者apply(或者bind)调用?绑定到指定的对象。
3. 由上下文对象调用?绑定到那个上下文对象。
4. 默认:在严格模式下绑定到undefined，否则绑定到全局对象。

一定要注意，有些调用可能在无疑中使用默认绑定规则。如果想"更安全"地忽略`this`绑定，你可以使用你个 DMZ对象，比如 $ = Object.create(null), 以保护全局对象。
ES6中的箭头函数并不会使用四条标准的绑定规则，而是根据当前的词法作用域来决定`this`，具体来说，箭头函数会继承外层函数调用的`this`绑定(无论this 绑定到什么)。 这其实和ES6之前的代码中的`self = this`机制一样。

## 第三章 对象

### 3.1 语法

对象可以通过两种形式定义:声明(文字)和构造形式。
对象的文字语法大概是这样:

```js
var myObj ={
    key:value
    //...
}
```

构造形式大概是这样:

```js
var myObj = new Object();
myObj.key = vlaue;
```

构造形式和文字形式生成对象是一样的。唯一的区别是，在文字声明中你可以添加多个键/值对，但是在构造形式中你必须逐个添加属性。

### 3.2 类型

对象是JavaScript的基础。在JavaScript中一共有六种主要类型(术语是"语言类型"):

* string
* number
* boolean
* null
* undefined
* object

`注意`: 简单基本类型(string, boolean,number,null和undefined) 本身并不是对象。null有时会被当作一种对象类型，但是这其实只是语言本身的一个bug,即对null执行typeof null时会返回字符串"object" 。事实上，null本身是基本类型。
有一种常见的`错误说法`是 "JavaScript中万物皆是对象",这显然是错误的。
实际上，JavaScript中有许多特殊的对象子类型，我们可以称之为复杂基本类型。
`函数`就是对象的一个子类型(从技术角度来说就是"可调用的对象")。JavaScript中的函数是 "一等公民"，因为它们本质上和普通对象一样(只是可以调用)，所以可以像操作其他对象一样操作函数(比如当作赢一个函数的参数)。
数组也是对象的一种类型，具备一些额外的行为，数组中内容的组织方式比一般的对象要稍微复杂一些。

#### 内置对象

JavaScript中海油一些对象子类型，通常被称为内置对象。有些内置对象的名字看起来和简单基础类型一样，不过实际上它们的关系更复杂，稍后介绍。

* String
* Number
* Boolean
* Object
* Function
* Array
* Date
* RegExp
* Error

原理性:不同对象在底层都表示为二进制，在JavaScript中二进制前三位都为0的话会被判断为object类型，null的二进制表示全是0，自然前三位也是0，所以执行typeof时会返回"object"。

原始值 "I am a string" 并不是一个对象，它只是一个字面量，并且是一个不可变的值。如果要在这个字面量上执行一些操作，比如获取长度、访问其中某个字符等，那需要将其装换为String对象。
幸好，在必要时语言会自动把字符串字面量转换成一个String对象，也就是说你并不需要显式的创建一个对象。JavaScript社区中的大多数人都认为能使用文字形式时就不要用构造形式。
`null` 和 `undefined`没有对象的构造形式，他们只有文字形式。想法，Date只有构造，没有文字形式。
对于`Object`,`Array`,`Function`,和`RegExp`(正则表达式)来说，无论使用文字形式还是构造形式，它们都是对象，不是字面量。在某些情况下，相比文字形式创建对象，构造形式可以体用一些额外选项。由于这两种形式都可以创建对象，所以沃恩首选更简单的文字形式。建议只在需要那些额外选项时使用构造形式。
Error 对象很少在代码中显式创建，一般是在抛出异常时被自动创建。也可以使用`new Error(...)`这种构造形式来创建，不过一般来说用不着。

### 3.3 内容

之前提到过，对象的内容是由一些存储在特定命名位置的(任意类型的)值组成的，我们称之为属性。
需要强调的一点是，当我们说"内容"时，似乎在暗示这些值实际上被存储在对象内部，但是只是它的表现形式。在引擎内部，这些值的存储方式是多种多样的，一般并不会存储在对象容器内部。存储在对象容器内部的是这些属性的名称，它们就像指针(从技术的角度来说就是引用)一样，指向这些值真正的存储位置。

#### 3.3.1 可计算属性名

ES6增加了可计算属性名，可以在文字形式中使用 []包裹一个表达式来当作属性名:

```js
var prefix = "foo";
var myObject = {
    [prefix + "bar"]:"hello",
    [prefix + "baz"]:"world"
};
myObject["foobar"];  //hello
myObject["foobaz"]; //world
```

可计算属性名最常用的场景可能是ES6的符号 (Symbol)。

#### 3.3.2 属性和方法

如果访问的对象属性时一个函数，有些开发者喜欢使用不一样的叫法以作区分。由于函数很容易被人分为是属于某个对象，在其他语言中，属于对象(也被称为"类")的函数通常被称为"方法",因此把"属性访问" 也说成是"方法访问"也就不奇怪了。
有意思的是，JavaScript的语法规范也做出了同样的区分。
从技术的角度来说，函数永远不会"属于" 一个对象，所以把对象内部引用的函数称为"方法"似乎有点不妥。
最保险的说法可能是,"函数" 和"方法"在JavaScript中是可以互换的。

#### 3.3.3 数组

数组也支持[]访问形式，不过我们之前提到过的，数组有一套更加结构化的值存储机制(不过仍然不限制值的类型)。数组期望的是数值下标，也就是存储的位置(通常称为索引)是整数。
注意:如果你试图向数组添加一个属性，但是属性名"看起来"像一个数字，那它会变成一个数值下标(因此会修改数组的内容而不是添加一个属性):

```js
var myArray = ["foo", 42, "bar"];
myArray["3"] = "baz";
myArray.length; //4
myArray[3];  //"baz"
```

#### 3.3.4 复制对象

#### 3.3.5 属性描述符

在ES5之前，JavaScript语言本身并没有提供可以直接检测属性特性的方法，比如判断属性是否是可读的。

思考下面的代码:

```js
var myObject = {
    a:2
}
Object.getOwnPropertyDescriptor(myObject, "a");
// {
//     value:2,
//     writable:true,
//     enumerable:true,
//     configurable:true
// }
```

这个普通的对象属性对应的属性描述符(也被称为"数据描述符",因为它只保存了一个数据值)可不仅仅只是一个2.它还包含另外三个特性: writable(可写),enumerable(可枚举)和configurable(可配置)。
在创建普通属性时属性描述符会使用默认值，我们也可以使用`Object.defineProperty(...)` 来添加一个新属性或者修改一个已有的属性(如果它是configurable) 并对特性进行设置。
举例来说:

```js
var myObject = {};
Object.defineProperty(myObject,"a",{
    value:2,
    writable:true,
    configurable:true,
    enumerable:true
});
myObject.a;  //2
```

我们使用 `defineProperty(...)`给myObjce添加一个普通的属性并显式指定了一些特性。然而，一般来说你不会使用这种方式，除非你想修改属性描述符。

1. Writable
    writable 决定是否可以修改属性的值。
    如果设置为不可写，对于修改属性值的行为如果在严格模式下会出现错误，而非严格模式下会出现修改静默失败。
2. Configurable
    只要配置是可配置的，就可以使用 defineProperty(...)方法来修改属性描述符。
    值得注意的是，不管是不是处于严格模式，尝试修改一个不可配置的属性描述符都会出现错误。另外把configurable修改成false是单向操作，无法撤销。
    除了无法修改外，configurable:false 还会禁止删除这个属性。
3. Enumerable
    用户定义的所有的普通属性默认都是 `enumerable`，这通常就是你想要的。但是如果你不希望某些属性出现在枚举中，那么就把它设置成 `enumerable:false`。

#### 3.3.6 不可变

1. 对象常量
    结合writable:false 和 configurable:false 就可以创建一个真正的常量属性(不可改变、重定义或者删除):

    ```js
    var myObject = {};
    Object.defineProperty(myObject, "FAVORITE_NUMBER",{
        value:42,
        writable:false,
        configurable:false
    });
    ```
2. 禁止扩展
    如果你想禁止一个对象添加新属性并且保留已有属性，可以使用`Object.preventExtensoins(...)`:

    ```js
    var myObject ={
        a:2
    };
    Object.preventExtensions(myObject);
    myObject.b =3;
    myObject.b;   //undefined
    ```
    在非严格模式下，创建属性b会静默失败。在严格模式下，将会抛出`TypeError`错误。
3. 密封
    Object.seal(...) 会创建一个 "密封"的对象，这个方法实际上会在一个现有对象上调用`Object.preventExtensions(...)`并把所有现有属性标记为 `configurable:false`。
4. 冻结
    `Object.freeze(...)` 会创建一个冻结对象，这个方法实际上会在一个现有对象调用`Object.seal(...)`并把所有"数据访问"属性标记为 `writable:false`，这样就无法修改它们的值。
    这个方法是你可以应用在对象级别最高的不可变性，它会禁止对于对象及其任意直接属性的修改(不过就像我们之前说过的，这个对象的其他对象是不受影响的)。
    你可以 "深度冻结" 一个对象，具体方法为，首先在这个对象上调用`Object.freeze(...)` ,然后遍历它引用的所有对象并在这些对象上调用`Object.freeze(...)`。但是一定要小心，因为这样做有可能会在无疑中冻结其他(共享)对象。

#### 3.3.7 [[Get]]

属性访问在实现时有一个微妙却非常重要的细节，思考下面的代码:

```js
var myObject ={
    a:2
};
myObject.a; //2
```

在语言规范中，myObject.a 在 myObject 上实际上是实现了`[[Get]]`操作(有点像函数调用:`[[Get]]`())。对象默认的内置`[[Get]]`操作首先在对象中查找是否有名称相同的属性，如果找到就会返回这个属性的值。
然而，如果没有找到名称相同的属性，按照 `[[Get]]`算法的定义会执行另外一种非常重要的行为。我们会在之后介绍这个行为(其实就是遍历可能存在额`[[Prototype]]`) 链,也就是原型链。
如果无论如何都没有找到相同的属性，那`[[Get]]`undefined:

```js
var myObject = {
    a:2
};
myObject.b;  // undefined
```

注意:这种方法和访问遍历时是不一样的，如果你引用了一个当前词法作用域中不存在的变量，并不会像对象属性一样返回undefined，而是会抛出一个 ReferenceError 异常:

```js
var myObject = {
a:undefined
};
myObject.a; //undefined
myObject.b; //undefined
```

从返回值的角度来说，这两个引用没有区别-它们都返回了undefined。然而，尽管乍看之下没有什么区别，实际上底层的`[[Get]]`操作对myObject.b进行了更复杂的处理。

#### 3.3.8 [[Put]]

既然有可以获取属性值的 `[[Get]]`操作，就一定有对应的`[[Put]]`操作。
你可能会认为给对象的属性赋值会触发`[[Put]]`操作，但是实际情况并不完全是这样。
`[[Put]]`被触发时，实际的行为取决于许多因素，包括对象中是否已经存在这个属性(这是最重要的因素)。
如果已经存在这个属性，`[[Put]]`算法大致会检查下面这些内容。

1. 属性是否是访问描述符 ? 如果是并且存在setter就调用setter。
2. 属性的数据描述符中`writable`是否是`false`？如果是，在非严格模式下静默失败，在严格模式下抛出`TypeError`异常。
3. 如果都不是，将该值设置为属性的值。

如果对象不存在这个属性，`[[Put]]`操作会更加复杂，后续介绍。

#### 3.3.9 Getter 和Setter

在 ES5中可以使用 `getter` 和`setter`部分改写默认操作，但是只能应用在单个属性上，无法应用整个对象上。`getter`是一个隐藏函数，会在获取属性值时调用。`setter`也是一个隐藏函数，会在设置属性值是调用。
当你给一个属性定义 `getter` ,`setter`或者两者都有时，整个属性会被定义为 "访问描述符" ("数据描述符"相对)。对于访问描述符来说，JavaScript会忽略它们的`value`和 `writable` 特性，取而代之的是关心 `get` 和 `set` (还有`configurable`和 `enumerable`) 特性。
不管是对象文字语法中的 `get a(){...}`,还是 `defineProperty(...)` 中的显示定义，二者都会在对象中创建一个不包含值的属性，对于这个属性的访问会自动调用一个隐藏函数，它的返回值会被当作属性访问的返回值:

#### 3.3.10 存在性

前面我们介绍过，如 `myObject.a` 的属性访问值可能是undefined ，但是这个值有可能是属性中存储的undefined，也可能是因为属性不存在所以返回undefined。那么如何区分这两种情况呢?

```js
var myObject = {
    a:2
};
("a" in myObject) ; // true
("b" in myObject); //false
myObject.hasOwnProperty("a");  //true
myObject.hasOwnProperty("b");  //false
```

`in` 操作符会检查属性是否在对象及其`[[Prototype]]`原型链中。相比之下，`hasOwnProperty(...)` 只会检查属性是否在 myObject 对象中，不会检查`[[Prototype]]`链。
所有的普通对象都可以通过对于`Object.prototype`的委托来访问`hasOwnProperty`，但是有的对象可能没有连接到 `Object.prototype` 比如使用 `Object.create(null)`来创建。在这种情况下，形如 `myObject.hasOwnProperty(..)`就会失败。
这时可以使用一种更加强硬的方法来进行判断: `Object.prototype.hasOwnProperty.call(myObject, "a")`,它借用基础的 `hasOwnProperty(...)`方法并把它显式绑定到 myObject上。

1. 枚举
    `propertyIsEnumerable(...)`会检查给定的属性名是否直接存在于对象中(而不是在原型链上)并且满足 `enumerable:true`。
    `Object.keys(...)` 会返回一个数组，包含所有可枚举的属性，`Object.getOwnPropertyNames(..)` 会返回一个数组，包含所有属性，无论他们是否可枚举。
    `in` 和 `hasOwnProperty`的区别在于是否查找`[[Prototype]]`链，然而，`Object.keys(...)` 都只会查找对象直接包含的属性。
    (目前)并没有内置的方法可以获取 in 操作符的属性列表(对象本身的属性以及[[Property]]链中的所有属性)。不过你可以递归遍历某个对象的整条`[[Prototype]]`链并保存每一层中使用 `Object.keys(...)`得到的属性列表-只可包含可枚举属性。

### 3.4 遍历

`for ... in` 循环可以用来遍历对象的可枚举属性列表(包括 `[[Prototype]]`链)
ES5中增加了一些数组辅助迭代器，包括 `forEach(...)`,`every(...)`和`some(...)`。 每种辅助迭代器都可以接受一个回调函数并把它应用到数组的每个元素上，唯一的区别就是它们对于回调函数返回值的处理方式不同。
`forEach(...)`会遍历数组中的所有值并忽略回调函数的返回值。`every(...)`会一直运行直到回调函数返回`false`(或者"假"值)， `some(...)`会一直运行直到回调函数返回`true`(或者 "真"值)。
`every(...)`和`some(...)`中特殊的返回值和普通for循环中的break语句类似，它们会提前终止遍历。
使用 `for ..in` 遍历对象是无法直接获取属性值的，因为它实际上遍历的是对象中的所有可枚举属性，你需要手动获取属性值。
`注意`: 遍历数组下标时采用的是数字顺序(for 循环或者是其他迭代器)，但是遍历对象属性时的顺序是不确定的，在不同的JavaScript引擎中可能不一样。因此，在不同的环境中需要保持一致时，一定不要相信任何观察到的顺序，它们是不可靠的。
那么如何直接遍历值而不是数组的下标(或者对象属性)呢? 幸好，ES6增加了一种用来遍历数组的 `for ..of`循环语法(如果对象本身定义了迭代器的话也可以遍历对象):

```js
var myArray = [1,2,3];
for(var v of myArray){
    console.log(v);
}
//1
//2
//3
```

`for ..of` 循环首先会向被访问对象请求一个迭代器对象，然后通过调用迭代器对象的`next()`方法来遍历所有返回值。
数组内置的 `@@iterator` ,因此 `for ..of` 可以直接应用在数组上。我们使用内置的 `@@iterator`来手动遍历数组，看看它是怎么工作的。

```js
var myArray = [1,2,3];
var it = myArray[Symbol.iterator]();
it.next();  //{value:1, done:false}
it.next();  //{value:2, done:false}
it.next();  //{value:3, done:false}
it.next(); //{done:true}
```

和数组不同，普通的对象没有内置的 `@@iterator` ,所以无法自动完成 `for ...of` 遍历。`之所以要这样做，有许多非常复杂的原因，不过简单来说，这样做是为了避免影响未来的对象类型`。
当然，你可以给任何想遍历的对象定义 `@@iterator` ，举例来说 :

```js
var myObject = {
    a:2,
    b:3
};
Object.defineProperty(myObject, Symbol.iterator,{
    enumerable:false,
    writable:false,
    configurable:true,
    value:function(){
        var o = this;
        var idx = 0;
        var ks = Object.keys(o);
        return {
            next:function(){
                return {
                    value:o[ks[idx++]],
                    done:(idx > ks.length)
                };
            }
        };
    }
});
//手动遍历 myObject
var  it = myObject[Symbol.iterator]();
it.next();// {value:2, done:false}
it.next();// {value:3, done:false}
it.next();// {value:undefined, done:true}
//用 for...of 遍历 myObject
for( var v of myObject){
    console.log(v);
}
//2
//3
```

实际上，甚至可以定义一个 "无限"迭代器，它永远不会"结束"并且总会返回一个新值(比如随机数、递增值、唯一标识符，等等)。你可能永远不会在 `for ...of`循环中使用这样的迭代器，因为它永远不会结束，你的陈故乡会被挂起:

```js
var randoms  ={
    [Symbol.iterator]:function(){
        return {
            next:function(){
                return {value:Math.random()};
            }
        };
    }
};
var randoms_pool =[];
for(var n of randoms){
    randoms_pool.push(n);
    //防止无限运行!
    if(randoms_pool.length ===100) berak;
}
```

这个迭代器会生成"`无限个`"随机数，因此我们添加了一条`break`语句，防止程序被挂起。

### 3.5 小结

JavaScript中的对象有字面形式(比如 `var a={...}`)和构造形式(比如 `var a =new Array(...)`) 。字面形式更常用，不过有时候构造形式可以提供更多选项。
许多人都以为"JavaScript中万物都是对象"，这是错误的。对象是6个 (或者是7个，取决于你的观点)基础类型之一。对象有包括function在内的子类型，不同子类型具有不同的行为，比如内部标签`[object Array]` 表示这是对象的子类型数组。
对象就是 键/值对的集合。可以通过 `.propName` 或者 `["propName"]` 语法来获取属性值。访问属性时，引擎实际上会调用内部的默认`[[Get]]`操作(在设置属性值时是`[[Put]]`),`[[Get]]` 操作会检查对象本身是否包含这个属性，如果没找到的话还会查找`[[Prototype]]`链。
属性的恶性可以通过属性描述符来控制，比如 `writable` 和`configurable`。此外，可以使用 `Object.preventExtensions(...)`, `Object.seal(...)`和`Object.freeze(...)`来设置对象(及其属性)的不可变性级别。
属性不一定包含值——它们可能是就别 getter/setter 的 "访问描述符" 。 此外，属性可以是 可枚举或者不可枚举的，这决定了它们是否会出现 在 `for ...in` 循环中。
你可以使用 ES6的 `for ...of` 语法来遍历数据结构(数组，对象等等) 中的值，`for ...of`会寻找内置或者自定义的 `@@iterator`对象并调用它的 `next()`方法来遍历数据值。

## 第四章 混合对象和类

### 4.1 类理论

类/继承描述了一种代码的组织结构形式——一种在软件中对真实世界中问题领域的建模方法。
面向对象编程强调的是数据和操作数据的行为本质上是互相关联的(当然，不同的数据有不同的行为)，因此好的设计就是把数据以及和它相关的行为打包(或者说封装)起来。这在正式的计算机科学中有时被称为数据结构。
类理论强烈建议父类和子类使用相同的方法名来表示特定的行为，从而让子类重写父类。我们之后会看到，在JavaScript代码中这样做会降低代码的可读性和健壮性。

#### 4.1.1 "类"设计模式

你可能从来没把类作为设计模式看待，讨论得最多的是面向对象设计模式，比如迭代器模式，观察者模式，工厂模式，单例模式，等等。从这个角度来说，我们似乎是在(低级)面向对象类的基础上实现了所有(高级)设计模式，可能听说过过程化编程，这种代码值包含过程(函数)调用 ，没有高层的抽象。或许老师还教过你最好使用类把过程化风格的"意大利面代码"转换成结构清晰、组织良好的代码。
当然，如果你有`函数式编程` (比如Monad) 的经验就会知道类也是非常常用的一种设计模式。但是对于其他人来说，这可能是第一次知道类并不是必须的编程基础，而是一种可选的代码抽象。
有些语言(比如Java)并不会给你选择的机会，类并不是可选的——万物皆是类。其他语言(比如C/C++或者PHP) 会提供过程化和面向类这两种语法，开发者可以选择其中一种风格或者混用两种风格。

#### 4.1.2 JavaScript中的 "类"

JavaScript属于哪一种类呢? 在相当长的一段时间里，`JavaScript`只有一些近似类的语法元素(比如 new 和instanceof)，不过在后来的 ES6中新增了一些元素，比如class关键字。
这是不是意味着 JavaScript中实际上有类呢? 简单来说 :不是。
由于类是一种设计模式，所以你可以用一些方法(本章之后会介绍)近似实现类的功能。
为了满足对于类设计模式的最普遍需求，JavaScript提供了一些近似类的语法。
虽然有近似类的语法，但是JavaScript的机制似乎一直在阻止你使用类设计模式。在近似类的表象这下，JavaScript的机制其实和类完全不同。语法糖和(广泛使用的) JavaScript "类" 库视图掩盖这个现实，但是你迟早会面对它:其他语言中的类和JavaScript中的 "类"并不一样。
总结一下，在软件设计中类是一种可选的模式，你需要自己决定是否在JavaScript中使用它。由于许多开发者都非常喜欢面向类的软件设计，我们会在本章的剩余部分中介绍如何在JavaScript中实现类以及存在的一些问题。

### 4.2 类的机制

在许多面向类的语言中，"标准库" 会提供 Stack类，他是一种 "栈"数据结构(支持压入，弹出，等等)。Stack类内部会有一些变量来存储数据，同时会提供一些公有的可访问行为("方法")，从而让你的代码可以和(隐藏的)数据进行交互(比如添加、删除数据)。
但是在这些语言中，你实际上并不是直接操作`Stack`(除非创建一个静态类成员引用，这超出了我们的讨论范围)。`Stack` 类仅仅是一个抽象的表示，它描述了所有 "栈" 需要做的事，但是它本身并不是一个 "栈"你必须先实例化`Stack`类然后才能对它进行操作。

#### 4.2.1 建造

"类" 和 "实例" 的概念来源于房屋建造。

#### 4.2.2 构造函数

类实例是由一个特殊的类方法构造的，这个方法名通常和类名相同，被称为`构造函数`。这个方法的任务就是初始化实例所需的所有信息(状态)。
类构造函数属于类，而且通常和类同名。此外，构造函数大多需要用 `new`来调，这样语言引擎才知道你想要构造一个新的类实例。

### 4.3 类的继承

在面向类的语言中，你可以先定义一个类，然后定义一个继承前者的类。
后者通常被称为 "子类" ，前者通常被称为"父类" 。

#### 4.3.1 对态

Car重写了继承自父类的drive() 方法，但是之后 Car调用了 inherited: drive() 方法，这表明Car可以引用继承来的原始drive()方法。快艇的 pilot() 方法同样引用了原始的 drive()方法。
这个技术被称为`多态`或者`虚拟多态`。在本例中，更恰当的说法是相对多态。
多态是一个非常广泛的话题，我们现在所说的"相对"只是多态的一个方面:任何方法都可以引用继承层次中高层的方法(无论高层的方法名和当前方法名是否相同)。之所以说"相对"是因为我们并不会定义想要访问的绝对继承层次(或者说类)，而是使用相对引用"查找上一层"。

#### 4.3.2 多重继承

JavaScript要简单的多:它本身并不提供"多重继承"功能。许多人认为这是件好事，因为使用多重继承的代价太高。然而这无法阻挡开发者们的热情，他们会尝试各种各样的方法来实现多重继承。

### 4.4 混入

在继承或者实例化时，`JavaScript`的对象机制并不会自动执行复制行为。简单的来说`JavaScript`中只有队形，并不存在可以被实例化的 "类" 。一个对象并不会被复制到其他对象，它们会被关联起来。
由于在其他语言中类表现出来的都是赋值行为，因此`JavaScript`开发者也想了一个方法来模拟类的复制行为，这个方法就是`混入`。接下来我们会看到两种类型的混入:`显示`和`隐式`。

#### 4.4.1 显式混入

```js
//非常简单的 mixin(...)例子:
function mixin(sourceObj, targetObj){
    for(var key in sourceObj){
        //只会在不存在的情况下复制
        if(!(key in targetObje)){
            targetObj[key] = sourceObj[key];
        }
    }
    return targetObj;
}
var Vehicle ={
    engines:1,
    ignition:function(){
        console.log("Turning on my engine");
    },
    drive:function(){
        this.ignition();
        console.log("Steering and moving forward!");
    }
};
var Car = mixin(Vehicle, {
    wheels: 4,
    drive:function(){
        Vehicle.drive.call(this);
        console.log("Rolling on all " + this.wheels + "wheels!");
    }
});
```

1. 在说多态
    我们来分析一下这条语句:`Vehicle.drive.call(this)` 。这就是我所说的`显式多态`。还记得吗，在之前的伪代码中对应的语句是 `inherited:drive()`.我们之为`相对多态`。
    `JavaScript` (在 ES6之前)并没有相对多态机制。所以，由于`Car`和`Vehicle`中都有 `drive()`函数，为了指明调用对象，我们必须使用绝对(而不是相对)引用。我们通过名称显示指定Vehicle 对象并调用它的drive()函数。
    但是如果直接执行 Vehicle.drive()，函数调用中的this会被绑定到 `Vehicle` 对象而不是Car对象，这并不是我们想要的。因此，我们会使用 .call(this) 来确保drive()在Car对象的上下文中执行。
    但是在JavaScript中(由于屏蔽)使用显示伪多态会在所有需要使用(伪)多态引用的地方创建一个函数关联，这会极大地增加维护成本。此外，由于显示伪多态可以模拟多重继承，所以它会进一步增加代码的复杂度和维护难度。
2. 混合复制
    `JavaScript`中的函数无法(用标准、可靠的方法)真正的复制，所以你只能复制对共享函数对象的引用(函数就是对象;)。如果你修改了共享函数对象(比如ignition()) ，比如添加了一个属性，那Vehicle 和 Car都会受到影响。
    `显式混入`是JavaScript中一个很棒的机制，不过它的功能也没有看起来那么的强大。虽然它可以把一个对象的属性复制到另一个对象中，但是这其实并不能带来太多的好处，无非就是少几条定义语句，而且还会带来我们刚才提到的函数对象引用问题。
    如果你向目标对象中显式混入超过一个对象，就可以部分模仿多重继承行为，但是仍没有直接的方式来处理函数和属性的同名问题。有些开发者/库提出了"晚绑定"技术和其他的一些解决方法，但是从根本上来说，使用这些"诡计"通常会(降低性能并且)得不偿失。
    一定要注意，只在能够提高代码可读性的前提下使用显示混入，避免使用增加代码理解难度或者让对象更加复杂的模式。
    如果使用混入时感觉越来越困难，那或许你应该停止使用它了。实际上，如果你必须使用一个复杂的库或者函数来实现这些细节，那就标志着你的方法是有问题的或者是不必要的。
3. 寄生继承
    `显式混入模式`的一种变体被称为"`寄生继承`"，它既是显式的又是隐式的，主要推广者是 `Douglas Crockford`。
    下面是它的工作原理:

    ```js
    //传统的 JavaScript类 Vehicle
    function Vehicle(){
        this.engines =1;
    }
    Vehicle.prototype.ignition = function(){
        console.log("Turning on my engine");
    };
    Vehicle.prototype.drive = function(){
        this.ignition();
        console.log("Steering and moving forward!");
    };
    //寄生类 Car
    function Car(){
        //首先 ，car是一个 Vehicle
        var car = new Vehicle();
        //接着我们队car进行定制
        car.wheels = 4;
        //保存到 Vehicle::drive() 的特殊引用
        var vehDrive = car.drive;
        //重写 Vehicle::drive()
        car.drive = function(){
            vehDrive.call(this);
            console.log("Rolling on all "+ this.wheels + "wheels");
        }
        return car;
    }
    var myCar = new Car();
    myCar.drive();
    //发动引擎 ;
    //手握方向盘!
    //全速前进!
    ```

    如你所见，首先我们赋值一份Vehicle父类(对象) 的定义，然后混入子类(对象)的定义(如果需要的话保留父类的特殊引用),然后用这个复合对象构建实例。

### 4.4.2 隐式混入

隐式混入和之前提到的显式伪多态很像，因此也具备同样的问题。
思考下面的代码:

```js
var Something ={
    cool:function(){
        this.greeting = "Hello World";
        this.count = this.count ? this.count +1 :1;
    };
    var Anothr = {
        cool:function(){
            //隐式把 Something 混入 Another
            Something.cool.call(this);
        }
    };

}
Another.cool();
Another.greeting;// "Hello World"
Another.count; //1 (count 不是共享状态)
```

通过在构造函数调用或者方法调用中 使用 `Something.cool.call(this)`, 我们实际上 "借用"了函数 `Something.cool()`并在 `Another`的上下文中调用了它(通过this绑定;)。最终的结果是 `Something.cool()`中的赋值操作都会应用在Another对象上而不是`Something` 对象上。
因此，我们把 `Something` 的行为 "`混入`" 到了 Another 中。
虽然这类技术利用了 this的重新绑定功能，但是 `Something.cool.call(this)`仍然无法变成相对(而且更灵活的)引用 ，所以使用时千万要小心，通常来说，尽量避免使用这样的结构，以保证代码的整洁和可维护性。

### 4.5 小结

`类` 是一种设计模式。许多语言提供了对于面向类软件设计的原始语法。JavaScript 也有类似的语法，但是和其他语言中的类完全不同。
类意味着复制。
传统的类被实例化时，它的行为被复制到了实例中。类被继承时，行为也会被复制到了子类中。
`多态` (在继承链的不同层次名称相同但是功能不同的函数) 看起来似乎是从子类引用父类，但是本质上引用的骑士是复制的结果。
JavaScript并不会(像类那样) 自动创建对象的副本。
`混入模式`(无论显式还是隐式) 可以用来模拟类的复制行为，但是通常会产生丑陋并且脆弱的语法，比如显式伪多态(OtherObj.methodName.call(this, ...)),这会让代码更加难懂并且难以维护。
此外，显式混入实际上无法完全模拟类的复制行为，因为对象(和函数! 别忘了函数也是对象)只能复制引用，无法复制引用的对象或者函数本身。忽视这一点会导致许多问题。
总的来说，在`JavaScript`中模拟`类`是得不偿失的，虽然能解决当前的问题，但是可能会埋下更多的隐患。

## 第五章 原型

### 5.1 [[Prototype]]

JavaScript中的对象有一个特殊的`[[Prototype]]`内置属性，其实就是对于其他对象的引用。几乎所有的对象在创建时`[[Prototype]]`属性都会被赋予一个非空的值。
对象的`[[Prototype]]`链接可以为空，虽然很少见。
思考下面的代码:

```js
var myObject ={
    a:2
};
myObject.a; //2
```

`[[Prototype]]` 引用有什么用呢?
当你试图引用对象的属性时会触发`[[Get]]` 操作，比如 `myObject.a`。对于默认的 `[[Get]]` 操作来说，第一步是检查对象本身是否有这个属性，如果有的话就使用它。
但是 如果 a 不在myObject 中，就需要使用对象的`[[Prototype]]`链了。
对于默认的 `[[Get]]`操作来说，如果无法在对象本身找到需要的属性，就会继续访问对象的`[[Prototype]]`链:

```js
var anotherObject = {
    a:2
};
//创建一个关联到anotherObject的对象
var myObject = Object.create(anotherObject);
myObject.a; //2
```

现在myObject对象的 `[[Prototype]]`关联到了 anotherObject。显然myObject.a并不存在，但是尽管如此，属性访问仍然成功地(在anotherObject中)找到了值2。
但是，如果 anotherObject中也找不到 a并且`[[Prototype]]`链不为空的话，就会继续查找下去。
这个过程会持续到找到匹配的属性名或者查找完整条`[[Prototype]]`链，如果是后者的话，`[[Get]]`操作的返回值是 `undefined`。
使用 `for ...in`遍历对象时原理和查找`[[Prototype]]`链类似，任何可以通过原型链访问到(并且是 `enumerable`)的属性都会被枚举。使用 `in` 操作符来检查属性在对象中是否存在，同样会查找对象的整个原型链(无论属性是否可枚举)。

#### 5.1.1 Object.prototype

但是到哪里是`[[Prototype]]`的"尽头"呢 ?
所有普通的`[[Prototype]]`链最终都会指向内置的`Object.prototype`。由于所有的"普通"(内置，不是特定主机的扩展) 对象都 "源于" (或者说把`[[Prototype]]`链的顶端设置为)这个`Object.prototype`对象，所以包含`JavaScript`中许多通用的功能。

#### 5.1.2 属性设置和屏蔽

第3章提到过，给一个对象设置属性并不是仅仅是添加一个新属性或者修改已有的属性值现在我们完整地讲解一下这个过程:
`myObject.foo ="bar"`;
如果 myObject 对象包含名为 foo 的普通数据访问属性，这条赋值语句只会修改已有的属性值。
如果 foo不是直接存在于 myObject 中，`[[Prototype]]`链就会被遍历，类似`[[Get]]`操作。
如果原型链上找不到 foo，foo 就会被直接添加到 myObject上。
然而，如果foo存在于原型链上层，赋值语句`myObject.foo = "bar"`的行为就会有些不同(而且可能很出人意料)。
如果属性名 foo 既出现在 myObject中也出现在 myObject 的`[[Prototype]]`链上层，那么就会发生屏蔽。myObject中包含的 foo属性会屏蔽原型链上层的所有foo属性，因为 myObject.foo 总是会选择原型链中最低的foo属性。
屏蔽比我们想想中的更加复杂。下面我们分析一下如果foo不直接存在于myObject中而是存在于原型链上层时`myObject.foo="bar"`会出现三种情况。

1. 如果在`[[Prototype]]`链上层存在名为`foo`的普通数据访问属性 并且没有别标记为只读(`writable:false`),那么就会直接在myObject中添加一个名为foo的新属性，它是`屏蔽属性`。

2. 如果在`[[Prototype]]`链上层存在foo,但是它别标记为只读(`writable:false`),那么无法修改已有属性或者在myObject上创建`屏蔽属性`。如果运行在`严格模式`下，代码会抛出一个错误。否则，这条赋值语句就被忽略。总之，不会发生屏蔽。

3. 如果在 `[[Prototype]]`链上层存在`foo`并且它是一个`setter`，那就一定会调用这个`setter`。`foo`不会被添加到(或者说屏蔽于)`myObject`，也不会重新定义`foo`这个 `setter`。

大多数开发者都认为如果向`[[Prototype]]`链上层已经存在的属性(`[[Put]]`)赋值，就一定会触发屏蔽，但是如你所见，三种情况中只有一种(第一种)是这样的。
如果你希望在第二种和第三种情况下也屏蔽foo,那就不同使用 = 操作符来赋值，而是使用 `Object.defineProperty(...)`来向myObject添加foo。

注意 : 第二种情况可能是最令人以意外的，只读属性会阻止`[[Prototype]]`链下层隐式创建(屏蔽)同名属性。这样做主要是为了模拟类属性的继承。你可以把原型链上层的foo看作是父类中的属性，它会被myObject继承(复制),这样一来 myObject 中的foo属性也是只读，所以无法创建。但是一定要注意，实际上并不会发生类似的继承复制。这看起来有点奇怪，myObject对象竟然会应为其他对象中有一个只读foo就不能包含foo属性。更奇怪的是，这个限制值存在于`=`赋值中，使用`Object.defineProperty(...)`并不会受影响。

如果需要对屏蔽方法进行委托的话就不得不使用丑陋的显式伪多态。通常来说，使用屏蔽得不偿失，所以应尽量避免使用。第6章会介绍一种不适用屏蔽的更加简洁的设计模式。
有些情况下会隐式产生屏蔽，一定要当心。思考下面的代码:

```js
var anotherObject = {
    a:2
};
var myObject = Object.create(anotherObject);
anoterObject.a; //2
myObject.a;   //2
anotherObject.hasOwnProperty("a") ;  //true
myObject.hasOwnProperty("a");  //false
myObject.a ++;  //隐式屏蔽!
anotherObject.a;  //2
myObject.a;  //3
myObject.hasOwnProperty("a");  // true
```

尽管`myObject.a++`看起来应该(通过委托)查找并增加`anotherObject.a`属性，但是别忘了`++`操作相当于 `myObject.a = myObject.a +1`。因此`++`操作首先会通过 `[[Prototype]]`查找属性 `a` 并从 `anotherObject.a` 获取当前属性值2，然后给这个值加1，接着用 `[[Put]]`将值3赋值给`myObject`中新建的屏蔽属性`a`，天呐 !
修改委托属性时一定小心。如果想染 `anotherObject.a` 的值增加，唯一的办法是 `anotherObject.a++`。

### 5.2 "类"

现在你可能会好奇: 为什么一个对象需要关联到另一个对象? 这样做有什么好处?这个问题非常好，但是在回答之前我们首先要理解`[[Prototype]]` "不是" 什么 。
第 4章我们说过，JavaScript和面向类的语言不通，它并没有类来作为对象的抽象模式或者说蓝图。JavaScript中只有对象。
实际上，JavaScript才是真正应该被称为"面向对象" 语言，因为它是少有的可以不通过类，直接创建对象的语言。
在JavaScript中，类无法藐视对象的行为，(因为根本就不存在类!)对象直接定义自己的行为。在说一遍，JavaScript中只有对象。

#### "类"函数

多年依賴，JavaScript中有一种奇怪的行为一直在被无耻的滥用，那就是模仿类。
这种奇怪的 "类似类"的行为利用函数的一种特殊特性:所有的函数默认都会拥有一个名为prototype的公有并且不可枚举的属性，它会指向另一个对象。
在面向类的语言中，类可以被复制(或者说实例化)多次，就像用模具制作东西一样。我们在第4章看到过，之所以会这样式因为实例化(或者继承)一个类就意味着"把类的行为复制到物理对象中"，对于每一个新实例来说都会重复这个过程。
但是在JavaScript中，并没有类似的复制机制。你不能创建一个类的多个实例，只能创建多个对象，它们`[[Prototype]]` 关联的是同一个对象。但是在默认情况下并不会进行复制，因此这些对象之间并不会完全失去联系，它们是相互关联的。
最后我们得到了两个对象，它们之间互相关联，就是这样。我们并没有初始化一个类，实际上我们并没有从"类"中复制任何行为到一个对象中，只是让两个对象互相关联。
如果你把JavaScript中对象的所有委托行为都归结到对象本身并且把对象看作是实物的话，那就(差不多)可以理解差异继承了。
但是和原型继承一样，差异继承会更多的是你脑中构建出的模型，而非真实情况。它忽略了一个事实，那就是对象B实际上并不是被差异构造出来的，我们只是定义了B的一些指定特性，其他没有定义的东西都变成了"洞" 。而这些洞(或者说缺少定义的空白处)最终会被委托行为"填满"。
默认情况下，对象不会想差异继承暗示的那样通过复制生成。因此，差异继承也不适合用来描述JavaScript的`[[Prototype]]`机制。
当然如果你喜欢，完全可以使用差异继承这个术语，但是无论如何它只适用于你脑中的模型，并不符合引擎的真实行为。

#### 5.2.2 "构造函数"

安装JavaScript世界的惯例 ，"类"名首字母要大写
这个管理影响力非常大，以至于如果你用new来调用小写方法或者不用new调用首字母大写的函数，许多JavaScript开发者都会责怪你。这很令人吃惊，我们竟然会如此女里地维护JavaScript(假)"面向类"的权利，尽管对于JavaScript引擎来说首字母大写没有任何意义。

1. 构造函数还是调用
    在JavaScript中对于"构造函数"最准确的解释是，所有带new的函数调用。

#### 5.2.3 技术

我们是不是已经介绍了JavaScript中所有和"类"相关的问题了呢?
不是。JavaScript开发者绞尽脑汁想要模仿类的行为:

```js
function Foo(name){
    this.name = name;
}
Foo.prototype.myName = function(){
    return this.name;
};
var a = new Foo("a");
var b = new Foo("b");
a.myName();  // "a"
b.myName();  //"b"
```

这段代码展示了另外两种 "面向类" 的技巧:

1. `this.name = name` 给每个对象(也就是 a 和b ，参见第2章中的this绑定)都添加了 `.name`属性，有点像类实例封装的数据值。
2. `Foo.prototype.myName = ...` 可能更有趣的技巧，它会给`Foo.prototype`对象添加一个属性(函数)。现在，`a.myName()` 可以正常工作，但是你可能会觉得很惊讶，这是什么原理呢?

在这段代码中，看起来似乎创建 a 和b时会把`Foo.prototype`对象复制到这两个对象中，然而事实并不是这样。
在本章开头介绍默认`[[Get]]`算法时我们介绍过 `[[Prototype]]`链，以及当属性不直接存在于对象中时如何通过它来进行查找。
因此，在创建的过程中 ，a 和b的内部 `[[Prototype]]`都会关联到`Foo.prototype`上找到。

#### 回顾"构造函数"

之前讨论 `.constructor`属性时我们说过，看起来`.constructor === Foo`为真意味着a确实一个指向Foo的`.constructor`属性，但是事实不是这样的。
这里一个很不幸的误解。实际上，`.constructor` 引用同样被委托了`Foo.prototype`,而 `Foo.prototype.constructor`默认指向Foo。
把 `.constructor`属性指向Foo看作是 a 对象由 Foo "构造"非常容易理解，但这只不过是一种虚假的安全感。`a.constructor`只是通过默认的`[[Prototype]]`委托指向Foo ，这和 "构造毫无关系。相反，对于`.consructor`的误解理解很容易对你自己产生误导。
举例来说，`Foo.prototype`的`.constructor`属性只是Foo函数在声明时的默认属性。如果你创建了一个新对象替换了函数默认的`.prototype`对象引用，那么新对象并不会自动获得`.constructor`属性。
思考下面的代码:

```js
function Foo(){ /* ..*/}
Foo.prototype = { /* ..*/}; //创建一个新原型对象
var a1 = new Foo();
a1.constructor ===Foo ; //false!
a1.construcotr ===Object; //false
```

`Object(...)` 并没有 "构造" a1 ,对吧? 看起来应该是 Foo() "构造" 了它。大部分开发者都认为是 Foo() 执行了构造工作，但是问题在于，如果你认为"constructor"表示 "由 .....构造"的话，a1.constructor 应该是 Foo，但是它并不是 Foo！
到底怎么回事?
a1并没有 .constructor 属性，所有它会委托`[[Prototype]]`链上的`Foo.prototype`。但是这个对象也没有 `.constructor`属性(不过默认的`Foo.prototype`对象有这个属性!),所以它会继续委托，这次会委托给委托链顶端的`Object.Prototype`。这个对象有 `.constructor`属性。指向内置的`Object(...)`函数。
当然，你可以给 `Foo.prototype`添加一个 `.constructor`属性，不过这需要手动添加一个符合正常行为的不可枚举(参见第3章)属性。
举例来说:

```js
function Foo(){ /*..*/}
Foo.prototype ={/* ..*/};  //创建一个新原型对象
//需要在 Foo.prototype 上 "修复" 丢失的 .constructor属性
//新对象属性起到 Foo.prototype 的作用
// 关于 defineProperty(...), 参见第3章
Object.defineProperty(Foo.prototype,"constructor",{
    enumerable:false,
    writable:true,
    configurable:true,
    value:Foo  // 让 .constructor 指向 Foo
});
```

修复`.constructor`需要很多手动操作。所有这些工作都是源于把 "constructor" 错误地理解为 "由....构造"，这个误解的代价是在太高了。
实际上，对象的 `.construcotr`会默认指向一个函数，这个函数可以通过对象的 `.prototype`引用。"constructor"和"prototype"这两个词本身的含义可能使用也可能不适用。最好的办法是记住这一点"constructor并不表示被构造"。
`.constructor`并不是一个不可变属性。它是不可枚举(参见上面的代码)的，但是它的值是可写的(可以被修改)。此外，你可以给任意`[[Prototype]]`链中的任意对象添加一个名为 `constructor`的属性或者对其进行修改，你可以任意对其赋值。
和`[[Get]]`算法查找`[[Prototype]]` 链的机制一样, `.constructor`属性引用的目标可能和你想的完全不同。
结论 ?
一些随意的对象属性引用，比如`a1.constructor`,实际上是不被信任的，它们不一定会指向默认的函数引用。此外很快沃恩就会看到，稍不留神`a1.construcotr`就可能会指向你意想不到的地方。
`a1.constructor`是一个非常不可靠并且不安全的引用。通常来说要尽量避免使用这些引用。

### 5.3 (原型) 继承

我们已经看过了许多JavaScript程序中常用的模拟类行为的方法，但是如果没有"继承"机制的话，JavaScript中的类就只是一个空架子。
经典的`原型风格` 就是下面这种:

```js
function Foo(name){
    this.name = name;
}
Foo.prototype.myName = function(){
    return this.name;
};
function Bar(name, label){
    Foo.call(this, name);
    this.label = label;
}
// 我们创建了一个新的 Bar.prototype 对象并关联到 Foo.prototype
Bar.prototype = Object.create(Foo.prototype) ;
//注意! 现在没有 Bar.prototype.constructor 了
//如果你需要这个属性的话 可能需要手动修复一下它
Bar.prototype.myLable = function(){
    return this.label;
};
var a = new Bar("a", "obj a");
a.myName();  // "a"
a.myLabel();   // "obj a"
```

这段代码的核心部分就是 语句`Bar.prototype = Object.create(Foo.prototype)` 。 调用 `Object.create(...)`会凭空创建一个 "新"对象并把新对象内部的`[[Prototype]]`关联到你指定的对象(本例中是 Foo.prototype) 。
换句话说，这句话的意思是:"创建一个新的Bar.prototype对象并把它关联到Foo.prototype"。
声明 `function Bar(){...}`时，和其他函数一样，Bar会有一个.prototype 关联到默认对象，但是这个对象并不是我们想要的Foo.prototype 。因此我们创建一了一个新对象并把它关联到我们希望的对象上，直接把原始的关联对象抛弃掉。
注意 ，下面的这两种方式是常见的错误做法，实际上它们都存在一些问题:

```js
//和你想要的机制不一样!
Bar.prototype = Foo.prototype;
//基本上满足你的需求，但是可能会产生一些副作用:
Bar.prototype = new Foo();
```

`Bar.prototype = Foo.prototype`并不会创建一个关联到`Bar.prototype`的新对象 ，它只是让`Bar.prototype`直接引用 Foo.prototype 对象。因此当你执行类似Bar.prototype.myLable = .... 的赋值语句时会直接修改Foo.prototype 对象本身。显然这不是你想要的结果，否则你根本不需要Bar对象，直接使用Foo就可以了，这样代码也会更简单一些。
`Bar.prototype = new Foo()`的确会创建一个关联到Bar.prototype的新对象。但是它使用了Foo(...)的"构造函数调用" ，如果函数Foo有一些副作用(比如写日志、修改状态、注册到其他对象、给this添加数据属性，等等)的话，就会影响到Bar()的 "后代"，后果不堪设想。
因此，要创建一个合适的关联对象，我们必须使用`Object.create(...)`而不是使用具有副作用的Foo(...)。这样做唯一的缺点就是需要创建一个新对象然后把旧对象抛弃掉，不能直接修改已有的默认对象。
如果能有一个标准并且可靠的方法来修改对象的 `[[Prototype]]` 关联就好了。在ES6之前，我们只能通过设置`.__proto__`属性来实现，但是这个方法并不是标准并且无法兼容所有浏览器。ES6添加了辅助函数 `Object.setPrototypeOf(...)`,可以用标准并且可靠的方法来修改关联。
我们来对比一下两种把 Bar.prototype关联到 Foo.prototype 的方法:

```js
//ES6之前需要抛弃默认的Bar.prototype
Bar.prototype =Object.create(Foo.prototype);
//ES6开始可以直接修改现有的Bar.prototype
Object.setPrototypeOf(Bar.prototype, Foo.prototype);
```

如果忽略掉`Object.create(...)`方法带来的轻微性能损失(抛弃的对象需要进行垃圾回收)，实际上比ES6极其之后的方法更短而且可读性更高。不过无论如何，这是两种完全不同的语法。

#### 检查"类"关系

假设有对象a，如何寻找对象a委托的对象(如果存在的话)呢? 在传统的面向类环境中，检查一个实例(JavaScript中的对象) 的继承祖先(JavaScript中的委托关联)通常被称为内省(或者反射)。
思考下面的代码:

```js
function Foo(){
    //...
}
Foo.prototype.blah = ...;
var a = new Foo();
```

我们如何通过内省找出a的"祖先" (委托关联)呢?第一种方法是站在 "类"的角度来判断:

```js
a instanceof Foo; // true
```

`instanceof`操作符的左操作数是一个普通对象，右操作数是一个函数。`instanceof`回答的问题是: 在 a的整条`[[Prototype]]`链中是否有指向`Foo.prototype`的对象？
可惜，这个方法只能处理对象(a)和函数 (带.prototype 引用的Foo)之间的关系。如果想判断两个对象(比如a和b)之间是否通过`[[Prototype]]`链关联，只用`instanceof`无法实现。
注意:如果使用内置的.bind(...)函数来生成一个硬绑定函数的话，改函数是没有 `.prototype`属性的。在这样的函数上使用`instanceof`的话，目标好书的`.prototype`会代替硬绑定函数的`.prototype`。
通常我们不会在"构造函数调用"中使用硬绑定函数，不过如果你这么做的话，实际上相当于直接调用目标函数。同理，在硬绑定函数上使用instanceof也相当于直接在目标函数上使用instanceof。
下面这段荒谬的代码试图站在"类"的角度使用instanceof 来判断两个对象的关系:

```js
//用来判断 o1是否关联到 (委托)o2的辅助函数
function isRelatedTo(o1,o2){
    function F(){}
    F.prototype = o2;
    return o1 instanceof F;
}
var a = {};
var b = Object.create(a);
isRelatedTo(b, a);  // true
```

在 isRelatedTo(..)内部我们声明了一个一次性函数F， 把它的`.prototype`重新赋值并指向对象o2,然后判断o1 是否是F的一个"实例"。显而易见，o1实际上并没有继承F也不是由F构造，所以这种方法非常愚蠢并且容易造成误解。问题的关键在于思考的角度，强行在JavaScript中应用类的语义(在本例中就是使用instanceof)就会造成这种尴尬的局面。
下面是第二种判断`[[Prototype]`反射的方法，它更简洁:

```js
Foo.prototype.isPrototypeOf(a);   //true
```

注意，在本例中，我们实际上并不关心(甚至不需要)Foo, 我们只需要一个可以用来判断的对象(本例中是Foo.prototype )就行。isPrototypeOf(...)回答的问题是: 在a的在整条[[Prototype]]链中事否出现过Foo.prototype ?
同样的问题，同样的答案，但是在第二种方法中并不需要简介引用函数(Foo),它的.prototype 属性会被自动访问。
我们只需要两个对象就可以判断它们之间的关系。举例来说:

```js
//非常简单: b 是否出现在 c 的 [[Prototype]]链中?
b.isPrototypeOf(c);
```

注意，这个 方法并不需要使用函数("类") ，它直接使用b和c之间的对象引用来判断它们的关系。换句话说，语言内置isPrototypeOf(...)链。在ES5中，标准的方法是:

```js
Object.getPrototypeOf(a);
```

可以验证一下，这个对象引用是否和我们想的一样:

```js
Object.getPrototypeOf(a) === Foo.prototype;  //true
```

绝大多数(不是所有!) 浏览器也支持一种非标准的方法来访问内部`[[Prototype]]`属性:
`a.__proto__ === Foo.prototype ; //true`
和我们之前说过的`.constructor`一样，`._proto__`实际上并不存在于你正在使用的对象中(本例中是a)。实际上，它和其他的常用函数(.toString(), .isPrototypeOf(...),等等)一样，存在于内置的`Object.prototype`中。(它们是不可枚举的)
此外，`.__proto__`看起来像一个属性，但是实际上它更像一个 `getter/setter`。
`.__proto__`的实现大致上是这样的:

```js
Object.defineProperty(Object.prototype, "__proto__",{
    get:function(){
        return Object.getPrototypeOf(this);
    },
    set:function(o){
        //ES6中的 setPrototypeOf(...)
        Object.setPrototypeOf(this, o);
        return o;
    }
});
```

因此，访问(获取值) `a.__proto__`时，实际上是调用了`a.__proto__()`(调用`getter` 函数)。虽然 getter函数存在于`Object.prototype`对象中，但是它的this指向对象a，所以和 `Object.getPrototypeOf(a)` 结果相同。
`.__proto__`是可设置属性，之前的代码中使用 ES6的`Object.setPrototypeOf(...)`进行设置。然而，通常说你不需要修改已有对象的`[[Prototype]]`。
一些框架会使用非常复杂和高级的技术来实现"子类"机制，但是通常来说，我们不推荐这种用法，因为这会极大地增加代码的阅读难度和维护难度。
我们只有在一些特殊情况下(我们前面讨论过) 需要设置函数默认.prototype 对象的`[[Prototype]]`,让它引用其他对象(除了Object.prototype)。这样可以避免使用全新的对象替换 默认对象。此外最好把`[[Prototype]]`对象关联看作是只读特性，从而增加代码的可读性。
延伸:
JavaScript社区对于双下划线有一个非官方的称呼，他们会把类似`__proto__`的属性称为"笨蛋(`dunder`)"。所以，JavaScript潮人会把`__proto__`叫做 "笨蛋proto"

### 对象关联

现在沃恩知道了，`[[Prototype]]`机制就是存在于对象中的一个内部链接，它会引用其他对象。
通常来说，这个连接的作用是:如果在对象上没有找到需要的属性或者方法引用，引擎就会继续在`[[Prototype]]`关联的对象上进行查找。同理，如果在后者中也没有找到需要的引用就会继续查找它的`[[Prototype]]`，以此类推。这一系列对象的链接被称为"原型链"

#### 5.4.1 创建关联

我们已经明白了为什么JavaScript的`[[Prototype]]`机制和类不一样，也明白了它如何建立对象间的关联。那`[[Prototype]]`机制的意义是什么呢? 为什么JavaScript开发者费这么大力气(模拟类)在代码中创建这些关联呢?
还记得吗，本章前面曾经说过`Object.create(...)`是一个大英雄,现在是时候来弄明白为什么了:

```js
var foo = {
    something:function(){
        console.log("Tell me something good...");
    };
    var bar = Object.create(foo);
    bar.something();   //Tell me something good ....
}
```

Object.create(...) 会创建一个新对象 (bar) 并把它关联到我们制定的对象(foo) ,这样我们就可以充分发挥`[[Prototype]]` 机制的威力(委托)并且避免不必要的麻烦(比如使用new的构造函数调用会生成.prototype 和.constructor引用)。

扩展: `Object.create(null)` 会创建一个拥有空(或者说null)`[[Prototype]]`链接的对象，这个对象无法进行委托。由于这个对象没有原型链，所以instanceof操作符(之前解释过)无法进行判断，因此总是会返回false。这些特殊的空`[[Prototype]]`对象通常被称作为"字典"，它们完全不会受到原型链的干扰，因此非常适合用来存储数据。
我们并不需要类来创建两个对象之间的关系，只需要通过委托来关联对象就足够了。而`Object.create(...)`不包含任何"类的诡计" ，所以它可以完美地创建想要的关联关系。
`Object.create(...)`的`polyfill`代码
Object.create(...)是在 ES5中新增的函数，所以在ES5之前的环境中(比如旧IE)如果要支持这个功能的话就需要使用一段简单的polyfill代码，它部分实现了`Object.create(...)`的功能:

```js
if(!Object.create){
    Object.create = function(o){
        function F() {}
        F.prototype = o;
        return new F();
    }
}
```

这段polyfill 代码使用了一个一次性函数F ，我们通过改写它的.prototype 属性使其指向想要关联的对象，然后使用 new F() 来创造一个新对象进行关联。
由于 `Object.create(..c)` 可以被模拟，因此这个函数被应用的fico广泛。标准ES5中内置的`Object.create(...)`函数还提供了一系列附加功能，但是ES5之前的版本不支持这些功能。通常来说，这些功能的应用范围要小的多，但是出于完整性考虑，我们还是介绍一下：

```js
var anotherObject ={
    a:2
};
var myObject = Object.create(anotherObject, {
    b:{
        enumerable:false,
        writable:true,
        configurable:false,
        value:3
    },
    c:{
        enumerable:true,
        writable:false,
        configurable:false,
        value:4
    }
});
myObject.hasOwnProperty("a");   //false
myObject.hasOwnProperty("b");   //true
myObject.hasOwnProperty("c");   //true
myObject.a;  //2
myObject.b; //3
myObject.c;  //4
```

`Object.create(...)`的第二个参数指定了需要添加到新对象中属性名以及这些属性的属性。
描述符。因为ES5之前的版本无法模拟属性操作符，所以polyfill 代码无法实现这个附加功能。
通常来说并不会使用`Object.create(...)`的附加功能，所以对于大多数的开发者来说，上面那段polyfill代码就足够了。
有些开发者更加严谨，我们认为只有能被完全模拟的函数才应该使用polyfill代码。由于`Object.create(...)`是只能部分模拟的函数之一，所以这些狭隘的人认为如果你需要在ES5之前的环境中使用`Object.create(..)`的特性，那不要使用polyfill代码，而是使用一个自定义函数并且名字不能是`Object.create` 。你可以把你自己的函数定义成这样:

```js
function createAndLinkObject(o){
    function F(){}
    F.prototype = o;
    return new F();
}
var anotherObject = {
    a:2
};
var myObject = createAndLinkObject(anotherObject);
myObject.a;  // 2
```

我并不赞同这个严格的观点，相反，我很赞同在ES5中使用上面那段polyfill代码。如何选择取决于你。

#### 5.4.2 关联关系是备用

看起来对象之间的关联关系是处理"缺失"属性或者方法时的一种备用选项。这个说法有点道理，但是我认为这并不是`[[Prototype]]`的本质。
思考下面的代码:

```js
var anotherObject = {
    cool:function(){
        console.log("cool!");
    }
};
var myObject = Object.create(anotherObject);
myObject.cool();  //"cool !"
```

由于存在`[[Prototype]]` 机制，这段代码可以正常工作。但是如果你这样写只是为了让myObject 在无法处理属性或者方法时可以使用备用的anotherObject, 那么你的软件就会变得有点"神奇"，而且很难理解和维护。
这并不是说任何情况下都不应该选择备用这种设计模式，但是这在JavaScript中并不是很常见，所以如果你使用的是这种模式，那或许应当退后一步并重新思考一下这种模式是否合适。
提示:
在 ES6中有一个被称为"代理(Proxy)"的高端功能，它实现额就是"方法无法找到时"的行为。
千万不要忽略这个微妙但是非常重要的区别。
当你给开发者设计软件时，假设要调用myObject.cool() ，如果myObject中不存在cool()时这条语句也可以正常工作的话，那你的 API设计就会变得很"神奇" ，对于未来维护你软件的开发者来说这可能不太好理解。
但是你可以让你的API设计不那么"神奇"，同时仍然能发挥`[[Prototype]]`关联的威力:

```js
var anotherObject = {
    cool:function(){
        console.log("cool!");
    }
};
var myObject = Object.create(anotherObject);
myObject.doCool = function(){
    this.cool();  //内部委托
};
myObject.doCool();   // "Cool!"
```

这里我们调用 myObject.doCool() 是实际存在于myObject 中的，这可以让我们的API设计更加清晰(不那么神奇)。从内部来说，我们的实现遵循的是委托设计模式,通过`[[Prototype]]` 委托到`anotherObject.cool()` 。
换句话说，内部委托比起直接委托可以让API接口设计更加清晰。下一章我们会详细解释委托。

### 5.5 小结

如果要访问对象中并不存在的一个属性，`[[Get]]`操作就会查找对象内部`[[Prototype]]`关联关系实际上定义了一条"原型链"(有点像嵌套的作用域链)，在查找属性时会对它进行遍历。
所有普通对象都有内置的`Object.prototype`，指向原型链的顶端(比如说全局作用域)，如果在原型链中找不到指定的属性就会停止。toString()、valueOf()和其他一些通用的功能都存在于Object.prototype 对象上，因此语言中所有的对象都可以使用它们。
关联两个对象最常用的方法是使用new关键词进行函数调用，在调用的4个步骤(第二章)中会创建一个关联其他对象的新对象。
使用new调用函数时会把新对象的.prototype 属性关联到 "其他对象"。带new的函数调用通常被称为"构造函数调用" ，尽管它们实际上和传统面向类语言中的类构造函数不一样。
虽然这些JavaScript机制和传统面向类语言中的"类初始化" 和"类继承"很相似但是JavaScript中的机制有一个核心区别，那就是不会进行复制，对象之间是通过内部的 `[[Prototype]]`链关联的。
出于各种原因，以"继承"结尾的术语，因为对象之间的关系不是复制而是委托。

## 6 行为委托

回顾一下第5章的结论:`[[Prototype]]`机制就是指对象中的一个内部链接引用另一个对象。
如果在第一个对象上没有找到需要的属性或者方法引用，引擎就会继续在`[[Prototype]]`关联的对象上进行查找。同理，如果在后者中也没有找到需要的引用就会继续查找它的`[[Prototype]]`,以此类推。这一系列对象的连接被称为"原型链"。
换句话说，JavaScript中的这个机制的本质就是对象之间的关联关系。

### 6.1 面向委托的设计

为了更好地学习如何更直观地使用`[[Prototype]]`,我们必须认识到它代表的是一种不同于类的设计模式。
注意:面向类的设计中有些原则依然有效，因此不要把所有的只是都抛掉。(只需要抛掉大部分就够了!)举例来说，封装是非常有用的，它同样可以应用在委托中(虽然不太常见)。
我们需要试着把思路从类和继续的设计模式转换到委托行为的设计模式。如果你在学习或者工作的过程中几乎一直在使用类，那转换思路可能不太自然并且不太舒服。你可能需要多重复几次才能熟悉这种思维模式。

### 6.1.1 类理论

假设我们需要在软件中建模一些类似的任务("XYZ", "ABC" 等)。
如果使用类，那设计方法可能是这样的:定义一个通用父(基类)，可以将其命名为Task,在Task类中定义所有认为都有的行为。接着定义子类XYZ, ABC, 它们都继承自Task并且会添加一些特殊的行为来处理对应的任务。
非常重要的是，类设计模式鼓励你在继承时使用方法重写(和多态)，比如说在XYZ任务中重写Task中定义的一些通用方法，甚至在添加新行为时通过super调用这个方法的原始版本。你会发现许多行为可以先"抽象"到父类然后在用子类 进行特殊化(重写)。
下面是对应的伪代码:

```js
class Task{
    id;
    //构造函数Task()
    Task(ID){id=ID;}
    outputTask(){output(id); }
}
class XYZ inherits Task{
 label;
 //构造函数XYZ()
 XYZ(ID,Label) {super(ID); Label =Label;}
 outputTask(){super(); output(label)}
}
class ABC inherits Task{
    // ...
}
```

现在你可以实例化子类XYZ的一些副本然后使用这些实例来执行任务 "XYZ"。这些实例会复制Task定义的通用行为以及XYZ定义的特殊行为。同理，ABC类的实例也会复制Task的行为和ABC的行为。在构造完成后，你通常只需要操作这些实例(而不是类)，因为每个实例都有你需要完成任务的所有行为。

#### 6.1.2 委托理论

但是现在我们试着来使用委托行为而不是类来思考同样的问题。
首先你会定义一个名为`Task`的对象(和许多`JavaScript`开发者告诉你的不同，它既不是`类`也不是`函数`)，它会包含所有任务都可以使用(写作使用，读作委托)的具体行为。接着，对于每个任务("`XYZ`", "`ABC`")你都会定义一个对象来存储对应的数据和行为。你会把特定的任务对象都关联到Task功能对象上，让它们在需要的时候可以进行委托。
基本上你可以想象成，执行任务"XYZ" 需要把两个兄弟对象(XYZ和Task)协作完成。但是我们并不需要把这些行为放在一起，通过类的复制，我们可以把它们分别放在各自独立的对象中，需要时可以运行XYZ对象委托给Task。
下面是推荐的代码行为，非常简单:

```js
Task = {
    setID:function(ID){this.id = ID;} ,
    outputID:function() {console.log(this.id);}
};
// 让XYZ委托 Task
XYZ = Object.create(Task);
XYZ.prepareTask = function(ID,Label){
    this.setID(ID);
    this.label = Label;
};
XYZ.outputTaskDetails = function (){
    this.outputID();
    console.log(this.label);
}
//ABC = Object.create(Task);
//ABC .... =  ....
```

在这段代码中，Task和XYZ 并不是类(或者函数)，它们是对象。XYZ通过 Object.create(...)创建，它的`[[Prototpye]]`委托了Task对象。
相比于面向类(或者说面向对象)，我会把这种编码风格称为"`对象关联`"(OLOO,objects linked to other objects)。我们真正关心的只是XYZ对象(和ABC对象)委托了Task对象。
在JavaScript中，`[[Prototype]]`机制会把对象关联到其他对象。无论你多么努力地说服自己，JavaScript中就是没有类似"类"的抽象机制。这有点像逆流而上:你确实可以这么做，但是如果你选择对抗事实，那要达到目的就显然会更加困难。
对象关联风格的diamante还有一些不同之处。

1. 在上面的代码中，id和label数据成员都是直接存储在XYZ上(而不是Task)。通常来说，在`[[Prototype]]`委托中最好把状态保存在委托者(XYZ,ABC)而不是委托目标(Task)上。
2. 在类设计模式中，我们故意让父类(Task)和子类(XYZ)中都有outputTask方法，这样就可以利用重写(多态)的优势。在委托行为中则恰好相反:我们会尽量避免在`[[Prototype]]`链的不同级别由使用相同命名。否则就需要使用笨拙并且脆弱的语法来消除引用歧义。这个设计模式要求尽量少使用容易被重写的通用方法名，提倡使用更有描述性的方法名，尤其是要写清相应对象行为的类型。这样做实际上可以创建出更容易理解和维护的代码，因为方法名(不仅在定义的位置，而是贯穿整个代码)更加清晰。
3. this.setID(ID); XYZ 中的方法首先会寻找XYZ自身是否有setID(...)，但是XYZ中并没有这个方法名。此外，由于调用位置触发了this的隐式绑定规则，因此虽然setID(...)方法在Task中，运行时this仍然会绑定到XYZ，这正是我们想要的。在之后的代码中我们还会看到this.outputID(),原理相同。

换句话说，我们和XYZ进行交互时可以使用Task中的通用方法，因为XYZ委托了Task。
`委托行为`意味着某些对象(XYZ)在找不到属性或者方法引用时会把这个请求委托给另一个对象(Task)。
这是一种极其强大的设计模式，和父类、子类、继承、多态等概念完全不同。在你的脑海中对象并不是按照父类到子类的关系垂直组织的，而不是通过任意方向的委托关联并排组织的。
注意: 在API接口的设计中，委托最好在内部实现，不要直接暴露出去。在之前的例子中我们并没有让开发者通过API直接调用XYZ.setID().(当然，可以这么做!)相反，我们把委托隐藏在了API的内部，XYZ.prepareTask(..)会委托Task.setID(...)。

1. 互相委托(禁止)
    你无法在两个或两个以上互相(双向)委托对象之间创建循环委托。如果你把B关联到A然后试着把A关联到B，就会出错。
    很遗憾(并不是非常出乎意料，但是有点烦人)这种方法是被禁止的。如果你引用了一个两边不存在的属性或者方法，那就会在`[[Prototype]]`链上产生一个无线递归的循环。但是如果所有的引用都被严格限制的话，B是可以委托A的，反之亦然。因此，互相委托理论上是可以正常工作的，在某些情况下这是非常有用的。
    之所有要禁止互相委托，是因为引擎的开发者们发现在设置时检查(并禁止!)一次无限循环引用要更加高效，否则每次从对象中查找属性时都需要进行检查。
2. 调试
    我们来简单介绍一个容易让开发者感到迷惑的细节。通常来说，JavaScript规范并不会控制浏览器中开发者工具对于特定值或者结构的表示方式，浏览器和引擎可以自己选择合适的方式来进行解析，因此浏览器和工具的解析结果并不一定相同。比如，下面这段代码的结果只能在chrome的开发者工具中才能看到。
    这段传统的 "类构造函数"JavaScript代码在Chrome开发者工具的控制台中结果如下所示:

    ```js
    function Foo(){}
    var a1 = new Foo();
    a1; //Foo{}
    ```
    我们看到代码的最后一行:表达式a1的输出是Foo{}。如果你在Firefox中运行同样的代码会得到Object{}.为什么会这样呢?这些输出是什么意思呢?
    chrome实际上想说的是"{}是一个空对象,由名为Foo的函数构造"。Firefox想说的是"{}是一个空对象，由Object构造"。之所以有这种细微的差别，是因为chrome会动态跟踪并把实际执行构造过程的函数名当作一个内置属性，但是其他浏览器并不会跟踪这些额外的信息。
    看起来可以用JavaScript的机制来解释chrome的跟踪原理:
    ```js
    function Foo(){}
    var a1 =new Foo();
    a1.constructor ; // Foo(){}
    a1.constructor.name; //"Foo"
    ```

    Chrome是不是直接输出了对象的`.constructor.name` 呢?令人迷惑的是，答案是"既是又不是"。
    思考下面的代码:

    ```js
    function Foo(){}
    var a1 = new Foo();
    Foo.prototype.constructor = function Gotcha(){};
    a1.constructor ; // Gotcha(){}
    a1.constructor.name; // "Gotcha"
    a1; //Foo{}
    ```

    即使我们把`a1.constructor.anem`修改为另一个合理的值(Gotcha),chrome控制台仍然会输出Foo。
    看起来之前那个为题(是否使用`.constructor.name`?)的答案是 "不是";Chrome在内肯定是服务另一种方式进行跟踪。
    别着急!我们先看看下面这段代码:

    ```js
    var Foo = {};
    var a1 = Object.create(Foo);
    a1; // Object{}
    Object.defineProperty(Foo, "constructor",{
        enumerable:false,
        value:function Gotcha()
    });
    a1; //Gotcha{}
    ```

    啊哈!抓到你了(Gotcha的意思就是抓到你了)!本例中chrome的控制台确实使用了`.constructor.name`。实际上，在编写本书时，这个行为被认定是chrome的一个bug，当你读到此书时，它可能已经被修复了。所以你看到的可能a1; //Object{}。
    除了这个bug,chrome内部跟踪(只用于调试输出) "构造函数名称"的方法是chrome自身的一种扩展行为，并不包含在JavaScript的规范中。
    如果你并不是使用"构造函数"来生成对象，比如使用本章介绍的对象关联风格来编写代码，那chrome就无法跟踪对象内部的"构造函数名称"，这样的对象输出是Object {} , 意思是 "Object()构造出的对象"。
    当然，这并不是对象关联风格diamante的缺点。当年使用对象关联风格编写带啊并使用行为委托设计模式时，并不需要关注是谁"构造了"对象(就是使用new调用的那个函数)。只有使用类风格来编写代码时 chrome内部到的 "构造函数名称"跟踪才有意义，使用对象关联时这个功能不起任何作用。

#### 6.1.3 比较思维模型

现在你已经明白了"类"和"委托"这两种设计模式的理论区别，接下来我们看看它们在思维模型方面的区别。
我们会通过一些示例(Foo,Bar)代码来比较一下两种设计模式(面向对象和对象关联)具体的实现方法。下面是典型("原型")面向对象风格:

```js
function Foo(who){
    this.me = who;
}
Foo.prototype.identify = function(){
    return "I am " + this.me ;
};
function Bar(who){
    Foo.call(this, who);
}
Bar.prototype = Object.create(Foo.prototype);
Bar.prototype.speak = function(){
    alert("Hello, " + this.identify() + ".");
};
var b1 = new Bar("b1");
var b2 = new Bar("b2");
b1.speak();
b2.speak();
```

子类Bar继承了父类 Foo ，然后生成了b1和b2两个实例。b1委托了`Bar.prototype`,后者委托了Foo.prototype .这种风格很常见，你应该很熟悉了。
下面我们看看如何使用对象关联风格来编写功能完全相同的代码:

```js
Foo = {
    init:function(who){
        this.me = who;
    },
    identify:function(){
        return "I am" + this.me;
    }
}
Bar = Object.create(Foo);
Bar.speak = function(){
    alert("Hello, "+ this.identify() + ".");
};
var b1 = Object.create(Bar);
b1.init("b1");
var b2 = Object.create(Bar);
b2.init("b2");
b1.speak();
b2.speak();
```

这段代码找那个我们同样利用`[[Prototype]]`把`b1`委托给`Bar`并把`Bar`委托个`Foo`，和上一段代码一模一样。我们仍然实现了三个对象之间的关联。
但是非常重要的一点是，这段代码简洁了许多，我们只是把对象关联起来，并不需要那些既复杂又令人困惑的模仿类的行为(构造函数、原型以及`new`)。
问问你自己:如果对象关联风格的代码能够实现类风格代码的所有功能并且更加简洁易懂，那它是不是比类风格更好?
下面我们看看两段代码对应的思维模型。
首先，类风格代码思维模型强调实体以及实体间的关系:
    ![avatar](https://raw.githubusercontent.com/lucky51/doc/master/javascript/You%20don't%20konw%20js/images/6-1.png)
实际上这张图有点不清晰/误导人，因为它还展示了许多技术角度不需要关注的细节(但是你必须理解它们)!从图中可以看出这是一张十分复杂的关系图。因此，如果你跟着图中的箭头走就会发现，JavaScript机制有很强的内部连贯性。
举例来说，JavaScript中的函数之所以可以访问 `call(...)`、`apply(...)`和`bind(..)`，就是因为函数本身是对象。而函数对象同样有`[[Prototype]]`属性并且关联到`Function.prototype`对象，因此所有函数对象都可以通过委托调用这些默认方法。JavaScript能做到这一点，你也可以!
好，下面我们来看一张简化版的图，它更"清晰"一些————只展示了必要的对象和关系:
![avatar](https://raw.githubusercontent.com/lucky51/doc/master/javascript/You%20don't%20konw%20js/images/6-2.png)
仍然很复杂，是吧?虚线表示的是`Bar.prototype`继承`Foo.prototype`之后丢失的`.constructor`属性引用，它们还没有被修复。即使移除这些虚线，这个思维模型在你处理对象关联时非常复杂。

现在我们看看对象关联风格代码的思维模型:
![avatar](https://raw.githubusercontent.com/lucky51/doc/master/javascript/You%20don't%20konw%20js/images/6-3.png)
通过比较可以看出，对象关联风格的代码显然更加简洁，因为这种代码值关注一件事:`对象之间的关联关系`。
其他的"类"技巧都是非常复杂并且令人困惑的。去掉它们之后，事情会变得简单许多(同时保留所有功能)。

### 6.2 类与对象

我们已经看到了"类"和"行为委托"在理论和思维模型方面的区别,现在看看在真是场景中如何应用这些方法。首先看看web开发中非常典型的一种前端场景:创建UI控件(按钮、下拉列表，等等)。

### 6.2.1 控件 "类"

你可能已经习惯了面向对象设计模式，所以很快会想到一个包含所有通用控件行为的父类(可能叫作`Widget`)和继承父类的特殊控件子类(比如Button)。
注意: 这里将使用jQuery来操作DOM和CSS,因为这种操作和我们现在讨论的内容没有关系，这些代码并不关注你是否使用，或使用哪种`JavaScript`框架(jQuery,Dojo,YUI,等等) 来解决问题。
下面这段代码展示的是如何在不使用任何"类"辅助库或者语法的情况下，使用纯`JavaScript`实现类风格的代码:

```js
//父类
function Widget(width, height){
    this.width = width || 50;
    this.height = height || 50;
    this.$elem = null;
}
Widget.protoype.render = function($where){
    if(this.$elem){
        this.$elem.css({
            width:this.width+"px",
            height:this.height+"px"
        }).appendTo($where);
    }
}
//子类
function Button(width, height,label){
    //调用 "super" 构造函数
    Widget.call(this, width,height);
    this.label = label || "Default";
    this.$elem  = $("<button>").text(this.label);
}
//让Button "继承" Widget
Button.prototype = Object.create(Widget.prototype);
//重写 render(...)
Button.prototype.render = function($where) {
    // "super" 调用
    Widget.prototype.render.call(this, $where);
    this.$elem.click(this.onClick.bind(this));
};
Button.prototype.onClick = function(evt){
    console.log("Button '" + this.label + "' clicked !");
};
$(document).ready(function(){
    var $body =$(document.body);
    var btn1 = new Button(125, 30, "Hello");
    var btn2 = new Button(150,40, "World");
    btn1.render($body);
    btn2.render($body);
});
```

在面向对象设计模式中我们需要先在父类中定义基础的`render(...)`,然后在子类中重写它。子类并不会替换基础的`render(..)`，只是添加一些按钮特有的行为。
可以看到代码中出现了丑陋的显式多态，即通过`Widget.call`和`Widget.prototype.render.call`从"子类"方法中引用"父类"中的基础方法。

#### ES6的class语法糖

附录A会详细介绍ES6的class语法糖，不过这里可以简单介绍一下如何使用class来实现相同的功能:

```js
class Widget{
    constructor(width, height){
        this.width = width || 50;
        this.height = height || 50;
        this.$elem = null;
    }
    render($where){
        if(this.$elem){
            this.$elem.css({
                width:this.width+"px",
                height:this.height + "px"
            }).appendTo($where);
        }
    }
}
class Button extends Widget{
    constructor(width,height,label){
        super(width, height);
        this.label = label || "Default";
        this.$elem = $("<button>").text(this.label);
    }
    render($where){
        super($where);
        this.$elem.click(this.onClick.bind(this));
    }
    onClick(evt){
        console.log("Button '" + this.label + "'clicked ! ");
    }
}
$(document).ready(function(){
    var $body = $(document.body);
    var btn1 = new Button(125, 30, "Hello");
    var btn2 = new Button(150,40,"World");
    btn1.render($body);
    btn2.render($body);
});
```

毫无疑问，使用 `ES6`的`class`之后，上一段代码中许多丑陋的语法都不见了，`super(...)`函数棒极了。(尽管深入探究就会发现并不是那么完美!).
尽管语法上得到了改进，但实际上这里并没有真正的类，class任然是通过`[[Prototype]]`机制实现的，因此我们仍然面临这第4章至第6章提到的思维模式不匹配问题。附录A会详细介绍`ES6`的`class`语法极其实现细节，我们会看到为什么解决语法上的问题无法真正接触对于JavaScript中类的误解，尽管它看起来非常像一种解决办法!
无论你使用的是传统的原型语法还是`ES6`中的新语法糖，你仍然需要用 "类"的概念来对问题(UI控件)进行建模。就像前几章试图证明的一样，这种做法会为你带来新的麻烦。

#### 6.2.2 委托控件对象

下面的例子使用对象关联风格委托来更简单地实现`Widget/Button`:

```js
var Widget = {
    init:function(width,height){
        this.width = width||50;
        this.height = height || 50;
        this.$elem = null;
    },
    insert:function($where){
        if(this.$elem){
            this.$elem.css({
                width:this.width + "px",
                height:this.height+ "px"
            }).appendTo($where);
        }
    }
};
var Button = Object.create(Widget);
Button.setup = function(width, height,label){
    //委托调用
    this.init(width, height);
    this.label = label || "Default";
    this.$elem = $("<button>").text(this.label);
};
Button.build = function($where){
    //委托调用
    this.insert($where);
    this.$elem.click(this.onClick.bind(this));
};
Button.onClick = function(evt){
    console.log("Button '" + this.label + "'clicked! ");
};
$(document).ready(function(){
    var $body = $(document.body);
    var btn1 =Object.create(Button);
    btn1.setup(125, 30, "World");
    var btn2 = Object.create(Button);
    btn2.setup(150, 40, "World");
    btn1.build($body);
    btn2.build($body);
});
```

使用对象关联风格来编写代码时不需要把`Widget`和`Button`当作父类和子类。相反，`Widget`只是一个对象，包含一组通用的函数，任何类型的控件都可以委托，Button同样只是一个对。(当然，它会通过委托关联到`Widget`)

从设计模式的角度来说，我们并没有像类一样在两个对象中都定义相同的方法名`render(...)`，相反，我们定义了两个更具描述性的方法名(`insert(...)`和`build(...)`)。同理，初始化方法分别叫做`init(...)`和`setup(...)`。
在委托设计模式中，除了建议使用不相同且根据描述性的方法名之外，还要通过对象关联避免丑陋的显示伪多态调用(`Widget.call` 和`Widget.prototype.render.call`),代之以简单的相对委托调用`this.init(...)`和`this.insert(...)`。
从语法角度来说，我们同样没有使用任何构造函数、`.prototype`或`new`,实际上也没必要使用它们。
如果你仔细观察就会发现，之前的一次调用(`var btn1 =new Button(..)`)现在变成了两次(`var btn1 = Object.create(Button)`和`btn1.setup(...)`)。咋一看这似乎是一个缺点(需要更多的代码)。
但是这一点其实也是对象关联风格代码相比传统原型风格代码有优势的地方。为什么呢?
使用类构造函数的话，你需要(并不是硬性需求，但是强烈建议)在同一个步骤中实现构造和初始化。然而，在许多情况下把这两步分开(就像对象关联代码一样)更灵活。
举例来说，假如你在程序启动时创建了一个实例池，然后一直等到实例被取出并使用时才执行特定的初始化过程。这个过程中两个函数调用是挨着的，但是完全可以根据需要让它们出现在不同的位置。
`对象关联可以更好地支持关注分离(separation of concerns)原则，创建和初始化并不需要合并为一个步骤`。

### 6.3 更简洁的设计

对象关联除了能让代码开起来更简洁(并且更具有扩展性)外还可以通过行为委托模式简化代码结构。我们来看最后一个例子，它展示了对象关联如何简化整体设计。
在这个场景中我们有两个控制器对象，一个用来操作网页中的登陆表单，另一个用来与服务器进行验证(通信)。
我们需要一个辅助函数来创建Ajax通信。我们使用的是jQuery(尽管其他框架也做的不错),它不仅可以处理Ajax并且会返回一个类`Promise`的结果，因此我们可以使用`.then(...)`来监听响应。

在传统的类设计模式中，我们会把基础的函数定义在名为`Controller`的类中，然后派生两个子类`LoginController`和`AuthController`,它们都继承自`Controller`并且重写了一些基础行为:

```js
//父类
function Controller(){
    this.errors =[];
}
Controller.prototype.showDialog= function(title, msg){
    //给用户显示标题和消息
};
Controller.prototype.success = function(msg){
    this.showDialog("Success", msg);
};
Controller.prototype.failure = function(err){
    this.errors.push(err);
    this.showDialog("Error", err);
};
//子类
function LoginController(){
    Controller.call(this);
}
//把子类关联到父类
LoginController.prototype = Object.create(Controller.prototype);
LoginController.prototype.getUser = function(){
    return document.getElementById("login_username").value;
};
LoginController.prototype.getPassword = function(){
    return document.getElementById("login_password").value;
}
LoginController.prototype.validateEntry = function(user,pw){
    user = user || this.getUser();
    pw = pw || this.getPassword();
    if(!(user && pw)){
        return this.failure("Please enter a username & password!");
    }else if(user.length < 5){
        return this.failure("Please must be 5 + characters! ");
    }
    //如果执行到这说明通过验证
    return true;
};
//重写基础的failure()
LoginController.prototype.failure = function(err){
    // "super" 调用
    Controller.prototype.failure.call(this, "Login invalid: " +err);
}
//子类
function AuthController (login){
    Controller.call(this);
    //合成
    this.login = login;
}
//把子类关联到父类
AuthController.prototype = Object.create(Controller.prototype);
AuthController.prototype.server = function(url,data){
    return $.ajax({
        url:url,
        data:data
    })
};
AuthController.prototype.checkAuth = function(){
    var user = this.login.getUser();
    var pw = this.login.getPassword();
    if(this.login.validateEntry(user, pw)){
        this.server("/check-auth",{
            user:user,
            pw:pw
        })
        .then(this.success.bind(this))
        .fail(this.failure.bind(this));
    }
};
//重写基础的success()
AuthController.prototype.success = function(){
    // "super" 调用
    Controller.prototype.success.call(this, "Authenticated! ");
};
//重写基础的failure()
AuthController.prototype.failure=function(err){
    //"super" 调用
    Controller.prototype.failure.call(this, "Auth Failed: " + err);
};
var auth = new AuthController();
auth.checkAuth(
    //除了继承，我们还需要合成
    new LoginController()
)
```

所有控制器共享的基础行为是 `success(...)`、`failure(...)`和`showDialog(...)`。子类`LoginController` 和 `AuthController`通过重写`failure(...)`和`success(...)`来扩展默认基础类行为。此外，注意`AuthController`需要一个`LoginController`的实例来和登陆表单进行交互，因此这个实例变成了一个数据属性。
另一个需要注意的是我们在继承的基础上进行了一些合成。`AuthController`需要使用`LoginController`,因此我们实例化后者(`new LoginController()`)并用一个类成员属性`this.login`来引用它，这样`AuthController`就可以调用`LoginController`的行为。
注意: 你可能想让`AuthController`继承`LoginController`或者相反，这样我们就通过继承链实现了真正的合成。但是这就是类继承在问题领域建模时会产生的问题，因为AuthController和LoginController 都不具备对方的基础行为，所以这种继承关系是不恰当的。我们的解决办法是进行一些简单的合成从而让它们不必互相合作。

如果你熟悉面向类设计，你一定觉得以上你毁容非常亲切和自然。

#### 反类

但是，我们真的需要用一个Controller父类，两个子类加上合成来对这个问题进行建模吗? 能不能使用对象关联风格的行为委托来实现更简单的设计呢?当然可以!

```js
var LoginController = {
    errors:[],
    getUser:function(){
        return document.getElementById("login_username").value;
    },
    getPassword:function(){
        return document.getElementById("login_password").value;
    },
    validateEntry:function(user,pw){
        user = user||this.getUser();
        ps = pw || this.getPassword();
        if(!(user && pw)){
            return this.failure("Please enter a username & password!");
        }else if(user.length < 5){
            return this.failure("Password must be 5+ characters!");
        }
        //如果执行到这里说明通过验证
        return true;
    },
    showDialog:function(title, msg){
        //给用户显示标题和消息
    },
    failure:function(err){
        this.errors.push(err);
        this.showDialog("Error", "Login invalid: " + err);
    }
};
//让AuthController 委托LoginController
var AuthController = Object.create(LoginController);
AuthController = Object.create(LoginController);
AuthController.errors =[];
AuthController.checkAuth = function(){
    var user = this.getUser();
    var pw = this.getPassword();
    if(this.validateEntry(user, pw)){
        this.server("/check-auth".{
            user:user,
            pw:pw
        })
        .then(this.accepted.bind(this))
        .fail(this.rejected.bind(this));
    }
};
AuthController.server = function(url ,data){
    return $.ajax({
        url:url,
        data:data
    });
};
AuthController.accepted = function(){
    this.showDialog("Success", "Authenticated!");
};
AuthController.rejected = function(err){
    this.failure("Auth Failed: " + err);
}
```

由于`AuthController`只是一个对象(`LoginController`也一样)，因此我们不需要实例化(比如new AuthController()),值需要一行代码就行:
`AuthController.checkAuth();`
借助对象关联，你可以简单地向委托链上添加一个或多个对象，而且同样不需要实例化:

```js
var controller1 =Object.create(AuthController);
var controller2 =Object.create(AuthController);
```

在行为委托模式中，`AuthController`和`LoginController`只是对象，它们之间是兄弟关系，并不是父类和子类的关系，代码中AuthController 委托了LoginController ,它向委托也完全没问题。
这种模式的重点在于只需要两个实体(`LoginController`和`AuthController`),而之前的模式需要三个。
我们不需要Controller基类来"共享"两个实体之间的行为，因为委托足以满足我们需要的功能。同样，前面提到过的，我们也不需要实例化类，因为它们根本就不是类，它们只是对象，此外，我们也不需要合成，因为两个对象可以通过委托合作。
最后，我们避免了面向类设计模式中的多态。我们在不同的对象中没有相同的函数名`success(...)`和`failure(...)`,这样就不需要使用丑陋的显示伪多态。相反，在AuthController 中它们的名字是`accepted(...)`和 `rejected(...)`————可以更好地描述它们的行为。

`总结`: 我们用一种(极其)简单的设计实现了同样的功能，这就是对象关联风格代码和行为委托设计模式的力量。

### 6.4 更好的语法

`ES6`的`class`语法可以间接地定义类方法，这个特性让class看起来更具有吸引力:

```js
class Foo{
    methodName(){ /* ...*/}
}
```

我们终于可以抛弃定义中的关键字function了，对所有JavaScript开发者来说真是大快人心!
你可能注意到了，在之前推荐的对象关联语法中出现了很对function，看起来违背了对象关联的间接性，但是实际上大可不必如此!
在ES6中我们可以在任意对象的字面形式中使用简洁方法声明(concise method declaration),所以对象关联风格的对象可以这样声明(和calss的语法糖一样):

```js
var LoginController = {
    errors:[],
    getUser(){ //在也不用担心代码里有function了
        //..
    },
    getPassword(){
        //...
    }
    //...
};
```

唯一的区别是对象的字面形式仍然需要使用","来分割元素，而class语法不需要。这个区别对于整体的设计来说无关紧要。
此外，在ES6中，你可以使用对象的字面形式(这样就可以使用简洁方法定义)来改写之前繁琐的属性赋值语法(比如AuthController的定义)，然后用`Object.setPrototypeOf(...)`来修改它的`[[Prototype]]`:

```js
var AuthController = {
    errors:[],
    checkAuth(){
        //...
    },
    server(url, data){
        //...
    }
    //...
}
//现在把 AuthController 关联到LoginController
Object.setPrototypeOf(AuthController, LoginController);
```

是用ES6的简洁方法可以让对象关联风格更加人性化(并且仍然比典型的原型风格代码更加简洁和优秀)。你完全不需要使用类就能享受简洁的对象语法。

#### 反词法

简洁方法有一个非常小但是非常重要的缺点。思考下面的代码:

```js
 var Foo ={
     bar() { /* ...*/}
     baz:function baz(){ /*..*/}
 }
```

去掉语法糖之后的代码如下所示:

```js
var Foo = {
    bar:function(){ /**/},
    baz:function baz(){ /**/}
};

```

看到区别了吗?由于函数对象本身没有名称标识符，所以bar()的缩写形式(function()..)实际上会变成一个匿名函数表达式并赋值给bar属性。相比之下，具名函数表达式(function baz()...)会额外给.baz属性附加一个词法名称标识符baz。
然后呢?在本书第一部分"作用域和闭包"中我们分析了匿名函数表达式的三大主要缺点，下面我们会简单介绍下这三个缺点,然后和简洁方法定义进行对比。
匿名函数没有name表示符，这会导致:

1. 调试栈更难追踪;
2. 自我引用(递归、事件(解除)绑定，等等)更难;
3. 代码(稍微)更难理解。

简洁方法没有第1个和第3个缺点。
去掉语法糖的版本使用的是你忙函数表达式，通常来说并不会在追踪栈中添加name,但是简洁方法很特殊，会给对应的函数对象设置一个内部的name属性，这样理论上可以用在追踪栈中。(但是追踪的具体实现时不同的，因此无法保证可以使用。)
很不幸，简洁方法无法避免第2个缺点，它们不具备可以自我引用的词法标识符。思考下面的代码:

```js
var Foo ={
    bar:function(x){
        if(x<10){
            return Foo.bar(x * 2);
        }
        return x;
    },
    baz:function baz(x){
        if(x<10){
            return baz(x*2);
        }
        return x;
    }
}

```

在本例中使用 Foo.bar(x*2)就足够了，但是在许多情况下无法使用这种方法，比如多个对象通过代理共享函数、使用this绑定，等等。这种情况下最好的办法就是使用函数对象的name标识符来进行真正的自我引用。
使用简洁方法时一定要小心这一点。如果你需要自我引用的话，那最好使用传统的具名函数表达式来定义对应的函数( baz:function baz(){...}) ，不要使用简洁方法。

### 6.5 内省

如果你写过许多面向类的程序(无论是使用JavaScript还是其他语言)，那你可能很熟悉自省。自省就是坚持实例的类型。类实例的自省目的是通过创建方式来判断对象的结构和功能。
下面的代码使用`instanceof`来推测对象a1的功能:

```js
function Foo(){
//...
}
Foo.prototype.something = function(){
    //...
}
var a1 = new Foo();
//之后
if(a1 instanceof Foo){
    a1.something();
}
```

因为Foo.prototype(不是Foo!)在a1 的`[[Prototype]]`链上，所以instanceof操作(会令人困惑地)告诉我们a1是Foo "类"的一个实例。知道了这点后，
我们就可以认为a1有Foo "类"描述功能。
当然，Foo类并不存在，只有一个普通的函数Foo，它引用了a1委托的对象(Foo.prototype)。从语法角度来说，instanceof似乎是检查a1和Foo的关系，但是实际上它想说的是a1和Foo.prototype(引用的对象)是互相关联的。
instanceof 语法会产生语义困惑而且非常不直观。如果你想检查对象a1和某个对象的关系，那必须使用另一个引用该对象的函数才行————你不能直接判断两个对象是否关联。
还记得本章之前介绍的抽象的Foo/Bar/b1例子吗，简单来说是这样的:

```js
function Foo(){ /* ..*/}
Foo.prototype ...
function Bar() {/* ..*/}
Bar.prototype=Object.create(Foo.prototype);
var b1 = new Bar("b1");
```

如果使用instanceof 和.prototype 语义来检查本例中实体的关系，那必须这样做:

```js
//让Foo和Bar互相关联
Bar.prototype instanceof Foo; //true
Object.getPrototypeOf(Bar.prototype) ===Foo.prototype; //true
Foo.prototype.isPrototypeOf(Bar.prototype); //true
//让b1关联到 Foo和Bar
b1 instanceof Foo;//true
b1 instanceof Bar; //true
Object.getPrototypeOf(b1) ===Bar.prototype; //true
Foo.prototype.isPrototypeOf(b1); //true
Bar.prototype.isPrototypeOf(b1); //true

```

显然这是一种非常糟糕的方法。举例来说，(使用类时)你最直观的想法可能是使用Bar instanceof Foo (因为容易把 "实例"理解成"继承"),但是在JavaScript中这是行不通的，你必须使用Bar.prototype instanceof Foo。
这是一种常见但是可能更加脆弱的内省模式，许多开发者认为它比instanceof更好。这种模式被称为"鸭子类型"。这个术语源自这句格言"`如果看起来像鸭子，叫起来像腰子，那就一定是鸭子。`"
举例来说:
if(a1.something){
    a1.something();
}
我们并没有检查a1和委托something()函数的对象之间的关系，而是建设如果a1通过了测试a1.something的话，那a1就一定能调用.something()(无论这个方法存在于a1自身还是委托到其他的对象)。这个假设的风险其实并不算高。
但是"鸭子类型"通常会在测试之外做出许多关于对象功能的假设，这当然会带来许多风险(或者说脆弱的设计)。
ES6的Promise就是典型的"鸭子类型"，出于各种原因，我们需要判断一个对象引用的是否是Promise，但是判断的方法是检查对象是否具有then()方法。换句话说，如果对象有then()方法，ES6的Promise就会认为这个对象是 "可持续(thenable)的，因此会期望它具有Promise的所有标准行为。
如果有一个不是Promise但是具有then()方法的对象，那你千万不要把它用在ES6的Promise机制中，否则会出错。
这个例子清楚地解释了"鸭子类型"的危害。你应该尽量避免使用这个方法，即使使用也要保证条件是可控的。
现在回到本章想说的对象关联风格代码，其内省更加简洁。我们先回顾一下之前的Foo/Bar/b1对象关联的例子(只包含关键代码):

```js
var Foo = {/* ...*/}
var Bar = Object.create(Foo);
Bar....
var b1 = Object.create(Bar);

```

使用对象关联时，所有的对象都是通过`[[Prototype]]`委托互相关联，下面是内省的方然，非常简单:

```js
//让Foo 和Bar互相关联
Foo.isPrototypeOf(Bar); //true
Object.getPrototypeOf(Bar) ===Foo;//true
//让b1关联到 Foo和Bar
Foo.isPrototypeOf(b1); //true
Bar.isPrototypeOf(b1); //true
Object.getPrototypeOf(b1) ===Bar; //true
```

我们没有使用instanceof，因为它会产生一些和类有关的误解。现在我们想问的问题是"你是我的原型吗?" 我们并不需要使用间接的形式，比如`Foo.prototype`或者繁琐的`Foo.prototype.isPrototypeOf(...)`。
我觉得和之前的方法比起来，这种方法显然更加简洁并且清晰。再说一次，我们恩威JavaScript中对象关联比类风格的代码更加简洁(而且功能相同)。

### 6.6 小结

在软件架构中你可以选择是否使用类和继承设计模式。大多数开发者利索当然地认为类是唯一的(合适)代码组织方式，但是本章中我们看到了另一种更少见但是更强大的设计模式:`行为委托`。
行为委托认为对象之间是兄弟关系，互相委托，而不是父类和子类的关系。JavaScript的`[[Prototype]]`机制本质上就是行为委托机制。也就是说，我们可以选择在JavaScript中努力实现类机制，也可以拥抱更自然的`[[Prototype]]`委托机制。
当你只用对象来设计代码时，不仅可以让语法更加简洁，而且可以让代码结构更加清晰。
对象关联(对象之前互相关联)是一种编码风格。它倡导的是直接创建和关联对象，不把它们抽象成类。对象关联可以用基于`[[Prototype]]`的行为委托非常自然地实现。
