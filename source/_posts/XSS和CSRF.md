---
title: XSS 和CSRF
tags:
  - http
  - 前端
categories:
  - 安全
date: 2017-08-27 20:30:39
---


# XSS：跨站脚本（Cross-site scripting）
- 示例
比如 别人在论坛中提交了一段代码
`
while(true){
	alert('1')
}
`
当自己打开这个论坛的时候，就会一直执行这段js，这段js可以用来盗号或者执行未授权的操作

- 原理
XSS 其实就是所谓的 HTML 注入，攻击者的输入没有经过后台的过滤直接进入到数据库，最终显示给来访的用户。如果攻击者输入一段 js 脚本，就能窃取来访者的敏感信息（比如 Cookie），实现伪装成来访者对网站发送危险请求。

- 防御
避免 XSS 的方法之一主要是对用户输入的内容进行过滤，比如 PHP 里面的 htmlspecialchars() 函数。



# CSRF（Cross-site request forgery）跨站请求伪造，又叫XSRF

>XSS 是实现 CSRF 的诸多途径中的一条，但绝对不是唯一的一条。一般习惯上把通过 XSS 来实现的 CSRF 称为 XSRF。

- 原理:
要完成一次CSRF攻击，受害者必须依次完成以下步骤：
	- 登录受信任网站A，并在本地生成Cookie。
	- 在不登出A的情况下，访问危险网站B。危险网站B，它里面有一段HTML的代码如下：　　
`
<img src=http://www.mybank.com/Transfer.php?toBankId=11&money=1000>
`
	- 你登录了银行网站A，然后访问危险网站B，噢，这时你会发现你的银行账户少了1000块！
    
- 防御
CSRF攻击是源于WEB的隐式身份验证机制！WEB的身份验证机制虽然可以保证一个请求是来自于某个用户的浏览器，但却无法保证该请求是用户批准发送的！
通过下面途径可以防御：
	- 敏感动作使用post，不要用get，post 受到跨域限制，浏览器在发送post请求前，发送option 询问服务端是否允许该源（域名+端口）的访问，假如自己站点存在让黑客发送危险请求的 漏洞，该方法就没用了。
	- 给每个表单加入随机 Token 进行验证，这样B页面无法获取A页面的 Token 导致请求验证失败，从而防止了 CSRF。

