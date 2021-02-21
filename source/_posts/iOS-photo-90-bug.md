---
title: iOS手机拍摄照片会旋转90度的bug原理与解决方案
date: 2017-03-01 17:10:32
tags:
  - 工程实践
  - 移动端
---
---

#### 背景
在网页应用中，**允许用户上传图片** 是一个非常常见的需求。

但是，在实际开发过程中，我们将用户上传的图片转换成了 base64 格式，然后发送给服务端，却会发现偶尔会出现问题，有一些图片本来是这样的：

![柴犬](dog.webp)

服务端收到时却变成了这样：

![柴犬2](dog2.webp)

经过测试发现，只有 **iOS手机竖着拍摄** 的照片才会出现这样的问题，而iOS手机横着拍摄的照片、Android手机拍摄的照片以及通过屏幕截图、网络下载等途径获得的图片，都不会产生这个问题。

#### 解决方案

由于项目上线时间比较紧迫，我直接在 Google 上进行“提问”，找到了一个在 github 上开源的项目 [lrz.js](https://github.com/think2011/localResizeIMG)。

这个工具的主要用途是保证图片质量的同时压缩图片体积，但神奇的是，只要按照文档上的说明将图片进行一次压缩，那么图片旋转90度的 bug 也被修复了！

#### 原理

那么它是如何解决这个问题的呢？

项目上线后，我阅读了 [lrz.js](https://github.com/think2011/localResizeIMG) 的源代码，发现它引入了一个名为 [exif.js](https://github.com/exif-js/exif-js) 的库，这个工具库能够读取图片的很多原始数据信息，包括：拍照方向、相机设备型号、拍摄时间、ISO 感光度、GPS 地理位置等。

**而拍照方向就是关键所在！**

[exif.js](https://github.com/exif-js/exif-js) 获取图像的拍照方向的代码如下：

```
EXIF.getData(IMG_FILE, function () { // IMG_FILE为图像数据
  var orientation = EXIF.getTag(this, "Orientation");
  console.log("Orientation:" + orientation); // 拍照方向
});
```

获取拍照方向的结果为 1-8 的数字：

![拍照方向信息](photo-direction.webp)

`注意：上表中加了*的方向并不常见，因为它们代表的是镜像方向，如果不做任何处理，手机在自然拍摄的情况下，是不会出现镜像情况的。`

这个表格代表什么意义？我们来看第一行，值为1时，右边两列的值分别为：
```
Row #0 is Top
Column #0 is Left side
```
其实很好理解，它表示照片的第一行位于顶端，而第一列位于左侧，那么这张照片自然就是以正常角度拍摄的。

而这8种结果，就是第一行与第一列所在的位置的8种组合。

那么，我们来测试一下 **iOS手机横着拍摄** 的照片，来看看它的 **拍照方向** 是什么呢？

![测试1](demo-img.webp)

结果是 1，即以正常角度拍摄的，其实也就是原图啦~

那么，我们再测试一下 **iOS手机竖着拍摄** 的照片，来看看它的 **拍照方向** 是什么呢？

![测试2](demo-img2.webp)

原来是 6！即第一行位于右侧，第一列位于顶端，其实相当于将照片顺时针旋转了90度！

所以，实际上iOS手机 **竖着** 拍摄的照片与 **横着** 拍摄的照片其本质上是一样的，只不过竖着拍摄的照片被添加了一个 **顺时针旋转90°** 的 **拍照方向**，所以显示的时候，就变成了上下边窄左右边宽的状态，其实也就是横着拍摄的照片 **顺时针旋转90°** 而成的~

那么明白了这些，文章开头所说的照片旋转bug的原因，也就很简单啦~

其实就是当我们在前端对图片进行 drawInRect 处理或者 base64 转换等操作之后，照片的 Orientaion 信息，即为拍照方向信息被删除了，所以 **iOS手机竖着拍摄** 的照片又回到了 **横着** 的状态，看起来也就是 **逆时针旋转了90°**！

而要想解决这个 bug 的思路也很简单：在处理图片之前，先读取并保存图片的 **拍照方向** 信息，然后在处理图片之后，再根据拍照方向，对图片进行相应的调整，[lrz.js](https://github.com/think2011/localResizeIMG) 中的代码如下：

```
switch (orientation) {
  case 3:
    ctx.rotate(180 * Math.PI / 180);
    ctx.drawImage(img, -resize.width, -resize.height, resize.width, resize.height);
    break;
  case 6:
    ctx.rotate(90 * Math.PI / 180);
    ctx.drawImage(img, 0, -resize.width, resize.height, resize.width);
    break;
  case 8:
    ctx.rotate(270 * Math.PI / 180);
    ctx.drawImage(img, -resize.height, 0, resize.height, resize.width);
    break;
  case 2:
    ctx.translate(resize.width, 0);
    ctx.scale(-1, 1);
    ctx.drawImage(img, 0, 0, resize.width, resize.height);
    break;
  case 4:
    ctx.translate(resize.width, 0);
    ctx.scale(-1, 1);
    ctx.rotate(180 * Math.PI / 180);
    ctx.drawImage(img, -resize.width, -resize.height, resize.width, resize.height);
    break;
  case 5:
    ctx.translate(resize.width, 0);
    ctx.scale(-1, 1);
    ctx.rotate(90 * Math.PI / 180);
    ctx.drawImage(img, 0, -resize.width, resize.height, resize.width);
    break;
  case 7:
    ctx.translate(resize.width, 0);
    ctx.scale(-1, 1);
    ctx.rotate(270 * Math.PI / 180);
    ctx.drawImage(img, -resize.height, 0, resize.height, resize.width);
    break;
  default:
    ctx.drawImage(img, 0, 0, resize.width,resize.height);
}
```

其中，translate 是平移变换，scale(-1,1) 是向左翻转，rotate 是顺时针旋转。

举例说明 **case 2**，当图片的拍照方向为 2 时，即第一行位于顶端，而第一列位于右侧，其实相当于把照片进行了左右的翻转。所以，这里对图片的操作是，先向右平移等于图片宽度的距离，再向左翻转，这相当于以图片水平方向的对称轴为轴进行了左右翻转，然后再以 (0,0) 为起始点绘制原宽高的图片，即完成了对拍照方向的纠正。

#### 最后
经过一系列的测试，发现确实只有iOS手机拍摄的照片是通过 **拍照方向** 来区别横竖的，Android手机无论竖拍还是横拍的照片，**拍照方向** 都为 1，也就是说即使丢失了 **拍照方向** 这一信息，也不会影响到图片的呈现。

而屏幕截图、网络上的图片、通过PS制作的图片等就更没有 **拍照方向** 这一信息了。