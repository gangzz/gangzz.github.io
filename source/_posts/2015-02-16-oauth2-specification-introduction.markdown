---
layout: post
title: "OAuth2 Specification Introduction"
date: 2015-02-16 22:28:30 +0800
comments: true
categories: ["Oauth2", "Others"]
---
#OAuth 2.0#
- - -
  
##1.1 Role##
*	Resource Owner：资源的拥有者，当资源拥有者指人的时候一般是终端用户。
*	Resource Server：资源不熟的服务器。负责响应合法Access_Token的资源请求。
*	Client：请求资源的客户端，可以使其它应用的服务器、桌面应用、其它设备等。
*	Authorization Server：授权服务器。在【资源拥有者】授权后颁发给Client合法的Access_Token，并保存授权信息。

##1.2 Protocol Flow##

>     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+

                     
Figure 1: Abstract Protocol Flow

1.	Client直接或者间接向Resource Owner索取资源的授权。
2.	Resource Owner返回授权信息给Client（grant code）。
3.	Client向Authorization Server发送包含授权信息的请求。
4.	Authorization Server颁发Access_Token给Client。
5.	Client使用Access_Token向Resource Server请求感兴趣的资源。
6.	Resource Server验证Access_Token的有效性（向Authorization Server）后，响应Client请求。


##1.3 Authorization Grant
认证授权是一张代表了资源拥有者授权的证书，Client用它获取Access_Token。OAuth2提供了四种授权类型：authorization code， implicit， resource owner password credentials(UserName, Password)，client credential——第四种提供客户端的可扩展性。

###1.3.1 Authorization Code
通过介于Client和Resource Owner之间的Authorization Server获取授权码。与直接向Resource Owner索取授权不同，Client把Resource Owner导向授权服务器，在获取到授权后该服务器将Resource Owner和授权码导向Client。
  
在把Resource Owner和授权码导向Client之前，Authorization Server对其鉴权并获得授权。由于Resource Owner只需要和Authorization Server打交道，Resource Owner的证书不需要和Client共享的。

授权码的机制提供了几个安全方面重要的好处，比如鉴定Client、直接把Access_Token传递给Client，而不需要经过Resource Owner——那将存在暴露给第三方的潜在风险。

###1.3.2 Implicit（简化）
Implicit流程是Authorization Code的简化版本，主要用于浏览器中脚本实现。在Implicit流程中不是颁发授权码（Authorization Code）而是直接传递Access_Token给Client。这个授权过程是暗含的，这其中没有中间证书的发布（如Authorization Code）。

在传递Access_Token给Client的过程中，Authorization Server是不对Client鉴定的。有些情况下，可能通过Redirect Url鉴定Client，但是在发布过程中Access_Token可能会被Resource Owner和任何有权获得Client User-Agent的设备截获。

暗含授权机制提升了一些客户端（如浏览器内App）的响应性和效率，因为它降低了获得Access_Token过程中的交互旅程。然而，必须权衡该便利性带来的安全方面的损失，特别是在Authorization Code方式可选的时候，这部分将在[10.3](http://tools.ietf.org/html/rfc6749#section-10.3)和[10.6](http://tools.ietf.org/html/rfc6749#section-10.16)节讲述。

###1.3.3 Resource Owner Password Credentials
Resource Owner的密码证书（如用户名+密码）可以作为直接获得Access_Token的授权方式。这种授权方式应该只在Resource Owner和Client存在充分信任的情况下使用，而且是其它授权机制无法使用时（比如Client是客户设备操作系统的一部分或者高优先级的程序）。

虽然这种认证方式要求Client能够直接访问Resource Owner的密码证书，但是还是要求在单独的请求中传递Resource Owner的证书来换取密码。该机制应避免在Client中记录Resource Owner的密码，可以通过使用长期的Access_Token或者Refresh_Token方式代替。

###1.3.4 Client Credentials
当被请求资源是Client自己控制的或者被请求资源是预先与Authorization Server商定好的场景下，可以采用客户端（Client)认证。Client Credential被用作鉴权方案，通常是Client就是Resource Owner自身或者是被请求资源的授权已经预先在Authorization Server设定好的。

##1.4 Access Token
Access Token是用来获取被保护资源的凭证。一个Access Token是一段代表了向client授权的字符串。该字符串对Client通常非透明的。它规定了授权的区间和范围，由Resource Owner授权，由Authorization Server和Resource Server执行。

Access Token可能是用于检索权限信息的Id，也可能是符合某种校验规格的包含权限信息的数据（比如包含数据和签名）。客户端（Client）在使用Access Token时可能还要引入额外的鉴权机制——这超出了该规格的讨论范围。


Access Token提供了一个授权的抽象层，使用可被Resource Server理解的单一token替代了其它不同的验证机制。在这个抽象中发布证书的动作比授权证书的动作更严格，同时免去了Resource Server要了解一大堆授权方法的痛苦。

基于Resource Server对安全性的要求，Access tokens可以存在不同的格式、构造和可用方法。Access Token的属性和方法超出了本规格的范围，在一些同伴规格比如[\[RFC6750\]](http://tools.ietf.org/html/rfc6750)中有其相关的定义。

##1.5 Refresh Token
Refresh Token是一种用于获取Access Token的凭证。Refresh Token由Authorization Server发布，用于获取新的Access Token（当原有变为非法或失效后），或者用于获得具备同样或者更窄范围的额外Access Token（一个生命周期更短，或者范围比Resource Owner授权的更窄的Access Token）。是否发布Refersh Token是Authorization Server自主决定的。Refresh Token的发布是包含在Access Token的发布中的。

Refresh Token是代表了Resource Owner授权的字符串，通常对Client是非透明的。这个字符串代表了获取授权信息的Id。与Access Token不同 Refresh Token只用于和Authorization Server交互，不会用于Resource Server。

	  +--------+                                           +---------------+
  	  |        |--(A)------- Authorization Grant --------->|               |
	  |        |                                           |               |
	  |        |<-(B)----------- Access Token -------------|               |
	  |        |               & Refresh Token             |               |
	  |        |                                           |               |
	  |        |                            +----------+   |               |
	  |        |--(C)---- Access Token ---->|          |   |               |
	  |        |                            |          |   |               |
	  |        |<-(D)- Protected Resource --| Resource |   | Authorization |
	  | Client |                            |  Server  |   |     Server    |
	  |        |--(E)---- Access Token ---->|          |   |               |
	  |        |                            |          |   |               |
	  |        |<-(F)- Invalid Token Error -|          |   |               |
	  |        |                            +----------+   |               |
	  |        |                                           |               |
	  |        |--(G)----------- Refresh Token ----------->|               |
	  |        |                                           |               |
	  |        |<-(H)----------- Access Token -------------|               |
	  +--------+           & Optional Refresh Token        +---------------+

               Figure 2: Refreshing an Expired Access Token

图标2中展示的流程：

1. 客户端向权限服务器发送授权信息和鉴定请求，请求一个Access Token。
2. 权限服务器鉴定客户端并验证授权信息的合法性，如果合法发布一个Access Token和一个Refresh Token。
3. 客户端提交对受保护资源的请求和Access Token。
4. 资源服务器鉴定Access Token合法性，如合法响应请求。
5. 步骤（3）和步骤（4）不断进行，直到Access Token失效。如果客户端发觉Access Token失效它会跳转到步骤（7），否则继续（3）。
6. 因为Access Token已经失效，资源服务器返回token失效错误。
7. 客户端传递鉴定请求和Refresh Token给权限服务器，获取新的Access Token。客户端的授权需求基于客户端类型和权限服务器的策略。
8. 权限服务器鉴权Client并验证Refresh Token，如果合法，发布新的Access Token（有可能包括新的Refresh Token）。

步骤3，4,5,6超出了本规格范围。


##1.6 TLS Version
TLS(Transport Layer Security)。无论该协议选择哪个TLS，随着广泛部署和安全弱点的暴露，都有会心的TLS出现。到目前为止TLS 1.2是最新的版本，但是它还存在较多部署限制并且可能还不适合实现。TLS 1.0[RFC2246](http://tools.ietf.org/html/rfc2246)是目前最广泛部署的版本，并且提供了良好的协同性。

不同的实现可能还选择了额外的TLS机制满足他们的安全需要。

##1.7 HTTP Redirections
在协议中扩展使用了Http重定向，客户端或者权限服务器使用这种方式彼此传递Resource Owner的user-agent。在当前协议中使用了Http的302状态码的方式，其它经由user-agent的方式也是被允许的并且可以视为一种具体实现。

##1.8 Interoperability(互操作性)
OAuth 2.0提供了一个丰富的框架和良好定义的安全特性。然而，作为一个包含众多可选组件的丰富且高可扩展的框架，就自身而言，这份协议倾向于产生一个广泛的不可互操作的实现。

除此以外，协议中有少量的必要组件是部分实现或未实现（如客户端注册、权限服务器扩展性、终端发现）。缺失了这些组件，客户端只能人工并且针对权限服务器和资源服务器配置才能实现互操作性。

协议的定义存在一个明确的期待——将来的工作会定义规范的资料和必要的扩展，以实现全面的网络层互操作性。

##1.9 Notational Conventions(标记约定)
关键字"MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL"的示意在[RFC2119](http://tools.ietf.org/html/rfc2119)中定义.

本协议使用Augmented Backus-Naur Form (ABNF)[RFC5234](http://tools.ietf.org/html/rfc5234)标记。此外, 从"Uniform Resource Identifier (URI): Generic Syntax“ [RFC3986](http://tools.ietf.org/html/rfc3986)引入了URI-reference的规定。


某些安全相关的短语采用[RFC4949](http://tools.ietf.org/html/rfc4949)中的场景理解.这些短语包括但不限于："attack", "authentication", "authorization", "certificate","confidentiality", "credential", "encryption", "identity", "sign","signature", "trust", "validate", and "verify".

除非特殊说明，左右协议的参数名和值都是大小写敏感的。
