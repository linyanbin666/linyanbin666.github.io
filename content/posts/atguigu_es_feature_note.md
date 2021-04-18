---
title: "ES6-11新特性学习笔记（B站）"
date: 2021-04-18T16:09:33+08:00
tags: ["ES6", "ES新特性"]
categories: ["前端"]
draft: false
---

> 此文为学习 [尚硅谷Web前端ES6教程，涵盖ES6-ES11](https://www.bilibili.com/video/BV1uK411H7on) 记录的笔记

### ES6
#### let变量声明
- 变量不能重复声明
```
let i = 1;
let i = 2; // error
```
- 块级作用域
```
{
    let i = 1;
}
console.log(i); // error
```
- 不存在变量提升
```
console.log(i); // error
let i = 1;
```
- 不影响作用域链
```
{
    let i = 1;
    function fn() {
        console.log(i); // 1
    }
    fn();
}
```
#### const声明常量
- 常量声明时需要赋初始值，一般常量名使用大写
```
const ERROR; // error
```
- 常量值不能修改
```
const ERROR = "error";
ERROR = ""; // error
```
- 块级作用域（同let）
- 对于数组和对象值的修改，不算对常量的修改
```
const ARR = ["a", "b"];
ARR[0] = "c";
ARR.push("d"); 
const OBJ = {
    name: "a"
}
OBJ[name] = "b";
OBJ.age = 10; 
```
#### 变量的解构赋值
- 数组解构
```
let arr = ["a", "b"];
const [a, b] = arr;
```
- 对象解构
```
let obj = {
    name: 'horizon',
    age: 23
};
const {name, age} = obj;
```
#### 模板字符串
```
// 使用 `` 定义模板字符串
const key = "key"
const str = `${key}-value`
```
#### 对象的简化写法
```
const name = "horizon";
const age = 23;
const user = {
    name,
    age
    eat() {
        console.log('eat');
    }
};
```
#### 箭头函数
- this是静态的。this 始终指向函数声明时所在作用域下的 this 的值
```
function getName() {
    console.log(this.name);
}
const getName2 = () => {
    console.log(this.name);
} 
window.name = 'horizon';

getName(); // horizon
getName2(); // horizon

const user = {
    name: 'jeson'
}
getName.call(user); // jeson
getName2.call(user); // horizon
```
- 不能作为构造实例化对象
```
const Persion = (name) => {
    this.name = name;
}
const me = new Persion('horizon'); // error
```
- 不能使用 arguments 变量
```
const fn = () => {
    console.log(arguments); // error
}
```
- 箭头函数的简写
```
// 1. 当形参有且只有一个时，可以省略小括号
const fn = n => {
    return n + n;
}
// 2. 当代码体只有一条语句时，可以省略花括号，且如果该语句是返回值的语句，必须省略 return
const pw = n => n * n; 
```
- 箭头函数的使用场景  
> - 适合与 this 无关的回调，如定时器、数组的方法回调
> - 不适合与 this 有关的回调，如事件回调，对象的方法
#### 函数参数的初始值，一般位置要靠后
- 形参初始值， 
```
function incr(a, b = 1) {
    return a + b;
}
console.log(incr(10)); // 11
```
- 与解构赋值结合
```
function getName({name = 'horizon'}) {
    console.log(name);
}
getName(); // horizon
getName({name: 'jeson'}); // jeson
```
#### rest参数
rest参数用于获取函数的实参，用来代替 arguments
```
// arguments是对象
function output() {
    console.log(arguments);
}
// args是数组，必须放在参数的最后一个位置
const output2 = (...args) {
    console.log(args);
}
```
#### 扩展运算符
[...] 扩展运算符能够将【数组】转换为逗号分隔的【参数序列】
```
function output(a, b) {
    return a + b;
}
const arr = [1, 2];
console.log(output(...arr)); // 3
```
- 扩展运算符的应用
  - 数组的合并
  - 数组的克隆 
  - 将伪数组转为真正的数组
#### Symbol  
新的原始数据类型，表示独一无二的值，Symbol的特点：
- Symbol的值是唯一的，用来解决命名冲突的问题
- Symbol值不能与其他数据进行运算
- Symbol定义的对象不能使用 for...in 循环符，但是可以使用 Reflect.ownKeys 来获取对象的所有键名
#### 迭代器
迭代器是一种接口，为各种不同的数据结构提供统一的访问机制，任何数据结构只要部署了 Iterator 接口（对象里的一个属性：Symbol.iterator），就可以完成遍历操作
- ES6创造了一种新的遍历命令 for...of 循环，Iterator 接口主要供 for...of 消费
- 原生具备 Iterator 接口的数据（可用 for...of 循环）
  - Array
  - Arguments
  - Set
  - Map
  - String
  - TypeArray
  - NodeList
- 工作原理
  - 创建一个指针对象，指向当前数据结构的起始位置
  - 第一次调用对象的 next 方法，指针自动指向数据的第一个成员
  - 接下来不断调用 next 方法，指针一直往后移动，直到指向最后一个
  - 每调用 next 方法返回一个包含 value 和 done 属性的对象
```
const user = {
    name: 'horizon',
    skill: ['java', 'js', 'ts'],
    [Symbol.iterator]() {
        let i = 0;
        return {
            next: () => {
                if (i < this.skill.length) {
                    const result = {
                        value: this.skill[i],
                        done: false
                    }
                    i++;
                    return result;
                } else {
                    return {
                        value: undefined,
                        done: true
                    }
                }
            }
        }
    }
}
for (const each of user) {
    console.log(each);
}
```
#### 生成器函数
生成器函数是ES6提供的一种异步编程解决方案，语法行为与传统函数完成不同
- 使用
```
// 加"*"表明是生成器函数
function * gen() {
  console.log('1'); // code1
  yield '1';
  console.log('2'); // code2
  yield '2';
}
let iterator = gen(); // 调用生成器函数返回的是一个迭代器对象
iterator.next(); // 执行code1
iterator.next(); // 执行code2

// 输出 yield 返回的结果
for (const v of iterator) {
    console.log(v);
}
```
- 实例
模拟获取：用户数据 -> 订单数据 -> 商品数据
```
// 调用生成器函数
let iterator = gen();
iterator.next();
function getUsers() {
    setTimeout(() => {
        let data = "用户数据";
        iterator.next(data);
    }, 1000)
}
function getOrders() {
    setTimeout(() => {
        let data = "订单数据";
        // 第二次调用 next 传入的实参将作为第一个 yield 语句返回的结果
        iterator.next(data);
    }, 1000)
}
function getProducts() {
    setTimeout(() => {
        let data = "商品数据";
        iterator.next(data);
    }, 1000)
}
function * gen() {
    const users = yield getUsers();
    const orders = yield getOrders();
    const products = yield getProducts();
}
```
#### Promise
Promise是ES6引入的异步编程的新解决方案，语法上Promise是一个构造函数，用来封装异步操作并可以获取成功或失败的结果。
#### Set
常用API
- add
- delete
- has
- clear
- size（属性）
```
const set = new Set([1, 2, 3, 1]);
for (const num of set) {
    console.log(num);
}
```
#### Map
常用API
- set
- delete
- get
- clear
- size（属性）
```
let m = new Map();
m.set('name', 'horizon');

for (const o of m) {
    console.log(o); // 输出长度为2的数组，第一个元素为key，第二个元素为value
}
```
#### Class类
通过class关键字可以定义类，基本上ES6的class可以看做是一种语法糖，它的绝大部分功能ES5都可以做到，新的class写法只是让对象原型的写法更加清晰、更像面向对象编程的语法而已。
- class声明类
- constructor定义构造函数初始化
- extends继承父类
- super调用父类构造方法
- static定义静态方法和属性
- 父类方法可以重写
```
// ES5实现类的写法
function Person(name, age) {
    this.name = name;
    this.age = age;
}
Person.prototype.eat = function() {
    console.log('eat');
}
// ES6实现类的写法
class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }
    
    eat() {
        console.log('eat');
    }
}
Person person = new Person('horizon', 23);
person.call();

// ES5实现继承
function Phone(brand, price) {
    this.brand = brand;
    this.price = price;
}
Phone.prototype.call = function() {
    console.log('I can call');
}
function SmartPhone(brand, price, color) {
    Phone.call(this, brand, price);
    this.color = color
}
// 设置子级构造函数的原型
SmartPhone.prototype = new Phone;
SmartPhone.prototype.contructor = SmartPhone
SmartPhone.prototype.photo = function() {
    console.log('I can photo');
}

// ES6实现继承
class Phone {
    constructor(brand, price) {
        this.brand = brand;
        this.price= price;
    }
    call() {
        console.log('I can call');
    }
}
class SmartPhone extends Phone {
    constructor(brand, price, color) {
        super(brand, price);
        this.color = color;
    }
    photo() {
        console.log('I can photo');
    }
}
```
#### 数值扩展
- Math.EPSILON：js表示的最小精度
- 二进制和八进制
- Number.isFinite：检测一个数值是否为有限数
- Number.isNaN：检测一个数值是否为NaN
- Number.isInteger：判断一个数是否为整数
- Math.trunc：将数字的小数部分抹掉
- Math.sign：判断一个数是正数、零还是负数
#### 对象方法扩展
- Object.is：判断两个值是否相等
```
console.log(Object.is(1, 1)); // true
console.log(NaN === NaN); // false
console.log(Object.is(NaN, NaN)); // true
```
- Object.assign：对象的合并
- Object.setPropertyOf、Object.getPropertyOf：设置原型对象
#### 模块化
模块化是指将一个大的程序文件，拆分成许多小的文件，然后将小文件组合起来
- 模块化的好处
  - 防止命名冲突
  - 代码复用
  - 高维护性
- 模块化规范产品
  - CommonJS => NodeJS、Browserify
  - AMD => requireJS
  - CMD => seaJS
- ES6模块化语法
  - export命令用于规定模块的对外接口
  - import命令用于输入其他模块提供的功能
### ES7
#### Array.prototype.includes
includes 方法用来检测数组中是否包含某个元素，返回布尔类型值
#### 指数操作符
在ES7中引入了指数运算符[**]，用来实现幂运算，功能与Math.pow结果相同
### ES8
#### async和await
async和await两种语法结合可以让异步代码像同步代码一样
##### async函数
- async函数的返回值为Promise对象
- Promise对象的结果由async函数执行的返回值决定
##### await表达式
- await必须写在async函数中
- await右侧的表达式一般为Promise对象
- await返回的是Promise成功的值
- await的Promise失败了，就会抛出异常，需要通过try...catch捕获处理
#### 对象方法扩展
- Object.values()：返回一个给定对象的所有可枚举属性值的数组
- Object.entries()：返回一个给定对象自身可遍历属性[key,value]的数组
- Object.getOwnPropertyDescriptors()：返回指定对象所有自身属性的描述对象
### ES9
#### 扩展运算符与rest参数
扩展运算符与rest参数在ES6中已引入，但只针对数组，在ES9中为对象提供了像数组一样的扩展运算符与rest参数
#### 正则扩展
- 命名捕获分组（?<name>）
```
let str = '<a href="http://www.baidu.com">百度</a>';
const reg = /<a href="(?<url>.*)">(?<text>.*)<\/a>/;
const result = reg.exec(str)
console.log(result.groups.url);
console.log(result.groups.text);
```
- 反向断言（?<=）
```
let str = "ES110哔哔120哩哩";
// 正向断言
const reg = /\d+(?=哩)/;
console.log(reg.exec(str)); // 120
// 反向断言
const reg2 = /(?<=哔)\d+/;
console.log(reg2.exec(str)); // 120
```
- dotAll模式  
正则的dot(.)匹配除换行符以外的任意一个字符，dotAll模式下dot(.)将可以匹配任意字符（包括换行符）。
```
// 正则表达式的最后加了s符号
const reg = /.*/s;
```
### ES10
#### 对象扩展方法Object.fromEntries
接收一个二维数组或Map将其转换为对象，与ES8的Object.entries方法互为逆运算
```
const res = Object.fromEntries([
    ['name', 'horizon']
])
console.log(res); // {name: 'horizon'}
const m = new Map();
m.set('name', 'horizon');
const obj = Object.fromEntries(m);
console.log(obj); // {name: 'horizon'}
```
#### 字符串扩展方法trimStart与trimEnd
- trimStart：清除字符串左侧空白字符
- trimEnd：清除字符串右侧空白字符
#### 数组扩展方法flat与flatMap
- flat：将多维数组转化为低维数组，参数可传入需要降几维，默认值为1
- flatMap：将map的结果进行维度降低
```
const array = [1,2,[3,4,[5,6]]];
console.log(array.flat()); // [1,2,3,4,[5,6]]
console.log(array.flat(2)); // [1,2,3,4,5,6]
const arr = [1,2,3,4,5];
const newArr = arr.flatMap(each => [each * 10]); // [10,20,30,40,50]
```
#### Symbol.prototype.description
```
const s = Symbol('horizon');
console.log(Symbol.description); // horizon
```
### ES11
#### 私有属性
```
class Person {
    // 公有属性
    name;
    // 私有属性
    #age;
    constructor(name, age) {
        this.name = name;
        this.#age = age;
    }
}
const person = new Person('horizon', 23);
console.log(person.name); // horizon
console.log(person.#age); // error
```
#### Promise.allSettled
接收一个Promise数组，不管Promise数组内的Promise是否成功或失败，allSettled返回的Promise都是成功状态的，这与all不同。
#### 字符串扩展String.matchAll
将所有的匹配结果取出来，返回一个可迭代的对象，可用for...of循环遍历
#### 可选链操作符（?.）
```
function main(config) {
    // const host = config && config.db && config.db.host;
    const host = config?.db?.host;
}
const config = {
    db: {
        host: '127.0.0.1'
    }
}
```
#### 动态import
使用 import('/path') 动态导入模块，返回一个Promise对象
#### BigInt类型
```
const num1 = 123n;
const num2 = BigInt(123);
```
#### 绝对全局对象globalThis