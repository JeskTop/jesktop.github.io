---
layout: post
title: 〈译〉使用Redis处理Rails Model缓存
date: 2015-07-17 10:02:30
disqus: y
---

- - -
原文：[Rails Model Caching with Redis](http://www.sitepoint.com/rails-model-caching-redis/)

Model层的缓存常常都会被忽略，甚至是经验丰富的码农。当你对视图层做缓存时，你不需要进行底层缓存，这是一个非常常见的误解。虽然在Rails里大部分的瓶颈在于视图层，但是总有个别情况不是这样的。

底层缓存是非常灵活的，可以工作于任何一个应用程序。在本教程中，我将演示如何使用Redis来缓存你的model层。

## 缓存是如何工作的？

过去，访问磁盘的成本已经非常高了。而且从磁盘访问数据经常会对性能产生不利的影响。为了解决这个问题，我们可以在应用程序和数据库服务器之间加上缓存层。

缓存层在初始化时是没有任何数据的。当它接收到数据请求时，它会调用数据库并将结果存储在内存中（缓存）。所有后续的请求将从缓存层直接读取数据，所以可以避免不重复往返的访问数据库服务器，从而提高性能。

## 为什么使用Redis？

Redis是一个基于内存、Key-Value存储系统。它的速度极快，几乎是瞬间完成数据检索。Redis支持先进的数据结构，如链表，哈希表，集合，并能持续保存到磁盘。

虽然大多数码农更喜欢使用Memcache和Dalli去处理他们的缓存化的需求，但我发现Redis非常容易的安装和方便管理。另外，如果你使用的是resque或Sidekiq管理你的队列，你很可能已经安装了Redis了。对于那些有兴趣了解何时使用Redis的朋友们，可以到这个 讨论 里了解更多相关信息。

## 前提

我假设你的项目正在使用Rails，文章中的例子是使用Rails 4.2.rc1，使用haml渲染视图和MongoDB作为数据库，但是本教程的片段应该适用于任何版本的Rails。

在开始之前，你需要安装和运行Redis。进入你的应用程序目录，并执行以下命令：

{% highlight html %}
$ wget http://download.redis.io/releases/redis-2.8.18.tar.gz
$ tar xzf redis-2.8.18.tar.gz
$ cd redis-2.8.18
$ make
{% endhighlight %}

这个命令将需要一段时间才能完成。一旦完成了，就可以开启Redis服务了：

{% highlight html %}
$ cd redis-2.8.18/src
$ ./redis-server
{% endhighlight %}

使用gem “rack-mini-profiler” 可以测量性能提升，这个gem可以帮助我们正确的体现出性能的改善。

## 开始

例如，让我们构建一个虚拟的在线故事书阅读书店。这个书店有各种各样的书籍和语言。首先，让我们创建模型：

{% highlight ruby %}
# app/models/category.rb
 
class Category
  include Mongoid::Document
  include Mongoid::Timestamps
  include Mongoid::Paranoia
 
  include CommonMeta
end
 
# app/models/language.rb
 
class Language
  include Mongoid: :Document
  include Mongoid::Timestamps
  include Mongoid::Paranoia
 
  include CommonMeta
end
 
# app/models/concerns/common_meta.rb
 
module CommonMeta
  extend ActiveSupport::Concern
  included do
    field :name, :type => String
    field :desc, :type => String
    field :page_title, :type => String
  end
end
{% endhighlight %}

我在[这里](https://github.com/skmvasu/redis_cache_sitepoint/blob/master/db/seeds.rb)包括了一个seed数据文件。只要复制粘贴到你的seeds.rb和运行rake seed任务，数据就会加载到我们的数据库中。

{% highlight html %}
rake db:seed
{% endhighlight %}

现在，让我们创建一个简单的Category列表页面，该页面显示了所有Categories的描述和标记信息。

{% highlight ruby %}
# app/controllers/category_controller.rb
 
class CategoryController < ApplicationController
  include CategoryHelper
  def index
    @categories = Category.all
  end
end
 
# app/helpers/category_helper.rb
 
module CategoryHelper
  def fetch_categories
    @categories = Category.all
  end
end
 
# app/views/category/index.html.haml
 
%h1
  Category Listing
%ul#categories
  - @categories.each do |cat|
      %li
        %h3
          = cat.name
        %p
          = cat.desc
 
# config.routes.rb
 
Rails.application.routes.draw do
  resources :languages
  resources :category
end
{% endhighlight %}

当你打开浏览器并将其地址指向/category时，你会发现mini-profiler benchmarking显示在后端执行每一个动作的时间。这些正确的数据是告诉你，你的应用程序哪部分比较缓慢和应该如何优化它们。本页面执行了两条SQL语句并且使用了5ms的时间完成查询。

虽然起初看起来好像5ms是无关紧要的，特别是在需要更多时间去渲染视图时，但在一个生产级别的应用程序中有多次数据库查询时，它们可以明显的降低软件的性能。

![](https://ruby-china-files.b0.upaiyun.com/photo/2015/853480fe43430b40618f7cc08e4987da.png)

由于元数据模型是不太可能发生改变的，这样就可以避免不必要的数据库切换。这一点也是底层缓存的用武之地。

## 安装Redis

使用Redis基于Ruby的客户端来帮助我们非常方便的链接Redis实例：

{% highlight ruby %}
gem 'redis'
gem 'redis-namespace'
gem 'redis-rails'
gem 'redis-rack-cache'
{% endhighlight %}

一但安装好这些gem后，就可以配置Rails使用Redis来作为缓存存储：

{% highlight ruby %}
# config/application.rb
 
#...........
config.cache_store = :redis_store, 'redis://localhost:6379/0/cache', { expires_in: 90.minutes }
#.........
{% endhighlight %}

使用redis-namespace gem可以让我们创建一个更好的Redis命名空间：

{% highlight ruby %}
# config/initializers/redis.rb 
 
$redis = Redis::Namespace.new("site_point", :redis => Redis.new)
{% endhighlight %}

现在所有的Redis功能都可以通过`$redis`进行全局使用了。以下的一个例子是体现如何访问在redis服务器上的值（运行于Rails console）：

{% highlight ruby %}
$redis.set("test_key", "Hello World!")
{% endhighlight %}

这个命令创建了一个key：“test_key”和value：“Hello World”保存在Redis中。要取这个值，只做：

{% highlight ruby %}
$redis.get("test_key")
{% endhighlight %}

现在，我们有了基础知识，让我们开始重写我们的helper方法：

{% highlight ruby %}
# app/helpers/category_helper.rb
 
module CategoryHelper
  def fetch_categories
    categories =  $redis.get("categories")
    if categories.nil?
      categories = Category.all.to_json
      $redis.set("categories", categories)
    end
    @categories = JSON.load categories
  end
end
{% endhighlight %}

在第一次执行这部分代码时，内存/缓存中是没有任何东西的。所以我们请求Rails把数据从数据库推送到Redis中。注意到 `to_json`的调用了吗？ 当要写对象进Redis，我们多种方式。一种选择是遍历对象中的每个属性，然后将它们保存为一个哈希函数，但是这种方式较为缓慢。最简单的方法是将它们保存为一个JSON编码的字符串。解码，只需使用`JSON.load`。

然而，这有一个意想不到的副作用。当我们正在检索这个值时，一个简易的对象符号不工作。我们需要更新视图并使用哈希语法来显示该类型：

{% highlight html %}
# app/views/category/index.html.haml
 
%h1
  Category Listing
%ul#categories
  - @categories.each do |cat|
    %li
      %h3
        = cat["name"]
      %p
        = cat["desc"]
{% endhighlight %}

重新启动浏览器，并看看性能是否有所不同。首次访问，我们仍然访问数据库，但随后的重新加载将不在访问数据库了。以后所有的请求都将直接从缓存中读取。这个简单的变化非常有效。

![](https://ruby-china-files.b0.upaiyun.com/photo/2015/fa943998d309ec7184d200795c4d8425.png)

## 管理缓存

我刚发现一个关于categories的错误。让我们先解决它：

{% highlight ruby %}
$ rails c
 
c = Category.find_by :name => "Famly and Frends"
c.name = "Family and Friends"
c.save
{% endhighlight %}

重新加载并查看该更新是否显示在视图中：

![](https://ruby-china-files.b0.upaiyun.com/photo/2015/8a2c5ac24c03612698f75fcf390f766b.png)

很遗憾，我们的视图上并没有体现出这个变化。因为我们并没有访问数据库，所有的数据都直接从缓存中读取。唉，现在的缓存已经过期，直到Redis重启前被更新都数据都无法使用。这个对于大多数应用程序来说真是一个破坏者啊。我们偶尔可以使用缓存到期来解决这个问题：

{% highlight ruby %}
# app/helpers/category_helper.rb
 
module CategoryHelper
  def fetch_categories
    categories =  $redis.get("categories")
    if categories.nil?
      categories = Category.all.to_json
      $redis.set("categories", categories)
      # Expire the cache, every 3 hours
      $redis.expire("categories",3.hour.to_i)
    end
    @categories = JSON.load categories
  end
end
{% endhighlight %}

缓存将会在每3个小时就失效。虽然这对大多数情况下工作，缓存中的数据将滞后于现在数据库。这种工作方式很可能不抬适合你。如果你喜欢保持缓存的更新，我们可以使用`after_save`这个回调：

{% highlight ruby %}
# app/models/category.rb
 
class Category
  #...........
  after_save :clear_cache
 
  def clear_cache
    $redis.del "categories"
  end
  #...........
end
{% endhighlight %}

每次模型的更新，我们都将通知Rails去清除缓存。这样可以确保缓存是最新的。Yay!

> 你应该使用类似`cache_observers`在生产环境中，为了保持简洁，我们在这里坚持使用`after_save`。如果你不知道哪种方法最适合你，这里的[讨论](http://stackoverflow.com/questions/15165260/rails-observer-alternatives-for-4-0)可能会对你有所启发。

## 结论

底层缓存是非常简单的，如果使用得当，它是非常有价值的。它可以在你花费最小的努力下瞬间提高你的系统的性能。在这篇文章中所有的[代码片断](https://github.com/skmvasu/redis_cache_sitepoint)可以在GitHub上找到。 

希望喜欢读篇文章。欢迎在评论中分享你的想法。
