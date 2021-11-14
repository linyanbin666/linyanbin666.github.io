---
title: "Promise学习笔记（B站）"
date: 2021-04-18T16:27:31+08:00
tags: ["Promise"]
categories: ["前端"]
draft: false
---
### Promise是什么？
#### 理解
1. Promise是ES6引入的一种异步编程的解决方案
2. 具体表达
- 从语法上来说，Promise是一个构造函数
- 从功能上来说，promise对象用来封装一个异步操作并可以获取其返回结果
#### promise的状态改变
- pending 变为 resolved（fulfilled）
- pending 变为 rejected
PS：一个promise对象只能改变一次，无论变为成功或失败，都会有一个结果数据，成功的结果一般称为 value，失败的结果一般称为 reason
#### promise的基本运行流程
![Promise运行流程](/images/atguigu_promise_note-promise_run_process.png)
#### promise的基本使用
```
const p = new Promise((resolve, reject) => {
    setTimeout(() => {
        const time = Date.now();
        if (time % 2 === 0) {
            resolve('成功的数据，time=' + time)
        } else {
            reject('失败的数据，time=' + time)
        }
    }, 1000)
})
p.then(value => {
    console.log('成功的回调', value)
}, reason => {
    console.log('失败的回调', value)
})
```
### 为什么要用Promise
#### 指定回调函数的方式更加灵活
- 旧的：必须在启动异步任务前指定
- promise：启动异步任务 => 返回promise对象 => 给promise对象绑定回调函数（甚至可以在异步任务执行结束后再指定）
#### 支持链式调用，解决回调地狱问题
- 什么是回调地狱？
回调函数嵌套调用，外部回调函数异步执行的结果是嵌套的回调函数执行的条件
- 回调地狱的缺点？
不便于阅读/不便于异常处理
- 解决方案：promise链式调用
- 终极解决方案：async/await
### 如何使用Promise？
#### API
- Promise构造函数：Promise(executor)
  - executor函数：执行器 (resolve, reject) => {}
  - resolve函数：成功时调用的函数 resolve => {}
  - reject函数：失败时调用的函数 reject => {}
- Promise.prototype.then
- Promise.prototype.catch
- Promise.prototype.finally
- Promise.resolve
- Promise.reject
- Promise.race
- Promise.all
- ....
#### promise的几个关键问题
1. 如何改变promise的状态？
- resolve(vale)：如果当前是 pending 就会变为 resolved
- reject(reason)：如果当前是 pending 就会变为 rejected
- 抛出异常：如果当前是 pending 就会变为 rejected

2. 一个 promise 指定多个成功/失败回调函数，都会调用吗？
单 promise 改变为对应状态时都会调用

3. 改变 promise 状态和指定回调函数谁先谁后？
- 都有可能，正常情况下是先指定回调再改变状态，但也可以先改状态再指定回调
- 如何先改状态再指定回调？
  - 在执行器中直接调用resolve()/reject()
  - 延迟更长时间才调用then()

4. promise.then() 返回的新 promise 的结果状态由什么决定？
- 简单表达：由 then() 指定的回调函数执行的结果决定
- 详细表达：
  - 如果抛出异常，新 promise 变为 rejected，reason 为抛出的异常
  - 如果返回的是非 promise 对象，新 promise 变为 resolved，value 为返回的值
  - 如果返回的是一个 promise 对象，此 promise 对象的结果会成为新 promise 的结果

5. promise 如何串连多个操作任务？
promise 的 then() 返回一个新的 promise，通过 then() 的链式调用串连多个同步/异步任务

6. promise 异常传透
- 理解：当使用 promise 的 then 链式调用时，可以在最后指定失败的回调，前面任何操作出了异常，都会传到最后失败的回调中处理
- 原理：then 方法未指定 onRejected 回调函数时，其默认值会是：reason => Promise.reject(reason)，将异常传递下去

7. 中断 promise 链
- 理解：当使用 promise 的 then 链式调用时，在中间中断，不在调用后面的回调函数
- 办法：在回调函数中返回一个 pending 状态的 promise 对象（如：new Promise(() => {})）

### async与await
- async 函数
  - 函数的返回值是 promise 对象
  - promise 对象的结果由 async 函数执行的返回值决定
- await 表达式
  - await 右侧表达式一般为 promise 对象，但也可以是其他值
  - 如果表达式是 promise 对象，await 返回的是 promise 成功的值
  - 如果表达式是其他值，直接将此值作为 await 的返回值
- 注意点
  - await 必须写在 async 函数中，但 async 函数可以没有 await 
  - 如果 await 的 promise 失败了，会抛出异常，需要通过try...catch捕获处理
### JS异步之宏队列与微队列
#### 原理图
> 图片来源：https://www.cnblogs.com/BAHG/p/12921321.html
![JS异步之宏队列与微队列](/images/atguigu_promise_note-macro_queue_and_micro_queue.png)

#### 说明
1. JS中用来存储执行回调函数的队列包含2个，分别为宏队列和微队列
2. 宏队列：用来保存待执行的宏任务（回调），比如：定时器回调/DOM事件回调/Ajax回调
3. 微队列：用来保存待执行的微任务（回调），比如：promise的回调、MutationObserver的回调
4. JS执行时会区分这两个队列
- JS引擎首先必须先执行所有的初始化同步任务代码
- 每次准备取出第一个宏任务执行前，都要将所有的微任务取出执行，即微队列的优先级比宏任务高，且与微任务所处的代码位置无关
