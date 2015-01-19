---
layout: post
title: 微信开发简介
date: 2015-01-18 17:28:40
disqus: y
---

## 推荐使用GEM

在开始前，推荐使用两个gem：  
[gem 'weixin_rails_middleware'](https://github.com/lanrion/weixin_rails_middleware)    
[gem 'weixin_authorize'](https://github.com/lanrion/weixin_authorize)   
具体可以查看gem的文档，使用后可以很方便的处理一些问题，例如签名，加密和接口参数拼接等。  

## 基础接口
如果只使用基础接口，只需要使用`weixin_rails_middleware`这个gem就可以了。基础接口指的是一般用户主动发出的请求，例如：发送文本或图片等消息到公众帐号时，公众帐号发出的回应消息。而相比于高级接口来说，基础接口要简单很多。

### 验证微信请求
在开始时，微信需要先验证你的接口，根据官方文档的验证方式大概是：  
在微信后台填入服务器地址（URL）、Token和EncodingAESKey，然后微信通过提供的参数和一个随机数（echostr）进行字典序排序后，进行sha1加密，然后发送至服务器端，服务器端只要对其验证并且返回随机数（echostr），即表示验证成功。（具体内容参考微信文档）   
而使用gem后，整个验证流程基本不需要自己操心，只需要添加正确的secret_key和token，验证就通过了。

### 业务逻辑实现
使用`weixin_rails_middleware`后，执行 `rails generate weixin_rails_middleware:install`，就生成了整个业务逻辑的简单实现了，包括发送来的是图片，文本或者是关注之类的操作。只要是用户主动发起的操作和请求，基本都实现了简单的回复，所以如果不是特别复杂的操作，基础接口中业务逻辑的实现还是相当方便的。

## 高级接口
这里主要想说一下高级接口，因为在第一次开发微信公众号时，还是走了不少弯路。   

### 简介
在使用高级接口时，我们使用`gem 'weixin_authorize'`。高级接口和基础接口有什么不一样呢？我觉得最大的区别在于高级接口基本上都是服务器端主动发起的请求，因为是服务器端主动发起的请求，所以每次需要带上`access_token`来提供微信验证此信息是否是服务器端发送的。所以我们先来看看`access_token`是如何获得的。

### 获取access_token
{% highlight html %}
access_token是公众号的全局唯一票据，公众号调用各接口时都需使用access_token。开发者需要进行妥善保存。access_token的存储至少要保留512个字符空间。access_token的有效期目前为2个小时，需定时刷新，重复获取将导致上次获取的access_token失效。
{% endhighlight %}
因为`access_token`是会失效的，所以`gem 'weixin_authorize'`中使用Redis来存储`access_token`是一个不错的方式，根据文档配置，然后通过：
{% highlight ruby %}
$client ||= WeixinAuthorize::Client.new(ENV["APPID"], ENV["APPSECRET"])
{% endhighlight %}
即可获得对应公众号的`access_token`（公众号可以使用AppID和AppSecret调用接口来获取access_token）。  
现在咱们手持`access_token`自然如有神助，可以大显身手了。

### 自定义菜单
因为使用了开发者模式，所以就算是希望完成自定义菜单这样的事情也变得困难起来。那我们应该如何自定义菜单呢？  
建议参考：[自定义菜单的实现](https://github.com/lanrion/weixin_rails_middleware/wiki/DIY-menu)    
文档中讲的很详细，表的结构可以参考文档。这里主要说一下，如何把定义好的菜单结构发送给微信，并且生成对应的菜单。  
参考：  

{% highlight ruby %}
def generate_menu
  weixin_client = WeixinAuthorize::Client.new(@current_public_account.app_key, @current_public_account.app_secret)
  menu   = @current_public_account.build_menu
  result = weixin_client.create_menu(menu)
  set_error_message(result["errmsg"]) if result["errcode"] != 0
  redirect_to public_account_diymenus_path(@current_public_account)
end
{% endhighlight %} 
其实就是带着`access_token`，把微信需要的菜单JSON结构拼装好，发送到微信对应的接口，然后微信就会生成菜单了。因为`gem 'weixin_authorize'`把事情都处理好了，所以自定义菜单对于我们来说，就变得非常的简单了。

### 关联用户和微信帐号
大家如果使用过银行的一些公众帐号，一定会用过这样的功能，就是把微信帐号和银行卡关联起来，然后银行卡发生交易时，微信都会收到相应的提示。那么它们是如何关联起来的呢？  
我这里介绍一个关联方式，就是用户通过公众帐号的菜单，点击“登录”，然后进行关联。那么公众帐号菜单的登录和直接用网页登录究竟有什么不一样呢？

#### 请求授权页面
在微信菜单中的“登录”按钮跳到的页面也是一个网站的登录页面，但是他们不一样的地方是，这里需要微信做一个授权，也就是菜单中的“登录”按钮指向的链接是这样的： 
 
{% highlight html %}
https://open.weixin.qq.com/connect/oauth2/authorize?appid=APPID&redirect_uri=REDIRECT_URI&response_type=code&scope=SCOPE&state=STATE#wechat_redirect
{% endhighlight %}
里面两个必须的参数是APPID和REDIRECT_URI，其中REDIRECT_URI是需要跳转的链接地址。为什么需要这样的地址呢？   
因为随着微信的REDIRECT_URI，去到登录页面时会带着一个code值，而我们需要用code来获取用户的openid。

{% highlight html %}
code说明 ：
code作为换取access_token的票据，每次用户授权带上的code将不一样，code只能使用一次，5分钟未被使用自动过期。

openid说明：
普通用户的标识，对当前公众号唯一
{% endhighlight %}
有了code值，就可以立刻向微信获取用户的openid值了：

{% highlight ruby %}
sns_info = $client.get_oauth_access_token(params[:code])
if sns_info.is_ok?
  user.openid = sns_info.result[:openid]
end
{% endhighlight %}
只要这样，当用户通过该页面登录成功后，就可以把openid和user关联起来，如果往后有消息需要推送给用户的话就相当方便了，调用微信对应的接口加上openid，就可以发送消息到对应的微信用户上了。 

以上就是这些时间里对微信公众帐号开发的一些看法和认识，都是一些比较简单的介绍。下面在介绍大家开发时记得使用微信提供的沙箱进行调试：
[http://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login](http://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login)
