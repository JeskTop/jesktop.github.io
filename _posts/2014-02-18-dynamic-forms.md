---
layout: post
title: 如何创建动态表单
date: 2014-02-18 13:38:21
disqus: y
---

介绍
=============

参考RailsCasts上的[403-dynamic-forms](http://railscasts.com/episodes/403-dynamic-forms)完成了一个动态表单，说一说整个过程。

动态表单是指用户可以自定义表单需要的内容，然后自己建立表单，让用户进行填写来收集自己需要收集的信息。典型的例子是“制作在线问卷调查”，例如：[金数据](https://www.jinshuju.net/)

根据RailsCasts上的流程，我根据视频上学到的内容使用`Rails4.03`和`Ruby2.1`完整的完成了一遍，对于表`(Product和ProductType)`的CRUD操作均通过脚手架添加所以就不多贴代码，大概说说流程和思路。

添加Model
=============
添加整个动态表单需要用到的model，这里使用Product做介绍，需要3张表：

*   Product`(字段：name:string，price:decimal，properties:text，product_type_id:integer)`
*   ProductType`(字段：name:string)`
*   ProductField`(字段：name:string，field_type:string，required:boolean，product_type_id:integer)`

`Product`是产品表，`ProductType`是动态表单的类型表（用户可以定义多种表单类型），`ProductField`是动态表单包含的信息（表单内需要填写的信息）。

`ProductType`和`Porduct`为一对一关系，而和`ProductField`则为一对多（一个表单内足以包含一种或更多的需要填写的信息内容）

`ProductField`的字段分别存入名称`(name)`、输入框的类型`(field_type)`和是否为必填字段`(required)`。

而动态表单的数据则存入`Product`的`properties`字段中，使用了Rails提供了serialize（序列化）存入数据表中，只需要在`Product`里添加的`serialize :properties, Hash`即可把`properties`作为hash存入数据库中。

```
虽然序列化很方便可以让你储存任意的物件，但是缺点是序列化资料就失去了透过资料库查询索引的功效，你无法在SQL的where条件中指定序列化后的资料。
```

创建ProductType时添加ProductField
=============
ProductType Model设置
-------------
我们需要在创建`ProductType`的同时添加`ProductField`，需要设置`ProductField`可通过`ProductType`进行添加，`ProductType`的Model代码如下：

    class ProductType < ActiveRecord::Base
      has_many :fields, class_name: "ProductField"
      accepts_nested_attributes_for :fields, allow_destroy: true
    end

通过`accepts_nested_attributes_for`进行设置，可以点击查看文档[accepts_nested_attributes_for](http://api.rubyonrails.org/classes/ActiveRecord/NestedAttributes/ClassMethods.html#method-i-accepts_nested_attributes_for)。

ProductType View设置
-------------
因为在创建`ProductType`时添加`ProductField`，所以我们需要在`ProductType`的form中添加嵌入式的表单，并且需要动态添加操作，因为对于每个`ProductType`来说`ProductField`的数量是未知的，所以这里需要使用coffeescript在`ProductType`的`_form.html.erb`中进行异步的增加和删除。整体操作分为3步:

1.  在`_form`中添加创建`ProductField`的表单

    新建`_field_fields.html.erb`，加入`ProductField`需要添加的信息：
    
        <fieldset>
          <%= f.select :field_type, %w[text_field check_box] %>  <!-- 提供添加"text_field"和"check_box"两种选择 -->
          <%= f.text_field :name, placeholder: "field_name" %>
          <%= f.check_box :required %> <%= f.label :required %>
          <%= f.hidden_field :_destroy %>     <!-- accepts_nested_attributes_for的删除操作需要提供"_destroy"  ->
          <%= link_to "[remove]", "#", class: "remove_fields" %>
        </fieldset>

    在`_form.html.erb`中加入`_field_fields.html.erb`
    
        <%= form_for(@product_type) do |f| %>
          ...
          <%= f.fields_for :fields do |builder| %>
            <%= render 'field_fields', f: builder %>
          <% end %>
          ...
        <% end %>
        
2.  添加动态添加和删除`_field_fields`的操作：

    添加一个helper进行动态操作：
    
        def link_to_add_fields(name, f, association)
          new_object = f.object.send(association).klass.new     #相当于ProductField.new
          id = new_object.object_id
          fields = f.fields_for(association, new_object, child_index: id) do |builder|  
            render(association.to_s.singularize + "_fields", f: builder)
          end
          link_to(name, '#', class: "add_fields", data: {id: id, fields: fields.gsub("\n", "")})  #把需要添加的html加入该链接的data-fields中
        end
        
    [fields_for文档](http://api.rubyonrails.org/classes/ActionView/Helpers/FormHelper.html#method-i-fields_for)
    
    在`_form.html.erb`添加`link_to_add_fields`如：
    
        <%= link_to_add_fields "Add Field", f, :fields %>  
        
    使用coffeescript对`ProductField`进行异步的添加和删除操作：
    
        $(document).on 'click', 'form .remove_fields', (event) ->    #把_destroy设置为1，并且隐藏fieldset
          $(this).prev('input[type=hidden]').val('1')
          $(this).closest('fieldset').hide()
          event.preventDefault()
        
        $(document).on 'click', 'form .add_fields', (event) ->     #把data-fields的信息，插入form中
          time = new Date().getTime() 
          regexp = new RegExp($(this).data('id'), 'g')
          $(this).before($(this).data('fields').replace(regexp, time))
          event.preventDefault()
          
3.  controller中加入嵌套表单的参数保护（白名单）：

        def product_type_params
          params.require(:product_type).permit(:name).tap do |whitelisted|
            whitelisted[:fields_attributes] = params[:product_type][:fields_attributes]
          end
        end
        
在Product使用ProductType
=============
