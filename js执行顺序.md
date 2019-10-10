# js 执行顺序

## 关于 javaacript

javascript 是一门单线程语言，所以 javascript 是按语句的执行顺序执行的。
虽然 js 是单线程，但是我们可以将任务分成两类

1. 同步任务：需要执行的任务在主线程上排队，一次执行
2. 异步任务：没有立马执行但是需要被执行的任务，放在 任务队列里面，

## javascript 事件循环

当我们打开网站的时候，网页的渲染其实是一堆同步任务，不如页面骨架和页面元素的渲染，但是想图片音乐等占用资源大耗时久的任务就是异步任务，
异步执行：

1. 所有同步任务偶在主线程上执行，形成一个很执行栈
2. 主线程之外，还存在一个任务队列（task queue）只要异步任务有了运行结果，就在“任务队列”之中放置一个事件。
3. 一旦“执行栈”中的所有同步任务执行完毕，系统就会读取“任务队列”，看看里面有哪些事件。那些对应的异步任务，就结束等待状态，进入执行栈开始被执行。
4. 主线程不断重复以上三步。

js 引擎存在 monitoring process 进程，会持续不断的检查主线程执行栈是否为空，一旦为空，就会去 Event Queue 那里检查是否有等待被调用的函数。
![N]M]\_N{OL(HBZ`B89REKMQ9.png](https://upload-images.jianshu.io/upload_images/9374643-598ff6c029970ff9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
同步和异步分别进入到不同的执行场所，同步进入到主线程，异步的进入到 event table 并注册函数
当指定事件完成之后，event table 会将函数移入到 event queue
当主线程的任务执行完毕之后，会把 event queue 里面读取对应的函数进入主线程执行
上述过程会不断循环，形成 event loop(事件循环)

## setTimeout

在使用 setTimeout 的时候，经常会发现设定的时间与自己设定的时间有差异，贴段代码看一下

```js
setTimeout(() => {
  task();
}, 3000);

console.log('执行 console');

// 执行 console
// task()
```

上文所说的 setTimeout 是一个异步的所以会先执行 console 这个同步任务
但是，如果改成下面这段会发现执行时间远远超过预定的时间

```js
setTimeout(() => {
  task();
}, 3000);

sleep(10000000);
```

这是为啥？？
我们来看一下是怎么执行的：

task()进入到 event table 里面注册计时
然后主线程执行 sleep 函数，但是非常慢。计时任然在继续

3 秒到了。task()进入 event queue 但是主线程依旧没有走完
终于过了 10000000ms 之后主线程走完了，task()进入到主线程
所以可以看出其真实的时间是远远大于 3 秒的
还会遇到一种情况，就是 setTimeout(fn(),0),这样的代码其含义主要是在这个任务会在主线程最早可得的空闲时间执行，换句话说就是主线程的任务执行结束之后立马执行

```js
console.log('先执行这里');
setTimeout(() => {
  console.log('执行啦');
}, 0);

// 先执行这里
// 执行啦
```

HTML5 标准规定了 setTimeout()的第二个参数的最小值（最短间隔），不得低于 4 毫秒，如果低于这个值，就会自动增加。在此之前，老版本的浏览器都将最短间隔设为 10 毫秒。另外，对于那些 DOM 的变动（尤其是涉及页面重新渲染的部分），通常不会立即执行，而是每 16 毫秒执行一次。

4.promise 和 process.nextTick(callback)
process.nextTick(callback)类似 node.js 版的"setTimeout"，在事件循环的下一次循环中调用 callback 回调函数。

除了广义的同步任务和异步任务，我们可以分的更加精细一点：

1. macro-task(宏任务)：包括整体代码 script，setTimeout，setInterval
2. micro-task(微任务)：Promise，process.nextTick
   不同的任务会进入到不同的 event queue。比如 setTimeout 和 setInterval 会进入相同的 Event Queue。

事件循环，宏任务，微任务的关系图

![](https://imgs.react.mobi/FoS-s3FqmJBNzWf-8uTgeVoFesk6)

不哔哔，搞段代码瞅瞅：

```js
setTimeout(function() {
  console.log('setTimeout');
});

new Promise(function(resolve) {
  console.log('promise');
}).then(function() {
  console.log('then');
});

console.log('console');

// promise
// console
// then
// setTimeout
```

首先会遇到 setTimeout,将其放到红任务 event queue 里面
然后回到 promise ， new promise 会立即执行， then 会分发到微任务
遇到 console 立即执行
整体宏任务执行完成，接下来判断是否有微任务
，刚刚放到微任务里面的 then，执行
ok，第一轮事件结束，进行第二轮，刚刚我们放在 event queue 的 setTimeout 函数进入到宏任务，立即执行
结束
终于结束了，我们来贴段巨复杂的代码搞一搞

```js
console.log('1');

setTimeout(function() {
  console.log('2');
  process.nextTick(function() {
    console.log('3');
  });
  new Promise(function(resolve) {
    console.log('4');
    resolve();
  }).then(function() {
    console.log('5');
  });
});
process.nextTick(function() {
  console.log('6');
});
new Promise(function(resolve) {
  console.log('7');
  resolve();
}).then(function() {
  console.log('8');
});

setTimeout(function() {
  console.log('9');
  process.nextTick(function() {
    console.log('10');
  });
  new Promise(function(resolve) {
    console.log('11');
    resolve();
  }).then(function() {
    console.log('12');
  });
});

// 1,7,6,8,2,4,3,5,9,11,10,12
```

惊不惊喜，意不意外
我们来分析一下

首先先执行 console.log(1) 然后将 setTimeout 放到宏任务 event queue 里面 记作 setTimeout 1 ，接着 看到 process.nextTick ，将其放到微任务里面 ，记作 process 1，然后 看到 new promise 立即执行输出 9 ，将里面的 then 放到 微任务里面 记作 then 2, 继续，遇到 setTimeout 放到宏任务里面记作 setTimeout 2 。目前输出的是：1,7，
OK, 接下来，开始判断是否有微任务，刚刚放入到微任务 event queue 的进入到主程序开始执行，process 1 ， then 2 目前输出的是：6,8、
接下来，微任务的 event queue 空了，进行下一轮事件，将刚刚放到宏任务的 setTimeout 1 进入到主线程
遇到 console 立即执行， 遇到 process.nextTick 放到微任务 event queue 里面 记作 process1， 接着遇到 new Promise 立即执行， 将 then 放到 event queue 里面 记作 then 2，OK，当前宏任务里的任务执行完了，判断是否有微任务，发现有 process1， then 2 两个微任务 ， 一次执行 目前输出的是：2,4,3,5、
目前主线程里的任务都执行结束了,又开始第三轮事件循环，同上（字太多，省略。。。。） 目前输出的是：9,11,10,12、
注意: 以上所说只能是在浏览器中的执行顺序，

## node.js 的 Event Loop

Node.js 也是单线程的 Event Loop，但是它的运行机制不同于浏览器环境。浏览器的 Event loop 是在 HTML5 中定义的规范，而 node 中则由 libuv 库实现。
node.js 运行机制

1. v8 引擎解析 JavaScript 脚本。
2. 解析后的代码，调用 Node API。
3. libuv 库负责 Node API 的执行。它将不同的任务分配给不同的线程，形成一个 Event Loop（事件循环），以异步的方式将任务的执行结果返回给 V8 引擎。
4. V8 引擎再将结果返回给用户。
