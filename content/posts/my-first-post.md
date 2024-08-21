+++
title = 'Extension Types e o Newtype Pattern em Dart'
date = 2024-08-21T11:45:12-03:00
draft = true
+++

Quando eu falo Newtype, o que você pensa? Parece algo muito complicado, mas é bem simples. Você literalmente usa todo dia sem pensar sobre, pois é algo que já faz sentido pra você, desde o começo da sua jornada como programador. Mas, quando a gente coloca no papel essa técnica e fala sobre isso, parece algo de outro mundo.

Ok, mas o que é Newtype? Apenas um wrapper em algum outro tipo que permite que você estenda sua funcionalidade. Simples, né?

Tá bom, eu não te culpo se você não entendeu. Quando eu coloco dessa maneira fica estranho mesmo. Vamos ver um exemplo pra entender o que eu estou falando:

```dart
List<int> format(int value) {
 //
}
```

Você já usou esta função centenas de vezes. Eu não estou brincando. Eu tenho certeza que você já usou esta função. Vamos ver ela com uma perspectiva diferente:

```dart
String format(DateTime value) {
 //
}
```

Melhor agora, né? Uma função de formatação de data. Mas por que eu troquei a `String` por `List<int>` e  o `DateTime` por uma `int`?

Mas aí é que está: eu não troquei nada. Debaixo dos panos, uma `String` é uma `List<int>`,  e o `DateTime` é uma `int`.

Na classe `DateTime`, temos o [getter `milisecondsSinceEpoch`](https://github.com/dart-lang/sdk/blob/b9479eb440de7af2c9946931a1ecaabf457b31af/sdk/lib/core/date_time.dart#L740), que retorna um `int`, e em sua implementação nós temos o [campo `_value`](https://github.com/dart-lang/sdk/blob/b9479eb440de7af2c9946931a1ecaabf457b31af/sdk/lib/_internal/vm_shared/lib/date_patch.dart#L33), o valor deste `DateTime`.

E, com `String`, é ainda mais simples. A primeira frase da [documentação oficial de `String` no Dart](https://api.flutter.dev/flutter/dart-core/String-class.html) é sobre a `String` ser uma sequência de "UTF-16 code units". Pra pegar esses code units que compõe a `String`, é só chamar o getter `codeUnits`, que retorna uma `List<int>`:

```dart
void main(List<String> args) {
 print('Hello World'.codeUnits); // [72, 101, 108, 108, 111, 32, 87, 111, 114, 108, 100]
}
```

Isto é um Newtype. A `String` envolve uma `List<int>` e estende funcionalidade à este tipo, da mesma maneira que `DateTime` faz com `int`.

Adicionar funcionalidade à um tipo mais primitivo é apenas a ponta do iceberg das vantagens do Newtype.

Você também pode ter percebido que, com este pattern, não conseguimos criar valores ilegais para uma `String` ou para um `DateTime`. Mas o que são valores ilegais?

Como vimos, uma String tem o formato UTF-16. Isto significa que os valores válidos para uma String no Dart são entre 0 e 1114111, inclusivo. Então, vamos tentar construir uma String com um valor inválido, e ver o que acontece:

```dart
void main(List<String> args) {
 print(String.fromCharCode(1114112)); // Throws RangeError: Invalid value: Not in inclusive range 0..1114111: 1114112
}
```

O nosso código falha, e não nos deixa construir uma String quebrada, e **isso é ótimo!** Nós não queremos uma String inválida rondando no nosso código, esperando o pior momento pra fazer um erro indecifrável.

Da mesma maneira, não podemos criar um DateTime errado:

```dart
void main(List<String> args) {
 print(DateTime.parse('Not a date')); // Throws FormatException: Invalid date format
}
```

Além de adicionar uma camada de funcionalidade para os tipos primitivos, nós também adicionamos uma camada de **validação**. Não é possível criar algum valor inválido para seus tipos, um dos pontos mais importantes do Newtype.

"Mas, -", você, desenvolvedor Flutter, me diz, "eu não estou criando uma linguagem de programação. Eu faço telinha! Por que eu iria me preocupar como uma validação de DateTime é feita?". Se você pensou isso, parabéns, você me deixou triste.

Vamos ver uma função que você provavelmente tem na sua base de código:

```dart
Future<void> signUp(String email, String password) async {
 if (!isEmailValid(email)) {
  throw InvalidEmailException();
 }

 if (!isPasswordValid(password)) {
  throw InvalidPasswordException();
 }

 logger.info('Creating user: $email');

 try {
  await repository.signUpUser(email, password);
 } catch (e) {
  throw SignUpApiException(e);
 }

 try {
  await cache.saveUser(email, password);
 } catch (e) {
  throw CacheException(e);
 }
}
```

Você gosta disso? Você olha pra essa função e fala "Sim, esta é a função mais bonita do mundo!"? Se você pensa isso, não tem nenhum problema. Ela faz validação de cada input e, caso algum input esteja inválido, ou alguma operação falhar, solta uma exceção sobre o que deu errado. Como implementação, ela está sólida.

Mas, imagine quem vai chamar esta função, como ele terá que trabalhar:

```dart
try {
 final String email = controller.email;
 final String password = controller.password;

 await singUp(password, email);
} on InvalidEmailException {
 // handle email exception
} on SignUpApiException {
 // handle sign up api exception
} on CacheException {
 // handle cache exception
}
```

Podemos ver os problemas: a pessoa passou a senha no lugar do e-mail, o e-mail no lugar da senha e ela esqueceu de tratar o erro de senha inválida.

Mas, você me diz: "Esta função que eu estou fazendo é pra mim, então esses erros não vão acontecer!", e sim, se é um código privado, onde você, e, *com toda certeza do mundo, apenas você*, vai ver este código, então não tem problema. O problema vem quando estamos fazendo código para outras pessoas, seja num contexto corporativo ou open-source, onde este tipo de código mostra seus problemas, pois, daqui a 5, 10 20 anos, alguém vai cometer esses erros, e daí é uma bomba relógio no código, pronta para estourar e fazer alguns pobres desenvolvedores passarem o feriado corrigindo isso.

E nem falamos do pior: está vendo como temos um *log* do e-mail? Neste problema que acabamos de ver, vamos *logar* a senha do usuário. Está vendo como este código é frágil?

O Newtype pattern é uma ótima ferramenta para impedir que isto aconteça. Para isso, vamos criar um Newtype próprio.

### Classe ou Extension Type

Vamos criar umas regras para os nossos Newtypes:

- Eles precisam ser **válidos**, nós não podemos construir um Newtype com valor interno inválido;
- Ser **imutável**, caso precisamos alterar o valor de um Newtype, então criamos outro;
- Ter **igualdade** com outro Newtype construído a partir do mesmo valor;
- Ser **livre de *side-effects*** em sua construção.

Nós temos duas maneiras de criar um Newtype: usando classes, ou usando extension types.

> **Aviso:**  Este artigo não é uma introdução a extension types, e é altamente desejável que você já tenha um entendimento do que eles são. Caso não tenha, recomendo:
>
> - [A documentação oficial do Dart em Extension Types](https://dart.dev/language/extension-types);
> - [Este vídeo do Professor Diego Antunes explicando certinho o que são Extension Types](https://www.youtube.com/watch?v=2TJIOpBDMnU).

#### Usando classes

Antes de continuar, vou conversar sobre um pattern que já vi sendo usado, **e altamente aconselho contra o uso**. Eu não sei se ele possuí um nome oficial, então resolvi chamar de "**Build First, Validate Later**".

Como funciona: nós construímos nosso Newtype **sem validação**, e depois checamos para ver se é um Newtype válido:

```dart
class Email {
 String value;  

 Email(this.value);

 bool isValid() => _isEmailValid(value);
}

void main(List<String> args) {
 final email = Email('invalid email');
}
```

O problema desta abordagem é óbvio: a validação não é mais obrigatória, e sim opcional. Quando nós construímos software robusto que precisa durar, e, na minha opinião, isto seria considerado um anti-pattern.

Além disso, ele viola o primeiro dos nossos pontos: **Ele precisa ser válido**. Da mesma maneira que não queremos que nossa String tenha algum valor que não seja UTF-16 válido, também não queremos que nosso Email tenha um valor interno inválido.

Vamos ver uma maneira de melhorar a construção do nosso Newtype Email:

```dart
import 'package:email_validator/email_validator.dart';

class Email {
 String value;

 Email(this.value) : assert(EmailValidator.validate(value));
}
```

Está melhorando! Agora, ao construir um e-mail, nós chamamos a função `validate` do package [email_validator](https://pub.dev/packages/email_validator) e, usando o nosso `assert`, garantimos que o e-mail passado é válido.

Porém, ainda temos problemas. Primeiramente, o `assert` só vai ser chamado no modo `debug` do seu projeto, ou seja, quando seu projeto estiver em modo `release`, e seu usuário final realizar o cadastro, não terá validação de e-mail. Segundo, caso o `assert` falhar, nós não teremos um `InvalidEmailException`, e sim um `AssertionError`.

Então, vamos melhorar:

```dart
import 'package:email_validator/email_validator.dart';

class Email {
 String value;

 Email(this.value) {
  if (!EmailValidator.validate(_value)) {
   throw InvalidEmailException();
  }
 }
}

class InvalidEmailException implements Exception {}
```

Agora sim, esta implementação resolveu esses dois erros: O bloco do construtor será executado tanto em `debug` quanto `release`, e, caso seja inválido, soltará um `InvalidEmailException`.

Porém, se você está acostumado com Dart, você provavelmente tem um problema com sua implementação. Quando nós estamos convertendo um valor em Dart, normalmente de uma String, para algum outro tipo, temos dois construtores: o `parse` e o `tryParse`:

```dart
void main(List<String> args) {
 final bool parse = bool.parse('true');
 final int? tryParse = int.tryParse('10');
}
```

E esses dois métodos já explicam o que pode acontecer: eles podem **falhar**. No caso do parse, caso falhe, solta alguma exceção, normalmente um `FormatException`, enquanto o `tryParse` retorna um `null` quando o parseamento falha.

Vamos implementar isso no nosso Newtype:

```dart
import 'package:email_validator/email_validator.dart';

class Email {
 String value;

 Email._(this.value);

 factory Email.parse(String source) {
  if (!EmailValidator.validate(source)) {
   throw InvalidEmailException();
  }

  return Email._(source);
 }
 
 static Email? tryParse(String source) {
  if (!EmailValidator.validate(source)) {
   return null;
  }

  return Email._(source);
 }
}

class InvalidEmailException implements Exception {}
```

Perfeito! A construção do nosso Newtype está perfeita! Nós expomos dois métodos que quem chama a função pode usar para construir um e-mail, com a validação sendo feita em ambos, dando a opção de escolher qual é a melhor dependendo do caso. Além disso, os detalhes de implementação estão escondidos. Quem chama a função `Email.parse` não faz ideia como a validação do e-mail está sendo feita, muito menos que estamos usando um package externo para isso, e ela nem precisa!

Vamos ver como nosso Newtype está se adequando as regras que estabelecemos anteriormente:

- Eles precisam ser **válidos**, nós não podemos construir um Newtype com valor interno inválido; ✅
- Ser **imutável**, caso precisamos alterar o valor de um Newtype, então criamos outro; ❌
- Ter **igualdade** com outro Newtype construído a partir do mesmo valor; ❌
- Ser **livre de *side-effects*** em sua construção. ✅

Conseguimos metade até agora! Nosso construtor só permite que Newtypes válidos sejam construídos, além de se manter livre de *side effects*. Vamos ver a parte de **imutabilidade**.

Vemos que nosso valor interior, `value`, tem um grande problema: ele é mutável. Com nossa implementação atual, nós podemos alterar o valor interno para algum valor inválido **após** sua construção.

```dart
void main(List<String> args) {
 final Email email = Email.parse('valid@email.com');
 email.value = 'invalid email';
}
```

Para ajustar isto, nós podemos fazer uma simples mudança: deixar `value` como `final`.

```dart
class Email {
 final String value;

 const Email._(this.value);

 //
}
```

Com isso, mudar o valor original é um erro de compilação, além de conseguir deixar a classe Email como `const`.

Ótimo. Para finalizar, nós precisamos implementar a **igualdade**. Essa parte é fácil, só precisamos dar `override` no `operator ==` e `hashcode`:

```dart
@override
bool operator ==(covariant Email other) {
 if (identical(this, other)) return true;

 return other.value == value;
}

@override
int get hashCode => value.hashCode;
```

Agora, nosso Newtype está seguindo todas as regras que estabelecemos:

- Eles precisam ser **válidos**, nós não podemos construir um Newtype com valor interno inválido; ✅
- Ser **imutável**, caso precisamos alterar o valor de um Newtype, então criamos outro; ✅
- Ter **igualdade** com outro Newtype construído a partir do mesmo valor; ✅
- Ser **livre de *side-effects*** em sua construção. ✅

Vamos ver como nossa implementação final ficou de nossa classe:

```dart
import 'package:email_validator/email_validator.dart';

class Email {
 final String value;

 const Email._(this.value);

 factory Email.parse(String source) {
  if (!EmailValidator.validate(source)) {
   throw InvalidEmailException();
  }

  return Email._(source);
 }

 static Email? tryParse(String source) {
  if (!EmailValidator.validate(source)) {
   return null;
  }

  return Email._(source);
 }

 @override
 bool operator ==(covariant Email other) {
  if (identical(this, other)) return true;

  return other.value == value;
 }

 @override
 int get hashCode => value.hashCode;
}

class InvalidEmailException implements Exception {}

void main(List<String> args) {
 final Email email1 = Email.parse('valid@email.com'); // Builds successfully
 final Email email2 = Email.parse('invalid email'); // Throws InvalidEmailException
 final Email? email3 = Email.tryParse('valid@email.com'); // Builds successfully
 final Email? email4 = Email.tryParse('invalid email'); // Returns null
}
```

Bastante coisa, não acha? Sim, um dos meus principais problemas com este método é a quantidade de boilerplate que precisa ser escrito para seguir essas regras. Além disso, temos o overhead que uma classe tem, então caso a gente tenha uma situação onde precisamos instanciar milhares de Newtypes ao mesmo tempo, podemos ter uma penalidade de performance.

#### Usando Extension Types

Usando Extension Types, a definição do nosso Newtype fica muito mais simples:

```dart
import 'package:email_validator/email_validator.dart';

extension type const Email._(String _value) {
 factory Email.parse(String source) {
  if (!EmailValidator.validate(source)) {
   throw InvalidEmailException();
  }

  return Email._(source);
 }

 static Email? tryParse(String source) {
  if (!EmailValidator.validate(source)) {
   return null;
  }

  return Email._(source);
 }
}

class InvalidEmailException implements Exception {}
```

É só isso. Simples assim. Não precisa dar `override` em nada. Além de ganhar o benefício de não precisar escrever tanto código, nós ganhamos o benefício de performance: Extension Types são uma abstração em **tempo de compilação**. Isso significa que não tem nenhum custo de performance usar eles.

Se lembra da função `signUp`? Vamos ver como podemos melhorar ela usando nossos Newtypes.

> Eu vou deixar a implementação do Newtype `Password` para você implementar, mas sua implementação vai ser bem parecida com a do `Email`.

```dart
Future<void> signUp(Email email, Password password) async {
 logger.info('Creating user: $email');

 try {
  await repository.signUpUser(email, password);
 } catch (e) {
  throw SignUpApiException(e);
 }

 try {
  await cache.saveUser(email, password);
 } catch (e) {
  throw CacheException(e);
 }
}
```

```dart
try {
 final Email? email = Email.tryParse(controller.email);
 final Password? password = Password.tryParse(controller.password);

 if (email == null || password ==== null) {
  // handle invalid email or password
  return;
 }

 await singUp(password, email);
} on SignUpApiException {
 // handle sign up api exception
} on CacheException {
 // handle cache exception
}
```

Fontes:
<https://rust-unofficial.github.io/patterns/patterns/behavioural/newtype.html>
<https://www.howtocodeit.com/articles/ultimate-guide-rust-newtypes>
