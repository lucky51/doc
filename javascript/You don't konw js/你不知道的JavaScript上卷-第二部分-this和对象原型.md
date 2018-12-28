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

这个迭代器会生成"无限个"随机数，因此我们添加了一条bread语句，防止程序被挂起。

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

JavaScript属于哪一种类呢? 在相当长的一段时间里，JavaScript只有一些近似类的语法元素(比如 new 和instanceof)，不过在后来的 ES6中新增了一些元素，比如class关键字。
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

在继承或者实例化时，JavaScript的对象机制并不会自动执行复制行为。简单的来说JavaScript中只有队形，并不存在可以被实例化的 "类" 。一个对象并不会被复制到其他对象，它们会被关联起来。
由于在其他语言中类表现出来的都是赋值行为，因此JavaScript开发者也想了一个方法来模拟类的复制行为，这个方法就是`混入`。接下来我们会看到两种类型的混入:`显示`和`隐式`。

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
        我们来分析一下这条语句:Vehicle.drive.call(this) 。这就是我所说的`显式多态`。还记得吗，在之前的伪代码中对应的语句是 inherited:drive().我们之为`相对多态`。
        JavaScript (在 ES6之前)并没有相对多态机制。所以，由于Car和Vehicle中都有 drive()函数，为了指明调用对象，我们必须使用绝对(而不是相对)引用。我们通过名称显示指定Vehicle 对象并调用它的drive()函数。
        但是如果直接执行 Vehicle.drive()，函数调用中的this会被绑定到 Vehicle 对象而不是Car对象，这并不是我们想要的。因此，我们会使用 .call(this) 来确保drive()在Car对象的上下文中执行。
        但是在JavaScript中(由于屏蔽)使用显示伪多态会在所有需要使用(伪)多态引用的地方创建一个函数关联，这会极大地增加维护成本。此外，由于显示伪多态可以模拟多重继承，所以它会进一步增加代码的复杂度和维护难度。
2. 混合复制
    JavaScript中的函数无法(用标准、可靠的方法)真正的复制，所以你只能复制对共享函数对象的引用(函数就是对象;)。如果你修改了共享函数对象(比如ignition()) ，比如添加了一个属性，那Vehicle 和 Car都会受到影响。
    `显式混入`是JavaScript中一个很棒的机制，不过它的功能也没有看起来那么的强大。虽然它可以把一个对象的属性复制到另一个对象中，但是这其实并不能带来太多的好处，无非就是少几条定义语句，而且还会带来我们刚才提到的函数对象引用问题。
    如果你向目标对象中显式混入超过一个对象，就可以部分模仿多重继承行为，但是仍没有直接的方式来处理函数和属性的同名问题。有些开发者/库提出了"晚绑定"技术和其他的一些解决方法，但是从根本上来说，使用这些"诡计"通常会(降低性能并且)得不偿失。
    一定要注意，只在能够提高代码可读性的前提下使用显示混入，避免使用增加代码理解难度或者让对象更加复杂的模式。
    如果使用混入时感觉越来越困难，那或许你应该停止使用它了。实际上，如果你必须使用一个复杂的库或者函数来实现这些细节，那就标志着你的方法是有问题的或者是不必要的。
3. 寄生继承
    显式混入模式的一种变体被称为"寄生继承"，它既是显式的又是隐式的，主要推广者是 `Douglas Crockford`。
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

通过在构造函数调用或者方法调用中 使用 `Something.cool.call(this)`, 我们实际上 "借用"了函数 `Something.cool()`并在 Another的上下文中调用了它(通过this绑定;)。最终的结果是 `Something.cool()`中的赋值操作都会应用在Another对象上而不是Something 对象上。
因此，我们把 Something 的行为 "混入" 到了 Another 中。
虽然这类技术利用了 this的重新绑定功能，但是 `Something.cool.call(this)`仍然无法变成相对(而且更灵活的)引用 ，所以使用时千万要小心，通常来说，尽量避免使用这样的结构，以保证代码的整洁和可维护性。

### 4.5 小结

`类` 是一种设计模式。许多语言提供了对于面向类软件设计的原始语法。JavaScript 也有类似的语法，但是和其他语言中的类完全不同。
类意味着复制。
传统的类被实例化时，它的行为被复制到了实例中。类被继承时，行为也会被复制到了子类中。
`多态` (在继承链的不同层次名称相同但是功能不同的函数) 看起来似乎是从子类引用父类，但是本质上引用的骑士是复制的结果。
JavaScript并不会(像类那样) 自动创建对象的副本。
`混入模式`(无论显式还是隐式) 可以用来模拟类的复制行为，但是通常会产生丑陋并且脆弱的语法，比如显式伪多态(OtherObj.methodName.call(this, ...)),这会让代码更加难懂并且难以维护。
此外，显式混入实际上无法完全模拟类的复制行为，因为对象(和函数! 别忘了函数也是对象)只能复制引用，无法复制引用的对象或者函数本身。忽视这一点会导致许多问题。
总的来说，在JavaScript中模拟类是得不偿失的，虽然能解决当前的问题，但是可能会埋下更多的隐患。
