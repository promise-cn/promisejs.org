@title
  @primary
    API 指南
  @secondary
    Forbes Lindesay 作
    刘诚 / 闫畅 译

## 方法

###[Promise_all] Promise.all(iterable)

参数:
  1 iterable 对象

返回:
  一个 Promise 对象。等待 `iterable` 中所有的 promise 执行成功（fulfill）后，生成出一个执行结果的数组（与传入的 `iterable` 顺序相同），再将该数组做为参数传给 Promise 对象、

:js-example
  var promise = Promise.resolve(3);
  Promise.all([true, promise]).then(values => {
    console.log(values); // [true, 3]
  });

:js-polyfill
  Promise.all = function (arr) {
    // 待完成: 该 polyfill 暂时只支持类似数组的 iterable 元素
    //       应该能支持所有形式的iterable元素
    var args = Array.prototype.slice.call(arr);

    return new Promise(function (resolve, reject) {
      if (args.length === 0) return resolve([]);
      var remaining = args.length;
      function res(i, val) {
        if (val && (typeof val === 'object' || typeof val === 'function')) {
          var then = val.then;
          if (typeof then === 'function') {
            var p = new Promise(then.bind(val));
            p.then(function (val) {
              res(i, val);
            }, reject);
            return;
          }
        }
        args[i] = val;
        if (--remaining === 0) {
          resolve(args);
        }
      }
      for (var i = 0; i < args.length; i++) {
        res(i, args[i]);
      }
    });
  };

###[Promise_denodeify] Promise.denodeify(fn, length) @non-standard

部分 promise 实现提供了一个 `.denodeify` 方法，使与 node.js 代码的交互操作变得更加容易。它会在 `fn` 上加上一个 `callback`，并通过它来 fulfill 或 reject 这个 promise。

参数：
  1. 一个函数。如果该函数返回一个 promise，那么 `denodeify` 函数执行返回的结果将会使用这个 promise 的状态。

  2. length（可选）：传入函数 fn 的参数长度。如果提供了 `length` 这个参数，而且传入的参数超过了 `length`，那么超过部分的参数将不会被传入fn。

返回：
  promise

:js-example
  var Promise = require('promise');
  var readFile = Promise.denodeify(require('fs').readFile);

  module.exports = function readJson(filename) {
    return readFile(filename, 'utf8').then(JSON.parse);
  };

:js-polyfill
  Promise.denodeify = function (fn, argumentCount) {
    argumentCount = argumentCount || Infinity;
    return function () {
      var self = this;
      var args = Array.prototype.slice.call(arguments);
      return new Promise(function (resolve, reject) {
        while (args.length && args.length > argumentCount) {
          args.pop();
        }
        args.push(function (err, res) {
          if (err) reject(err);
          else resolve(res);
        })
        var res = fn.apply(self, args);
        if (res &&
          (
            typeof res === 'object' ||
            typeof res === 'function'
          ) &&
          typeof res.then === 'function'
        ) {
          resolve(res);
        }
      })
    }
  }

###[Promise_race] Promise.race(iterable)

参数：
  1. iterable 对象

返回：
  promise，这个 promise 在 `iterable` 中任何一个 promise 成功完成（fulfill）或被中断（rejected）之后就会完成（resolve）。（并得到相应的返回值或错误原因）

:js-example
  var p1 = new Promise(function(resolve, reject) {
    setTimeout(resolve, 500, "one");
  });
  var p2 = new Promise(function(resolve, reject) {
    setTimeout(resolve, 100, "two");
  });

  Promise.race([p1, p2]).then(function(value) {
    console.log(value); // "two"
    // 两者都会完成（resolve），但是 p2 会快一些
  });

  var p3 = new Promise(function(resolve, reject) {
    setTimeout(resolve, 100, "three");
  });
  var p4 = new Promise(function(resolve, reject) {
    setTimeout(reject, 500, "four");
  });

  Promise.race([p3, p4]).then(function(value) {
    console.log(value); // "three"
    // p3 更快，所以它会被 resolve
  }, function(reason) {
    // Not called
  });

  var p5 = new Promise(function(resolve, reject) {
    setTimeout(resolve, 500, "five");
  });
  var p6 = new Promise(function(resolve, reject) {
    setTimeout(reject, 100, "six");
  });

  Promise.race([p5, p6]).then(function(value) {
    // 未被调用
  }, function(reason) {
    console.log(reason); // "six"
    // p6 更快，所以 promise 会被中断
  });

:js-polyfill
  Promise.race = function (values) {
    // TODO: this polyfill only supports array-likes
    //       it should support all iterables
    return new Promise(function (resolve, reject) {
      values.forEach(function(value){
        Promise.resolve(value).then(resolve, reject);
      });
    });
  };

###[Promise_reject] Promise.reject(reason)

返回一个 rejected promise，并得到相应的 `reason`

:js-example
  Promise.reject(new Error("fail")).then(function(error) {
    // 未被调用
  }, function(error) {
    console.log(error); // 执行栈
  });

:js-polyfill
  Promise.reject = function (value) {
    return new Promise(function (resolve, reject) {
      reject(value);
    });
  };

###[Promise_resolve] Promise.resolve(value)

返回一个带有相应值的，被 resolve 的 promise。

如果传入的参数值是一个 promise，那么它会被更进一步解开（unwrapped），以便于返回的 promise 能够使用传入的 promise 的状态。这一点对于转化其他库实现的 promises 非常有帮助。

:js-example
  Promise.resolve("Success").then(function(value) {
    console.log(value); // "Success"
  }, function(value) {
    // 未被调用
  });
  var p = Promise.resolve([1,2,3]);
  p.then(function(v) {
    console.log(v[0]); // 1
  });
  var original = Promise.resolve(true);
  var cast = Promise.resolve(original);
  cast.then(function(v) {
    console.log(v); // true
  });

:js-polyfill
  Promise.resolve = function (value) {
    return new Promise(function (resolve) {
      resolve(value);
    });
  };

## Prototype Methods

###[Promise_prototype_catch] Promise.prototype.catch(onRejected)

等同于 `Promise.prototype.then(undefined, onRejected)`

:js-example
  var p1 = new Promise(function(resolve, reject) {
    resolve("Success");
  });

  p1.then(function(value) {
    console.log(value); // "Success!"
    throw "oh, no!";
  }).catch(function(e) {
    console.log(e); // "oh, no!"
  });

  p1.then(function(value) {
    console.log(value); // "Success!"
    throw "oh, no!";
  }).then(undefined, function(e) {
    console.log(e); // "oh, no!"
  });

:js-polyfill
  Promise.prototype['catch'] = function (onRejected) {
    return this.then(null, onRejected);
  };

###[Promise_prototype_done] Promise.prototype.done(onFulfilled, onRejected) @non-standard

参数：
  1. onFulfilled promise 执行成功时被调用，且将执行结果做为参数传入
  2. onRejected promise 执行出现异常时呗调用，会将异常做为参数传入

不像 `Promise.prototype.then`，它并不返回一个 Promise。如果有错误的话，它并不会像 promise 一样捕获（catch）错误，而会在下次函数执行时抛出（throw）错误。在 node.js 中，这会使线程崩溃（以便于以一个新状态重启）；在浏览器端，错误会在 console 合理地中显示出来。

值得注意的是，`promise.done`并未被标准化。然而这个方法被大多数 promise 库支持，也有助于避免在意外情况下遗漏或未能捕获抛出的异常。

我推荐使用以下 polyfill (#[a(href="/polyfills/promise-done-" + versions.promise + ".min.js") minified] / #[a(href="/polyfills/promise-done-" + versions.promise + ".js") unminified]):

:html
  <script src="https://www.promisejs.org/polyfills/promise-done-#{versions.promise}.min.js"></script>

:js-example
  var Promise = require('promise');
  var p = Promise.resolve('foo');

  p.done(function (value) {
    console.log(value); // "foo"
  });

  p.done(function (value) {
    throw new Error('Ooops!'); // 在下一次执行中被抛出
  });

:js-polyfill
  Promise.prototype.done = function (onFulfilled, onRejected) {
    var self = arguments.length ? this.then.apply(this, arguments) : this
    self.then(null, function (err) {
      setTimeout(function () {
        throw err
      }, 0)
    })
  }

###[Promise_prototype_finally] Promise.prototype.finally(onResolved) @non-standard

一些 promise 库实现了一个（未被标准化的）`.finally` 方法。

参数：

  1. 它接受一个函数作为参数。无论最终 promise 被 fulfill 还是 rejected，这个函数都会被调用。

:js-example
  var Promise = require('promise');
  var p = Promise.resolve('foo');
  var disposed = false;
  p.then(function (value) {
    if (Math.random() < 0.5) throw new Error('oops!');
    else return value;
  }).finally(function () {
    disposed = true; // always called
  }).then(function (value) {
    console.log(value); // => "foo"
  }, function (err) {
    console.log(err); // => oops!
  });

:js-polyfill
  Promise.prototype['finally'] = function (f) {
    return this.then(function (value) {
      return Promise.resolve(f()).then(function () {
        return value;
      });
    }, function (err) {
      return Promise.resolve(f()).then(function () {
        throw err;
      });
    });
  }

###[Promise_prototype_nodeify] Promise.prototype.nodeify(callback, ctx) @non-standard

在一些 promise 的实现中提供了 `.nodeify` 方法，目的是为了更简单的和 node.js 的代码交互。

参数：
  1. callback 回调函数
  2. ctx 指定回调函数调用时的作用域

返回：
  1. 如果 `callback` 参数不是一个函数，那么仅返回原来的 promise 而不进行任何改动
  2. 如果 `callback` 是一个函数，那么它就会被执行并且返回 `undefined`

:js-example
  var Promise = require('promise');
  var readFile = Promise.denodeify(require('fs').readFile);

  // callback 是可选参数
  module.exports = function readJson(filename, callback) {
    return readFile(filename, 'utf8').then(JSON.parse).nodeify(callback);
  };

:js-polyfill
  Promise.prototype.nodeify = function (callback, ctx) {
    if (typeof callback != 'function') return this;

    this.then(function (value) {
      asap(function () {
        callback.call(ctx, null, value);
      });
    }, function (err) {
      asap(function () {
        callback.call(ctx, err);
      });
    });
  }

###[Promise_prototype_then] Promise.prototype.then(onFulfilled, onRejected)

根据 promise 执行的情况，适时调用 `onFulfilled` 函数， `onRejected` 函数

参数：

  1. onFulfilled promise 执行成功时被调用，且将执行结果做为参数传入
  2. onRejected promise 执行出现异常时呗调用，会将异常做为参数传入

返回：

  1. 如果 promise 正常执行完成，返回 `onFulfilled` 执行的结果
  2. 如果 promise 执行抛出异常，返回 `onRejected` 执行的结果
  3. 如果 `onFulfilled` 不是一个函数，当 promise 执行成功后，返回 promise 的执行结果
    （比如 `function (value) { return value; }`）
  4. 如果 `onRejected` 不是一个函数，当 promise 执行异常后，不进行捕捉，继续抛出异常
    （比如 `function (reason) { throw reason; }`）

:js-example
  var p1 = new Promise(function(resolve, reject) {
    resolve("Success!");
    // 或者
    // reject ("Error!");
  });

  p1.then(function(value) {
    console.log(value); // 执行成功！
  }, function(reason) {
    console.log(reason); // 执行失败！
  });

  var p2 = new Promise(function(resolve, reject) {
    resolve(1);
  });

  p2.then(function(value) {
    console.log(value); // 1
    return value + 1;
  }).then(function(value) {
    console.log(value); // 2
  });

##[apendix] 更多阅读

 - [Promises Introduction](/) - 解释了 Promise 的必要性，还有仅使用 callback 的弊端
 - [Patterns](/patterns/) - promise 的用法, 许多辅助方法让你的 promise 事半功倍
 - [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) - mozilla 开发者社区关于 promise 的佳作

@pager
  @previous(href="/")
    introduction
  @next(href="/patterns/")
    patterns
