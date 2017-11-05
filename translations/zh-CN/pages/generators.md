@title
  @primary
    生成器
  @secondary
    Forbes Lindesay 作
    Cheng Liu 译

##[intro] 基本介绍

生成器是 ES6 规范中最激动人心的一个特性。 主要的应用场景是表述之后还会运行的执行步骤（可能是无限次地）。例如，下列的函数返回了前 n 个正整数。

:js
  function count(n){
    var res = []
    for (var x = 0; x < n; x++) {
      res.push(x)
    }
    return res
  }

  for (var x of count(5)) {
    console.log(x)
  }

我们可以用生成器特性来实现相似功能，返回所有的正整数。

:js
  function* count(){
    for (var x = 0; true; x++) {
      yield x
    }
  }

  for (var x of count()) {
    console.log(x)
  }

代码解读，`count` 函数在此处被用来惰性求值， 他会在每次 `yield` 时暂停，等待下一个值被请求。这意味着 for-of 循环会无限的执行下去，持续地打印出下一个整数。

真正让人兴奋的是，我们能从中看到生成器有能力有序地暂停函数执行，以此来帮助我们书写异步执行的代码。特别是， 生成器允许我们在已有的流程结构中，做一些异步的操作。（例如，循环、条件分支、try-catch 结构）

生成器**没有**提供一种能够表达异步操作结果的方式（成功或者失败）。为此，我们需要 promise 。

这篇文章的目的就是教会你能够像下面那样书写代码：

:js
  var login = async(function* (username, password, session) {
    var user = yield getUser(username);
    var hash = yield crypto.hashAsync(password + user.salt);
    if (user.hash !== hash) {
      throw new Error('Incorrect password');
    }
    session.setUser(user);
  });

上述的代码阅读起来就好像是同步执行的代码，但实际上他在每次 `yield` 关键词的地方都在处理异步逻辑。调用 `login` 函数的结果将会是一个 promise 。

##[fulfilling] 正常是如何运作的？

如你在基本介绍中所看到的那样，我们能够用 `yeild` 关键词来暂停函数的执行，并等待 promise 的执行。现在我们只需要一种能够更好的对生成器函数进行控制的方法，每当 promise 执行完成后，让生成器函数能够再执行一次。幸运的是，我们可以通过生成器的 `.next` 方法达到这个目的。

:js
  function* demo() {
    var res = yield 10;
    assert(res === 32);
    return 42;
  }

  var d = demo();
  var resA = d.next();
  // => {value: 10, done: false}
  var resB = d.next(32);
  // => {value: 42, done: true}
  // 此时再调用 next 方法就会抛出异常。

代码解读：我们先执行一次 `d.next()` 让生成器运行到 `yield` 的代码，然后我们再一次执行 `d.next()` 时，手动给定 `yield` 后表达式的结果。然后函数执行到了 `return` 的地方返回最终结果。

我们可以利用这个特点，通过调用 `.next(result)` 的方式表述一个 promise 对象是执行成功的。

##[rejecting] 异常是如何运作的？

我们也需要一种方式来表述一个 promise 对象，在被生成器调用时抛出了异常。这就需要在生成器中调用 `.throw(error)` 方法来完成。

:js
  var sentinel = new Error('foo');
  function* demo() {
    try {
      yield 10;
    } catch (ex) {
      assert(ex === sentinel);
    }
  }

  var d = demo();
  d.next();
  // => {value: 10, done: false}
  d.throw(sentinel);

就如之前说的那样，我们调用 `d.next()` 运行到了第一个 `yield` 关键词。然后调用 `d.throw(error)` 表示我们代码执行失败的状态，这样就会让生成器表现得像是在 `yield` 的地方抛出了异常。在这个例子里，就会被 `catch` 块捕捉到。

##[both] 结合使用正常与异常

把上述例子结合起来使用的话，我们只需要不断地手动将代码执行下去，让 promise 的结果来改变代码的流程。我们下面来看一个简单的实现。

:js
  function async(makeGenerator){
    return function () {
      var generator = makeGenerator.apply(this, arguments);

      function handle(result){
        // result => { done: [Boolean], value: [Object] }
        if (result.done) return Promise.resolve(result.value);

        return Promise.resolve(result.value).then(function (res){
          return handle(generator.next(res));
        }, function (err){
          return handle(generator.throw(err));
        });
      }

      try {
        return handle(generator.next());
      } catch (ex) {
        return Promise.reject(ex);
      }
    }
  }

注意，我们用到了 `Promise.resolve` 来确保我们总是能够使用我们所期望的 promise 对象，同时也用到了 `Promise.reject` 结合 try/catch 代码块来确保同步的异常错误总是被转化成异步的异常错误。

##[apendix] Further Reading

 - [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*) - mozilla 开发者社区的生成器文档
 - [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) - mozilla 开发者社区的 promise 文档
 - [regenerator](http://facebook.github.io/regenerator/) - Facebook 关于增加生成器函数支持的项目
 - [gnode](https://github.com/TooTallNate/gnode) - 使用 regenerator 项目来为以前版本的 node.js 提供生成器函数支持的命令行应用
 - [then-yield](https://github.com/then/yield) - 配合生成器函数使用 promise 来撰写代码的库
 - [Task.js](http://taskjs.org/) - 另一个结合生成器函数与 promise 使用的库
 - [YouTube](https://www.youtube.com/watch?v=qbKWsbJ76-s) - 在 JSConf.eu 上这篇文章主旨思想讨论的视频

@pager
  @previous(href="/patterns/")
    用法
  @next(href="/implementing/")
    实现

