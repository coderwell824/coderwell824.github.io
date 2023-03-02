---
title: 手写发布订阅模式
category: Javascript
cover: http://lc-llaarrzl.cn-n1.lcfile.com/AShhJ9SY7WhHOyrXTanHdwRqVhI24pTj/emit.png
date: 2023-02-24
---
# 发布订阅模式的定义

首先说明一下什么是发布-订阅模式都由统一的一个调度中心来处理, 那也就是说这个模式是由三部分组成的

发布者: 将消息事件发布到调度中心

订阅者: 把自己想关注的消息事件, 注册到调度中心

调度中心: 处理事件注册与发布

<img src="http://lc-llAarrZL.cn-n1.lcfile.com/qptpsyOs9dm886K8Jf8XzCgTPWG5zwaj/image-20230301175521145.png" alt=" " style="zoom:50%;" />

## 发布订阅模式的思路

定义一个事件变量对象来存储消息事件, 因为订阅者不止一个

定义一个添加方法, 将事件添加到事件变量对象中

定义一个删除方法, 将事件在事件变量对象中删除

定义一个派发方法, 调用事件变量对象中的事件

```javascript
class EventSubscription{

  constructor(){
    //定义事件容器, 用来存储事件数组
    this.eventObject={}
  }

  //事件添加方法
  addEventListener(){

  }

  //派发方法
  dispatchEvent(){

  }

  //事件移除方法
  removeEventListener(){

  }
}
```

## addEventListener事件添加方法

添加消息事件是需要一个事件名以及消息事件的事件回调函数

需要将消息事件添加到事件变量对象中, 并且不止一个变量

添加事件变量时需要判断是否存在当前事件, 做不同的处理

```javascript
  /**
   * @description:事件添加方法
   * @param {*} eventName 事件名
   * @param {*} callback 事件回调函数
   */
  addEventListener (eventName, callback) {

    //判断事件变量对象中是否存在当前事件, 如果没有的话初始化一个空数组
    if (!this.eventObject[eventName]) {
      this.eventObject[eventName] = []
    } else {
      //如果存在, 那就说明该事件有多个订阅者, 继续往后push
      this.eventObject[eventName].push(callback)
    }
  }
```

## removeEventListener事件移除方法

删除消息事件是需要一个事件名的, 以及消息事件的事件回调函数

在事件变量对象中, 对应的事件回调函数不止一个, 需要对符合条件的callback进行删除

如果事件回调函数不存在的话, 直接删除事件

```javascript
  /**
   * @description: 移除事件方法
   * @param {*} eventName 事件名
   * @param {*} callback 事件回调函数
   */
  removeEventListener (eventName, callback) {

    //判断该事件在事件变量对象中是否存在
    if (!this.eventObject[eventName]) {
      return new Error("事件无效")
    }

    //事件回调函数不存在直接移除该事件的订阅和发布
    if (!callback) {
      delete this.eventObject[eventName]
    } else {

      //移除事件
      const index = this.eventObject[eventName].findIndex(item => item === callback);
      if (index === -1) return new Error("无该绑定事件")
      this.eventObject[eventName].splice(index, 1)

      //无事件回调函数移除该事件的订阅和发布
      if (!this.eventObject[eventName].length) {
        delete this.eventObject[eventName];
      }
    }
  }
```

## diapatchEvent事件派发方法

事件派发方法只需要方法名, 确定是哪个事件, 也可以传递参数

将事件变量对象中对应的事件回调函数依次执行

```javascript
  /**
   * @description: 派发方法
   * @param {*} eventName 事件名
   * @param {*} params 参数
   */
  dispatchEvent (eventName, params) {

    //若没有注册该事件则抛出错误
    if (!this.eventObject[eventName]) {
      return new Error("该事件未注册")
    }

    //触发事件
    this.eventObject[eventName].forEach((callback) => {
      callback(...params)
    })
  }
```

## 完整代码

```javascript
class EventSubscription {

  constructor() {
    //定义事件容器, 用来存储事件数组
    this.eventObject = {}
  }

  /**
   * @description:事件添加方法
   * @param {*} eventName 事件名
   * @param {*} callback 事件回调函数
   */
  addEventListener (eventName, callback) {

    //判断事件变量对象中是否存在当前事件, 如果没有的话初始化一个空数组
    if (!this.eventObject[eventName]) {
      this.eventObject[eventName] = []
    } else {
      //如果存在, 那就说明该事件有多个订阅者, 继续往后push
      this.eventObject[eventName].push(callback)
    }
  }

  /**
   * @description: 移除事件方法
   * @param {*} eventName 事件名
   * @param {*} callback 事件回调函数
   */
  removeEventListener (eventName, callback) {

    //判断该事件在事件变量对象中是否存在
    if (!this.eventObject[eventName]) {
      return new Error("事件无效")
    }

    //事件回调函数不存在直接移除该事件的订阅和发布
    if (!callback) {
      delete this.eventObject[eventName]
    } else {

      //移除事件
      const index = this.eventObject[eventName].findIndex(item => item === callback);
      if (index === -1) return new Error("无该绑定事件")
      this.eventObject[eventName].splice(index, 1)

      //无事件回调函数移除该事件的订阅和发布
      if (!this.eventObject[eventName].length) {
        delete this.eventObject[eventName];
      }
    }
  }

  /**
   * @description: 派发方法
   * @param {*} eventName 事件名
   * @param {*} params 参数
   */
  dispatchEvent (eventName, params) {

    //若没有注册该事件则抛出错误
    if (!this.eventObject[eventName]) {
      return new Error("该事件未注册")
    }

    //触发事件
    this.eventObject[eventName].forEach((callback) => {
      callback(...params)
    })
  }
}
```

## 测试代码

```javascript
const eventSubscription = new EventSubscription();

function test1 (params) {
  console.log("测试test1:", params);
}
function test2 (params) {
  console.log("测试test2:", params);
}

// 添加订阅内容
eventSubscription.addEventListener('test', test1);
eventSubscription.addEventListener('test', test2);
console.dir(eventSubscription)
// 输出结果
// EventSubscription {
//     eventObject: { test: [ [Function: test1], [Function: test2] ] }
// }

// 派发订阅事件
eventSubscription.dispatchEvent('test', 'tests事件触发')
// 输出结果
// 测试test1: tests事件触发
// 测试test2: tests事件触发

// 删除订阅事件
eventSubscription.removeEventListener('test', test2)
console.log(eventSubscription)
// 输出结果: EventSubscription { eventObject: { test: [ [Function: test1] ] } }

eventSubscription.dispatchEvent('test')
console.log(eventSubscription)
// 输出结果: EventSubscription { eventObject: {} }

```

