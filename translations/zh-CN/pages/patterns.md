@title
  @primary
    用法
  @secondary
    Forbes Lindesay 作
    Cheng Liu 译

##[intro] 基本介绍

我们已经见识到了当你想在异步执行的代码中做错误处理的复杂之处。我们也看到通过使用 promise 中的 `.then` 方法能帮减轻这些痛苦。 在这篇文章中，我们会涵盖一些 promise 的高级用法，以及辅助方法让你的代码更加精准。

##[resovle] Promise.resolve(value)

有时候，你仅仅只想让某个值转变成一个 promise 对象，又或者你当前的值并不确定是否是一个你所想的 promise，其执行行为会跟你认为的不一样（比如 jQuery 的 promise），最后，你会想把他们都转换成真正的 promise 。

:js
  var value = 10;
  var promiseForValue = Promise.resolve(value);
  // 等价于
  var promiseForValue = new Promise(function (fulfill) {
    fulfill(value);
  });

:js
  var jQueryPromise = $.ajax('/ajax-endpoint');
  var realPromise = Promise.resolve(jQueryPromise);
  // 等价于
  var realPromise = new Promise(function (fulfill, reject) {
    jQueryPromise.then(fulfill, reject);
  });

:js
  var maybePromise = Math.random() > 0.5 ? 10 : Promise.resolve(10);
  var definitelyPromise = Promise.resolve(maybePromise);
  // 等价于
  var definitelyPromise = new Promise(function (fulfill, reject) {
    if (isPromise(maybePromise)) {
      maybePromise.then(fulfill, reject);
    } else {
      fulfill(maybePromise);
    }
  });

##[reject] Promise.reject

最好是能够避免在异步执行的方法中抛出一个同步的异常。总是返回一个 promise 对象的好处就是你可以用一种相同的方式来做异常处理的环节。为了实现这个目的，就有一个快捷方式来生成会抛出错误的 promise 。

:js
  var rejectedPromise = Promise.reject(new Error('Whatever'));
  // 等价于
  var rejectedPromise = new Promise(function (fulfill, reject) {
    reject(new Error('Whatever'));
  });

##[parallel] Parallel operations

并发执行一些方法总是会让代码写得特别复杂。下面的代码尝试读取一系列的文件（给定文件名），然后抓取成 JSON 格式，最后回调一个结果数组。

:js
  function readJsonFiles(filenames, callback) {
    var pending = filenames.length;
    var called = false;
    var results = [];
    if (pending === 0) {
      // 当没有文件需要被读取时，我们应该提前结束。
      // 但是，如果我们此时立即返回，就成为了同步执行的代码，会破坏原本设想的执行顺序，程序就会有不确定性。
      // 所以，这里增加了 setTimeout，伪装成了异步代码，并尽早返回。
      return setTimeout(function () { callback(); }, 0);
    }
    filenames.forEach(function (filename, index) {
      readJSON(filename, function (err, res) {
        if (err) {
          if (!called) callback(err);
          return;
        }
        results[index] = res;
        if (0 === --pending) {
          callback(null, results);
        }
      });
    });
  }

写一个这么简单的异步函数就要用这么惊人的代码量。这段代码可以写成库函数来，来实现一个异步的 map 操作，但是这样如此繁琐的写法却只能解决了某部分特殊的情况。

##[all] Promise.all

参数：
  1. 需要执行的 promise 的数组

返回：
  1. 一个崭新的 promise ，并且将所有 promise 执行结果组成数组传入
  2. 如果其中某个 promise 执行异常，则会将第一个抛出的异常传入进行异常处理

:js
  function readJsonFiles(filenames) {
    // 特别注意这里传入的 readJSON 是一个函数，并不是调用该函数后的执行结果
    return Promise.all(filenames.map(readJSON));
  }
  readJsonFiles(['a.json', 'b.json']).done(function (results) {
    // results 是 a.json，b.json 中的值组成的数组
  }, function (err) {
    // 读取任何文件失败时，err 代表了第一个错误
  });

`Promise.all` 是一个内置方法，所以你无需担心要自己实现他， 更何况这个示例展示了 promise 的简约之美

:js
  function all(promises) {
    var accumulator = [];
    var ready = Promise.resolve(null);

    promises.forEach(function (promise, ndx) {
      ready = ready.then(function () {
        return promise;
      }).then(function (value) {
        accumulator[ndx] = value;
      });
    });

    return ready.then(function () { return accumulator; });
  }

代码解读，我们先定义了一个值来存放结果（就叫他 `accumulator`），还有一个值用来追踪最新的结果（叫他 `ready`）。 在每次循环遍历传入的 promise 数组时更新 `ready`，并将执行结果按顺序的存放在 `accumulator` 中。 遍历完成后， `ready` 就成为了一个会等待所有传入的 promise 完成的 promise，而各个 promise 的执行结果又会被存放到 `accumulator`。

我们现在就只是等待 `ready` promise 执行完成，再返回给我们 `accumulatro`。

原生的实现会比这个示例运行效率更好，不过这个例子告诉了你 promise 是用一种怎样有趣的方式链接起来。

##[race] Promise.race

有些应用场景需要两个 promise 互相竞争。来看下面的应用场景，实现一个超时方法。

:js
  function delay(time) {
    return new Promise(function (fulfill) {
      setTimeout(fulfill, time);
    });
  }
  function timeout(promise, time) {
    return new Promise(function (fulfill, reject) {
      // 跟 delay promise 进行竞争
      promise.then(fulfill, reject);
      delay(time).done(function () {
        reject(new Error('Operation timed out'));
      });
    });
  }

Promise.race 使这样的竞争写起来更简单。

:js
  function timeout(promise, time) {
    return Promise.race([promise, delay(time).then(function () {
      throw new Error('Operation timed out');
    })]);
  }

无论哪个 promise 有了返回（不管成功或者失败），就会赢得竞争，决定返回的结果。

##[apendix] Further Reading

 - [Generators](/generators/) - 学会如何同时用 generator 和 promise 让 promise 写起来更简单（在支持 ES6 特性的浏览器）
 - [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) - mozilla 开发者社区关于 promise 的文档
 - [then/promise](https://github.com/then/promise) - 所有这些辅助方法在 JavaScript 中的实现

@pager
  @previous(href="/api/")
    API 指南
  @next(href="/generators/")
    生成器

