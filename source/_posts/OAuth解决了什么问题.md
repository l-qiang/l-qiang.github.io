---
title: OAuth解决了什么问题？
date: 2019-10-15 19:47:15
categories:
 - OAuth
tags:
 - OAuth
---

{% blockquote RFC 6749 https://tools.ietf.org/html/rfc6749#section-1  section-1 %}
In the traditional client-server authentication model, the client
   requests an access-restricted resource (protected resource) on the
   server by authenticating with the server using the resource owner's
   credentials.  In order to provide third-party applications access to
   restricted resources, the resource owner shares its credentials with
   the third party.  This creates several problems and limitations:

<!-- more -->

   o  Third-party applications are required to store the resource
      owner's credentials for future use, typically a password in
      clear-text.

   o  Servers are required to support password authentication, despite
      the security weaknesses inherent in passwords.

   o  Third-party applications gain overly broad access to the resource
      owner's protected resources, leaving resource owners without any
      ability to restrict duration or access to a limited subset of
      resources.

   o  Resource owners cannot revoke access to an individual third party
      without revoking access to all third parties, and must do so by
      changing the third party's password.
   o  Compromise of any third-party application results in compromise of
      the end-user's password and all of the data protected by that
      password.

{% endblockquote  %}

大概意思就是

- 第三方应用需要存储资源拥有者的密码，并且通常是明文密码
- 尽管密码不安全，但是服务器必须支持密码验证
- 资源拥有者不能限制第三方应用对资源访问的范围和持续时间
- 资源拥有着不能撤销单个第三方应用的权限，只能是通过修改密码的方式来撤销所有第三方应用的权限
- 任何第三方应用遭到破坏会导致用户密码和所有受到该密码保护的数据受到损害

