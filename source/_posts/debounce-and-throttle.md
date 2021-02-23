---
title: 函数防抖与函数节流
date: 2017-05-08 15:46:09
tags:
  - 性能优化
---
---

Web 开发中经常遇到这样的场景，监听一个事件，然后进行相应的 DOM 操作。

但是，当所监听的事件触发频率非常高的时候，会因为 DOM 的频繁操作导致浏览器卡顿甚至卡死，类似的事件如下：
1. window 对象的 resize、scroll
2. 拖拽时的 mouseover
3. 射击游戏的 mousedown、keydown
4. 文字输入自动补全时的 keyup 事件

以上的情况解决方案又分为两类，第一类就是 window 的 resize 事件，第二类是其他所有情况。两种解决方案分别是Debounce 和 Throttle。

#### 函数防抖(Debounce)
当你用手按住弹簧的时候，它不会弹起直到你松开手。
也就是说，函数会在事件触发 N 毫秒后被调用，但前提是你已经“松开了手”；如果在 N 毫秒内事件再次被触发，会清空之前的计时器重新计时。
```
function debounce(idle, action) {
  const args = [].slice.call(arguments, 2);
  let timer;
  return function() {
    const ctx = this;
    const total_args = args.concat([].slice.call(arguments)); // 参数柯里化
    clearTimeout(timer); // 事件被触发就清空计时器
    timer = setTimeout(function() {
      action.apply(ctx, total_args);
    }, idle);
  }
}

// 示例
function action() {
  // do some action
}
const doAction = debounce(100, action, arg0, arg1);
window.onresize = function() {
  doAction(arg2, arg3); // 此处参数可以分批次传入
}
```

#### 函数节流(Throttle)
平时，水龙头的水是不间断的流出的，你可以将水龙头拧紧，实现水一滴一滴的流出。
也就是设定一个周期，函数触发的周期不会小于这个周期。
```
function throttle1(idle, action) {
  const args = [].slice.call(arguments, 2);
  let flag = true;
  let timer;
  return function() {
    const ctx = this;
    const total_args = args.concat([].slice.call(arguments));
    if(flag) { // flag为true时才开始计时器
      flag = false; // 开始计时器后，flag置为false，直到动作执行完
      timer = setTimeout(function() {
        action.apply(ctx, total_args);
        flag = true; // 进行一次动作后，将flag置为true，再进行下一次动作
      }, idle);
    }
  }
}

function throttle2(idle, action) {
  const args = [].slice.call(arguments, 2);
  let last = 0;
  return function() {
    const ctx = this;
    const total_args = args.concat([].slice.call(arguments));
    const cur = +new Date();
    if(cur - last > idle) { // 周期大于idle时，才会进行动作
      action.apply(ctx, total_args);
      last = cur; // 重新计算周期
    }
  }
}

// 示例
function action() {
  // do some action
}
const doAction = throttle1(100, action, arg0, arg1);
window.onscroll = function() {
  doAction(arg2, arg3); // 此处参数可以分批次传入
}
```