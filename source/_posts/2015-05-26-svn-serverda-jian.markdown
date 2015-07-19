---
layout: post
title: "svn server搭建"
date: 2015-05-26 20:33:04 +0800
comments: true
categories: ["others"]
---

参考:https://www.rosehosting.com/blog/install-and-configure-svn-webdav-server-on-a-centos-6-vps/

# 介绍
本文目标为实现http方式远程svn repository访问，实现借助于apache、webDAV、subversion，操作环境为centos。
#	安装subversion
使用管理员账号执行。

	yum install subversion
	
完成后创建svn repository，由于使用了apache，选择执行以下命令：

	svnadmin create /var/www/svn/repo
#	安装webdav
webdav（Web Distributed Authoring and Versioning ）是超文本协议的一个扩展，允许远程客户端执行web内容鉴权。DAV提供了一系列文件系统操作命令，各平台存在多种webDav的实现，其中包括svn。

执行以下命令安装

	yum install mod_dav_svn
#	配置Apache
完成后在/etc/httpd/conf.d中新建针对svn的配置subversion.conf，这么做是因为httpd默认的配置/etc/httpd/conf/httpd.conf中包含inclue conf.d/*.conf的引用。

subversion的核心内容
 >LoadModule dav\_svn\_module     modules/mod\_dav\_svn.so
 
 >LoadModule authz\_svn\_module   modules/mod\_authz\_svn.so
 >
 > \<Location /repos\>
 
 > DAV svn
 
 > SVNParentPath /var/www/svn
 
 > AuthType Basic
 
 > AuthName "some Realm"
 
 > AuthUserFile /etc/svn_htpasswd
 
 > AuthzSVNAccessFile /etc/svn_acl
 
 > Require valid-user
 
 > \</Location\>

 **LoadModule** 加载了webdav需要的模块。
 
 **\<Location\>**的使用定义了apache路径的闭包，在apache中包括\<Directory\>、\<File\>、\<Location\>三种命令用于定义url path的闭包，前两者用于映射文件系统，而Location用于非文件系统位置的映射。
 
 其中AuthUserFile、AuthzSVNAccessFile分别定义了全局的用户账号、SVN ACL配置文件的位置。
#	配置PASSWD
使用以下命令修改svn_htpasswd文件内容。

	#创建文件并加入user
	htpasswd -cm /etc/svn_htpasswd user1
	#新增user
	htpasswd -m /etc/svn_htpasswd user2
#	配置ACL
acl是Access Control List的缩写，提供了用户和分组范围的访问控制配置，配置可以复制/var/www/svn/repo/conf/auth文件，如果没有指定全局的ACL，svn就会使用每个仓库下conf/auth的配置。

ACL文件配置如下：

	[groups]
	dev_group = user1,user2

	[/]
	@dev_group = rw

	[repository:/repo]
	@dev_group = rw

至此完成，启动或重启apache httpd即可:

	service httpd start/restart