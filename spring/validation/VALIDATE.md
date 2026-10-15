# ✅ Validação de Dados em Java e Spring

<p align="center">
  <img src="https://img.shields.io/badge/Java-Validation-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white" alt="Java Validation" />
  <img src="https://img.shields.io/badge/Spring-Validation-6DB33F?style=for-the-badge&logo=spring&logoColor=white" alt="Spring Validation" />
</p>

---

## O que é validação?

A **validação** é o processo de verificar se um dado atende às regras esperadas antes que ele seja utilizado pela aplicação.

Por exemplo, ao receber os dados de um usuário, podemos verificar se:
* O nome foi informado.
* O e-mail possui um formato válido.
* A idade está dentro de um intervalo permitido.
* A senha possui o tamanho mínimo necessário.
* Uma lista contém os elements esperados.

> **Objetivo:** Impedir que dados inválidos avancem pelo fluxo da aplicação e causem erros ou inconsistências no banco de dados ou nas regras de negócio.

---

## Diferença entre @Valid, @Validated e validate()

Esses recursos estão intimamente relacionados à execução das regras, mas possuem finalidades e escopos diferentes:

### 🔹 @Valid
A anotação `@Valid` (vinda do pacote `jakarta.validation`) solicita explicitamente a validação de um objeto e permite validar objetos relacionados de forma recursiva.

```java
@PostMapping
public ResponseEntity<?> create(@Valid @RequestBody UserRequest request) {
    // ...
}
```
*Nesse exemplo, as restrições declaradas em `UserRequest` são automaticamente verificadas quando a infraestrutura de validação do Spring processa a requisição HTTP.*

> **Importante:** O `@Valid` não cria as regras de validação sozinhos; ele apenas ativa a validação das regras que foram definidas nos atributos (como `@NotBlank`, `@Email`, etc.).

### 🔹 @Validated
A anotação `@Validated` é uma variante própria do **Spring Framework**. Ela oferece suporte à seleção de **grupos de validação** (*validation groups*) e habilita a validação diretamente em parâmetros de métodos de qualquer componente Spring (como classes de serviço).

```java
@Validated
@Service
public class UserService {
    public void atualizarSaldo(@NotNull Long id, @Positive BigDecimal valor) {
        // ...
    }
}
```

### 🔹 validate()
O método `validate()` pode ser utilizado de forma programática chamando diretamente uma instância da interface `Validator` para rodar os testes e coletar os erros manualmente.

```java
Set<ConstraintViolation<UserRequest>> errors = validator.validate(request);
```
*Esse formato é amplamente utilizado em testes unitários, scripts isolados ou quando você precisa ter total controle sobre o fluxo caso ocorra uma falha de validação.*

---

## Principais anotações de validação

As principais anotações abaixo pertencem ao pacote padrão de especificações `jakarta.validation.constraints`.

| Anotação | Finalidade | Exemplo |
| :--- | :--- | :--- |
| **`@NotNull`** | Impede valores `null`. | `@NotNull Long id` |
| **`@NotEmpty`** | Impede valores nulos ou vazios em textos, coleções, mapas e arrays. | `@NotEmpty List<String> tags` |
| **`@NotBlank`** | Exige um texto não nulo com pelo menos um caractere que não seja espaço. | `@NotBlank String name` |
| **`@Size`** | Define o tamanho mínimo e/ou máximo de strings, coleções ou arrays. | `@Size(min = 3, max = 50)` |
| **`@Email`** | Verifica se o texto tem formato de e-mail sintaticamente válido. | `@Email String email` |
| **`@Min`** | Define o valor numérico mínimo permitido. | `@Min(18) int age` |
| **`@Max`** | Define o valor numérico máximo permitido. | `@Max(120) int age` |
| **`@Positive`** | Exige um valor numérico estritamente maior que zero. | `@Positive BigDecimal price` |
| **`@PositiveOrZero`**| Aceita zero ou valores numéricos positivos. | `@PositiveOrZero int quantity` |
| **`@Negative`** | Exige um valor numérico estritamente menor que zero. | `@Negative int adjustment` |
| **`@DecimalMin`** | Define o limite inferior de um número decimal (usando String). | `@DecimalMin("0.01")` |
| **`@DecimalMax`** | Define o limite superior de um número decimal (usando String). | `@DecimalMax("9999.99")` |
| **`@Digits`** | Limita a quantidade máxima de dígitos inteiros e decimais fracionados. | `@Digits(integer = 6, fraction = 2)` |
| **`@Pattern`** | Valida se o texto corresponde perfeitamente a uma expressão regular (Regex). | `@Pattern(regexp = "\\d{11}")` |
| **`@Past`** | Exige uma data estritamente anterior ao momento atual. | `@Past LocalDate birthDate` |
| **`@PastOrPresent`** | Aceita datas passadas ou exatamente o momento atual. | `@PastOrPresent LocalDate date` |
| **`@Future`** | Exige uma data estritamente futura. | `@Future LocalDate expiration` |
| **`@FutureOrPresent`**| Aceita datas futuras ou exatamente o momento atual. | `@FutureOrPresent` |
| **`@AssertTrue`** | Exige que o valor booleano avaliado seja obrigatoriamente `true`. | `@AssertTrue boolean accepted` |
| **`@AssertFalse`** | Exige que o valor booleano avaliado seja obrigatoriamente `false`. | `@AssertFalse boolean blocked` |
| **`@Null`** | Exige explicitamente que o valor seja `null`. | `@Null String generatedCode` |

###  Atenção às diferenças cruciais:
* `@NotNull` verifica apenas se a referência não é nula. Uma string vazia (`""`) passa por ele.
* `@NotEmpty` impede nulo e vazio (`""`), mas permite uma string contendo apenas espaços em branco (`"   "`).
* `@NotBlank` é o mais restritivo para textos: impede nulo, vazio e espaços em branco.
* `@Size` **não** torna o campo obrigatório por padrão. Se o valor enviado for `null`, ele é considerado válido.
* Para limites inteiros, utilize `@Min`/`@Max`. Para precisão de ponto flutuante (`BigDecimal`, `double`), utilize `@DecimalMin`/`@DecimalMax`.

Para criar validações completas, o ideal é **combinar anotações**:

```java
@NotBlank(message = "O nome é obrigatório")
@Size(min = 3, max = 80, message = "O nome deve ter entre 3 e 80 caracteres")
private String name;
```

**importante** : O parâmetro **`message`** é utilizado para customizar a mensagem de erro que será gerada caso o valor do campo não atenda à restrição da anotação. Quando omitido, o framework utiliza uma mensagem genérica padrão do sistema (como *"não deve estar em branco"* ou *"must not be blank"*). 

Ao definir uma string personalizada nesse atributo, você garante que a aplicação retorne mensagens claras, amigáveis e padronizadas no idioma dos seus usuários quando as validações falharem.

---

##  Como declarar as validações em um DTO

DTO (*Data Transfer Object*) é um padrão de projeto para objetos que transportam dados entre as camadas da aplicação. No Java moderno, os `records` são ideais para essa função.

```java
public record UserRequest(
    @NotBlank(message = "O nome é obrigatório")
    @Size(min = 3, max = 80)
    String name,

    @NotBlank(message = "O e-mail é obrigatório")
    @Email(message = "Informe um e-mail válido")
    String email,

    @NotNull(message = "A idade é obrigatória")
    @Min(value = 18, message = "A idade mínima é 18 anos")
    @Max(value = 120, message = "A idade máxima é 120 anos")
    Integer age
) {}
```

---

## 🔗 Validação de objetos relacionados com @Valid

Se o seu DTO principal possui um objeto aninhado ou composto por outra classe, você **deve** usar `@Valid` sobre o atributo para disparar a validação interna em cascata.

```java
public record AddressRequest(
    @NotBlank String street,
    @NotBlank String city,
    @NotBlank @Pattern(regexp = "\\d{8}", message = "O CEP deve conter 8 dígitos")
    String zipCode
) {}

public record UserRequest(
    @NotBlank String name,
    @Valid @NotNull AddressRequest address
) {}
```

Para validar tipos contidos **dentro de coleções**, declare a anotação diretamente no argumento de tipo da coleção:

```java
public record TeamRequest(
    List<@NotBlank String> members
) {}
```

---

## Como utilizar no Controller

Para automatizar a verificação com as respostas do ecossistema Web do Spring Boot, declare `@Valid` ao lado do `@RequestBody`:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping
    public ResponseEntity<Void> create(@Valid @RequestBody UserRequest request) {
        userService.create(request);
        return ResponseEntity.status(201).build();
    }
}
```
*Se os dados falharem nas restrições, o Spring interceptará o fluxo disparando automaticamente a exceção `MethodArgumentNotValidException` resultando em um erro HTTP `400 Bad Request`.*

---

## Validação manual com Validator

Se precisar rodar as validações sem o contexto de requisições web ou controladores, você pode usar a fábrica do Hibernate Validator programaticamente:

```java
import jakarta.validation.*;
import java.util.Set;

public class Main {
    public static void main(String[] args) {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();

        UserRequest request = new UserRequest("", "email-invalido", 15);
        Set<ConstraintViolation<UserRequest>> violations = validator.validate(request);

        for (ConstraintViolation<UserRequest> violation : violations) {
            System.out.println(violation.getPropertyPath() + ": " + violation.getMessage());
        }

        factory.close();
    }
}
```

Métodos úteis retornados pela lista de violações (`ConstraintViolation`):
* `getPropertyPath()`: Retorna o nome exato do campo que falhou.
* `getMessage()`: Retorna a mensagem de erro vinculada.
* `getInvalidValue()`: Retorna o dado incorreto enviado pelo cliente.

---

## Validações personalizadas

*(Nota: O texto original foi interrompido aqui. Adicione nesta seção a criação de uma anotação customizada estendendo `ConstraintValidator` para fechar o conteúdo do estudo).*
