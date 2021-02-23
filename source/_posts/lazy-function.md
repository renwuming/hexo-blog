---
title: 惰性函数
date: 2017-05-09 18:10:10
tags:
  - 性能优化
---
---

#### 定义与实现
**惰性函数表示，此函数有很多个分支判断，但这些分支判断只会在第一次调用时执行，执行后会修改此函数，再次调用时无须判断。**

这种函数的应用场景之一是在兼容性判断时，例如：
```
// 兼容的事件监听函数，非惰性
function addEvent(event, el, handler) {
  if(el.addEventListener) {
    el.addEventListener(event, handler);
  } else if (el.attachEvent) {
    el.attachEvent("on"+event, handler);
  } else {
    el["on"+event] = handler;
  }
}
// 兼容的事件监听函数，惰性
function addEvent(event, el, handler) {
  if(el.addEventListener) {
    addEvent = function(event, el, handler) {
      el.addEventListener(event, handler);
    }
  } else if (el.attachEvent) {
    addEvent = function(event, el, handler) {
      el.attachEvent("on"+event, handler);
    }
  } else {
    addEvent = function(event, el, handler) {
      el["on"+event] = handler;
    }
  }
  return addEvent(event, el, handler);
}
```
还有在单例模式中的例子：
```
function singleton() {
  // 单例
  const instance = {
    name: "singleton"
  };
  // 重写构造函数，用闭包实现单例
  singleton = function() {
    return instance;
  }
  return instance;
}
```
#### 惰性函数的特点
1. 应用频繁，如果只调用一次，则失去了意义
2. 复杂的分支判断，若无分支判断，也失去了意义
3. 环境固定，函数中的判断依据不会变化，所以只需要首次调用时判断即可


