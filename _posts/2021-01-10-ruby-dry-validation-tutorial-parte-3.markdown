---
layout: post
title:  "Ruby - dry-validation tutorial parte 3"
date:   2021-01-10 13:24:22 -0300
comments: true
categories: ruby dry-rb dry-validation
---
Chegamos na parte 3! Objetivo agora é mostrar como integrar o dry-validation com o Rails e mostrar um exemplo prático.

Se você ainda não leu os posts anteriores, [na parte 1][dry-validation-parte1-link] eu falo sobre a importância da validação de dados e apresentei a gem `dry-validation` como uma solução extremamente interessante. Na [parte 2][dry-validation-parte2-link] eu mostro mais exemplos e opções mais avançadas que permitem validar praticamente qualquer tipo de entrada.

Como falamos anteriormente, a gem dry-validation é independente e funciona em qualquer aplicação Ruby. Mas dado que iremos dar um exemplo em Rails, podemos usar a gem dry-rails e com isso ganharmos algumas facilidades e conveniências. Neste post vou focar na primeira funcionalidade chamada `safe_params`.

Inicie adicionando a gem `dry-rails` ao seu Gemfile:

{% highlight ruby %}
gem 'dry-rails'
{% endhighlight %}

Em seguida, crie um initializer em `config/initializers/system.rb` com o seguinte conteúdo:

{% highlight ruby %}
Dry::Rails.container do
   #cherry-pick features
   config.features = %i[safe_params]
end
{% endhighlight %}

Basicamente, estamos configurando a funcionalidade safe_params que falaremos a seguir.

Agora crie um controller com o seguinte conteúdo:

{% highlight ruby %}
class SignUpController < ApplicationController
  skip_forgery_protection

  before_action :validate

  schema(:create) do
    required(:name).filled
    required(:email).filled
    required(:interests).filled(:array, size?: 2).each(:filled?, :str?)
  end

  def create
    head 201
  end

  private

  def validate
    render json: safe_params.errors.to_h, status: 422 if safe_params.failure?
  end
end
{% endhighlight %}

Nos posts anteriores, nós definimos os Schemas em classes que herdavam de `Dry::Validation::Contract`.
Com a gem dry-rails, ganhamos a funcionalidade de definir estes mesmos Schemas dentro dos controllers. No exemplo anterior, estamos definindo um Schema para a action create. Dado que as actions dos controllers são as portas de entrada para a nossa aplicação, fica extremamente poderemos já definirmos estas regras aqui.

O método validate que está listado no before_action é o responsável em aplicar a validação e nos permitir tomar alguma decisão caso a validação falhe.

Quando chamamos o `safe_params` dentro desse método, é retornado um objeto do tipo `Dry::Schema::Result` que representa o resultado da validação. Por baixo dos panos, quando a request chega no controller, o Schema definido para a action create é validado pegando como entrada os `params` da request. Então é possível fazer `safe_params.failure?` para saber se algo falhou na validação.

E caso tenha falhado, `safe_params.errors.to_h` vai retornar um hash com os erros.

Se você trabalha em alguma API, esse tipo de validação pode te ajudar bastante. Pense que após o Schema ser validado, você ficará tranquilo em fazer as demais operações (criar registros no banco, enviar e-mails etc) da sua aplicação pois sabe que está lidando com dados válidos. A redução de bugs e comportamentos inesperados que você já barra na entrada é gigante. 

Qualquer dúvida ou feedback, comente abaixo =)

[dry-validation-parte1-link]: https://rinaldifonseca.github.io/ruby/dry-rb/dry-validation/2020/12/18/ruby-dry-validation-parte-1.html
[dry-validation-parte2-link]: https://rinaldifonseca.github.io/ruby/dry-rb/dry-validation/2020/12/26/ruby-dry-validation-tutorial-parte-2.html
