---
layout: post
title:【译+源码分析】Electron内部：整合Message Loop
date: 2018-03-13 18:11:24.000000000 +08:00
---

注：这是一篇来自Electron官方发布的博文，作者是Electron创始人zcbenz。[原文](https://electronjs.org/blog/electron-internals-node-integration)

这是解释Electron内部机制系列专栏的第一篇文章，我在这篇文章中主要介绍一下Electron中Node的event loop是如何整合到Chromium的。

已经有很多尝试使用Node进行GUI编程框架，例如用于GTK +绑定的node-gui和用于QT绑定的node-qt。 但是他们都没有在生产环境中工作，因为GUI开发套件有自己的消息循环，而Node使用libuv作为自己的事件循环，并且主线程只能同时运行一个循环。 因此，在Node中运行GUI消息循环的常用技巧是以非常小的时间间隔用interval把GUI的消息注入到Node消息循环中，但这会使GUI界面响应变慢并占用大量CPU资源。

在Electron的开发过程中，我们遇到了同样的问题，但方式相反：我们必须将Node的事件循环集成到Chromium的消息循环中。

## Main进程和渲染进程

在我们深入了解消息循环集成的细节之前，我将首先解释Chromium的多进程体系结构。

在Electron中，有两种类型的进程：主进程和渲染进程（实际上这非常简化，完整的视图请参阅多进程体系结构）。 主流程负责像创建窗口一样的GUI工作，而渲染器进程仅处理运行和呈现网页。

Electron允许使用JavaScript来控制Main进程和渲染进程，这意味着我们必须将Node集成到两个进程中。

## 用libuv来体会Chromium的message loop

我的第一次尝试是用libuv重新实现Chromium的消息循环。

在渲染器进程很容易做到，因为它的消息循环只监听文件描述符和定时器，而我只需要用libuv实现接口。

然而，主进程中显然更加困难。 每个平台都有自己的GUI消息循环。 macOS Chromium使用NSRunLoop，而Linux使用glib。 我尝试了很多窍门从本地GUI消息循环中提取底层文件描述符，然后将它们提供给libuv进行迭代，但仍遇到无法工作的边界情况。

所以最后我添加了一个定时器来以小的间隔轮询GUI消息循环。 结果是该进程占用大量CPU使用率，并且某些操作有很长时间的延迟。

## 轮询Node的事件循环在一个单独的线程中

随着libuv的成熟，开始有可能使用另一种方法。

backend fd的概念被引入到libuv中，可以理解为libuv轮询其事件循环的文件描述符（或句柄）。 因此，通过轮询backend fd，可以在libuv中发生新事件时收到通知。

所以在Electron中，我创建了一个单独的线程来轮询backend fd，因为我是使用系统调用来轮询而不是libuv API，所以它是线程安全的。 每当libuv事件循环中发生新事件时，都会将消息发布到Chromium的消息循环中，然后在主线程中处理libuv事件。

通过这种方式，我不用修改Chromium和Node源码，并且在主进程和渲染进程中都使用了相同的代码。

## 代码

您可以在electron/atom/common/下的node_bindings文件中找到消息循环集成的实现。 它可以很容易地用于想要集成Node的项目。


## 译者注

Electron`./atom/common/node_bindings.cc`中的`CreateEnvironment()`是Node环境的导入入口函数，分别在3个地方`./atom/renderer/atom_renderer_client.cc`、`./atom/browser/atom_browser_main_parts.cc`、`./atom/renderer/web_worker_observer.cc`，除了主进程和渲染进程，还有webwork环境，这是由于worker也是独立的js环境，并且缺少web环境的一些api,需要用message或事件与主进程进行数据交互。

画了一下node_bingding的流程图：

![img](../assets/18/node_binding.png)

PrepareMessageLoop()就是本篇文章提到的整合方法。
````C++
  // Add dummy handle for libuv, otherwise libuv would quit when there is
  // nothing to do.
  uv_async_init(uv_loop_, &dummy_uv_handle_, nullptr);

  // Start worker that will interrupt main loop when having uv events.
  uv_sem_init(&embed_sem_, 0);
  uv_thread_create(&embed_thread_, EmbedThreadRunner, this);
````

`EmbedThreadRunner()`里面实现了监听各平台的`PollEvents`，如果有新的事件，会调主进程的`TaskRunner->PostTask()`把消息同步进去，同时执行一下`uv_run(uv_loop_, UV_RUN_NOWAIT)`。以达到事件同步的目的

````C++
while (true) {
    // Wait for the main loop to deal with events.
    uv_sem_wait(&self->embed_sem_);
    if (self->embed_closed_)
      break;

    // Wait for something to happen in uv loop.
    // Note that the PollEvents() is implemented by derived classes, so when
    // this class is being destructed the PollEvents() would not be available
    // anymore. Because of it we must make sure we only invoke PollEvents()
    // when this class is alive.
    self->PollEvents();
    if (self->embed_closed_)
      break;

    // Deal with event in main thread.
    self->WakeupMainThread();
  }

void NodeBindings::WakeupMainThread() {
  DCHECK(task_runner_);
  task_runner_->PostTask(FROM_HERE, base::Bind(&NodeBindings::UvRunOnce,
                                               weak_factory_.GetWeakPtr()));
}

````

Electron的实现中用到了`Node`,`v8`,`chromium`的接口，需要一定的源码分析储备，慢慢来吧。

