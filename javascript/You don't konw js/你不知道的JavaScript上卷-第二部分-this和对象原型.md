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

3. 如果在 `[[Prototype]]`链上层存在`foo`并且它是一个`setter`，那就一定会调用这个`setter`。foo不会被添加到(或者说屏蔽于)myObject，也不会重新定义foo这个 setter。

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
var b = new Foo("b);
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
描述符。因为ES5之前的版本无法模拟属性操作符，所以polyfill diamante无法实现这个附加功能。
通常来说并不会使用`Object.create(...)`的附加功能，所以对于大多数的开发者来说，上面那段polyfill代码就足够了。
有些开发者更加严谨，我们认为只有能被完全模拟的函数才应该使用polyfill代码。由于`Object.create(...)`是只能部分模拟的函数之一，所以这些狭隘的人认为如果你需要在ES5之前的环境中使用`Object.create(..)`的特性，那不要使用polyfill代码，而是使用一个自定义函数并且名字不能是Object.create 。你可以把你自己的函数定义成这样:

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
