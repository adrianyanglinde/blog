---
layout: post
title: "webuploader"
subtitle: "webuploader使用总结"
date: 2023-07-30
author: "Adrian"
header-img: "img/post-bg-js-module.jpg"
tags:
    - 前端开发
    - JavaScript
---

为了满足客服日常上传大文件的需求，基于 Webuploader 开发了大文件上传组件，目前已集成到@customer/ProAdFileUpload 中；在此，谈谈初次使用 Webuploader 的感想以及踩到的坑。

## Webuploader 介绍

Webuploader 是一个文件上传组件，其天然内置了将大文件分片，并发上传等功能，开发者无需关心文件是如何切割上传。

### 内部窥探

Webuploader 内部包含了如下几个功能类：

-   Uploader 核心，负责具体上传逻辑
-   Base 基础类，提供一些常用帮组类方法
-   Mediator 中介者，处理事件相关
-   Queue 队列
-   File 文件
-   Widgets 将一些文件，队列等功能应用在 Uploader 上
    ...

可见，它的核心应该还是一个事件驱动模型。同时，它还引用了一些诸如 dnd，filepaste 等第三库来帮组实现拖拽，粘贴的效果。

## Webuploader 使用

### widgets

Webuploader 对外将功能模块拆分成好几个 widget，你可以通过 command 机制来通信

需要注意的是在实例化前通常需要在 Webuploader 注册一个 widget，告诉 Webuploader 需要该 widget 响应哪些命令以及添加处理这些命令的 handle。

如下是注册一个 widget 来处理 queue 相关

```js
WebUploader.Uploader.register(
	{
		// 整个文件上传前
		"before-send-file": "beforeSendFile",
		// 每个分片上传前
		"before-send": "beforeSend",
		// 分片上传完毕
		"after-send-file": "afterSendFile",
		// 标识
		name: "widgetName"
	},
	{
		beforeSendFile,
		beforeSend,
		afterSendFile
	}
);
```

在响应 before-send-file 的时候还没有对文件做分片，因此可以在 beforeSendFile 里做整个文件的 md5 验证

在响应 after-send-file 的时候表面所有分片已经上传完成了，通常在这个时候需要再做一次文件的完整型验证

上面的例子，从单个文件角度来看，你可以看成是文件的一个生命周期（加入队列->上传前->分片前->上传完毕）

### 实例

提供了一些配置项生成 uploader 实例

```js
uploader.current = WebUploader.create(comfig);
```

### 事件

```js
uploader.current.on('beforeFileQueued', (file:File) => {
  file.on('statuschange', (status) => {
    ...
  })
});
...
uploader.current.on('uploadComplete', (file:File) => {
  file.off('statuschange');
});
```

### UI

WebUploader 只包含文件上传的底层实现，不包括 UI 部分，所以这部分就交由 react 来处理。

```js
const statusChanged = (file:File): void => {
  ...
  onChange({ fileList: _fileList });
};
```

## 踩坑记录

### widgets 异步过程控制

处理 widgets 的 handle 如果是异步，返回 promise 会有问题，需要返回其内置的 deffer

```js
const beforeSendFile = (file:File):Promise<void> => {
  const deferred = Base.Deferred();
  ...
  return deferred.promise();
};
```

### 停止并销毁上传

记得在 effect 里去停止上传和销毁 widgets，否则当渲染关闭的时候，WebUploader 还在运作上传功能

```js
useEffect(() => {
  ...
  return () => {
    uploader.current.stop(true);
    uploader.current = null;
    WebUploader.Uploader.unRegister('widgetName');
  };
}, [uploader.current]);
```
