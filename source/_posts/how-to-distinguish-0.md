---
title: js如何区分+0与-0
date: 2017-12-18 12:39:21
tags:
  - JS基础
---
---

#### 常见的场景
javascript中的 0 值判断有许多坑，比如当你判断一个对象中某个 key 是否有值，你可能会这样写：
```
if(obj[key]) {
  // do sth.
}
```
但如果这个key所对应的值是 0，那么你就被坑了，因为在 if 判断中，0 相当于 false，因此大括号中的语句并不会如预想那样执行。
#### 新的场景
而相似的，我在开发过程中遇到这样一个场景，一个数组中的元素是大于等于 0 的数字，在某种情况下会让元素的值乘以 -1，然后判断元素为负时，进行一系列操作，如下：
```
// 0 <= arr[i]
const v = arr[n] * -1;
if(v < 0) {
  // do sth.
}
```
此时，新的坑出现了，当元素值为 0 时，-0 < 0 的结果为 false，所以语句并不会如预期一样执行：
```
// 0 <= arr[i]
const v = arr[n] * -1; // 若arr[n]为0，则v的值为-0
if(v < 0) { // 判断结果为 false
  // do sth.
}
```
所以，当判断语句中涉及到 0 值时，一定要慎重。
#### 解决方案
那么如何判断一个数字的值为 -0 还是 +0 呢？

首先，0 就是 +0。

而判断 -0 的方案有如下几种：
###### 商值法
```
function isNegativeZero(num) {
  return num === 0 && (1 / num < 0); // 1与+0的商为Infinite，1与-0的商为-Infinite
}  
```
###### 对象键值比较法
```
// 比较tricky，不建议使用
function isNegativeZero(num) {
  if (num !== 0) return false;
  const obj = {};
  Object.defineProperty(obj, 'num', { value: -0, configurable: false }); // 将对象设置为，不可配置
  try {
    Object.defineProperty(obj, 'num', { value: num }); // 因为对象是不可配置的，所以若改变了对象num键对应的value，就会报错。利用这种特性，来判断参数num是否为-0
  } catch (e) {  
    return false  
  };  
  return true;  
}  
```
###### Object.is
```
// ES6的新特性
function isNegativeZero(num) {
  return Object.is(num, -0);
}
```
