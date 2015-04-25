---
layout: post
title:  "Rails form one to many relation"
date:   2015-04-25 14:35:15
categories: complex-form
comments: true
---

Nesse post vou mostrar como criar um formulário para uma funcionalidade que tem um relacionamento 1xN. Confira como deve ficar no final [https://rails-complex-forms.herokuapp.com/people/new](https://rails-complex-forms.herokuapp.com/people/new).

Para o exemplo, vamos imaginar a seguinte situação:

* Precisamos criar um CRUD de usuários e cada usuário pode ter vários endereços ( 1xN ! )

## Tabelas

Vamos criar as `migrations` para nossa funcionalidade. No terminal, execute:

{% highlight Bash shell scripts %}
rails g migration create_person
rails g migration create_addresses
{% endhighlight %}

Edite os arquivos da seguinte maneira:

{% highlight ruby %}
# db/migrate/..._create_person.rb
class CreatePerson < ActiveRecord::Migration
  def change
    create_table :people do |t|
      t.string :name
    end
  end
end

# db/migrate/..._create_addresses.rb
class CreateAddresses < ActiveRecord::Migration
  def change
    create_table :addresses do |t|
      t.integer :number
      t.string :street
      t.references :person
    end
  end
end
{% endhighlight %}

Execute as migrations:

{% highlight Bash shell scripts %}
rake db:migrate
{% endhighlight %}

Nada muito especial aqui, criamos uma tabela `people` com o atributo nome e uma tabela `addresses` com os atributos number, street e person (faz referência a tabela `people`)

## Models

Agora vamos criar os `models` para as tabelas que criamos:

Primeiro o model `Person`

{% highlight ruby %}
# app/models/person.rb
class Person < ActiveRecord::Base
  has_many :addresses, dependent: :destroy
  accepts_nested_attributes_for :addresses, allow_destroy: true

  validates :name, presence: true
end
{% endhighlight %}

Aqui dizemos que um `Person` possui vários `Addresses`, que é o que queremos.
O `dependent: :destroy` irá remover todas os endereços relacionados quando apagarmos um `Person`, sem isso o banco fica com registros perdidos.

O `accepts_nested_attributes_for` cria um método chamado `addresses_attributes`, que iremos usar no formulário para criar os endereços.

Agora o model `Address`

{% highlight ruby %}
# app/models/address.rb
class Address < ActiveRecord::Base
  belongs_to :person

  validates :street, :number, presence: true
end
{% endhighlight %}

Aqui dizemos que um `Address` pertence a um `Person`.

Mapeados nosso relacionamento 1xN, vamos criar nossa view.

## View

Antes, vamos adicionar a seguinte linha no arquivo de rotas (`config/routes.rb`)

{% highlight ruby %}
resources :people
{% endhighlight %}

Agora adicione a gem [cocoon](https://github.com/nathanvda/cocoon) no `Gemfile` e execute `bundle install`. Ela nos ajudará a adicionar dinamicamente endereços no formulário.

No arquivo `application.js` adicione a linha:

{% highlight js %}
//= require cocoon
{% endhighlight %}

Vamos criar nosso formulário (Estou usando `simple_form` e `bootstrap 3`, mas isso não interfere em nada drasticamente)

O formulário fica assim:

{% highlight erb %}
<!-- app/views/people/_form.html.erb-->
<%= simple_form_for @person, wrapper: :vertical_form do |f|%>
  <%= f.error_notification %>

  <%= f.input :name %>

  <%= content_tag(:div, id: 'addresses') do %>
    <%= f.simple_fields_for :addresses do |addresses_form| %>
      <%= render 'address_fields', f: addresses_form %>
    <% end %>
    <%= link_to_add_association 'add address', f, :addresses, class: 'btn btn-primary' %>
  <% end %>

  <%= f.button :submit %>
  <%= link_to 'Voltar', people_path %>
<% end %>

<!-- app/views/people/_address_fields.html.erb-->
<%= content_tag(:div, class: "col-md-12 nested-fields") do %>
  <%= f.input :street, wrapper_html: { class: 'col-xs-3' }  %>
  <%= f.input :number, wrapper_html: { class: 'col-xs-3' } %>
  <%= link_to_remove_association '', f, class: 'glyphicon glyphicon-remove' %>
<% end %>

{% endhighlight %}

Aqui temos um formulário para criação de um `Person` com a possibilidade de adicionar endereços.

Para o model `Person` temos apenas um campo `:name`.

Para os atributos de `Address` temos um `simple_fields_for :addresses` (esse `:addresses` tem que ser o mesmo que adicionamos no `accepts_nested_attributes_for` la no model `Person`, lembra? ).

Estamos renderizando os campos de `Address` em uma partial para a gem `cocoon` conseguir criar outros endereços quando clicar no link `link_to_add_association`.

O parâmetro `:addresses` da função `link_to_add_association` é o id da div que criamos (`content_tag(:div, id: 'addresses')`)

Na partial temos os campos de endereço: `:street` e `:number`, e um link para remover o endereço `link_to_remove_association`

Para o `link_to_remove_association` funcionar corretamente, la no nosso model `Person` adicionamos no `accepts_nested_attributes_for` a opção `allow_destroy: true`. Ao clicar para remover, o `cocoon` vai esconder o elemento que tiver a classe `nested-fields` (por isso adicionamos essa clase na div da nossa partial) e adicionar um campo hidden com a opção de `_destroy`, quando salvarmos o registro ele vai excluir automáticamente. No html fica assim:

{% highlight html %}
<input type="hidden" name="person[addresses_attributes][0][_destroy]" id="person_addresses_attributes_0__destroy" value="1">
{% endhighlight %}

Para entender como as coisas vão para o controller, quando dermos submit no formulários, o `params` será algo como:

No create:

{% highlight ruby %}
"person"=>
  {
    "name"=>"Bruno Zeraik", 
    "addresses_attributes"=>
      {
        "1429996522431"=> {"street"=>"rua xyz", "number"=>"123", "_destroy"=>"false"}
      }
  }
{% endhighlight %}

No update:
{% highlight ruby %}
"person"=>
  {
    "name"=>"Bruno Zeraik",
    "addresses_attributes"=>
      {
        "1429996814586"=>{"street"=>"Rua nova", "number"=>"123", "_destroy"=>"false"},
        "0"=>{"street"=>"rua xyz", "number"=>"123", "_destroy"=>"1", "id"=>"10"}
      }
  }
{% endhighlight %}

Repare que no update estamos apagando um endereço (`"_destroy"=>"false"`) e inserindo outro (`"_destroy"=>"1"`).

## Controller

Nossas actions no controller ficam assim, nada muito especial:

{% highlight ruby %}

def create
  @person = Person.new(person_params)
  if @person.save
    redirect_to people_path
  else
    render 'new'
  end
end

def update
  binding.pry
  @person = Person.find(params[:id])
  if @person.update_attributes(person_params)
    redirect_to people_path
  else
    render 'edit'
  end
end

private

def person_params
  params.require(:person).permit(
    :name, 
    addresses_attributes: [:id, :number, :street, :_destroy]
  )
end

{% endhighlight %}

Confira o código completo [aqui](https://github.com/brunozrk/rails_complex_forms).
