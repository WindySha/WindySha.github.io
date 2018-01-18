---
title: Gmail，OutLook账户基于Oauth2.0协议授权登录邮箱客户端的功能实现
date: 2018-01-12 15:40:38
tags: 
- Oauth2.0
- Android邮箱客户端
- Imap协议
- gmail
- outlook
---

# 邮箱登录安全

考虑到邮箱登陆的安全性，使用这种登陆方法，用户不用暴露帐号密码给我们客户端就可以进行收发邮件，google推荐用户使用网页授权登陆的方式来登陆Gmail邮箱。
经过研究后，得知，这种授权登陆的方式都是给予Oauth2.0协议。很多第三方的邮件客户端都已实现了给予Oauth2.0授权登陆这一功能，例如：

WPS邮箱,QQ邮箱，网易邮箱大师实现了gmail的授权登陆
WeMail，myMail等客户端实现了gmail和outlook的授权登陆
微软的Outlook客户端实现了gmail,outlook，yahoo,office365的授权登陆
ios邮件实现了gmail,yahoo的授权登陆


# Oauth2.0协议流程

经研究，发现整个流程就是基于Oauth2.0协议和Imap协议的。Oauth2.0授权登陆流程具体如下：

Oauth2.0授权模式有多种，这里使用简化模式登陆最方便。
简化模式（implicit grant type）不通过第三方应用程序的服务器，直接在浏览器中向认证服务器申请令牌，跳过了"授权码"这个步骤，因此得名。所有步骤在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。
<!-- more -->

它的步骤如下：
（A）客户端将用户导向认证服务器。
（B）用户决定是否给于客户端授权。
（C）假设用户给予授权，认证服务器将用户导向客户端指定的"重定向URI"，并在URI的Hash部分包含了访问令牌。
（D）浏览器向资源服务器发出请求，其中不包括上一步收到的Hash值。
（E）资源服务器返回一个网页，其中包含的代码可以获取Hash值中的令牌。
（F）浏览器执行上一步获得的脚本，提取出令牌。
（G）浏览器将令牌发给客户端。

下面是上面这些步骤所需要的参数。
A步骤中，客户端发出的HTTP请求，包含以下参数：

> response_type：表示授权类型，此处的值固定为"token"，必选项。
client_id：表示客户端的ID，必选项。
redirect_uri：表示重定向的URI，可选项。
scope：表示权限范围，可选项。
state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值。

C步骤中，认证服务器回应客户端的URI，包含以下参数：

> access_token：表示访问令牌，必选项。
token_type：表示令牌类型，该值大小写不敏感，必选项。
expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。
state：如果客户端的请求中包含这个参数，认证服务器的回应也必须，一模一样包含这个参数。

如果用户访问的时候，客户端的"访问令牌"已经过期，则需要使用"更新令牌"申请一个新的访问令牌。
客户端发出更新令牌的HTTP请求，包含以下参数：

>granttype：表示使用的授权模式，此处的值固定为"refreshtoken"，必选项。
refresh_token：表示早前收到的更新令牌，必选项。
scope：表示申请的授权范围，不可以超出上一次申请的范围，如果省略该参数则表示与上一次一致。

# 使用Oauth2.0协议登陆Outlook的实现概述

基于这个流程，我们先尝试实现Outlook.com的授权登陆。
在此之前，查阅了大量microsoft提供给开发者的帮助文档：

> 1.用C#和VB实现的例子 OAuth 2.0 for Microsoft Accounts (installed applications)：
http://www.afterlogic.com/mailbee-net/docs/OAuth2MicrosoftRegularAccountsInstalledApps.html#RegisterMicrosoft
2.Windows Live Connect实现oauth2.0协议流程：
https://msdn.microsoft.com/en-us/library/hh243647.aspx#
3.Oauth2.0 scope参数详解
https://msdn.microsoft.com/en-us/library/hh243646.aspx#accessing
4.Live SDK developer guide：
https://msdn.microsoft.com/en-us/library/hh243641.aspx
5.获取用户info文档
https://msdn.microsoft.com/en-us/library/hh826533.aspx
6.Connect to Outlook.com IMAP using OAuth 2.0
https://msdn.microsoft.com/en-us/library/dn440163.aspx

通过对这写文档的仔细阅读，可以得知，microsoft账户授权登陆的参数如下：
```
          auth_endpoint="https://login.live.com/oauth20_authorize.srf"
          token_endpoint="https://login.live.com/oauth20_token.srf"
          refresh_endpoint="https://login.live.com/oauth20_token.srf"
          response_type="code"
          redirect_uri="https://login.live.com/oauth20_desktop.srf"
          scope="wl.basic wl.offline_access wl.emails wl.imap"
          state="state"
          client_id="000000004C187996"
          client_secret="YzfVeh1WllDzBuC7-t9lrhPM5AGrqXfX"
```
其中的client_id和client_secret需要先创建一个微软账户，然后登陆微软的Live.com Developer Center：https://account.live.com/developers/applications/


建立一个 applications，就可以取到合法的client_id和client_secret。

通过上面的参数使用Http Post，向服务器请求，就可以得到：
```
{
"token_type":"bearer",
"expires_in":3600,
"scope":"wl.basic wl.offline_access wl.emails wl.imap wl.signin",

"access_token":"EwCIAq1DBAAUGCCXc8wU/zFu9QnLdZXy+YnElFkAAednF24XQXBC+PBhDpfx2c7Z5/XKYAbNlItHSvK/CAuEhAew1YXyMGKUGHoe3g+mqEHQJO+eQqt/RFNnr7XY0362G/2FoSbClPl1s4YPs+oBF8vPlWlKl+5ORydw6i7AmzyxSO7Dz/IXlbtRoHYERgZayEYyJBNRd+7thcKivCtyHrpQI25Ue/jY+KIJEVMRzj35Ujjc3IbbSXkG8AEztyKmI/vDMHU2rRqYffmbZG3w7eGP1a4kB",

"refresh_token":"MCdn2!RmPgwz9FbjIlFiqygD4uEMdCoFPEWpxoT4GB6gILh9VzgF8hM2FMNyjrnHsjGR0LlH0LaS0BT4LowsI7Q$",
"user_id":"d9f34c758d648aac466bf1f40766ad7a"
}
```

里面包含我们需要的授权令牌access_token，和刷新令牌refresh_token。

但是，里面却没有包含用户的邮箱地址以及用户名，而邮箱地址是客户端一定需要的一个参数。
通过阅读文档：https://msdn.microsoft.com/en-us/library/hh826533.aspx


可以通过HTTP GET https://apis.live.net/v5.0/me?access_token=ACCESS_TOKEN
向服务器请求用户的信息。


最后，也是最终要的步骤，就是如何通过授权令牌access_token来进行收发邮件，收邮件是用Imap协议，发送用Smpt协议
microsoft对此也有帮助文档：https://msdn.microsoft.com/en-us/library/dn440163.aspx



上图就是封装access_token的方法，
具体的代码实现为：
```
final String accessToken = AuthenticationCache.getInstance().retrieveAccessToken(
        mImapStore.getContext(), mImapStore.getAccount());
if (mLoginPhrase == null || !TextUtils.equals(mAccessToken, accessToken)) {
    mAccessToken = accessToken;
    final String oauthCode = "user=" + mImapStore.getUsername() + '\001' +
            "auth=Bearer " + mAccessToken + '\001' + '\001';
    mLoginPhrase = ImapConstants.AUTHENTICATE + " " + ImapConstants.XOAUTH2 + " " +
            Base64.encodeToString(oauthCode.getBytes(), Base64.NO_WRAP);
}
```
上面就是使用Oauth2.0实现登陆Outlook基本流程，希望能对你有所帮助。