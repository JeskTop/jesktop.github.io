---
layout: post
title: 配置第三方邮件发送服务Mailgun
date: 2015-04-08 09:28:40
disqus: y
---

[Mailgun](https://mailgun.com/)是一个第三方的邮件发送服务，文档非常的详细，功能也非常强大。在使用Mailgun之前，因为用户量不大，所以一直用Gmail的SMTP进行发送邮件，但是这不是一个长久的办法，所以我们就使用Mailgun作为代替的方案。

参考了[Tower](https://tower.im/)和[Github](https://github.com/)的邮件服务，我们从原来使用Gmail成功迁移到了Mailgun，和添加了大量新的功能，例如通过回复邮件在系统中创建留言等功能。新的功能使用Mailgun完成都非常方便，可见Mailgun的功能非常的强大，在这篇文章中，主要介绍如何在项目中配置Mailgun，别的更强大的功能就等到往后的文章在做介绍啦。

### 添加子域名处理邮件服务

假设现在我们使用的域名是`abc.com`，因为考虑到如果在Mailgun中也使用xxx@abc.com，很可能会导致和企业邮箱发生冲突，所以我们需要添加一个子域名去专门处理Mailgun的事情，那我们这里添加一个`mail.abc.com`的子域名。

在Mailgun中，点击`Add Domain`，然后Mailgun就会告诉你如何进行配置，整个过程是非常方便的，等待验证通过后，我们就开始使用Mailgun了。

### 域名信息

添加好域名信息后，我们可以查看域名的相关信息，大概如下：
{% highlight html %}
State: Active
IP Address: 184.174.154.221
SMTP Hostname: smtp.mailgun.org
Default SMTP Login: postmaster@mail.abc.com
Default Password: xxxxx91750ccxxxxx
API Base URL: https://api.mailgun.net/v3/no-reply@mail.abc.com
API Key: key-91750-xxxxx
{% endhighlight %}

Mailgun提供多种方式让你去使用他的邮件服务，一开始我尝试的使用Mailgun的API服务，可是最后还是没有配置成功，在Rails里使用第三方的Gem，还是存在部分问题。所以我还是使用了比较简单的SMTP，具体他们两者之间有什么差异，我还没有深入的研究过。

### 使用SMTP方式

使用SMTP其实就和使用一般的邮件服务一样，只需要在Rails项目中`config/environments/production.rb`添加：
{% highlight ruby %}
Rails.application.configure do
  ...
  ActionMailer::Base.smtp_settings = {
    port:            587,
    address:         'smtp.mailgun.org',  
    user_name:       'postmaster@mail.abc.com',
    password:        'xxxxx91750ccxxxxx',
    domain:          'mail.abc.com',
    authentication:  :plain
  }
  ActionMailer::Base.delivery_method = :smtp
  ...
end
{% endhighlight %}

然后在Mailer中就可以使用Mailgun发送邮件了：

{% highlight ruby %}
class ApplicationMailer < ActionMailer::Base
  default from: "notification <notification@mail.abc.com>"
  layout 'mailer'
  ...
end
{% endhighlight %}
