# Promises/A+  

Promise 表示异步操作的最终结果。与Promise交互的主要方式是通过其then方法，该方法注册回调已接收Promise的最终值或无法完成Promise的原因。
This specification details the behavior of the then method, providing an interoperable base which all Promises/A+ conformant promise implementations can be depended on to provide. As such, the specification should be considered very stable. Although the Promises/A+ organization may occasionally revise this specification with minor backward-compatible changes to address newly-discovered corner cases, we will integrate large or backward-incompatible changes only after careful consideration, discussion, and testing

该规范详细说明了then方法的行为，提供了一个可互操作的基础，可以依赖所有Promises / A +符合的promise实现来提供。
因此，应该认为规范非常稳定。
尽管Promises / A +组织可能偶尔会修改此规范，并采用较小的向后兼容更改来以解决新发现的极端情况，但只有经过仔细考虑，讨论和测试后，我们才会集成大型或后向不兼容的更改。

从历史上看，Promises / A +澄清了早期Promises / A提案的行为条款，将其扩展到涵盖事实上的行为并省略了未指明或有问题的部分。

最后，核心Promises / A +规范没有涉及如何创建，实现或拒绝承诺，而是选择专注于提供可互操作的方法。配套规范中的未来工作可能涉及这些主题。

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
  * 2.1.2.1 一定不能转变为任何其他的状态。
  * 2.1.2.2 必须有一个不能被改变的值。
* 2.1.3 当状态为rejected:
  * 2.1.3.1 一定不可以转变到其他的任何状态
  * 2.1.3.2 必须有一个不能改变的理由

这里 的"绝不能改变" 意味着不可变的标识(即 ===),但是并不是深度的不变形。

### 2.2 Then 方法

Promise 必须提供一个 then 方法用来访问当前或者最终的 value或者是reason。
一个 Promise 的 then 方法接受两个参数:

```js
primise.then(onFulfilled, onRejected)
```

* 2.2.1 onFulfilled 和 onRejected 都是可选的参数:
  * 2.2.1.1 如果 onFulfilled 不是一个方法 ，它必须被忽略.
  * 2.2.1.2 如果 onRejected 不是一个方法 ，它必须被忽略。
* 2.2.2. 如果onFulfilled 是一个方法 :
  * 2.2.2.1 它必须在promise的状态为fulfilled 时被调用，将promise的值作为它的第一个参数。
  * 2.2.2.2 它一定不能再promise状态为fulfilled之前调用。
  * 2.2.2.3 它绝不能被多次调用  。
* 2.2.3 如果 onRejected 是一个方法。
  * 2.2.3.1 它必须在promise被rejected之后调用，通过promise的reason作为它的第一个参数
  * 2.2.3.2 它一定不能再promise被rejected之前调用。
  * 2.2.3.3 一定不能重复调用。
* 2.2.4 在执行上下文堆栈只包含平台代码之前，不能调用onFulfilled 或 onRejected
* 2.2.5 onFulfilled 和 onRejected 只能作为方法被调用(不能通过this)
* 2.2.6 在同一个promise中then方法也许会被调用多次.
  * 2.2.6.1 如果当promise为fulfilled时，所有各自的onFulfilled回调 必须以它们原始调用then方法的顺序执行。
  * 2.2.6.2 如果当promise为rejected，所有各自的onRejected 回调必须以它们原始调用then方法的顺序执行。
* 2.2.7 then 方法必须返回一个promise

    ```js
    promise2 = promise1.then(onFulfilled, onRejected);
    ```
  * 2.2.7.1 如果onFulfilled 或onRejected 返回了一个值 x,执行Promise Resolution Procedure(promise 解决程序) [[Resolve]](promise2, x)
  * 2.2.7.2 如果onFulfilled 或onRejected 抛出了一个异常 e, promise2 必须通过e作为reason来rejected。
  * 2.2.7.3 如果onFulfilled 不是一个方法并且promise1 为fulfilled，promise2 必须

原文地址 :https://promisesaplus.com/#point-42