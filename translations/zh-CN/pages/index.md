@title
  @primary
    Promises
  @secondary
    Forbes Lindesay 作，
    Enix, Zhang.Zhao 译。

##[Motivation] 动机

请看以下读取文件并解析为JSON的异步 Javascript 函数。虽然它简单易懂，但大多数情况下，你不会想要使用这种阻塞式的代码。因为它意味着当你从硬盘读取文件（一个耗时较长的操作）时，你不能进行任何其他操作。

:js
  function readJSONSync(filename) {
    return JSON.parse(fs.readFileSync(filename, 'utf8'));
  }

为了让我们的应用高效且响应迅速，我们需要令所有涉及到 IO 的操作以异步的方式进行。最简单的方法是使用回调方法(callback)。然而，下面这种简单的实现方式可能不够好：

:js
  function readJSON(filename, callback){
    fs.readFile(filename, 'utf8', function (err, res){
      if (err) return callback(err);
      callback(null, JSON.parse(res));
    });
  }

 - 额外的 `callback` 参数令人困惑：它是谁，从哪来，到哪去？
 - 它违反了控制流的基本原则
 - 它不能处理 `JSON.parse` 抛出的错误。

我们需要处理 `JSON.parse` 抛出的错误，但同时要避开由 `callback` 抛出的错误。而当我们达成这些要求时，代码也变得一团糟：

:js
  function readJSON(filename, callback){
    fs.readFile(filename, 'utf8', function (err, res){
      if (err) return callback(err);
      try {
        res = JSON.parse(res);
      } catch (ex) {
        return callback(ex);
      }
      callback(null, res);
    });
  }

除了上述错误处理代码带来的混乱，我们仍未解决额外的 `callback` 带来的问题。

Promises 可以帮助你自然地处理错误并写出不含 `callback` 的更干净的代码，并且不需要修改底层的架构（你可以通过纯 Javascript [实现](/implementing) 它们，并用于包装已有的异步操作）。

##[Defination] 什么是 Promise?

Promise 背后的核心思想是：一个 Promise 代表一个异步操作的结果。它处于以下三种状态中的一种：

 - pending - Promise 的初始状态
 - fulfilled - 表示操作成功的状态
 - rejected - 表示操作失败的状态

一旦 promise 处于 fulfilled 或 rejected 状态，它就不可再被改变。

##[Construction] 构建一个 Promise

当所有的 API 都返回 Promise，你将很少需要手动构建一个 Promise 。同时，我们需要一种 polyfill 已有API的方式，比如：

:js
  function readFile(filename, enc){
    return new Promise(function (fulfill, reject){
      fs.readFile(filename, enc, function (err, res){
        if (err) reject(err);
        else fulfill(res);
      });
    });
  }

我们使用 `new Promise` 构建 Promise。我们赋予构造函数一个并不实际起效的工厂函数， Promise 以两个参数立即调用它：第一个参数令 Promise 进入 fulfilled 状态，第二个参数令 Promise 进入 rejected 状态，你需要在异步操作完成时正确地调用它们。

##[done] 等待一个 Promise

想使用一个 Promise ，我们必须知道如何等待它直至它变成 fullfilled 或 rejected 。我们可以使用 `promise.done` 来实现这一要求（如果你试着运行这些示例，请查看位于本章节末的警告）。

知道了这一点，使用 Promise 重写前面的 `readJSON` 函数将易如反掌：

:js
  function readJSON(filename){
    return new Promise(function (fulfill, reject){
      readFile(filename, 'utf8').done(function (res){
        try {
          fulfill(JSON.parse(res));
        } catch (ex) {
          reject(ex);
        }
      }, reject);
    });
  }

这里仍然包含了大量的错误处理代码（你将在下一章看到如何优化这一问题），但我们已经可以少写不少错误处理代码了，而且我们也不再需要传入一个奇怪的额外参数： `callback` 。

@panel-danger
  @panel-title
    非标准
  @panel-body
    注意 `promise.done` （在本章示例中有被使用）并不是标准化的。尽管它被大部分主流 promise 库支持，且在教学辅助、实际开发中颇为有用，我仍建议你结合下面的 polyfill 使用它 ([minified](/polyfills/promise-done-$versions.promise$.min.js) / [unminified](/polyfills/promise-done-$versions.promise$.js))：

    :html
      <script src="https://www.promisejs.org/polyfills/promise-done-$versions.promise$.min.js"></script>

##[then] 转换 / 链式调用

继续看我们的示例，我们真正想要实现的是通过另一个操作转换 promise 。在我们的示例中，第二个操作是同步的，但有时它也可能是异步的。好消息是，Promise 提供一个（已完全标准化，[除了jQuery](#jquery) ）转换、衔接操作的方法。

简单来说， `.then` 之于 `.done` ，就像 `.map` 之于 `.forEach` 。或者说，当你需要获取之前异步操作的结果并对其做一些操作时（即使只是等待它执行结束），用 `.then` ，否则用 `.done` 。

现在我们能够轻易地重写原来的示例了：

:js
  function readJSON(filename){
    return readFile(filename, 'utf8').then(function (res){
      return JSON.parse(res)
    })
  }

因为 `JSON.parse` 只是一个函数，我们还可以这样重写它：

:js
  function readJSON(filename){
    return readFile(filename, 'utf8').then(JSON.parse);
  }

它非常像我们一开始在示例中看到的简单同步代码了。

##[Implementation] 实现 / Polyfills

Promise 在 node.js 和浏览器中都很实用。

###[jquery] jQuery

现在需要告诉你的是，事实上 jQuery 中的 Promise 是一种与其他库中的 Promise 完全不同的一个概念，它的 Promise 有令人困惑的 API 。不过你可以轻松地把 jQuery 的 Promise 转换成标准的 Promise ：

:js
  var jQueryPromise = $.ajax('/data.json');
  var realPromise = Promise.resolve(jQueryPromise);
  //现在可以大胆使用 `realPromise`

###[browser] 浏览器

Promises 现在只被一小部分浏览器支持([查看 kangax 兼容性列表](http://kangax.github.io/es5-compat-table/es6/#Promise))。
但好消息是它极易 polyfill ([minified](/polyfills/promise-$versions.promise$.min.js) / [unminified](/polyfills/promise-$versions.promise$.js)):

:html
  <script src="https://www.promisejs.org/polyfills/promise-$versions.promise$.min.js"></script>

目前没有浏览器支持 `Promise.prototype.done` ，所以如果你想在不用上述 polyfill 的情况下使用它，你必须至少使用这个 polyfill ([minified](/polyfills/promise-done-$versions.promise$.min.js) / [unminified](/polyfills/promise-done-$versions.promise$.js)):

:html
  <script src="https://www.promisejs.org/polyfills/promise-done-$versions.promise$.min.js"></script>

###[node] Node.js

通常，在 node.js 中 polyfill 不是一种通行的做法，更好的做法是使用 require 导入你所需要的库。

要安装 [promise](https://github.com/then/promise) ，可以运行该命令：

```
npm install promise --save
```

然后你可以使用 `require` 将它导入本地变量：

:js
  var Promise = require('promise');

"Promise"库同时提供一些非常实用的与 node.js 互动的扩展组件。

:js
  var readFile = Promise.denodeify(require('fs').readFile);
  // 现在 `readFile` 会返回一个 promise 
  // 而不需要使用一个 `callback`

  function readJSON(filename, callback){
    // 如果提供了 callback ，则以错误为第一个参数、结果为第二个参数
    // 调用它，然后返回 `undefined`
    // 如果没有提供回调函数，就返回 promise
    return readFile(filename, 'utf8')
      .then(JSON.parse)
      .nodeify(callback);
  }

##[apendix] 扩展阅读

 - [用法](/patterns/) - Promise 的使用方法, 介绍大量可以节省你时间的工具与方法。
 - [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) - 来自 MDN (Mozilla Developer Network) 的非常棒的 Promise 文档。
 - [YouTube](https://www.youtube.com/watch?v=qbKWsbJ76-s) - 一个我在 JSConf.eu 的演讲，关于许多本文中亦有提及的内容。

@pager
  @next(href="/api/")
    API 参考
