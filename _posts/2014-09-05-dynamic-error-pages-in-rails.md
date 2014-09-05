---
layout: post
title: Rails中如何自定义Error Pages
date: 2014-09-05 22:43:00
disqus: y
---

## 修改配置
首先需要修改项目的配置，让项目接收到错误提示时，从route中寻找，而不直接读public的文件：
{% highlight ruby %}
# config/application.rb   
config.exceptions_app = self.routes
{% endhighlight %}
加入配置之后，避免在读取到/404, /422, /500时，仍然从public中寻找，建议删除public下的这三个html文件。

## 加入controler，view和route
因为public的对应错误信息页面删除了，所以需要重新定义错误信息的Route：
{% highlight ruby %}
# config/routes.rb
%w(404 422 500).each do |code|
  get code, to: "errors#show", code: code
end
{% endhighlight %}
在加好Errors后，自然需要对应的Controller文件：
{% highlight ruby %}
class ErrorsController < ApplicationController
 
  def show
    render status_code.to_s, status: status_code
  end
 
protected
 
  def status_code
    params[:code] || 500
  end
 
end
{% endhighlight %}
这样就会根据错误的提示，去render对应的模板，如出现404错误，则去寻找errors/404.html.erb，所以加入相应的404.html.erb, 422.html.erb, 500.html.erb。参考：
{% highlight ruby %}
# app/views/errors/404.html.haml
%h1 404 - Not Found
{% endhighlight %}
## 开发环境下，测试错误信息页面
因为开发环境下出现错误，都会呈现对应的错误信息，包括代码位置等。而不会弹出404或500页面，其实要在开发环境下，检测这些页面也是很方便的：
{% highlight ruby %}
# config/environments/development.rb
config.consider_all_requests_local = false
{% endhighlight %}
只需要把上面的设置从true改为false就可以了。

## Rails内自带的500错误提示
当把错误自定义以route的方式展现后，如果本来的错误信息页面，例如404页面出错了，就会出现这样的错误：
{% highlight ruby %}
"500 Internal Server Error\n" \
"If you are the administrator of this website, then please read this web " \
"application's log file and/or the web server's log file to find out what " \
"went wrong."
{% endhighlight %}
这个问题就像刚刚上面说的，是因为你的错误提示信息页面出错了，无法展现404页面了，所以就调用了Rails下的一个500错误提示信息，源码位置在：https://github.com/rails/rails/blob/4-0-stable/actionpack/lib/action_dispatch/middleware/show_exceptions.rb#L18-L22
所以，如果出现了这样的错误，需要仔细看看自己的错误信息页面是否在哪里出了问题。

参考：
[DYNAMIC ERROR PAGES IN RAILS](http://wearestac.com/blog/dynamic-error-pages-in-rails)。