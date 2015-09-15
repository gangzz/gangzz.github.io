---
layout: post
title: "Event Page"
date: 2015-07-19 17:48:54 +0800
comments: true
categories: 
---

app和扩展一个常见的需求是长时间保存任务和状态。Event Page提供了解决方法。Event Page在需要是被加载，在不必要时卸载和释放空间。

从长期稳定版本Chrome 22开始支持，并且带来的显著的性能提升，特别是低配机器上。

## Manifest

	{
	  "name": "My extension",
	  ...
	  "background": {
	    "scripts": ["eventPage.js"],
	    "persistent": false
	  },
	  ...
	}

通过persistent声明background page是否是Event Page类型。如果不包含persistent参数，则为常规类型。

## LifeTime

Event Page会在它们需要时加载，当他们变为空闲时释放。发生以下事件时将导致Event Page加载：

* 首次安装或者更新到新版本。
* Event Page监听一个事件，并且该事件已经发出。
* content script或其他扩展发送了消息。
* 扩展插件的某个视图调用了getBackgroundPage()方法。

一旦Event Page被加载它会持续整个活跃期（比如调用一个扩展函数或者处理网络请求）。除此以为只有当其扩展的**所有**视图关闭并且所有消息接口也关闭的情况下才会卸载Event Page。注意启动一个View不会触发Event Page的加载，只会延迟Event Page的卸载。

当所有视图都关闭后Event Page变为空闲，在转为卸载前会发送runtime.onSuspend事件。Event Page会存活几秒保证在卸载前处理onSuspend事件，如果该事件处理过程中扩展开启了新的视图或是请求导致Event Page不能被卸载，则发出runtime.onSuspendCancled事件。


## Event registration

Chrome会跟踪所有已注册的事件，当它分发事件时响应的Event Page就会加载。同样当通过removeListener方法移除了所有的事件监听时，Event Page也不再会被加载。

Event Listener只存在于Event Page的上下文中，因此每次Event Page加载时都要调用注册，仅仅在onInstalled方法中调用是不够的。

## Convert background page to event page

1. 添加persistent:false参数到background中。
2. 如果存在调用JS的window.setTimeout()或window.setInterval()注册函数时，需要改为Chrome.* API的alarms API。
3. 同理其它HTML5中异步API， notifications和geolocation可能在Page卸载时还没有完成。需要替换成对等的notifications。
4. 如果使用了extension.getBackgroundPage()方法需要切换成runtime.getBackgroundPage()方法。新的方法是异步的，所以在返回之前它可以启动Event Page。

## Best practices when using event pages

1. 应该在每次页面加载时注册你感兴趣的事件。注册的逻辑应该放在Event Page的顶级作用域。
2. 如果你希望在扩展插件安装或升级时做一些初始化，应该放在onInstalled中。这个位置适合做一次性事件。
3. 如果你希望保存运行时状态，应该使用storage API或者是IndexDB，不能依赖Event Page。
4. 使用事件过滤器来明确过滤你需要的事件。
5. 监听onSuspend事件来做必要的退出清理，更建议定期保存的方式来保存数据以避免不能预知的崩溃。
6. 如果使用message，必须在不需要后关闭message port以便Event Page可以卸载。
7. If you're using the context menus API, pass a string id parameter to contextMenus.create, and use the contextMenus.onClicked callback instead of an onclick parameter to contextMenus.create.
8. Remember to test that your event page works properly when it is unloaded and then reloaded, which only happens after several seconds of inactivity. Common mistakes include doing unnecessary work at page load time (when it should only be done when the extension is installed); setting an alarm at page load time (which resets any previous alarm); or not adding event listeners at page load time
