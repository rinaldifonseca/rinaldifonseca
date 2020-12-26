---
layout: post
title:  "Ruby - dry-validation tutorial parte 2"
date:   2020-12-26 13:24:22 -0300
comments: true
categories: ruby dry-rb dry-validation
---
No post anterior, falei sobre a importância da validação de dados e apresentei a gem dry-validation como uma solução extremamente interessante. Se você ainda não leu, [corre aqui para conferir][dry-validation-parte1-link].

Agora, irei aprofundar em mais detalhes e mostrar como podemos resolver vários problemas de uma forma bem simples e elegante.

Vamos começar: Quando definimos um contrato de validação, temos duas partes principais:

- Definição de Schemas e pré-processamento dos dados: é a parte responsável por definir a estrutura que os dados deverão ter, verificar os tipos e também aplicar as coerções(transformações) dos dados. 
  Também podemos usar os chamados predicados para verificar lógicas simples, como por exemplo garantir que uma string tenha um tamanho mínimo.
- Definição de regras de validações (High-level Rules): é a parte responsável por definir regras mais elaboradas e de preferência regras de negócio. É aqui que iremos definir que uma idade deveria ser maior que 21 anos por exemplo. É importante destacar que as regras de validação só serão executadas caso os dados sejam validados pelo Schema que definimos antes.

Veja esse exemplo completo:

{% highlight ruby %}
class UserSignupContract < Dry::Validation::Contract
  params do
    required(:age).filled(:integer)
    required(:country).filled(:string, min_size?: 2, max_size?: 3)
    optional(:nickname).maybe(:string)
    required(:scores).filled(:array, size?: 3).each(:filled?, :int?, gt?: 0)
    required(:slug).filled(format?: /^[a-z0-9-]+$/)
    required(:role).filled(included_in?: ['guest', 'admin'])
  end

  rule(:age) do
    if values[:country] == 'US'
      key.failure('must be greater than 21') if value < 21
    end
  end
end
{% endhighlight %}

Agora vamos testar usando dados inválidos:


{% highlight ruby %}
data = { country: "US", nickname: "jowHOO", role: "none", scores: [1,2,0], slug: "my_slug", age: 10}

UserSignupContract.new.call(data).errors.to_h

{% endhighlight %}

Com isso teremos a seguinte lista de erros conforme o esperado:


{% highlight ruby %}
{:scores=>{2=>["must be greater than 0"]}, :slug=>["is in invalid format"], :role=>["must be one of: guest, admin"], :age=>["must be greater than 21"]}
{% endhighlight %}

Agora, vamos começar explicação mais aprodundada com a parte: `required(:age).filled(:integer)`

Na gem dry-validation, o `filled` é uma macro. Ou seja, por "de baixo dos panos" esse código vai ser transformato em: `required(:age) { int? & filled? }`
Inclusive, esse código poderia ser usado assim mesmo ao invés de usar a macro filled.

`Int?` e `filled?` são o que chamamos de predicados. Basicamente são métodos que retornam true ou false.
E toda a ideia de composição de regras da gem funciona com base neles.

Vejamos a parte: `required(:country).filled(:string, min_size?: 2, max_size?: 3)`

Nela, estamos usando os predicados `min_size?` e `max_size?` para determinar que a string deverá ter seu size entre 2 e 3.

`optional(:nickname).maybe(:string)`
Como vimos no post anterior, o optional significa que a chave nickname é opcional. A `macro maybe` significa que o valor da chave  nickname poderá não ser preenchido.

`required(:scores).filled(:array, size?: 3).each(:filled?, :int?, gt?: 0)`
Esta construção é interessante! Indica que scores precisa ser um array onde cada elemento precisa ser preenchido com um inteiro maior que 0. (gt significa greater than)

`required(:slug).filled(format?: /^[a-z0-9-]+$/)`
O `format?` Indica o formato que a o valor deverá serguir. E indicamos o formato com uma regex.

`required(:role).filled(included_in?: ['guest', 'admin'])`
Por fim, o `included_in?` indica que o valor deverá estar incluso na lista especificada.

Como você pode perceber, os predicados e as macros nos dão flexibilidade para validar praticamente qualquer tipo de estrutura de dados. Você pode conferir a lista completa [aqui][predicate-logic-link] e [aqui][macros-link]

E se ainda não for suficiente, você poderá [criar seus próprios predicados][custom-predicates].

Agora, passemos para a parte 2, regras de validação.

O objetivo das regras (rules) é conseguir expressar validações mais completas que normalmente estão ligadas a regras de negócio. Elas só serão executadas se todas as regras do bloco `params` forem verificadas.

No exemplo acima, nosso objetivo é garantir que a idade (age) seja maior que 21 anos apenas quando o país (country) seja os Estados Unidos (US).

Para isso, usamos a construção rule(:age) informando o nome da chave que queremos aplicar a regra e passamos um bloco que terá a implementação da regra propriamente dita.

Dentro do bloco, temos acesso a `values` que contem um hash com todos os valores do dado de entrada.
Também temos `value` que é exatamente o valor da chave em questão, que nesse caso é :age.
Por fim, podemos usar `key.failure` para apontar a falha caso a nossa condição não seja satisfeita.

No próximo post, irei mostrar o uso das validações em uma aplicação Rails.

Qualquer dúvida ou feedback, comente abaixo =)

[dry-validation-parte1-link]: https://rinaldifonseca.github.io/ruby/dry-rb/dry-validation/2020/12/18/ruby-dry-validation-parte-1.html
[predicate-logic-link]: https://dry-rb.org/gems/dry-validation/0.13/basics/predicate-logic/
[macros-link]: https://dry-rb.org/gems/dry-validation/0.13/basics/macros/
[custom-predicates]: https://dry-rb.org/gems/dry-validation/0.13/custom-predicates/
