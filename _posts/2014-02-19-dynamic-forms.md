---
layout: post
title: 如何创建动态表单
date: 2014-02-19 13:38:21
disqus: y
---

- - -
#介绍

参考RailsCasts上的[403-dynamic-forms](http://railscasts.com/episodes/403-dynamic-forms)完成了一个动态表单，说一说整个过程。

动态表单是指用户可以自定义表单需要的内容，然后自己建立表单，让用户进行填写来收集自己需要收集的信息。典型的例子是“制作在线问卷调查”，例如：[金数据](https://www.jinshuju.net/)

根据RailsCasts上的流程，我根据视频上学到的内容使用`Rails4.03`和`Ruby2.1`完整的完成了一遍，对于表(Product和ProductType)的CRUD操作均通过脚手架添加所以就不多贴代码，大概说说流程和思路。

- - -
#添加Model

添加整个动态表单需要用到的model，这里使用Product做介绍，需要3张表：

*   Product (字段：name:string，price:decimal，properties:text，product_type_id:integer)
*   ProductType (字段：name:string)
*   ProductField (字段：name:string，field_type:string，required:boolean，product_type_id:integer)

Product是产品表，ProductType是动态表单的类型表（用户可以定义多种表单类型），ProductField是动态表单包含的信息（表单内需要填写的信息）。

ProductType和Porduct为一对一关系，而和ProductField则为一对多（一个表单内足以包含一种或更多的需要填写的信息内容）

ProductField的字段分别存入名称(name)、输入框的类型(field_type)和是否为必填字段(required)。

而动态表单的数据则存入Product的properties字段中，使用了Rails提供了serialize（序列化）存入数据表中，只需要在Product里添加的`serialize :properties, Hash`即可把properties作为hash存入数据库中。

> 虽然序列化很方便可以让你储存任意的物件，但是缺点是序列化资料就失去了透过资料库查询索引的功效，你无法在SQL的where条件中指定序列化后的资料。

- - -
#创建ProductType时添加ProductField

##ProductType Model设置

我们需要在创建ProductType的同时添加ProductField，需要设置ProductField可通过ProductType进行添加，ProductType的Model代码如下：

{% highlight ruby %}
class ProductType < ActiveRecord::Base
  has_many :fields, class_name: "ProductField"
  accepts_nested_attributes_for :fields, allow_destroy: true
end
{% endhighlight %}

通过`accepts_nested_attributes_for`进行设置，可以点击查看文档[accepts_nested_attributes_for](http://api.rubyonrails.org/classes/ActiveRecord/NestedAttributes/ClassMethods.html#method-i-accepts_nested_attributes_for)。

##ProductType View设置

因为在创建ProductType时添加ProductField，所以我们需要在ProductType的form中添加嵌入式的表单，并且需要动态添加操作，因为对于每个ProductType来说ProductField的数量是未知的，所以这里需要使用coffeescript在ProductType的`_form.html.erb`中进行异步的增加和删除。整体操作分为3步:

*   在`_form`中添加创建ProductField的表单

  新建`_field_fields.html.erb`，加入ProductField需要添加的信息：
    
{% highlight ERB %}
<fieldset>
  <%= f.select :field_type, %w[text_field check_box] %>  <!-- 提供添加"text_field"和"check_box"两种选择 -->
  <%= f.text_field :name, placeholder: "field_name" %>
  <%= f.check_box :required %> <%= f.label :required %>
  <%= f.hidden_field :_destroy %>     <!-- accepts_nested_attributes_for的删除操作需要提供"_destroy" -->
  <%= link_to "[remove]", "#", class: "remove_fields" %>
</fieldset>
{% endhighlight %}

  在`_form.html.erb`中加入`_field_fields.html.erb`：
    
{% highlight ERB %}
<%= form_for(@product_type) do |f| %>
  ...
  <%= f.fields_for :fields do |builder| %>
    <%= render 'field_fields', f: builder %>
  <% end %>
  ...
<% end %>
{% endhighlight %}
        
*   添加动态添加和删除_field_fields的操作：

  添加一个helper进行动态操作：
    
{% highlight ruby %}
def link_to_add_fields(name, f, association)
  new_object = f.object.send(association).klass.new     #相当于ProductField.new
  id = new_object.object_id
  fields = f.fields_for(association, new_object, child_index: id) do |builder|  
    render(association.to_s.singularize + "_fields", f: builder)
  end
  link_to(name, '#', class: "add_fields", data: {id: id, fields: fields.gsub("\n", "")})  #把需要添加的html加入该链接的data-fields中
end
{% endhighlight %}
        
  [fields_for文档](http://api.rubyonrails.org/classes/ActionView/Helpers/FormHelper.html#method-i-fields_for)
    
  在`_form.html.erb`添加link_to_add_fields如：
  
{% highlight ERB %}
<%= link_to_add_fields "Add Field", f, :fields %>  
{% endhighlight %}
        
  使用coffeescript对ProductField进行异步的添加和删除操作：
    
{% highlight coffeescript %}
$(document).on 'click', 'form .remove_fields', (event) ->    #把_destroy设置为1，并且隐藏fieldset
  $(this).prev('input[type=hidden]').val('1')
  $(this).closest('fieldset').hide()
  event.preventDefault()

$(document).on 'click', 'form .add_fields', (event) ->     #把data-fields的信息，插入form中
  time = new Date().getTime() 
  regexp = new RegExp($(this).data('id'), 'g')
  $(this).before($(this).data('fields').replace(regexp, time))
  event.preventDefault()
{% endhighlight %}
          
*   controller中加入嵌套表单的参数保护（白名单）：

{% highlight ruby %}
def product_type_params
  params.require(:product_type).permit(:name).tap do |whitelisted|
    whitelisted[:fields_attributes] = params[:product_type][:fields_attributes]
  end
end
{% endhighlight %}
 
- - -       
#在Product使用ProductType

##创建Product时，选择ProductType类型
在创建Product的form中，可以根据ProductType的id进行生成form的类型，所以添加一个`form_tag`：

{% highlight ERB %}
<%= form_tag new_product_path, method: :get do %>
  <%= select_tag :product_type_id, options_from_collection_for_select(ProductType.all, :id, :name) %>
  <%= submit_tag "New Product" %>
<% end %>
{% endhighlight %}
 
##在Product的_form中，根据选择的ProductType生成表单
在`_form.html.erb`中添加：

{% highlight ERB %}
<%= f.fields_for :properties, OpenStruct.new(@product.properties) do |builder| %>
  <% @product.product_type.fields.each do |field| %>
    <%= render "products/fields/#{field.field_type}", field: field, f: builder %>
  <% end %>
<% end %>
{% endhighlight %}

创建`products/fields/_check_box.html.erb`：

{% highlight ERB %}
<div class="field">
  <%= f.check_box field.name %>
  <%= f.label field.name %>
</div>
{% endhighlight %}

创建`products/fields/_text_field.html.erb`：

{% highlight ERB %}
<div class="field">
  <%= f.label field.name %><br />
  <%= f.text_field field.name %>
</div>
{% endhighlight %}

> About OpenStruct   
> person = OpenStruct.new('name' => 'John Smith', 'age' => 70)   
> person[:age] = 42 # => equivalent to ostruct.age = 42   
> person.age # => 42

##Product保存过程中，同时保存product_type_id

在controller中，new方法下先建立product和product_type的关系：

{% highlight ruby %}
def new
  @product = Product.new(product_type_id: params[:product_type_id])
end
{% endhighlight %}

并且hidden在form中，create时一同创建：

{% highlight ERB %}
<%= f.hidden_field :product_type_id %>
{% endhighlight %}

##controller中加入嵌套表单的参数保护（白名单）

{% highlight ruby %}
def product_params
  params.require(:product).permit(:name, :price, :product_type_id).tap do |whitelisted|
    whitelisted[:properties] = params[:product][:properties]
  end
end
{% endhighlight %}

##在Product的show页面中，显示product动态表里的信息：

{% highlight ERB %}
<% @product.properties.each do |name, value| %>
  <p>
    <b><%= name.humanize %>:</b>
    <%= value %>
  </p>
<% end %>
{% endhighlight %}

- - -       
#ProductType的必填项在Product form中生效

剩下最后一步，当ProductType的`required`字段设置为true，则该项为必填字段，所以在新建Product的时候，应该对称进行限制，只需要添加一条validate即可：


{% highlight ruby %}
class Product < ActiveRecord::Base
  ...

  validate :validate_properties
  
  def validate_properties
    product_type.fields.each do |field|
      if field.required? && properties[field.name].blank?
        errors.add field.name, "must not be blank"
      end
    end
  end
end
{% endhighlight %}

- - -  
#总结

经过以上步骤，基本就完成了整个动态表单的创建。整个过程可以购买RailsCasts Pro进行观看，链接可以点击：[403-dynamic-forms](http://railscasts.com/episodes/403-dynamic-forms)。

根据RailsCasts的说明完成后，其实发现还是有很多需要改进的地方的，其中有下面两点需要注意：

*   上面方式创建表单，尚不支持上传文件的功能
*   用Rails提供的hash形式保存数据，最大的弊端是一个搜索的问题，用Ruby进行过滤要比SQL的查询效率差很多
