@title
  @primary
    实现
  @secondary
    Forbes Lindesay 作
    Cheng Liu 译

##[intro] 基本介绍

这篇文章原文发表在 [Stack Overflow](http://stackoverflow.com/questions/23772801/basic-javascript-promise-implementation-attempt/23785244#23785244).
本文希望能向你展示 `Promise` 在 JavaScript 中的实现，相应地，你将会更加理解 promise 是怎么运行的。

##[state] 状态机

其实一个 promise 就是一种状态机，我们应该考虑到我们接下来会用到的状态信息。

:js
  var PENDING = 0;
  var FULFILLED = 1;
  var REJECTED = 2;

  function Promise() {
    // 存储的状态值可以是 PENDING，FULFILLED 或者 REJECTED
    var state = PENDING;

    // 一旦状态变为 FULFILLED 或者 REJECTED 时，保存值或者抛出异常
    var value = null;

    // 保存通过调用 .then 或者 .done 挂载的正常响应和异常响应的函数
    var handlers = [];
  }

##[transitions] 状态转换

下面，我们一起看看两种状态转换，成功和失败：

:js
  var PENDING = 0;
  var FULFILLED = 1;
  var REJECTED = 2;

  function Promise() {
    // 存储的状态值可以是 PENDING，FULFILLED 或者 REJECTED
    var state = PENDING;

    // 一旦状态变为 FULFILLED 或者 REJECTED 时，保存值或者抛出异常
    var value = null;

    // 保存通过调用 .then 或者 .done 挂载的正常响应和异常响应的函数
    var handlers = [];

    function fulfill(result) {
      state = FULFILLED;
      value = result;
    }

    function reject(error) {
      state = REJECTED;
      value = error;
    }
  }

现在我们实现了最简单的状态转换，不过让我们一起想想怎么实现更高层次的状态转换，称为 `resolve`

:js
  var PENDING = 0;
  var FULFILLED = 1;
  var REJECTED = 2;

  function Promise() {
    // 存储的状态值可以是 PENDING，FULFILLED 或者 REJECTED
    var state = PENDING;

    // 一旦状态变为 FULFILLED 或者 REJECTED 时，保存值或者抛出异常
    var value = null;

    // 保存通过调用 .then 或者 .done 挂载的正常响应和异常响应的函数
    var handlers = [];

    function fulfill(result) {
      state = FULFILLED;
      value = result;
    }

    function reject(error) {
      state = REJECTED;
      value = error;
    }

    function resolve(result) {
      try {
        var then = getThen(result);
        if (then) {
          doResolve(then.bind(result), resolve, reject)
          return
        }
        fulfill(result);
      } catch (e) {
        reject(e);
      }
    }
  }

注意 `resolve` 是如何既能处理 promise 对象又能处理普通值的。如果被传入的是一个 promise，会等待其执行完毕，promise 绝不能直接返回另一个 promise 对象做为执行结果，如同在这个 `resolve` 方法中向我们揭示的那样。此处我们用了一些辅助方法，现在我们来定义他们：

:js
  /**
   * 检查传入的值是否是一个 promise 对象，
   * 若是，则返回其 `then` 方法
   *
   * @param {Promise|Any} value
   * @return {Function|Null}
   */
  function getThen(value) {
    var t = typeof value;
    if (value && (t === 'object' || t === 'function')) {
      var then = value.then;
      if (typeof then === 'function') {
        return then;
      }
    }
    return null;
  }

  /**
   * 消除潜在的不正常行为，而且确保 onFulfilled 与 onRejected 只被执行一次
   *
   * 这里不保证异步执行
   *
   * @param {Function} fn 一个不能完全信任的处理函数
   * @param {Function} onFulfilled
   * @param {Function} onRejected
   */
  function doResolve(fn, onFulfilled, onRejected) {
    var done = false;
    try {
      fn(function (value) {
        if (done) return
        done = true
        onFulfilled(value)
      }, function (reason) {
        if (done) return
        done = true
        onRejected(reason)
      })
    } catch (ex) {
      if (done) return
      done = true
      onRejected(ex)
    }
  }

##[constructing] 构造

现在，我们已经完成了内部的状态机，但是我们还没有执行处理函数，让我们开始处理 promise 。

:js
  var PENDING = 0;
  var FULFILLED = 1;
  var REJECTED = 2;

  function Promise(fn) {
    // 存储的状态值可以是 PENDING，FULFILLED 或者 REJECTED
    var state = PENDING;

    // 一旦状态变为 FULFILLED 或者 REJECTED 时，保存值或者抛出异常
    var value = null;

    // 保存通过调用 .then 或者 .done 挂载的正常响应和异常响应的函数
    var handlers = [];

    function fulfill(result) {
      state = FULFILLED;
      value = result;
    }

    function reject(error) {
      state = REJECTED;
      value = error;
    }

    function resolve(result) {
      try {
        var then = getThen(result);
        if (then) {
          doResolve(then.bind(result), resolve, reject)
          return
        }
        fulfill(result);
      } catch (e) {
        reject(e);
      }
    }

    doResolve(fn, resolve, reject);
  }

正如你所看到的，我们重用了 `doResolve` 方法，因为我们认为处理函数同样不能被完全信任。
`fn` 方法允许 `resolve` 和 `reject` 可以被多次调用，甚至抛出多个异常。
决定 promise 只能被处理一次、或者只能被执行失败一次，绝不能在不同的状态间多次跳转的职责在我们身上。

##[done] 观测 (通过 .done)

现在，我们有了一个完整的内部状态机，但是我们还没有任何方法能够观测到它的变化。
我们的终极目标是实现 `.then` 方法，不过语义上来说， `.done` 方法看上去容易实现，
我们就现在实现它吧。

这里我们要做的，就是实现如下的 `promise.done(onFulfilled, onRejected)` 方法：

 - 仅被调用一次
 - 不会在下一个tick之前被调用 （就是说，`.done` 方法已经返回）
 - 调用时与 promise 是否被处理完成无关

  var PENDING = 0;
  var FULFILLED = 1;
  var REJECTED = 2;

  function Promise(fn) {
    // 存储的状态值可以是 PENDING，FULFILLED 或者 REJECTED
    var state = PENDING;

    // 一旦状态变为 FULFILLED 或者 REJECTED 时，保存值或者抛出异常
    var value = null;

    // 保存通过调用 .then 或者 .done 挂载的正常响应和异常响应的函数
    var handlers = [];

    function fulfill(result) {
      state = FULFILLED;
      value = result;
      handlers.forEach(handle);
      handlers = null;
    }

    function reject(error) {
      state = REJECTED;
      value = error;
      handlers.forEach(handle);
      handlers = null;
    }

    function resolve(result) {
      try {
        var then = getThen(result);
        if (then) {
          doResolve(then.bind(result), resolve, reject)
          return
        }
        fulfill(result);
      } catch (e) {
        reject(e);
      }
    }

    function handle(handler) {
      if (state === PENDING) {
        handlers.push(handler);
      } else {
        if (state === FULFILLED &&
          typeof handler.onFulfilled === 'function') {
          handler.onFulfilled(value);
        }
        if (state === REJECTED &&
          typeof handler.onRejected === 'function') {
          handler.onRejected(value);
        }
      }
    }

    this.done = function (onFulfilled, onRejected) {
      // 确保此处始终模拟异步执行
      setTimeout(function () {
        handle({
          onFulfilled: onFulfilled,
          onRejected: onRejected
        });
      }, 0);
    }

    doResolve(fn, resolve, reject);
  }

我们确保 Promise 状态变更时通知所有处理函数。
这个过程仅发生在下一个 tick 时

##[then] 观测 (通过 .then)

现在，我们已经把 `.done` 方法实现好了，我们就能轻易的用同样的办法实现 `.then` 方法，不同的是，在过程中构造了一个新的 Promise 。

:js
  this.then = function (onFulfilled, onRejected) {
    var self = this;
    return new Promise(function (resolve, reject) {
      return self.done(function (result) {
        if (typeof onFulfilled === 'function') {
          try {
            return resolve(onFulfilled(result));
          } catch (ex) {
            return reject(ex);
          }
        } else {
          return resolve(result);
        }
      }, function (error) {
        if (typeof onRejected === 'function') {
          try {
            return resolve(onRejected(error));
          } catch (ex) {
            return reject(ex);
          }
        } else {
          return reject(error);
        }
      });
    });
  }

##[apendix] 更多阅读

 - [then/promise](https://github.com/then/promise/blob/master/src/core.js) 简单的实现一个 promise 对象
 - [kriskowal/q](https://github.com/kriskowal/q/blob/v1/design/README.md) 完全不同的方法实现了 promise， 并且一步步的解释了背后的原理
 - [petkaantonov/bluebird](https://github.com/petkaantonov/bluebird) 利用黑魔法实现高性能的 promise 对象，[Optimization Killers](https://github.com/petkaantonov/bluebird/wiki/Optimization-killers) 这个维基页面将为你解答其中的奥秘
 - [Stack Overflow](http://stackoverflow.com/questions/23772801/basic-javascript-promise-implementation-attempt/23785244#23785244) 原文链接

@pager
  @previous(href="/generators/")
    生成器

