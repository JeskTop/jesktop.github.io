---
layout: post
title: 使用ember-simple-auth和devise做登录验证
date: 2015-04-08 09:28:40
disqus: y
---

参考：[Ember Simple Auth Devise](https://github.com/simplabs/ember-simple-auth/tree/master/packages/ember-simple-auth-devise)  

* 文章内容里，已经对如何使用是讲得比较清楚的了。所以我这里只是记录一个我在使用ember-simple-auth和devise做登录时的过程。

## ember-simple-auth设置

### ember-cli中加入ember-simple-auth

package.json中加入：
{% highlight javascript %}
{
  ...
  "devDependencies": {
    ...
    "ember-cli-simple-auth": "^0.7.3",
    "ember-cli-simple-auth-devise": "^0.7.3"
  }
}
{% endhighlight %}

bower.json中加入：
{% highlight javascript %}
{
  "name": "link-us",
  "dependencies": {
    ...
    "ember-simple-auth": "0.7.3"
  }
}
{% endhighlight %}

environment.js中引入'simple-auth-authorizer:devise'：
{% highlight javascript %}
module.exports = function(environment) {
  ...
  ENV['simple-auth'] = {
    authorizer: 'simple-auth-authorizer:devise'
  };
  ...
};
{% endhighlight %}

以上，分别给npm和bower加入相关的依赖环境，完成后，就可以在前端使用`ember-simple-auth`了。

### ember中加入登录页面

route中加入login的路径：
{% highlight javascript %}
Router.map(function() {
  this.route('login');
});
{% endhighlight %}

添加routes/login.js：
{% highlight javascript %}
import Ember from 'ember';

export default Ember.Route.extend({
});
{% endhighlight %}

添加controllers/login.js：
{% highlight javascript %}
import Ember from 'ember';
import LoginControllerMixin from 'simple-auth/mixins/login-controller-mixin';

export default Ember.Controller.extend(LoginControllerMixin, {
  authenticator: 'simple-auth-authenticator:devise'
});
{% endhighlight %}

添加登录页面templates/login.js：
{% highlight html %}
<form {{action 'authenticate' on='submit'}}>
  <label for="identification">帐号</label>
  {{input value=identification}}
  <label for="password">密码</label>
  {{input value=password type='password'}}
  <button type="submit">登录</button>
</form>
{% endhighlight %}

### ember-simple-auth原理
加好上面的代码后，前端的操作基本就起来了。
整个原理是这样的：
*   登录时，帐号和密码会POST到接口"/users/sign_in"中，并且带参数`{"user"=>{"password"=>"[FILTERED]", "email"=>"admin@admin.com"}}`
*   如果服务器端验证通过后，就会把token发过来。然后这时候，`ember-simple-auth`会把token放到每次请求的header中，如：`Authorization: Token token="<token>", email="<email>"`

## Devise设置

Devise在3.1版本以后，就去掉原来自带的使用token登录方式，所以现在需要自己配置。可参考：[Simple Token Authentication Example](https://gist.github.com/josevalim/fb706b1e933ef01e4fb6)，或使用[gem 'devise_token_auth'](https://github.com/lynndylanhurley/devise_token_auth)
这里我们主要参考`ember-simple-auth`的例子。

### 添加authentication_token字段
使用token登录方式，其实就是每次访问请求的时候，都带着user的token到服务器，然后服务器对token和uer的email进行匹配，如果正确了则继续访问，如果不争取就返回401提示没有登录。

所以我们首先给user添加一个`authentication_token`字段：
{% highlight ruby %}
class AddAuthenticationTokenToUser < ActiveRecord::Migration
  def change
    add_column :users, :authentication_token, :string
  end
end
{% endhighlight %}

并且如果user的`authentication_token`为空时，添加`authentication_token`数据，其实`authentication_token`就是一个随机串，并且保证在user表中唯一就可以了，所以我们添加一个创建：`authentication_token`的方法：
{% highlight ruby %}
class User < ActiveRecord::Base
  before_save :ensure_authentication_token

  def ensure_authentication_token
    if authentication_token.blank?
      self.authentication_token = generate_authentication_token
    end
  end

  private

    def generate_authentication_token
      loop do
        token = Devise.friendly_token
        break token unless User.where(authentication_token: token).first
      end
    end
end
{% endhighlight %}

### 添加登录接口
因为我们使用了devise，所以做登录接口，还是相对简单的，只要通过继承`Devise::SessionsController`，只是修改Devise原来返回参数的方式即可：
{% highlight ruby %}
class SessionsController < Devise::SessionsController
  respond_to :html, :json

  def create
    super do |user|
      if request.format.json?
        data = {
          token: user.authentication_token,
          email: user.email
        }
        render json: data, status: 201 and return
      end
    end
  end
end
{% endhighlight %}

这样session的验证成功后，便会把data信息返回到用户，并且显示登录成功。现在还需要在route中设置登录的url地址：
{% highlight ruby %}
App::Application.routes.draw do
  devise_for :users, controllers: { sessions: 'sessions' }
end
{% endhighlight %}

### 验证用户token信息
由于用户的每一次访问，均会把token和email带上，所以我们需要对这两个参数进行匹配，也就是验证用户是否有访问权限，如果没有则告诉用户401的错误信息，提示用户需要登录后才能继续进行操作：
{% highlight ruby %}
class ApplicationController < ActionController::Base
  before_filter :authenticate_user_from_token!

  before_filter :authenticate_user!

  private

  def authenticate_user_from_token!
    authenticate_with_http_token do |token, options|
      user_email = options[:email].presence
      user = user_email && User.find_by_email(user_email)

      if user && Devise.secure_compare(user.authentication_token, token)
        sign_in user, store: false  #如果用户的email和token没有错，user标识为sign in状态
      end
    end
  end
end
{% endhighlight %}

## 总结
一开始我想着还是继续使用session或者cookies来做登录验证的，毕竟早期的时候，ember和rails这块，还是有几个库是使用session登录方式，但是随着时间长了后，似乎基本都不更新了，而且暴露的问题越来越多。所以我果断把整个项目的登录方式推到，使用了维护更积极的`ember-simple-auth`，而且token的登录方式另外一个好处就是，手机端的登录也是使用token进行验证的。所以大家在使用前端mvc框架的时候，建议大家第一时间考虑使用token登录的方式。但是token登录的方式，存在一个安全问题，就如`ember-simple-auth`的作者强烈要求大家使用https。

但是这些时间看了部分资料和深入了解https后，发现原来https因为涉及加密和解密的问题，所以使用https的成本会比较高（服务器消耗大和https证书需要年费）。而且也发现JWT这种玩意可以加强api的安全性，目前尚未深入了解，如果这东西确实可以在不使用https的前提下提高api的安全性，应该也算是一个不错的选择。
