+++
title = 'Não use Extension Types para exceções'
date = 2024-08-28T13:38:13-03:00
draft = true
+++

**_Aviso_: Este artigo não é uma introdução à Extension Types**. É esperado ter um conhecimento básico sobre esta funcionalidade para ter um melhor entendimento do artigo. [Documentação oficial](https://dart.dev/language/extension-types).

Extension Types, na minha opinião, são uma das melhores adições à linguagem Dart que já tivemos. Fica tão mais fácil para criar tipos simples que ajudam a deixar seu sistema mais robusto e livre de bugs, trocando a dor de cabeça de horas de depuração por algumas linhas de código.

Porém, não são perfeitos. O maior problema deles são como podem infiltrar silenciosamente no seu código em lugares onde eles não deveriam estar, e causar bugs que você pode achar sem sentido na primeira vez que encontra eles.

Essa frase pode parecer meio sem sentido, "infiltrar silenciosamente no código em lugares onde não deveriam estar". O que isso significa?

Extension Types **existem apenas em tempo de compilação**, ou seja, operações de _runtime_ passam despercebidas quando são feitas com Extension Types. Se você tentar fazer uma comparação de tipo com um Extension Type, o resultado pode ser o contrário do que você esperava:

```dart
extension type Id(int id) {}

void main() {
  final list = [0, Id(1), 2, Id(3)];

  final intList = list.whereType<int>().toList();

  print(intList); // [0, 1, 2, 3]
}
```

Neste trecho de código, dá a entender que os tipos `Id` seriam filtrados da lista, restando apenas os valores 0 e 2. Porém, como `whereType` realiza uma operação no _tipo_ do objeto, temos este resultado inesperado.

No final da documentação oficial de Extension Types, fala como **não é uma abstração segura**, e como nós podemos pegar o valor interno que o Extension Type abstrai. Porém, é difícil diferenciar o tempo de compilação ao tempo de execução do código. Isso fica muito mais aparente quando nós trabalhamos com exceções.

Vamos ver estas exceções construídas utilizando Extension Types:

```dart
extension type const NoInternetConectionException._(() _value)
    implements Object {
  const NoInternetConectionException() : _value = ();
}

typedef ApiError = ({int statusCode, String errorMessage});

extension type const ApiErrorException._(ApiError _apiError) implements Object {
  const ApiErrorException({
    required int statusCode,
    required String errorMessage,
  }) : _apiError = (statusCode: statusCode, errorMessage: errorMessage);

  int get statusCode => _apiError.statusCode;
  int get message => _apiError.statusCode;
}

extension type const ParseException(Map<String, dynamic> json)
    implements Object {}
```

Isso aqui são exceções para chamadas de Api, e conseguimos ver claramente o que cada uma representa. Além disso, nós usaríamos ela da mesma maneira que uma classe de exceção normal:

```dart
Future<User> getUserById(int id) async {
  if (!hasInternetConnection) {
    throw NoInternetConectionException();
  }

  try {
    final response = await httpClient.get('/users/$id');

    final data = response.data;

    try {
      return User.fromJson(data);
    } catch (_) {
      throw ParseException(data);
    }
  } on HttpClientException catch (e) {
    throw ApiErrorException(
      statusCode: e.statusCode,
      errorMessage: e.message,
    );
  }
}
```

Isso aqui parece uma função bem simples e, especialmente, normal. Você lendo ela não imagina que tem algum problema muito grande, que pode causar um bug infernal e fazer você perder seu fim de semana tentando corrigir.

Como Extension Types não existem em tempo de execução, e uma operação envolvendo seu tipo é feita com o tipo interno. Ou seja, ao escrever a expressão `throw ParseDataException(data);`, você está escrevendo apenas `throw data;`.

Da mesma maneira, a cláusula `catch` é comparada com o valor interno, então, quando você escreve `on ParseException catch (e)`, na verdade você está escrevendo `on Map<String, dynamic> catch (e)`.

Isso se torna um grande problema quando começamos a usar exceções que compartilham o mesmo tipo interno:

```dart
extension type Exception1(String message) implements Object {}

extension type Exception2(String message) implements Object {}

void main(List<String> args) {
  try {
    throw Exception2('Exception 2 message');
  } on Exception1 catch (e) {
    print('Caught Exception1. Message: ${e.message}'); // Caught Exception1. Message: Exception 2 message
  } on Exception2 catch (e) {
    print('Caught Exception 2. Message: ${e.message}'); // Doesn't get executed
  }
}
```

Nosso Extension Type `Exception2` é capturado pela primeira cláusula `catch`, que está tentando apenas capturar exceções do tipo `Exception 1`. Porém, como o tipo interno da `Exception1` e `Exception2` são o mesmo (`String`), a checagem é feita através dele.

Então, ao lançar a exceção `Exception2`, nós caímos na cláusula da `Exception1`, e, além de seu `print` ser executado, a cláusula de `catch`da `Exception2` não é executada.

O maior problema disso é: parece código normal. Nós não temos nenhum aviso, nenhum _lint_ para nos avisar que o que estamos fazendo é perigoso. Além disso, se eu não te contasse que os tipos `Exception1` e `Exception2` não são classes, você diria que este comportamento é completamente sem sentido, provavelmente pensando que eu tenho péssima lógica de programação, sou maluco, talvez ambos.

Mas funciona assim, e este é o comportamento desejado de Extension Types. Como eu disse, Extension Types são uma das minhas adições favoritas à linguagem, porém, é necessário saber onde são os melhores lugares para seu uso. Por isso, **não use Extension Types para exceções**.
