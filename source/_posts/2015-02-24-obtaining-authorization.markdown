---
layout: post
title: "Obtaining Authorization"
date: 2015-02-24 10:54:55 +0800
comments: true
categories: [others, oauth2]
---

# Obtaining Authorization（获取授权）

客户端通过向资源拥有者申请授权的方式获得Access Token。OAuth提供四种授权方式：授权码、简化授权、资源拥有者密码凭证、客户端凭证，并提供了扩展其它凭证的机制。

##	4.1 Authorization Code Grant（授权码授权）

授权码授权可以用于access token、refersh token的获取，并且非常适合机密客户端。因为这是一个重定向流程，客户都必须支持与资源拥有者的浏览器交互，并可以响应来自权限服务器的重定向请求。

	
     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |  Client |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     +---------+       (w/ Optional Refresh Token)

	   				注意：步骤A、B、C通过浏览器时分成两部分。

                     Figure 3: Authorization Code Flow
                     
  图示3中包含以下步骤：
  
  1.	客户端引导资源拥有者的浏览器到授权端口来初始化流程，客户端包括客户端Id、请求的范围、本地状态、用于重定向的URI。
  2. 权限服务器校验资源拥有者的身份，然后执行授权或者拒绝授权。
  3. 假设资源拥有者予以授权，权限服务器重定向浏览器到URI，该URI种包括之前提供了方位和本地状态。
  4. 客户单使用上一步骤中得到的授权码从服务器索取Access Token。发起请求时，客户端向权限服务器鉴权。客户端发送包含其重定向URI、授权码用于验证。
  5. 权限服务器鉴权客户端，校验授权码，确认重定向URI与注册时的匹配。如果验证通过，权限服务器发布Access Token，可选的发布刷新Token。

  ### 4.1.1 Authorization Request（授权请求）
  客户端添加以下参数到查询部分到授权端点的URI（"application/x-www-form-urlencoded"格式）来构造授权请求：
  
  *	 response_type：必须。值必须设置为“code”。
  *	 client_id：必须。客户端的身份Id。
  *	 redirect_uri：可选。
  *	 scope：可选。申请访问的范围。
  *	 state：建议使用。一个非透明的值用于客户端维护请求和回调时的状态。权限服务器在重定向回客户端时保留该值。该参数应该使用以避免跨站欺骗。

  客户端通过重定向响应或其他手段引导资源拥有者到期构造的URI。
  
  举例，客户端引导浏览器发起TLS请求：
  
	GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
	Host: server.example.com
    
  权限服务器校验请求确保所有参数已提供并合法。如果合法权限服务器鉴权资源拥有者，并获取授权结果。
  
  一旦授权结论确定，权限服务器使用重定向或其它方式返回响应给client。
  
 