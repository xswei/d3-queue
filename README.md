# d3-queue

**queue** 通过可配置的并发来执行 0 个或多个延迟的异步任务: 你可以控制同时进行多少个任务。当所有的任务完成或其中一个出错时候，队列会将结果传递给 *await* 回调。这个模块类似于[Async.js](https://github.com/caolan/async) 的 [parallel(并行执行)](https://github.com/caolan/async#paralleltasks-callback) (当 *concurrency* 为 `infinite` 时)，[series](https://github.com/caolan/async#seriestasks-callback) (当 *concurrency* 为 1 时)，以及 [queue](https://github.com/caolan/async#queue)，但是此模块占用空间更小: 从版本 2 起，d3-queue 在压缩后大概有 700 字节，而 Async 库却达到 4300。

每个任务被定义为将回调作为最后一个参数的函数。例如，下面为一个延时打印 `hello!` 的例子:

```js
function delayedHello(callback) {
  setTimeout(function() {
    console.log("Hello!");
    callback(null);
  }, 250);
}
```

当任务完成时，必须调用指定的回调。回调的第一个参数为 `null` (如果任务成功的话,否则为 `error`), 可选的第二个参数为任务的返回值(如果任务有多个返回值要将其包装在对象或数组中)。

要同时运行多个任务，则要创建一个队列，使用 `defer` 加入任务，然后注册一个 *await* 回调用来在所有任务完成或有任务出错时候执行:

```js
var q = d3.queue();
q.defer(delayedHello);
q.defer(delayedHello);
q.await(function(error) {
  if (error) throw error;
  console.log("Goodbye!");
});
```

当然你也可以使用 `for` 循环来加入多个任务:

```js
var q = d3.queue();

for (var i = 0; i < 1000; ++i) {
  q.defer(delayedHello);
}

q.awaitAll(function(error) {
  if (error) throw error;
  console.log("Goodbye!");
});
```

任务可以配置可选的参数，比如为延时打印的任务传入自定义参数:

```js
function delayedHello(name, delay, callback) {
  setTimeout(function() {
    console.log("Hello, " + name + "!");
    callback(null);
  }, delay);
}
```

在使用 [*queue*.defer](#queue_defer) 添加任务时，只需要将需要的参数传入 `defer` 即可。`defer` 会将参数传入指定的任务函数。你亦可以使用链式调用以避免创建本地变量:

```js
d3.queue()
    .defer(delayedHello, "Alice", 250)
    .defer(delayedHello, "Bob", 500)
    .defer(delayedHello, "Carol", 750)
    .await(function(error) {
      if (error) throw error;
      console.log("Goodbye!");
    });
```

[asynchronous callback pattern(异步回调模型)](https://github.com/maxogden/art-of-node#callbacks) 在 Node.js 中很常见，因此 `d3-queue` 可以直接与很多 Node APIs 协同工作。比如，并发获取[stat two files(两个文件信息)](https://nodejs.org/dist/latest/docs/api/fs.html#fs_fs_stat_path_callback):

```js
d3.queue()
    .defer(fs.stat, __dirname + "/../Makefile")
    .defer(fs.stat, __dirname + "/../package.json")
    .await(function(error, file1, file2) {
      if (error) throw error;
      console.log(file1, file2);
    });
```

你也可以将任务中断: 在定义任务队列时会返回一个带有 *abort* 方法的对象。因此，如果一个任务在开始时调用了 [setTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowTimers/setTimeout)，则中断时会执行 [clearTimeout](https://developer.mozilla.org/en-US/docs/Web/API/WindowTimers/clearTimeout) 来取消任务。例如:

```js
function delayedHello(name, delay, callback) {
  var id = setTimeout(function() {
    console.log("Hello, " + name + "!");
    callback(null);
  }, delay);
  return {
    abort: function() {
      clearTimeout(id);
    }
  };
}
```

当调用 [*queue*.abort](#queue_abort) 时，任何正在进行的任务会被立即停止，还未开始的任务不会启动。要注意的是你可以使用 *queue*.abort 终止但不取消任务，这种情况下未启动的任务被取消，但是已经执行的任务还会继续执行。比较方便的是 [d3-request](https://github.com/d3/d3-request) 库实现了 XMLHttpRequest 的终止。比如:

```js
var q = d3.queue()
    .defer(d3.request, "http://www.google.com:81")
    .defer(d3.request, "http://www.google.com:81")
    .defer(d3.request, "http://www.google.com:81")
    .awaitAll(function(error, results) {
      if (error) throw error;
      console.log(results);
    });
```

取消这些任务，调用 `q.abort()` 即可.

## Installing

NPM: `npm install d3-queue`. Bower: `bower install d3-queue`. 也可以下载 [latest release](https://github.com/d3/d3-queue/releases/latest). 可以直接从 [d3js.org](https://d3js.org) 以 [standalone library](https://d3js.org/d3-queue.v3.min.js) 或作为 [D3 4.0](https://github.com/d3/d3) 的一部分直接载入. 支持 AMD, CommonJS 以及基本的标签引入形式，如果使用标签引入则会暴露 `d3` 全局变量:

```html
<script src="https://d3js.org/d3-queue.v3.min.js"></script>
<script>

var q = d3.queue();

</script>
```

[在浏览器中测试 d3-queue.](https://tonicdev.com/npm/d3-queue)

## API Reference

<a href="#queue" name="queue">#</a> d3.<b>queue</b>([<i>concurrency</i>]) [<>](https://github.com/d3/d3-queue/blob/master/src/queue.js "Source")

使用指定的 *concurrency* 创建一个新的队列。如果没有指定 *concurrency* 则表示队列并发数为无穷大。此外 *concurrency* 必须为一个正整数。例如，如果 *concurrency* 为 1，则所有的任务会依次执行。如果 *concurrency* 为 3 则最多同时并发执行 3 个任务。当在 web 浏览器中加载资源时时非常有用的。

<a href="#queue_defer" name="queue_defer">#</a> <i>queue</i>.<b>defer</b>(<i>task</i>[, <i>arguments</i>…]) [<>](https://github.com/d3/d3-queue/blob/master/src/queue.js#L20 "Source")

将指定的异步 *task* 添加到队列中，可以指定可选的 *arguments* 。*task* 为一个函数以在任务开始执行时调用。它会将可选的参数以及外加一个 *callback* 传递给 *task* 函数，*task* 函数可以使用这些可选参数并在完成时自动调用 *callback* 。注意在添加任务的时候不需要指定 *callback* 只需要指定可选参数即可， *callback* 最后最后一个参数定义在具体的 *task* 函数中。在 *task* 中调用 *callback* 时候必须传入的第一个参数为 `error` 或 `null` 以表示任务是否成功执行。第二个参数可选，为任务的返回结果(多个返回结果时需要包装)。

例如，下面有一个简单的延时返回计算结果的任务:

```js
function simpleTask(callback) {
  setTimeout(function() {
    callback(null, {answer: 42});
  }, 250);
}
```

如果任务出错，则后续的未执行的任务将不会被执行。对于并发数为 1 的序列队列来说，这意味着某个任务只有在前序所有任务成功执行后才会执行。对于并发数高的队列来说，第一次出现任务出错就会将其报告给 *await* 回调，此时已经执行的任务会继续执行但是结果不会报告给 *await* 回调。

任务只能在[*queue*.await](#queue_await) 或 [*queue*.awaitAll](#queue_awaitAll) 之前被添加到队列中。如果一个任务在其之后添加，则会抛出错误。如果 *task* 不是函数则也会抛出错。

<a href="#queue_abort" name="queue_abort">#</a> <i>queue</i>.<b>abort</b>() [<>](https://github.com/d3/d3-queue/blob/master/src/queue.js#L29 "Source")

中断活动的任务，并调用活动任务的 *task*.abort 方法(如果有的话)。这个方法会阻止新的任务的执行，并立即调用[*queue*.await](#queue_await) 或 [*queue*.awaitAll](#queue_awaitAll) 回调并传递任务终止的错误表示。参考任务终止的实现[介绍](#d3-queue)。请注意如果你的任务不可终止，即使已经调用了 *await* 回调，已经运行的任务会继续执行。*await* 回调只会在终止时候调用一次，随后任务完成或失败都不会再次触发 *await* 回调。

<a href="#queue_await" name="queue_await">#</a> <i>queue</i>.<b>await</b>(<i>callback</i>) [<>](https://github.com/d3/d3-queue/blob/master/src/queue.js#L33 "Source")

设置所有任务完成时需要调用的 *callback* . 回调的第一个参数为 *error* (如果出错的话)，否则为 `null`。如果任务出错，则不会有其他的附加参数传递给 *callback*。除此之外，回调会依次传入每个任务的返回结果作为参数。例如:

```js
d3.queue()
    .defer(fs.stat, __dirname + "/../Makefile")
    .defer(fs.stat, __dirname + "/../package.json")
    .await(function(error, file1, file2) { console.log(file1, file2); });
```

如果所有 [deferred](#queue_defer) 任务都完成，则 *callback* 会立即执行。这个方法只能被调用一次，也就是只能使用一次 *queue*.await，如果多次指定 *await* 回调或在 [*queue*.awaitAll](#queue_awaitAll) 之后再次指定 *await* 回调则会抛出错误。如果 *cabblack* 不是一个函数也会抛出错误。

<a href="#queue_awaitAll" name="queue_awaitAll">#</a> <i>queue</i>.<b>awaitAll</b>(<i>callback</i>) [<>](https://github.com/d3/d3-queue/blob/master/src/queue.js#L39 "Source")

设置所有任务完成时执行的 *callback*。*callback* 的第一个参数为 `error` (如果有错误发生的话) 或者 `null` (没有出错)。如果出错的话 *callback* 不会再有其他的参数。此外，所有任务的返回结果会以数组的形式作为 *callback* 的第二参数传递。例如:

```js
d3.queue()
    .defer(fs.stat, __dirname + "/../Makefile")
    .defer(fs.stat, __dirname + "/../package.json")
    .awaitAll(function(error, files) { console.log(files); });
```

如果所有 [deferred](#queue_defer) 的任务都完成则会立即调用 *callback*。这个方法只能被调用一次，也就是只能指定一次 *queue*.awaitAll. 如果在 [*queue*.await](#queue_await) 之后再次调用或者多次调用都会抛出错误，如果 *callback* 不是函数类型也会抛出错误提示。
