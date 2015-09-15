---
layout: post
title: "chrome extension overview"
date: 2015-07-19 16:36:19 +0800
comments: true
categories: ["chrome"]
---


## The basics

插件的本质是一组有html、css、js和图片组成的bundle。

插件可以通过content script和cross-orgin XMLHttpRequest和当前页面互动，并且可以调用浏览器赋予页面的特性：书签、页签等。

### 插件UI

插件UI分为browser action和page action两种，browser action会时时出现而page action只在匹配到的页面时才出现。如果你的插件是对所有网站通用的就选择browser action，否则应该选择page action。

除此以外插件还有其它途径展现自己的界面比如增加菜单项、提供一个配置页面、利用content script修改当前页面。

### Files

插件包含的文件:

* manifest.json json格式的元数据文件。
* html文件。
* js文件（可选）。
* 其它你可能用到的文件，比如图片。

#### 文件引用

文件的访问：一般情况下一个扩展插件中的文件可以通过相对路径访问。你也可以使用chrome-extensions/extensionID/file的方式访问。

在你的扩展插件打包并且上传到应用市场后，它将获得永久ID。其它情况下（开发）每当其更改目录或者重新打包都会导致变更，这时可以使用预置变量@@extension_id来解决。


### manifest.json

manifest.json提供了扩展插件的重要信息以及它需要用到的能力。

	 {
	  "name": "My Extension",
	  "version": "2.1",
	  "description": "Gets information from Google.",
	  "icons": { "128": "icon_128.png" },
	  "background": {
	    "persistent": false,
	    "scripts": ["bg.js"]
	  },
	  "permissions": ["http://*.google.com/", "https://*.google.com/"],
	  "browser_action": {
	    "default_title": "",
	    "default_icon": "icon_19.png",
	    "default_popup": "popup.html"
	  }
	}
	
## 架构

一个扩展插件有一个不可见的background页面，用于保存其逻辑。除此还可能存在其它html页面作为UI。如果一个页面要和用户加载的页面交互就必须使用content script。

### The background page

无论browser action还是page action都有一个background page。background page分为 持久background page和事件 background page。除非你希望自己的插件要时刻允许，否则推荐使用事件background page。

### UI Pages

每个扩展插件都可以有自己的常规HTML界面。比如popup.html用于点击插件是弹出，options.html作为配置页面。另一个类型的页面是override页面。

一个扩展插件内部的页面是可以互相操作dom和调用函数的。因此同样的函数没有必要在插件中重复出现，它们都可以通过调用background page的函数完成。

### Content Script

如果要和用户页面交互就必须使用content script。content script是在用户加载页面的作用域内执行的脚本。

content script可以轻易的读取和修改用户页面的DOM，但是它不能修改其所在插件background page中的dom。

content script与扩展插件并非完全隔离，它们之间可以通过消息通信。

## Useing Chrome.* APIS

扩展插件还可以调用浏览器专有方法，实现和浏览器的紧密集成。比如如果想打开一个窗口你可以调用window.open()方法，但是如果想控制url应该展现在哪个窗口时，就要用到chrome的tabs.create方法。

### ansynchronous vs synchronous

大部分Chrome.* API都是异步的，这意味着它们的调用会立即返回。如果希望关心方法调用的输出可以使用callback。异步方法会在执行后调用callback方法。

	chrome.tabs.create(object createProperties, function callback)
	
其它同步的Chrome.* API则不具备callback参数，它们的调用会持续到结果返回。

	string chrome.runtime.getURL()
	
## Communication between pages

同一个扩展插件中的页面是可以互相调用的，这是因为它们都在同一个进程的同一个线程范围内执行的。

可使用chrome.extension查找同一个扩展插件中的页面，使用方法getViews()和getBackgroundPage()。一旦某个页面获得了另一个页面的引用，它就可以调用其函数，修改它的DOM。

## Saving Data And Incognito Mode

扩展插件可以使用storage API、 HTML5 web storage API或者通过云端通信存储数据。当保持数据时应该先考虑是否是在隐身模式下，默认情况下插件不应该在隐身环境中执行。

隐身模式保证浏览器不保留任何用户的踪迹，因此需要考虑隐身模式下用户会希望一款插件做什么、不做什么。比如记录浏览历史的动作不应该执行，但是插件配置如果在隐身模式下被修改也应该在之后起作用。

可以通过chrome.tabs或是window的API判断当前是否是隐身模式。

	function saveTabData(tab, data) {
	  if (tab.incognito) {
	    chrome.runtime.getBackgroundPage(function(bgPage) {
	      bgPage[tab.url] = data;      // Persist data ONLY in memory
	    });
	  } else {
	    localStorage[tab.url] = data;  // OK to store data
	  }
	}