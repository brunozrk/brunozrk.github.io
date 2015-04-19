---
layout: post
title:  "Configurando Code Climate integrado ao Travis"
date:   2015-05-19 14:35:15
categories: codeclimate
comments: true
---

O [Code Climate](https://codeclimate.com/) é usado para ser um "*code-reviewer*" automatizado do seu código, ele vai olhar coisas como qualidade, segurança, estilo e cobertura de testes do seu código.

Qualidade, segurança e estilo o Code Climate faz por si só. Para a cobertura de testes precisamos configurar um CI, nesse exemplo vamos utilizar o [Travis-CI](https://travis-ci.org/).

O Code Climate é compatível Ruby, JavaScript, PHP e Python. Nesse exemplo vou mostrar como configurar para projetos ruby.

Só lembrando que tanto o Code Climate quanto o Travis são serviços pagos, porém são free para projetos *open source*.

## Code Climate

Para configurar o Code Climate no seu repositório, primeiro você precisa se cadastrar e efetuar o login no [site](https://codeclimate.com/) (é possível utilizar a conta do GitHub).

Feito isso, clique em *Add Open Source Repo* e adicione o seu projeto.

Pronto, o Code Climate vai começar a analisar seu código e dentro de instantes você poderá ver a pontuação, que varia entre zero e quatro.

Se sua nota não for a máxima (quatro), você pode clicar no gráfico gerado para entender o motivo da pontuação perdida e melhorar seu código de acordo com a analise feita pelo Code Climate.

### Badge

Agora vamos adicionar o badge no arquivo `README.md` do seu repositório.

Vá em `settings` do repositório que acabamos de adicionar no Code Climate e clique na aba `badges`.

Copie o que está escrito em `Markdown` e adicione no seu arquivo `README.md`, deve ficar algo parecido com isso:

![]({{site.url}}/assets/images/2015-04-17-codeclimate/code-climate-badge.png)

## Travis

Para usar o travis você também precisa se cadastrar, e também é possível fazer isso com o usuário do GitHub!

Feito isso, vá para seu [profile](https://travis-ci.org/profile/). Nele estarão listados todos os seus repositórios da conta do GitHub que você usou para logar.

1 - Ative o mesmo repositório que você adicionou no Code Climate.

2 - Adicione o arquivo `travis.yml` no repositório do seu código. É nele que ficam as configurações do ambiente do Travis. Para esse exemplo vamos deixar o básico, você pode conferir mais informações nesse [link](http://docs.travis-ci.com/user/getting-started/).

{% highlight YAML %}
language: ruby
rvm:
  - 2.2.1
{% endhighlight %}

3 - Faça um push!

{% highlight ruby %}
git add travis.yml
git commit -m 'configure travis'
git push origin master
{% endhighlight %}

Agora o Travis vai fazer o build do seu projeto e tentar rodar os teste, se tudo estiver ok, no final do log você vai ver algo como "*Done. Your build exited with 0.*"

### Badge

Vamos adicionar o status do build no `README.md`.

Com seu repositório aberto no Travis, clique no `badge` ao lado do nome do seu repositório e selecione a opção `markdown`, copie o texto e coloque no seu `README.md`, igual fizemos com o code climate, agora ficamos com isso:

![]({{site.url}}/assets/images/2015-04-17-codeclimate/build-passing-badge.png)

## Test Coverage

Ok, estamos quase acabando, agora vamos integrar o Travis com o Code Climate!

Primeiro vamos preparar o projeto para enviar os dados para o Code Climate:

1 - No arquivo `Gemfile`, adicione:

{% highlight ruby %}
gem "codeclimate-test-reporter", group: :test, require: nil
{% endhighlight %}

2 - No `spec_helper.rb` ou `test_helper.rb`, adicione:

{% highlight ruby %}
require "codeclimate-test-reporter"
CodeClimate::TestReporter.start
{% endhighlight %}

3 - No repositório do seu projeto no Code Climate, vá em `settings -> Test Coverage`. Nele você vai encontrar um token (CODECLIMATE_REPO_TOKEN) que deverá ser utilizado pelo travis, vamos configurar logo mais, então guarde esse token!

Agora vamos configurar o travis para quando ele executar os testes, enviar a cobertura para o Code Climate. Edite o arquivo `travis.yml` adicionando o seguinte:

{% highlight YAML %}
addons:
  code_climate:
    repo_token: abcdefg123123...
{% endhighlight %}

O valor do `repo_token` deve ser o token que pegamos anteriormente (CODECLIMATE_REPO_TOKEN).

Não é legal manter esse valores expostos no código, para evitar isso, vamos criptografar nosso token, para isso execute os passos:

1 - Instale a gem do travis para ter acesso a alguns comandos importantes para essa etapa:

{% highlight Bash shell scripts %}
gem install travis
{% endhighlight %}

2 - Dentro do seu repositório, execute:

{% highlight Bash shell scripts %}
travis login
{% endhighlight %}

3 - Criptografe seu token!

{% highlight Bash shell scripts %}
travis encrypt <TOKEN>
{% endhighlight %}

O retorno será algo como:

{% highlight Bash shell scripts %}
secure: "NSdlTlGvw..."
{% endhighlight %}

4 - Vamos editar o arquivo `travis.yml` novamente, agora colocando um token seguro:

{% highlight YAML %}
addons:
  code_climate:
    repo_token:
      secure: "NSdlTlGvw..."
{% endhighlight %}

Pronto! Feito isso, faça o *push* dessas alterações que o Travis irá detectar e começará a executar o build. Ao finalizar, ele vai enviar a cobertura de testes para o Code Climate.

### Badge

Pra fechar, vamos adicionar o `badge` da cobertura de testes no arquivo `README.md`

No repositório do codeclimate vá em `settings -> Badges`

Copie o texto que está no markdown do label *Test Coverage Badge* e adicione no `README.md`:

![]({{site.url}}/assets/images/2015-04-17-codeclimate/coverage-badge.png)

## Conclusão

Agora temos 3 informações essenciais sobre o código em um mesmo lugar, a cada commit o Travis e o Code Climate irão analisar o código novo e esses `badges` podem ter seus valores alterados!
