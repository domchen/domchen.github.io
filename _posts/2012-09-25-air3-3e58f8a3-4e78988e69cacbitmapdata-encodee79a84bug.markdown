---
author: dom
comments: true
date: 2012-09-25 03:45:44+00:00
layout: post
slug: air3-3%e5%8f%8a3-4%e7%89%88%e6%9c%acbitmapdata-encode%e7%9a%84bug
title: AIR3.3及3.4版本BitmapData.encode()的Bug
wordpress_id: 365
categories:
- ActionScript
- AIR
- Flash
- FLEX
tags:
- AIR3.3
- AIR3.4
- BitmapData.encode()
- 色差
---

最近做位图转换的时候，发现一个矢量素材的绘制结果总是出现色差。如下图。

[![AIR位图编码BUG](http://blog.domlib.com/wp-content/uploads/2012/09/air_bug.jpg)](http://blog.domlib.com/wp-content/uploads/2012/09/air_bug.jpg)



开始以为是应用colorTransform过程出了bug。测试了之后发现，即使没有colorTransform也会出现色差。排除了所有可能后，发现是使用了AIR3.3开始提供的位图编码接口：BitmapData.encode()造成的。而使用Flex4.6自带的mx.graphics.codec.PNGEncoder或者mx.graphics.codec.JPEGEncoder编码的结果都是正常的。好不容易发现这个问题，就全面测试了下。结果如下：

AIR3.3版本：使用BitmapData.encode()编码PNG和JEPG都会出现色差。JEPG-XR正常。

AIR3.4版本：使用BitmapData.encode()编码只有JEPG会出现色差。<del>PNG</del>和JEPG-XR正常。

而Flex自带的mx.graphics.codec.PNGEncoder和mx.graphics.codec.JPEGEncoder始终是正常的。但是这两个编码器是纯AS实现的，编码效率很低。

希望以上可以给遇到同样问题的同学一个参考。:)

ps:[测试fla源文件下载](http://blog.domlib.com/wp-content/uploads/2012/09/Air_bug_fla.rar)

**[更新]后来在AIR3.4下测试另一个图像的PNG编码也出了问题，透明通道完全丢失。说明AIR3.4的Png编码还是有问题的。跟JPEG一样，都是碰到特殊的图像时才会体现出来。建议不要使用AIR里的PNG和Jpeg编码接口了。用mx.graphics.codec.PNGEncoder和mx.graphics.codec.JPEGEncoder替换吧。虽然慢点，起码编码结果是正确的。**
