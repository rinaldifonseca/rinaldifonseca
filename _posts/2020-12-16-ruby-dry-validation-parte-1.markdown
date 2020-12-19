---
layout: post
title:  "Ruby - dry-validation parte 1"
date:   2020-12-18 18:24:22 -0300
comments: true
categories: ruby dry-rb dry-validation
---
Você já pensou que a entrada de dados da sua aplicação normalmente não é tão valorizada?
Aqui, me refiro como entrada de dados as inúmeras maneiras que a aplicação recebe informações.

No ecossistema Ruby, estas entradas poderiam ser: params de um controller(requests de APIs o form objects), sidekiq/sneaker worker, dados de uma rake task ou até mesmo os dados de entrada de um cli(command line interface).

Normalmente, os dados de entrada são tratados apenas com uma validação mínima.
E as demais partes da aplicação que utilizam estes dados são obrigadas a confiar no que está sendo recebido e não possuem maneiras práticas para validar estruturas mais complexas nem mesmo fazer coerção de tipos.

Veja um exemplo prático: Vamos considerar que em um processo de cadastro (um POST por ex.) você precise receber os seguintes dados:

{% highlight ruby %}
{data: {username: "foo", birthdate: "2020-02-02", age: 10, address: {street: "foo", number: "bar"}}}
{% endhighlight %}

E que desejamos validar com as seguintes regras:

- precisa ter uma chave "data"
- age precisa ser um inteiro
- username precisa ser uma string
- birthdate é opcional mas se for preenchido precisa ser uma data válida
- address precisa ter as chaves street e number preenchidos com strings

Se você utilizar um framework como Rails por exemplo, como você validaria todas estas regras?
Utilizando `params.require` e `permit` poderia ajudar com alguns pontos, mas com certeza você vai precisar escrever bastante código para cobrir todas as 5 regras acima. E se a validação precisar ser feita fora de um controller, você teria ainda mais trabalho.

Perceba que as regras acima dizem respeito apenas a dados, formatos e estrutura.
Ou seja, existe um `schema` para ser respeitado. É como se a aplicação "falasse" assim: "Hey, se os dados não respeitarem estas regras básicas, nem iremos começar a falar de regras de negócio".
Regras de negócio nesse contexto poderia ser: verificar se Age é maior que 18 ou o username já existe.

Então, nosso objetivo aqui é ter um mecanismo sólido de verificação que nos permita confiar que os dados de entrada de qualquer fluxo esteja no formato(schema) que esperamos.
E só depois disso que iremos lidar com gravar dados no banco, enviar e-mails, ou fazer qualquer outra etapa do nosso caso de uso.

Como fazer isso na prática? Utilizando a gem [dry-validation][dry-validation-link].
Esta é uma das gems do projeto [dry-rb][dry-rb-link] que como o próprio slogan diz: `is a collection of next-generation Ruby libraries.`

Com a gem `dry-validation` podemos criar a seguinte classe para cuidar da nossa definição de schema e validação de dados:

{% highlight ruby %}
class UserSignupContract < Dry::Validation::Contract
  params do
    required(:data).hash do
      required(:username).filled(:string)
      optional(:birthdate).filled(:date)
      required(:age).filled(:integer)
      required(:address).hash do
        required(:street).filled(:string)
        required(:number).filled(:string)
      end
    end
  end
end
{% endhighlight %}

A elegante DSL "fala por si". Basicamente precisamos criar uma classe que herde de `Dry::Validation::Contract` e definir as regras para cada campo.

Para realizar a verificação, podemos fazer o seguinte:

{% highlight ruby %}
data = {data: {username: "foo", birthdate: "2020-01-01", age: "10", address: {street: "foo", number: "bar"}}}

result = UserSignupContract.new.call(data)
{% endhighlight %}

Estamos criando uma instância da classe `UserSignupContract` e passando o hash data para o método call.
Como resposta, teremos um objeto result com o seguinte:

{% highlight ruby %}
<Dry::Validation::Result{:data=>{:username=>"foo", :birthdate=>Wed, 01 Jan 2020, :age=>10, :address=>{:street=>"foo", :number=>"bar"}}} errors={}>
{% endhighlight %}

Perceba que o birthdate foi convertido para um objeto Date mesmo tendo recebido a data como string. E que age foi convertido para inteiro mesmo tendo recebido a idade como string.

Se fizermos `result.success?` vai retornar true.


Agora vamos validar enviando birthdate e age com formatos inválidos:


{% highlight ruby %}
invalid_data = {data: {username: "foo", birthdate: "2020", age: "age", address: {street: "foo", number: "bar"}}}

result = UserSignupContract.new.call(invalid_data)
{% endhighlight %}

Teremos o seguinte result:

{% highlight ruby %}
<Dry::Validation::Result{:data=>{:username=>"foo", :birthdate=>"2020", :age=>"age", :address=>{:street=>"foo", :number=>"bar"}}} errors={:user=>{:birthdate=>["must be a date"], :age=>["must be an integer"]}}>
{% endhighlight %}

Observe que agora temos uma lista de erros com os respectivos problemas de validação.
No caso, a string "age" não pode ser convertida em um inteiro. E birthdate precisa estar em um formato válido de data.

Se fizermos `result.success?` vai retornar false, e `result.errors.to_h` vai retornar os erros da seguinte forma:

{% highlight ruby %}
{:data=>{:birthdate=>["must be a date"], :age=>["must be an integer"]}}
{% endhighlight %}

Conclusão: Ter um mecanismo sólido para validar a entrada de dados é importante e deixa nossa aplicação muito mais consistente pois garantimos que nossas regras de negócio serão executadas em cima de informações já validadas com um formato que esperamos.

A gem `dry-validation` oferece uma maneira robusta e prática de realizar estas validações e você poderá utilizá-la em praticamente qualquer parte da sua aplicação.

No próximo post irei trazer exemplos de como poderemos criar estruturas para validar regras de negócio e outras variações mais avançadas na definição de Schemas.

Qualquer dúvida ou feedback, comente abaixo =)

[dry-validation-link]: https://dry-rb.org/gems/dry-validation/
[dry-rb-link]: https://dry-rb.org
