---
layout: post
title: "Content Scripts"
date: 2015-07-19 21:03:56 +0800
comments: true
categories: 
---

Content Scripts可以读取用户Page的所有Dom细节，或是修改它们。

Content Scripts面临的限制：

* 不可以访问extension中定义的函数。
* 不可调用Chrome.* API。
* 不可以调用用户页面定义的函数。

然而Content Scripts可以调用一下Chrome.* API：

* extension ( getURL , inIncognitoContext , lastError , onRequest , sendRequest )
* i18n
* runtime ( connect , getManifest , getURL , id , onConnect ,  onMessage , sendMessage )
* storage

除此以外content Script可以使用message和extension通信，可以调用cross-origin XMLHttpRequest。

## Manifest

如果你的content scripts在任何时候都需要注入，可以选择content script:

	{
	  "name": "My extension",
	  ...
	  "content_scripts": [
	    {
	      "matches": ["http://www.google.com/*"],
	      "css": ["mystyles.css"],
	      "js": ["jquery.js", "myscript.js"]
	    }
	  ],
	  ...
	}

如果你的content scripts只注入特定的URL，可以选择permission：

	{
	  "name": "My extension",
	  ...
	  "permissions": [
	    "tabs", "http://www.google.com/*"
	  ],
	  ...
	}

