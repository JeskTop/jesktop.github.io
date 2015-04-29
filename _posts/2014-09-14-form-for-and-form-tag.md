---
layout: post
title: 如何理解Form_for和Form_tag
date: 2014-09-14 12:12:00
disqus: y
---

- - -
## Form_for是什么
在我们使用Rails的过程中，必然离不开`Form_for`，甚至在刚入门没有多长的时间里就会接触到`Form_for`。本来`Form_for`并不是一个如何特别神奇的东西，无非就是一个Helper，Helper的作用是什么？对于Helper而言，无论它的逻辑多么的复杂，其实本质的作用就是生成HTML代码。所以我们应该以HTML的角度，去理解`Form_for`.  
为什么特别写一篇文章来讲`Form_for`呢？主要的原因是之前有个人问了我一个这样的问题：  
`<input type="reset"/> `是用于重置表单的，可是我并不知道应该怎么放入`Form_for`当中！  
关于这类型的问题，我已经遇到很多次了，例如说Boostrap中关于一些form的样式代码，不知道如何应用到`Form_for`中之类。 那究竟关于`<form>`里的一些事情，如何应用到`Form_for`中呢？ 
先看看官方的文档怎么介绍`Form_for`的：  
{% highlight ruby %}
<%= form_for @article, url: {action: "create"}, html: {class: "nifty_form"} do |f| %>
  <%= f.text_field :title %>
  <%= f.text_area :body, size: "60x12" %>
  <%= f.submit "Create" %>
<% end %>
{% endhighlight %}
生成如下的HTML：  
{% highlight html %}
<form accept-charset="UTF-8" action="/articles/create" method="post" class="nifty_form">
  <input id="article_title" name="article[title]" type="text" />
  <textarea id="article_body" name="article[body]" cols="60" rows="12"></textarea>
  <input name="commit" type="submit" value="Create" />
</form>
{% endhighlight %} 
其实如果你愿意的话，上面的`Form_for`是可以混着HTML写的，当然我觉得这不会是一个好的案例，如：  
{% highlight ruby %}
<%= form_for @article, url: {action: "create"}, html: {class: "nifty_form"} do |f| %>
  <%= f.text_field :title %>
  <textarea id="article_body" name="article[body]" cols="60" rows="12"></textarea>
  <%= f.submit "Create" %>
<% end %>
{% endhighlight %} 
所以对于之前如何添加`<input type="reset"/> `的问题来说，答案已经很清晰了。当然如何解决这个问题并不是主要想讨论的事情，关键的一点是非常多的初学者对`Form_for`的理解误区，他们普遍觉得在`Form_for`里写input是应该像这样的：`<%= f.text_field :title %>`，而不是`<input id="article_title" name="article[title]" type="text"/>`，关键是忽略了`Form_for`主要责任是生成HTML而已。所以特别需要记住的一点是，就算是在Rails里，HTTP的表单请求依然是依赖HTML的`<form></form>`标签，而不是Rails的`Form_for`Helper，而`Form_for`是用于生成`<form></form>`。所以这个问题的产生其实是源自于对HTML本身的不够熟悉。 

## Form_for与Form_tag
好了，那说完以上的问题后，在说说那`Form_for`和`Form_tag`的区别是什么？其实它们的区别就是生成的HTML不一样，而有非常一大部分的初学者会认为他们的区别是一个用于创建和更新用，对于搜索就使用`Form_tag`。初学者的观点也不能说不对，但是如果要了解为什么需要在不同的情况使用不同的Helper还是需要了解一下它们的区别。那它们生成的HTML究竟有什么不一样呢？  
关于`Form_for`的例子，可以看上面提到的那个例子，而现在看看官方文档中对`Form_tag`的描述是怎么样的：  
{% highlight ruby %}
<%= form_tag("/search", method: "get") do %>
  <%= label_tag(:q, "Search for:") %>
  <%= text_field_tag(:q) %>
  <%= submit_tag("Search") %>
<% end %>
{% endhighlight %}
生成如下的HTML：  
{% highlight html %}
<form accept-charset="UTF-8" action="/search" method="get"><div style="margin:0;padding:0;display:inline"><input name="utf8" type="hidden" value="&#x2713;" /></div>
  <label for="q">Search for:</label>
  <input id="q" name="q" type="text" />
  <input name="commit" type="submit" value="Search" />
</form>
{% endhighlight %}
它们最大的区别和最关键的区别就是input中的name值，分别是：`Form_for`是`name="article[body]"`，而`Form_tag`是`name="q"`，而这个name值决定了数据发往服务器时的格式是怎么的：`Form_for`传出去的格式是`{article: {body: value}}`，而`Form_tag`则是`{q: value}`。关于`article`这个名称的定义则来源于`form_for @article`中的`@article`。  
然后清楚了它们的两个的区别后，又会衍生另一个问题，就是，那我用`Form_for`做搜索不就也可以了？答案是：当然可以。但这里会有一个问题，如果你直接这样使用`form_for @search`的话，整个页面就会出错了。原因在于Rails并不知道`@search`是不是一个hash类型，而且并没有定义里面的参数值，所以你这样定义就可以顺利通过了：`@search = {q: ""}`。同理可得，其实`@article = Article.new`就相当是一个给予`article`默认key值和默认value值的过程。所以其实无论是`Form_for`或`Form_tag`都好，Rails无形中都帮我们把整个概念整理的更简洁了。  
所以在创建和更新方面常见的操作为`@article = Article.new(params[:article])`，而在search中则多为`@article = Article.search(name: params[:name])`。
