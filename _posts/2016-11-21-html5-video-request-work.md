---
layout: post
title: HTML5 Video标签是如何处理视频流的网络请求的
date: 2016-11-16 16:00:24.000000000 +08:00
---

[访问原文地址](http://njfeng.com/2016/11/html5-video-request-work/)

## 问题
如果你们的网页中用了一个HTML5的video标签，这就意味这你开启了与服务端上视频数据之间的请求。当你尝试调整进度条到那些还没有缓冲好的视频部分时，你会发现浏览器的网络流量中出现了一写视频流的数据请求。但是如果视频文件都是由一些可变二进制文件组成，HTML5是如何计算出时间点的信息与对应的视频流数据的网络请求之间的关系呢？

## 解答

首先，很遗憾的是，这个HTML5没什么关系了。这是视频流自身hold住的。

就拿最常见的视频格式MP4来说，mp4文件格式包含着不同层级关系的'数据块'组成。其中一个数据块叫做`Time-To-Sample(stts)`，这个数据块用压缩的包含了每一帧的时间值。从这里，你就通过`Sample-to-Chunk(stsc) atom`可以找到对应的视频帧的chunk。最后，通过`Chunk offset atom(stco)`可以把对应的字节位移传到视频文件里。

Video标签里的视频的总时长是存储在`Movie Header atom(mvhd)`中。当我们更改了进度条的事件句柄，浏览器将根据视频的总时长和移动进度句柄的位置估算出一个时间值，再结合预先下载好的视频头部信息得出文件位置信息，最后在根据位置信息发出网络请求。

浏览器都有自己的media管理模块，这样可以让APP层通过简单的调用seekToTime()来决定发送什么样的HTTP请求来获取指定的视频流文件。

## keywords

-	ISC/IEC 14496-12
- QuickTime File Format documentation