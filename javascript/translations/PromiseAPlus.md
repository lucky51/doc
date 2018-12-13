# Promises/A+  

Promise 表示异步操作的最终结果。与Promise交互的主要方式是通过其then方法，该方法注册回调已接收Promise的最终值或无法完成Promise的原因。
该规范详细说明了then方法的行为，提供了一个可互操作的基础，可以依赖所有符合Promises/A +的promise来提供实现。
因此，应该认为规范非常稳定。
尽管Promises/A+组织可能偶尔会修改此规范，并采用较小向后兼容的更改来以解决新发现的极端情况，但只有经过仔细考虑，讨论和测试后，我们才会集成大型的或不向后兼容的更改。
从历史上看，Promises/A+澄清了早期Promises/A提案的行为条款，将其扩展到涵盖事实上的行为并省略了未指明或有问题的部分。
最后，核心Promises/A +规范没有涉及如何创建，实现或拒绝承诺，而是选择专注于提供可互操作的方法。配套规范中的未来工作可能涉及这些主题。

## 1. Terminology 术语

* 1.1 "promise" 是具有then方法的对象或函数 ，这种行为确立这个规范。
* 1.2 "thenable" 是定义了then方法的对象或函数
* 1.3 "value" 是任何合法的JavaScript值(包括undefined，thenable 或promise)
* 1.4 "exception" 是使用throw语句抛出的值
* 1.5 "reson" 是表明promise被reject的原因

## 2. Requirements 要求

### 2.1 Promise States

一个 Promise 必须是这三个状态之一: pending  ,fulfilled, or rejected

* 2.1.1 当状态为pending :
  * 2.1.1.1 可以转变为已完成或者已拒绝的状态。
* 2.1.2 当状态为fulfilled:
  * 2.1.2.1 绝不能改变转变为任何其他的状态。
  * 2.1.2.2 必须有一个不能被改变的值。
* 2.1.3 当状态为rejected:
  * 2.1.3.1 绝不能转变到其他的任何状态
  * 2.1.3.2 必须有一个不能改变的理由

这里 的"绝不能改变" 意味着不可变的标识(即 ===),但是并不是深度的不变形。

### 2.2 Then 方法

Promise 必须提供一个 then 方法用来访问当前或者最终的 value或者是reason。
一个 Promise 的 then 方法接受两个参数:

```js
primise.then(onFulfilled, onRejected)
```

* 2.2.1 **onFulfilled** 和 **onRejected** 都是可选的参数:
  * 2.2.1.1 如果 **onFulfilled** 不是一个方法 ，它必须被忽略.
  * 2.2.1.2 如果 **onRejected** 不是一个方法 ，它必须被忽略。
* 2.2.2. 如果**onFulfilled** 是一个方法 :
  * 2.2.2.1 它必须在**promise**的状态为**fulfilled** 时被调用，将**promise**的值作为它的第一个参数。
  * 2.2.2.2 它一定不能再**promise**状态为**fulfilled**之前调用。
  * 2.2.2.3 它绝不能被多次调用  。
* 2.2.3 如果 **onRejected** 是一个方法。
  * 2.2.3.1 它必须在**promise**被**rejected**之后调用，通过**promise**的**reason**作为它的第一个参数
  * 2.2.3.2 它一定不能再**promise**被**rejected**之前调用。
  * 2.2.3.3 一定不能重复调用。
* 2.2.4 在执行上下文堆栈只包含平台代码之前，不能调用onFulfilled 或 onRejected
* 2.2.5 **onFulfilled** 和 **onRejected** 只能作为方法被调用(不能通过this)
* 2.2.6 在同一个**promise**中**then**方法也许会被调用多次.
  * 2.2.6.1 如果当**promise**为**fulfilled**时，所有各自的onFulfilled回调 必须以它们原始调用**then**方法的顺序执行。
  * 2.2.6.2 如果当**promise**为**rejected**，所有各自的**onRejected** 回调必须以它们原始调用**then**方法的顺序执行。
* 2.2.7 **then** 方法必须返回一个**promise**

    ```js
    promise2 = promise1.then(onFulfilled, onRejected);
    ```
  * 2.2.7.1 如果**onFulfilled** 或**onRejected** 返回了一个值 **x**,执行Promise Resolution Procedure(promise 决议程序) **[[Resolve]](promise2, x)**
  * 2.2.7.2 如果**onFulfilled** 或**onRejected** 抛出了一个异常 e, promise2 必须通过e作为reason来rejected。
  * 2.2.7.3 如果**onFulfilled** 不是一个方法并且**promise1** 状态为**fulfilled**，**promise2** 必须以**promise1**相同的 **value**(值)来**fulfilled**。
  * 2.2.7.4 如果**onRejected** 不是一个方法并且**promise1**状态为**rejected**，**promise2**必须用**promise1**相同的reason(理由)来**rejected**。

### 2.3 The Promise Resolution Procedure(Promise 决议程序)

The **promise resolution procedure** 是一个将**promise**和**value**作为输入的抽象操作，我们将其表示为**[[Resolve]](promise, x)**  如果x是一个 thenable,在x的行为至少有点像**promise**的前提下,试着将**promise**采用**x**的状态，除此以外，会通过x的值来完成 **promise**
对 **thenable** 的这种处理允许**promise**的实现进行互操作，只要它们公开一个Promise/A+ 兼容的方法，它也允许PromiseA+ 的实现通过合理的 **then**方法去"同化"不符合标准的实现。
运行 **[[Resolve]](promise, x)**,执行一下步骤:

* 2.3.1 如果**promise**和**x** 引用同一个对象,用 **TypeError**作为**reject** **promise**的原因(reason).
* 2.3.2 如果x是一个promise ，采用它的状态 **[3.4]**
  * 2.3.2.1 如果**x**的状态为**pending** ,直到**x**的状态为**fulfilled**或者**rejected**之前, **promise**状态必须保持pending。
  * 2.3.2.2 如果当 **x** 的状态为 **fulfilled**，通过相同的值来完成 **promise**
  * 2.3.2.3 如果当 **x** 的状态为 **rejected** ,通过相同的理由(reason)来取消 **promise**
* 2.3.3 此外，如果 **x** 是一个对象或方法,
  * 2.3.3.1 让 **then** 变为 **x.then** [3.5]
  * 2.3.3.2 如果访问属性 **x.then** 的结果抛出了一个异常 e ,则通过 e作为 原因(reason)取消 **promise**
  * 2.3.3.3 如果 **then** 是一个方法，将**x**作为**this**调用它,第一个参数 **resolvePromise** , 第二个参数**rejectPromise** ，where:
    * 2.3.3.3.1 如果/当 **resolvePromise** 以一个值 **y**去调用，运行 **[[Resolve]](promise, y)**
    * 2.3.3.3.2 如果/当 **rejectPromise** 以一个原因(reason)r 去调用，用 r取消(reject) **promise**
    * 2.3.3.3.3 如果 **resolvePromise** 和**rejectPromise** 被调用，或者多次对同一参数进行多次调用，则第一次调用优先，并且忽略掉任何再次的调用。
    * 2.3.3.3.4 如果调用 **then** 抛出一个异常 e,
      * 2.3.3.3.4.1 如果 **resolvePromise**或 **rejectPromise** 已经被调用，忽略它。
      * 2.3.3.3.4.2 此外，用这个 **e**作为原因(reason)来取消(reject) **promise** 。
  * 2.3.3.4 如果 **x** 不是一个对象或方法，用**x**来完成 **promise**
* 2.3.4 如果**x**不是一个对象或方法，用**x**来完成 **promise**

如果一个 **promise** 通过在循环的thenable链中的一个thenable 来完成，[[Resolve]](promise,thenable) 的递归特性会导致[[Resolve]](promise,thenable)再次被调用，根据上面的算法，这会导致无限制的递归。因此我们鼓励(但不强制要求)promise实现时能检测无限递归，并在出现这种情况时以一个 **TypeError**来 取消(reject) **promise**

## 3.说明

原文地址 :https://promisesaplus.com/#point-42