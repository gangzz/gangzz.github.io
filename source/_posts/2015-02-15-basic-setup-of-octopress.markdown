---
layout: post
title: "Create The First Post"
date: 2015-02-15 23:25:06 +0800
comments: true
categories: [Ruby] 
published: true
---

#安装
##前置依赖
1.	git
1.	Ruby 1.9.3及以上版本，可以通过rbenv或RVM搞定。
1.	安装[ExecJS](https://github.com/sstephenson/execjs)中支持的任意JavaScript运行时环境。

##设置Octopress
源文件下载

	git clone git://github.com/imathis/octopress.git octopress
	cd octopress

解决依赖关系
	
	gem install bundler
	rbenv rehash    # If you use rbenv, rehash to be able to run the bundle command
	bundle install
	
安装默认主题的Octopress
	
	rake install
	
##设置部署
这里选择的Github Pages，从略。

#创建Blog
使用命令创建一篇Blog，blog文件位于source/_posts/目录下：

	rake new_post[your blog name]
	
使用任意编辑器编辑blog，完成保存执行命令：

	rake generate

#本地测试
使用命令启动本地服务器，端口4000.
	
	rake preview

#部署

	rake deploy