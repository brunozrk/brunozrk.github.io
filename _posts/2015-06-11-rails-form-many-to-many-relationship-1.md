---
layout: post
title:  "Rails form many to many relationship [1]"
date:   2015-06-11 21:00:00
categories: complex-form
comments: true
---

Nesse post vou mostrar como criar um formulário para uma funcionalidade que tem um relacionamento MxN. Confira como deve ficar no final [https://rails-complex-forms.herokuapp.com/sales/new](https://rails-complex-forms.herokuapp.com/sales/new).

Para o exemplo, vamos imaginar a seguinte situação:

* Precisamos criar um CRUD de vendas, onde um vendedor vende vários produtos, e o mesmo produto pode ser vendido por vários vendedores. Também queremos guardar a quantidade de produtos vendidos por cada venda. ( MxN ! )
* Para esse exemplo, vamos considerar um página de "Venda". Teremos Produtos e Vendedores previamente cadastrados, nossa tela de inserção terá duas listas para escolher o produto e o vendedor, e um campo para inserir a quantidade de produtos vendidos.

## Tabelas

Vamos criar as `migrations` para nossa funcionalidade. No terminal, execute:

{% highlight Bash shell scripts %}
rails g migration create_seller
rails g migration create_product
rails g migration create_sale
{% endhighlight %}

Edite os arquivos da seguinte maneira:

{% highlight ruby %}
# db/migrate/..._create_seller.rb
class CreateSeller < ActiveRecord::Migration
  def change
    create_table :sellers do |t|
      t.string :name
    end
  end
end

# db/migrate/..._create_product.rb
class CreateProduct < ActiveRecord::Migration
  def change
    create_table :products do |t|
      t.string :name
    end
  end
end

# db/migrate/..._create_sale.rb
class CreateSale < ActiveRecord::Migration
  def change
    create_table :sales do |t|
      t.integer :quantity
      t.references :seller, :product
    end
  end
end

{% endhighlight %}

Execute as migrations:

{% highlight Bash shell scripts %}
rake db:migrate
{% endhighlight %}

A tabela `sale` tem a referência das tabelas `product` e `seller`, ela também nos permite salvar a quantidade de produtos vendidos por venda.

## Models

Agora vamos criar os `models` para as tabelas que criamos:

Primeiro o model `Product`

{% highlight ruby %}
# app/models/product.rb
class Product < ActiveRecord::Base
  has_many :sales
  has_many :sellers, through: :sales
end
{% endhighlight %}

Aqui dizemos que um `Produto` possui vários `Vendedores(sellers)` através das `Vendas(sales)`.
Com isso, podemos buscar todos os vendedores de um produto (`product.sellers`) ou todas as vendas (`product.sales`).

O model `Seller` tem a mesma ideia:

{% highlight ruby %}
# app/models/seller.rb
class Seller < ActiveRecord::Base
  has_many :sales
  has_many :products, through: :sales
end
{% endhighlight %}

E por fim, o model `Sale`, com algumas validações.

{% highlight ruby %}
# app/models/sale.rb
class Sale < ActiveRecord::Base
  belongs_to :seller
  belongs_to :product

  validates :seller_id, :product_id, :quantity, presence: true
end
{% endhighlight %}

## View

Antes, vamos adicionar a seguinte linha no arquivo de rotas (`config/routes.rb`)

{% highlight ruby %}
resources :sales
{% endhighlight %}

Vamos criar nosso formulário (Estou usando `simple_form` e `bootstrap 3`, mas isso não interfere em nada drasticamente)

O formulário fica assim:

{% highlight erb %}
<!-- app/views/sales/_form.html.erb-->
<%= simple_form_for @sale, wrapper: :vertical_form do |f|%>
  <%= f.error_notification %>

  <%= f.input :seller_id, collection: Seller.all %>
  <%= f.input :product_id, collection: Product.all %>
  <%= f.input :quantity %>

  <%= f.button :submit %>
  <%= link_to 'Voltar', sales_path %>
<% end %>

{% endhighlight %}

## Controller

Nossas actions no controller ficam assim, nada muito especial:

{% highlight ruby %}

def create
  if @sale.update_attributes(sale_params)
    redirect_to sales_path
  else
    render 'new'
  end
end

def update
  if @sale.update_attributes(sale_params)
    redirect_to sales_path
  else
    render 'edit'
  end
end

private

def sale_params
  params.require(:sale).permit(:seller_id, :product_id, :quantity)
end

{% endhighlight %}

Adicione no `seeds.rb` alguns registros para poder fazer o teste:

{% highlight ruby %}
Seller.create(name: 'seller 1')
Seller.create(name: 'seller 2')

Product.create(name: 'product 1')
Product.create(name: 'product 2')
{% endhighlight %}

No terminal, execute:

{% highlight Bash shell scripts %}
rake db:seed
{% endhighlight %}

Eeeeeeeeeeee pronto!

No próximo post vou mostrar como criar um CRUD MxN inserindo valores dinamicamente.

Abraço!

Confira o código completo [aqui](https://github.com/brunozrk/rails_complex_forms).
