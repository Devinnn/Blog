# 由`setTimeout`和`setImmediate`执行顺序的随机性窥探Node的事件循环机制

## 问题引入

接触过事件循环的同学大都会纠结一个点，就是在Node中`setTimeout`和`setImmediate`执行顺序的随机性。

比如说下面这段代码：

```javascript
setTimeout(() => {
    console.log('setTimeout');
}, 0);
setImmediate(() => {
    console.log('setImmediate');
})
```
执行的结果是这样子的：

![](http://oxybu3xjd.bkt.clouddn.com/18-2-2/76500835.jpg)

为什么会出现这种情况呢？别急，我们先往下看。

## 浏览器中事件循环模型

我们都知道，JavaScript是单线程的语言，对`I/O`的控制是通过异步来实现的，具体是通过“事件循环”机制来实现。

对于JavaScript中的单线程，指的是JavaScript执行在单线程中，而内部`I/O`任务其实是另有线程池来完成的。

在浏览器中，我们讨论事件循环，是以“从宏任务队列中取一个任务执行，再取出微任务队列中的所有任务”来分析执行代码的。但是在Node环境中并不适用。具体的浏览器事件循环解析：[传送门](http://www.yangzicong.com/article/3)

在Node中，事件循环的模型和浏览器相比大致相同，而最大的不同点在于Node中事件循环分不同的阶段。具体我们下面会讨论到。本文的核心也在这里。

## Node中事件循环阶段解析

下面是事件循环不同阶段的示意图：

![](http://oxybu3xjd.bkt.clouddn.com/18-3-12/4053975.jpg)

每个阶段都有一个先进先出的回调队列要执行。而每个阶段都有自己的特殊之处。简单来说，就是当事件循环进入某个阶段后，会执行该阶段特定的任意操作，然后才会执行这个阶段里的回调。当队列被执行完，或者执行的回调数量达到上限后，事件循环才会进入下一个阶段。

以下是各个阶段详情。

### timers

一个`timer`指定一个下限时间而不是准确时间，在达到这个下限时间后执行回调。在指定的时间过后，`timers`会尽早的执行回调，但是系统调度或者其他回调的执行可能会延迟它们。

> 从技术上来说，`poll`阶段控制`timers`什么时候执行，而执行的具体位置在`timers`。

下限的时间有一个范围：`[1, 2147483647]`，如果设定的时间不在这个范围，将被设置为1。

### idle, prepare 

系统内部的一些调用。

### I/O callbacks

这个阶段执行一些系统操作的回调，比如说TCP连接发生错误。

### poll

这是最复杂的一个阶段。

`poll`阶段有两个主要的功能：一是执行下限时间已经达到的`timers`的回调，一是处理`poll`队列里的**事件**。

注：Node很多API都是基于事件订阅完成的，这些API的回调应该都在`poll`阶段完成。

以下是Node官网的介绍：

![](http://oxybu3xjd.bkt.clouddn.com/18-2-2/76866942.jpg)

笔者把官网陈述的情况以不同的条件分解，更加的清楚。（如果有误，师请改正。）

当事件循环进入`poll`阶段：
- `poll`队列不为空的时候，事件循环肯定是先遍历队列并同步执行回调，直到队列清空或执行回调数达到系统上限。
- `poll`队列为空的时候，这里有两种情况。
    - 如果代码已经被`setImmediate()`设定了回调，那么事件循环直接结束`poll`阶段进入`check`阶段来执行`check`队列里的回调。
    - 如果代码没有被设定`setImmediate()`设定回调：
        - 如果有被设定的`timers`，那么此时事件循环会检查`timers`，如果有一个或多个`timers`下限时间已经到达，那么事件循环将绕回`timers`阶段，并执行`timers`的有效回调队列。
        - 如果没有被设定`timers`，这个时候事件循环是阻塞在`poll`阶段等待回调被加入`poll`队列。
    

### check

这个阶段允许在`poll`阶段结束后立即执行回调。如果`poll`阶段空闲，并且有被`setImmediate()`设定的回调，那么事件循环直接跳到`check`执行而不是阻塞在`poll`阶段等待回调被加入。

`setImmediate()`实际上是一个特殊的`timer`，跑在事件循环中的一个独立的阶段。它使用`libuv`的`API`来设定在`poll`阶段结束后立即执行回调。

**注：`setImmediate()`具有最高优先级，只要`poll`队列为空，代码被`setImmediate()`，无论是否有`timers`达到下限时间，`setImmediate()`的代码都先执行。**

### close callbacks

如果一个`socket`或`handle`被突然关掉（比如`socket.destroy()`），`close`事件将在这个阶段被触发，否则将通过`process.nextTick()`触发。

## 关于setTimeout和setImmediate

代码重现，我们会发现`setTimeout`和`setImmediate`在Node环境下执行是靠“随缘法则”的。

比如说下面这段代码：

```javascript
setTimeout(() => {
    console.log('setTimeout');
}, 0);
setImmediate(() => {
    console.log('setImmediate');
})
```
执行的结果是这样子的：

![](http://oxybu3xjd.bkt.clouddn.com/18-2-2/76500835.jpg)

为什么会这样子呢？

这里我们要根据前面的那个事件循环不同阶段的图解来说明一下：

首先进入的是`timers`阶段，如果我们的机器性能一般，那么进入`timers`阶段，一毫秒已经过去了（`setTimeout(fn, 0)`等价于`setTimeout(fn, 1)`），那么`setTimeout`的回调会首先执行。

如果没有到一毫秒，那么在`timers`阶段的时候，下限时间没到，`setTimeout`回调不执行，事件循环来到了`poll`阶段，这个时候队列为空，此时有代码被`setImmediate()`，于是先执行了`setImmediate()`的回调函数，之后在下一个事件循环再执行`setTimemout`的回调函数。

而我们在执行代码的时候，进入`timers`的时间延迟其实是随机的，并不是确定的，所以会出现两个函数执行顺序随机的情况。

那我们再来看一段代码：

```javascript
var fs = require('fs')

fs.readFile(__filename, () => {
    setTimeout(() => {
        console.log('timeout');
    }, 0);
    setImmediate(() => {
        console.log('immediate');
    });
});
```
这里我们就会发现，`setImmediate`永远先于`setTimeout`执行。

原因如下：

`fs.readFile`的回调是在`poll`阶段执行的，当其回调执行完毕之后，`poll`队列为空，而`setTimeout`入了`timers`的队列，此时有代码被`setImmediate()`，于是事件循环先进入`check`阶段执行回调，之后在下一个事件循环再在`timers`阶段中执行有效回调。

同样的，这段代码也是一样的道理：

```javascript
setTimeout(() => {
    setImmediate(() => {
        console.log('setImmediate');
    });
    setTimeout(() => {
        console.log('setTimeout');
    }, 0);
}, 0);
```
以上的代码在`timers`阶段执行外部的`setTimeout`回调后，内层的`setTimeout`和`setImmediate`入队，之后事件循环继续往后面的阶段走，走到`poll`阶段的时候发现队列为空，此时有代码被`setImmedate()`，所以直接进入`check`阶段执行响应回调（注意这里没有去检测`timers`队列中是否有成员到达下限事件，因为`setImmediate()`优先）。之后在第二个事件循环的`timers`阶段中再去执行相应的回调。

综上，我们可以总结：
- 如果两者都在主模块中调用，那么执行先后取决于进程性能，也就是随机。
- 如果两者都不在主模块调用（被一个异步操作包裹），那么`setImmediate`的回调永远先执行。

### process.nextTick() and Promise

对于这两个，我们可以把它们理解成一个微任务。也就是说，它其实不属于事件循环的一部分。

那么他们是在什么时候执行呢？

不管在什么地方调用，他们都会在其所处的事件循环最后，事件循环进入下一个循环的阶段前执行。

举个🌰：

```javascript
setTimeout(() => {
    console.log('timeout0');
    process.nextTick(() => {
        console.log('nextTick1');
        process.nextTick(() => {
            console.log('nextTick2');
        });
    });
    process.nextTick(() => {
        console.log('nextTick3');
    });
    console.log('sync');
    setTimeout(() => {
        console.log('timeout2');
    }, 0);
}, 0);
```

结果是：

![](http://oxybu3xjd.bkt.clouddn.com/18-2-2/10050363.jpg)

再解释一下：

`timers`阶段执行外层`setTimeout`的回调，遇到同步代码先执行，也就有`timeout0`、`sync`的输出。遇到`process.nextTick`后入微任务队列，依次`nextTick1`、`nextTick3`、`nextTick2`入队后出队输出。之后，在下一个事件循环的`timers`阶段，执行`setTimeout`回调输出`timeout2`。

### 最后

下面给出两段代码，如果能够理解其执行顺序说明你已经理解透彻。

代码1：

```javascript
setImmediate(function(){
  console.log("setImmediate");
  setImmediate(function(){
    console.log("嵌套setImmediate");
  });
  process.nextTick(function(){
    console.log("nextTick");
  })
});

// setImmediate
// nextTick
// 嵌套setImmediate
```

解析：事件循环`check`阶段执行回调函数输出`setImmediate`，之后输出`nextTick`。嵌套的`setImmediate`在下一个事件循环的`check`阶段执行回调输出嵌套的`setImmediate`。

代码2：

```javascript
var fs = require('fs');

function someAsyncOperation (callback) {
  // 假设这个任务要消耗 95ms
  fs.readFile('/path/to/file', callback);
}

var timeoutScheduled = Date.now();

setTimeout(function () {

  var delay = Date.now() - timeoutScheduled;

  console.log(delay + "ms have passed since I was scheduled");
}, 100);


// someAsyncOperation要消耗 95 ms 才能完成
someAsyncOperation(function () {

  var startCallback = Date.now();

  // 消耗 10ms...
  while (Date.now() - startCallback < 10) {
    ; // do nothing
  }

});
```

解析：事件循环进入`poll`阶段发现队列为空，并且没有代码被`setImmediate()`。于是在`poll`阶段等待`timers`下限时间到达。当等到`95ms`时，`fs.readFile`首先执行了，它的回调被添加进`poll`队列并同步执行，耗时`10ms`。此时总共时间累积`105ms`。等到`poll`队列为空的时候，事件循环会查看最近到达的`timer`的下限时间，发现已经到达，再回到`timers`阶段，执行`timer`的回调。

---

如果有什么问题，欢迎留言交流探讨。

参考链接：

https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/  

https://github.com/creeperyang/blog/issues/26


