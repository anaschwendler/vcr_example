# Usando VCR para simular requisições

Nos últimos meses, tenho trabalhado com muitas requisições externas de API. Como faço parte da equipe de logística da Marley Spoon, nosso desafio é garantir que nossas requisições sejam integradas a tempo e que tudo seja entregue da melhor forma possível.

Parte dessa tarefa, como desenvolvedora de software, é garantir que o código que foi escrito seja bem testado e faça o que deve ser feito, de forma correta e eficiente. Nosso projeto tem muitas integrações com parceiros de logística. Quando fazemos uma integração esses outros sistemas, não queremos realizar requisições reais toda vez que executamos nossos testes. Essa ação não só gera apenas requisições desnecessárias, mas também pode gerar problemas com limites de requisições com API externas além de outros efeitos colaterais inconvenientes, como testes tomando mais tempo para rodar. Para evitar isso, fingimos fazer requisições reais e há uma ferramenta que ajuda a simular essas chamadas: ao registrar dados reais de uma requisição real, somos capazes de fazer o esboço(stub) de uma resposta de serviço web externo. O arquivo salvo é chamados de "fita cassete (cassette tape)" e o nome da ferramenta a que me refiro é [VCR](https://github.com/vcr/vcr).

Neste post vou explicar como tenho aprendido a usá-lo e por que acho que é útil.

![](https://media.giphy.com/media/xT1R9RfuBqWvfo8oDe/giphy.gif)

## O que é uma simulação de requisição de teste?

Como este artigo não é sobre a técnica de simulação de requisições, vou apenas explicar brevemente e adicionar um link para um artigo útil sobre o assunto. [Neste artigo](https://circleci.com/blog/how-to-test-software-part-i-mocking-stubbing-and-contract-testing/)(em inglês) podemos observar que simulação de testes de requisição pode ser definido como:

> Simular testes de requisição significa uma versão simulada de um serviço externo ou interno que pode substituir o real, ajudando seus testes a serem executados de forma mais rápida e confiável. Quando sua implementação interage com as propriedades de um objeto, em vez de sua função ou comportamento, uma imitação pode ser usada.

… e é exatamente para isso que usamos o VCR.

## O que é VCR?

De acordo com a [documentação](https://github.com/vcr/vcr):

> Grave as interações HTTP do seu conjunto de testes e reproduza-as durante futuras execuções de teste para testes rápidos, determinísticos e precisos.

Que significa que, quando você executar o teste pela primeira vez com a sintaxe do VCR, ele registrará o que aconteceu e, na próxima vez que você executar o teste novamente, terá a versão gravada pronta para substituir a solicitação real.

## Como usar o VCR?

Eu tenho um exemplo básico aqui [here](https://github.com/anaschwendler/vcr_example).

Basicamente, sempre que você quiser testar uma parte do código que requer uma requisição externa, você usa o método `.use_cassette` para expor que você deseja que o VCR lide com isso como uma fita cassete. Se já houver dados com uma interação HTTP pré-gravada, o VCR o usará. Se não, o VCR **cria automáticamente** uma "fita cassete" (dessa vez fazendo uma requisição real) baseado na requisição feita no teste.

Mas ainda assim, se o que você quer realizar requisições reais todas as vezes que seus testes rodam, VCR também oferece diferente modos de gravação, que podem ser conferidos [aqui](https://relishapp.com/vcr/vcr/v/6-0-0/docs/record-modes)

Exemplo:

```ruby
require 'rubygems'
require 'test/unit'
require 'vcr'

VCR.configure do |config|
  config.cassette_library_dir = "fixtures/vcr_cassettes"
  config.hook_into :webmock
end

class VCRTest < Test::Unit::TestCase
  def test_example_dot_com
    VCR.use_cassette("synopsis") do
      response = Net::HTTP.get_response(URI('http://www.iana.org/domains/reserved'))
      assert_match /Example domains/, response.body
    end
  end
end
```

Aqui estamos requisitando o uso de uma fita chamada `synopsis`, que simula a requisição para a [iana.org](http://www.iana.org/domains/reserved).

Se o teste estiver rodando pela primeira vez, VCR irá gerar um arquivo, armazenado em `fixtures/vcr_cassettes`, que nesse exemplo se chamará `synopsis.yml`.

Após rodar o teste, o arquivo `fixtures/vcr_cassettes/synopsis.yml` se parecerá com (eu removi algumas partes por que o arquivo é grande):

```
---
http_interactions:
- request:
    method: get
    uri: http://www.iana.org/domains/reserved
    body:
      encoding: US-ASCII
      string: ''
    headers:
      Accept-Encoding:
      - gzip;q=1.0,deflate;q=0.6,identity;q=0.3
      Accept:
      - "*/*"
      User-Agent:
      - Ruby
  response:
    status:
      code: 200
      message: OK
    headers:
      Date:
      - Fri, 18 Jun 2021 08:06:40 GMT
      Server:
      - Apache
      Vary:
      - Accept-Encoding
      Last-Modified:
      - Thu, 21 May 2020 22:41:39 GMT
      X-Frame-Options:
      - SAMEORIGIN
      Expires:
      - Fri, 18 Jun 2021 09:56:45 GMT
      Referrer-Policy:
      - origin-when-cross-origin
      X-Content-Type-Options:
      - nosniff
      Age:
      - '595'
      Content-Type:
      - text/html; charset=UTF-8
      Cache-Control:
      - public, max-age=21603
      Content-Security-Policy: /*cropped security policy*/
      Transfer-Encoding:
      - chunked
    body:
      encoding: ASCII-8BIT
      string: !binary |-
        /*cropped binary string*/
    http_version:
  recorded_at: Fri, 18 Jun 2021 08:06:40 GMT
recorded_with: VCR 5.0.0
```

Então, a partir de agora sua suíte de testes usará o arquivo salvo. Isso ainda vai fazer seus testes rodarem mais rápido, pois agora temos o arquivo armazenado e não precisamos fazer a requisição real a aplicação externa, o que sempre pode tomar muito mais tempo.

## Exemplos da vida real

Um exemplo de projeto que usa VCR é esse cliente `site-search-ruby` da elastic: https://github.com/elastic/site-search-ruby

No contexto de `Search`, quando procuramos todos os `DocumentTypes` no mecanismo, eles usam um arquivo chamado `engine_search` para simular uma requisição:

https://github.com/elastic/site-search-ruby/blob/master/spec/client_spec.rb

Esse é o arquivo usado nesta instância: https://github.com/elastic/site-search-ruby/blob/master/spec/fixtures/vcr/engine_search.yml

## Conclusão

VCR parece ser uma ferramenta bastante confiável para simular requisições para API externas. Mas nós ainda precisamos estar conscientes de que API podem sempre mudar, e quando isso acontece, precisamos gravar nossos testes novamente. Esse processo deve ser simples e fácil de reproduzir, então para mais ficar sobre o uso do VCR, eu recomendo [esse](https://fabioperrella.github.io/10_tips_to_help_using_the_VCR_gem_in_your_ruby_test_suite.html) artigo.

### Escondendo credenciais confidenciais em vcr_cassettes

Há uma opção de configuração disponível para filtrar dados sensíveis que podem ser usados para evitar que sejam gravados nos arquivos de cassete, a documentação está [aqui](https://relishapp.com/vcr/vcr/v/5-0-0/docs/configuration/filter-sensitive-data). Isso é importante, por que não queremos credenciais secretas expostas publicamente nos nossos repositórios armazenados na internet.

## References
https://github.com/vcr/vcr

https://github.com/elastic/site-search-ruby

https://circleci.com/blog/how-to-test-software-part-i-mocking-stubbing-and-contract-testing/

https://fabioperrella.github.io/10_tips_to_help_using_the_VCR_gem_in_your_ruby_test_suite.html
