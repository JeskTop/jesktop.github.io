---
layout: post
title: 使用Form Objects处理结构复杂的表单
date: 2015-02-08 19:28:40
disqus: y
---

参考：[RailsCasts 416-form-objects](http://railscasts.com/episodes/416-form-objects)   
自从使用Form Objects处理复杂的表单后，我就不想在考虑使用Rails带的那套方法了。  
原来的方法是，如果一个Form需要同时处理两个Model的数据时，就需要考虑使用`accepts_nested_attributes_for`，来建立数据间的关系。但是如果同时需要处理三个或三个以上的Model数据时，就会变得很混乱，form页面需要各种嵌套，然后model层需要小心翼翼的设置`accepts_nested_attributes_for`。  
但是自从学会了Form Objects后，妈妈在也不担心我写复杂的表单了。  

## 模拟场景
模拟一个现实中的场景，我们需要一个Form处理三个model的事情，这里先介绍即将登场的三位Model：

{% highlight ruby %}
Class Product  # 产品
  has_many :product_prices
  belongs_to :product_color
end
{% endhighlight %}

{% highlight ruby %}
Class ProductPrice  # 产品价格（多种价格）
  belongs_to :product
  enum category: { sell: 0, cost: 1} 
end
{% endhighlight %}

{% highlight ruby %}
Class ProductColor  # 产品颜色
  has_many :products
end
{% endhighlight %}

![](https://ruby-china-files.b0.upaiyun.com/photo/2015/bf917af736056bfd93c59e6904f941d5.png)

Form里的操作是，当我创建一个Product时，需要同时添加多个Product Price（销售价和成本价），和关联一个Product Color（如果颜色存在，则直接进行关联，不存在就重新创建）。  
所以我们需要设计四个input：name, sell_price, cost_price, color；别看就这么简单的四个玩意，其实是同时涉及三张表的操作哦。

## Form的设计
这里谈的设计不是指怎么美化这个表单，这里我就不谈如何美化表单的事情了，简单的使用simple_form来处理这个表单把，接招：  
{% highlight html %}
= simple_form_for @product, url: products_path, method: :post, html: {class: 'form-horizontal'} do |f| 
  = f.input :name
  = f.input :sell_price
  = f.input :cost_price
  = f.input :color

  .form-actions
    = f.button :submit, class: 'btn btn-primary', value: "提交"
{% endhighlight %}

请先不要着急着去看这个form究竟长什么样了，如果你跑去看，估计就是满满的错误，因为@product还没定义呢！  
当然有同鞋们会说，@product不就是`@product = Product.new`吗？当然不是！！因为Product没有sell_price等字段啊！

## 建立Form Objects
好了，主菜来了！首先我们在`app/forms/`里建一个`products_create_form.rb`文件，然后咱们来看看应该如何完善。

### 使用ActiveModel::Model

{% highlight ruby %}
Class ProductsCreateForm
  include ActiveModel::Model
  
  def self.model_name
    ActiveModel::Name.new(self, nil, "Product")
  end
  
  class << self
    def i18n_scope
      :activerecord
    end
  end
end
{% endhighlight %}

为什么在这里要`include ActiveModel::Model`呢？因为这样可以使用大部分我们在Model中常用到的方法，这里我们主要要使用`validates`去验证必填项。  
然后另外`self.model_name`这里把ActiveModel::Name设置为'Product`了，那生成的form中input的name就是`product[name]`。  
而`i18n_scope`则是让你把对于model来说的i18n设置放回activerecord中，不然要设置Form Objects下的i18n就需要针对ActiveModel来写i18n信息，这个就主要看个人需求是怎么样，一般我都会加上。

### 完善Form Objects
然后我们需要补全剩余的信息了：  

{% highlight ruby %}
Class ProductsCreateForm
  include ActiveModel::Model
  
  def self.model_name
    ActiveModel::Name.new(self, nil, "Product")
  end
  
  class << self
    def i18n_scope
      :activerecord
    end
  end
  
  attr_accessor :name, :sell_price, :cost_price, :color

  validates :name, :sell_price, :cost_price, :color, presence: true

  def initialize
  end

  def submit(params)
    self.name = params[:name]
    self.sell_price = params[:sell_price]
    self.cost_price = params[:cost_price]
    self.color = params[:color]
    if valid?
      product_color = ProductColor.where(name: self.color).first_or_create
      product = Product.create(name: self.name, product_color: product_color)
      ProductPrice.create(category: 'sell', value: self.sell_price, product: product)
      ProductPrice.create(category: 'cost', value: self.cost_price, product: product)
      true
    else
      false
    end
  end
end
{% endhighlight %}

在attr_accessor中加入我们需要输入的参数，并且把必填项加入到validates中。在submit时，valid?方法就会生效了。这一个create的流程下来，还是非常简单的。

### 调用 ProductsCreateForm
所以在刚刚@product中，调用ProductsCreateForm就可以了：  

{% highlight ruby %}
@product = ProductsCreateForm.new
{% endhighlight %}

而在create方法时，也是非常的简单的：

{% highlight ruby %}
@product = ProductsCreateForm.new
if @product.submit(params[:product])
  ...
else
  ...
end
{% endhighlight %}

整个流程下来，controller的代码已经非常简单了，把需要用的逻辑单独放在ProductsCreateForm中，不需要放到Model中，避免Model的臃肿，而且非常直观。   
那么问题来了，如果我是需要Update这些信息，那么直接调用ProductsCreateForm可以吗？显然那是不行的，Update的话会比较复杂些。

## 使用Form Objects更新信息
我们重新建一个`ProductUpdateForm`来专门处理Update：  

{% highlight ruby %}
Class ProductsUpdateForm
  include ActiveModel::Model

  def self.model_name
    ActiveModel::Name.new(self, nil, "Product")
  end

  class << self
    def i18n_scope
      :activerecord
    end
  end

  validates :name, :sell_price, :cost_price, :color, presence: true

  def initialize product
    @product = product
    @product_sell_price = @product.product_price.find_by(category: ProductPrice::categories[:sell])
    @product_cost_price = @product.product_price.find_by(category: ProductPrice::categories[:cost])
  end

  def name
    @name ||= @product.name
  end

  def sell_price
    @sell_price ||= @product_sell_price.value
  end

  def cost_price
    @cost_price ||= @product_cost_price.value
  end

  def color
    @color ||= @product.product_color.name
  end

  def update(params)
    @name = params[:name]
    @sell_price = params[:sell_price]
    @cost_price = params[:cost_price]
    @color = params[:color]
    if valid?
      @product.update(name: self.name)
      @product_sell_price.update(value: self.sell_price)
      @product_cost_price.update(value: self.cost_price)
      @product.product_color.update(name: self.color)
      true
    else
      false
    end
  end
end
{% endhighlight %}

相比Create来说，主要不同的地方也就是每个参数的定义方式不在使用`attr_accessor`，因为参数已经不会为空的，而需要从数据库中读取。然后别的相比之下差别也不大，我这里就不另外做解释了，如果还想了解多一些，可以看看RailsCasts的视频，非常不错的哦。
