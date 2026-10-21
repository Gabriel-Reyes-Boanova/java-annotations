# Lombok no Spring Boot

![Java](https://img.shields.io/badge/Java-17%2B-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x%20%7C%204.x-6DB33F?logo=springboot&logoColor=white)
![Lombok](https://img.shields.io/badge/Lombok-1.18.x-BC4521)
![Status](https://img.shields.io/badge/status-estudo-blue)

## Sumário

- [O que é o Lombok](#o-que-é-o-lombok)
- [Para que serve](#para-que-serve)
- [Como ele funciona](#como-ele-funciona)
- [Instalação e configuração](#instalação-e-configuração)
- [Principais anotações](#principais-anotações)
- [Detalhando as anotações](#detalhando-as-anotações)
- [Lombok no Spring Boot](#lombok-no-spring-boot-1)
- [Lombok com JPA: cuidados](#lombok-com-jpa-cuidados)
- [Lombok vs record](#lombok-vs-record)
- [Exemplo completo explicado linha por linha](#exemplo-completo-explicado-linha-por-linha)
- [Armadilhas comuns](#armadilhas-comuns)
- [Como ver o código gerado](#como-ver-o-código-gerado)
- [Boas práticas](#boas-práticas)
- [Referências](#referências)

---

## O que é o Lombok

O **Lombok** é uma biblioteca Java que **gera código repetitivo automaticamente**, em tempo de compilação, a partir de anotações. Getters, setters, construtores, `toString`, `equals`, `hashCode`, *builders* e *loggers* deixam de ser escritos à mão.

Você escreve isto:

```java
@Getter
@Setter
public class Usuario {
    private Long id;
    private String nome;
}
```

E o compilador enxerga isto:

```java
public class Usuario {
    private Long id;
    private String nome;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
}
```

> [!NOTE]
> O Lombok **não é do Spring** e não é obrigatório. É uma biblioteca independente, muito usada em projetos Spring Boot porque combina bem com classes cheias de campos e injeção de dependência.

## Para que serve

| Código repetitivo | Anotação do Lombok |
|---|---|
| Getters e setters | `@Getter`, `@Setter` |
| `toString()` | `@ToString` |
| `equals()` e `hashCode()` | `@EqualsAndHashCode` |
| Construtores | `@NoArgsConstructor`, `@AllArgsConstructor`, `@RequiredArgsConstructor` |
| Tudo acima junto (para classes simples) | `@Data` |
| Construção passo a passo de objetos | `@Builder` |
| Variável de log (`Logger`) | `@Slf4j` |
| Verificação de nulo em parâmetros e campos | `@NonNull` |

> [!IMPORTANT]
> O Lombok reduz o **ruído** do código, mas não muda o que a aplicação faz. O comportamento é o mesmo de quem escreveu tudo à mão.

## Como ele funciona

O Lombok é um **annotation processor**: ele se conecta ao compilador (`javac`), lê suas anotações e **altera a árvore sintática do código** antes de gerar o `.class`.

```mermaid
flowchart LR
    A["Usuario.java<br/>com anotações Lombok"] --> B["javac<br/>compilador"]
    B --> C{{"Lombok<br/>annotation processor"}}
    C -->|"insere getters, construtores..."| D["Árvore sintática<br/>modificada"]
    D --> E["Usuario.class<br/>com os métodos gerados"]
```

| Fase | O que acontece |
|---|---|
| **Você escreve** | Classe com anotações; sem os métodos |
| **Compilação** | O Lombok adiciona os métodos ao código compilado |
| **Resultado** | O `.class` tem todos os métodos; o `.java` continua limpo |

> [!NOTE]
> Por isso a **IDE precisa conhecer o Lombok** (plugin ou processamento de anotações ligado). Sem isso, ela não "vê" os métodos gerados e marca erro em código que compila normalmente.

---

## Instalação e configuração

### Maven

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

A versão é gerenciada pelo Spring Boot, então não precisa informá-la. O `<optional>true</optional>` evita que o Lombok seja repassado como dependência a quem usar o seu projeto.

<details>
<summary>Excluir o Lombok do <code>.jar</code> final e configurar o processador (opcional)</summary>

O Lombok só é necessário **na compilação**. O Spring Initializr gera a configuração abaixo para deixá-lo fora do `.jar` executável:

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
            </exclude>
        </excludes>
    </configuration>
</plugin>
```

Se o Lombok não gerar nada (isso pode acontecer em JDKs recentes, que não ativam o processamento de anotações automaticamente), declare-o como processador:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

</details>

### Gradle

```groovy
dependencies {
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    testCompileOnly 'org.projectlombok:lombok'
    testAnnotationProcessor 'org.projectlombok:lombok'
}
```

### IDE

| IDE | O que fazer |
|---|---|
| **IntelliJ IDEA** | Plugin Lombok (já vem incluído nas versões recentes) + ativar **Enable annotation processing** em *Settings > Build > Compiler > Annotation Processors* |
| **VS Code** | Extension Pack for Java (já suporta o Lombok) |
| **Eclipse / STS** | Instalar o Lombok pelo `lombok.jar` (executá-lo e apontar a IDE) |

### Arquivo `lombok.config` (opcional)

Fica na raiz do projeto e muda o comportamento do Lombok para todo o código:

```properties
# Impede que o Lombok procure outros lombok.config nas pastas acima
config.stopBubbling = true

# Marca o código gerado com @Generated (ferramentas de cobertura, como o JaCoCo, o ignoram)
lombok.addLombokGeneratedAnnotation = true
```

---

## Principais anotações

| Anotação | O que gera | Onde usar |
|---|---|---|
| `@Getter` | Métodos `getX()` | Classe ou campo |
| `@Setter` | Métodos `setX()` | Classe ou campo |
| `@ToString` | `toString()` com os campos | Classe |
| `@EqualsAndHashCode` | `equals()` e `hashCode()` | Classe |
| `@NoArgsConstructor` | Construtor sem parâmetros | Classe |
| `@AllArgsConstructor` | Construtor com **todos** os campos | Classe |
| `@RequiredArgsConstructor` | Construtor com campos `final` e `@NonNull` | Classe (injeção de dependência) |
| `@Data` | `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor` | Classes simples (não entidades JPA) |
| `@Value` | Versão **imutável** do `@Data` (campos `final`, sem setters) | Classes imutáveis |
| `@Builder` | Padrão *Builder* (`Usuario.builder().nome("Ana").build()`) | Classe ou construtor |
| `@Slf4j` | Variável `log` | Classe (logs) |
| `@NonNull` | Verificação de nulo (lança `NullPointerException`) | Campo ou parâmetro |

## Detalhando as anotações

### `@Getter` e `@Setter`

Podem ser colocados na **classe** (vale para todos os campos) ou em **campos específicos**:

```java
@Getter                       // getters para todos os campos
public class Produto {

    private Long id;

    @Setter                   // só o "nome" ganha setter
    private String nome;

    @Setter(AccessLevel.PROTECTED)   // setter com visibilidade protected
    private BigDecimal preco;
}
```

> [!TIP]
> Evite `@Setter` na classe inteira quando possível. Deixar campos sem setter protege a classe de alterações indesejadas.

### `@ToString`

```java
@ToString
public class Usuario {
    private Long id;
    private String nome;

    @ToString.Exclude          // fora do toString (ex.: senha)
    private String senha;
}
// Usuario(id=1, nome=Ana)
```

> [!WARNING]
> Nunca deixe campos sensíveis (senha, token) no `toString()`: eles acabam vazando em logs.

### `@EqualsAndHashCode`

Por padrão, usa **todos os campos** não estáticos. Para escolher quais contam:

```java
@EqualsAndHashCode(onlyExplicitlyIncluded = true)   // só os campos marcados entram
public class Usuario {

    @EqualsAndHashCode.Include
    private Long id;

    private String nome;       // não entra na comparação
}
```

### Construtores

| Anotação | Gera | Exemplo |
|---|---|---|
| `@NoArgsConstructor` | `new Usuario()` | Exigido pelo JPA |
| `@AllArgsConstructor` | `new Usuario(1L, "Ana")` | Todos os campos, na ordem declarada |
| `@RequiredArgsConstructor` | Só campos `final` e `@NonNull` | Injeção de dependência |

```java
@RequiredArgsConstructor
public class UsuarioService {
    private final UsuarioRepository repository;   // final: entra no construtor
    private String ignorado;                      // não é final: fica de fora
}
```

> [!IMPORTANT]
> No `@RequiredArgsConstructor`, esquecer o `final` faz a dependência **não ser injetada** e causa `NullPointerException` ao usá-la.

### `@Data`

É o "pacote completo" para classes simples:

```java
@Data
public class Contato {
    private String nome;
    private String telefone;
}
```

<details>
<summary>O mesmo código escrito à mão</summary>

```java
public class Contato {
    private String nome;
    private String telefone;

    public String getNome() { return nome; }
    public void setNome(String nome) { this.nome = nome; }
    public String getTelefone() { return telefone; }
    public void setTelefone(String telefone) { this.telefone = telefone; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Contato)) return false;
        Contato outro = (Contato) o;
        return Objects.equals(nome, outro.nome) && Objects.equals(telefone, outro.telefone);
    }

    @Override
    public int hashCode() {
        return Objects.hash(nome, telefone);
    }

    @Override
    public String toString() {
        return "Contato(nome=" + nome + ", telefone=" + telefone + ")";
    }
}
```

</details>

> [!WARNING]
> **Não use `@Data` em entidades JPA.** Veja [Lombok com JPA: cuidados](#lombok-com-jpa-cuidados).

### `@Value`

Gera uma classe **imutável**: a classe vira `final`, os campos viram `private final`, e não há setters.

```java
@Value
public class Endereco {
    String rua;       // vira "private final String rua"
    String cidade;
}
```

> [!TIP]
> Para DTOs no Java moderno, prefira `record` (veja [Lombok vs record](#lombok-vs-record)).

### `@Builder`

Cria um construtor em etapas, útil para classes com muitos campos:

```java
@Builder
@Getter
public class Pedido {
    private Long id;
    private String cliente;
    private BigDecimal total;
}

// Uso
Pedido pedido = Pedido.builder()
        .cliente("Ana")
        .total(new BigDecimal("99.90"))
        .build();
```

| Recurso | Para que serve |
|---|---|
| `@Builder.Default` | Define valor padrão (sem ele, o valor inicial do campo é **ignorado**) |
| `@Singular` | Em coleções, permite adicionar item por item |
| `@Builder(toBuilder = true)` | Cria uma cópia modificável de um objeto existente |
| `@SuperBuilder` | Builder para classes com **herança** |

```java
@Builder
public class Pedido {
    @Builder.Default
    private String status = "NOVO";      // sem @Builder.Default, o builder deixaria null

    @Singular
    private List<String> itens;          // .item("a").item("b") ou .itens(lista)
}
```

### `@Slf4j`

Cria o `Logger` automaticamente:

```java
@Slf4j
@Service
public class UsuarioService {

    public void criar(String email) {
        log.info("Criando usuário com e-mail {}", email);   // "log" foi gerado pelo Lombok
    }
}
```

Equivale a escrever:

```java
private static final Logger log = LoggerFactory.getLogger(UsuarioService.class);
```

> [!NOTE]
> O Spring Boot já traz o SLF4J (via `spring-boot-starter-logging`), então basta o Lombok.

### `@NonNull`

```java
public void definirNome(@NonNull String nome) {   // gera: if (nome == null) throw new NullPointerException(...)
    this.nome = nome;
}
```

Em um campo, o `@NonNull` faz o setter e o construtor gerados validarem o valor.

---

## Lombok no Spring Boot

### Injeção por construtor com `@RequiredArgsConstructor`

O uso mais comum no dia a dia:

```java
@RestController
@RequestMapping("/api/usuarios")
@RequiredArgsConstructor                          // gera o construtor com o campo final
public class UsuarioController {

    private final UsuarioService service;         // injetado pelo construtor gerado
}
```

Como só existe **um construtor**, o Spring o usa direto, sem `@Autowired`.

### Usando `@Qualifier` com campos `final`

Para o `@Qualifier` chegar ao construtor gerado, adicione ao `lombok.config`:

```properties
lombok.copyableAnnotations += org.springframework.beans.factory.annotation.Qualifier
```

```java
@Service
@RequiredArgsConstructor
public class NotificacaoService {

    @Qualifier("emailSender")
    private final Sender sender;
}
```

### Onde o Lombok mais aparece

| Camada | Anotações comuns |
|---|---|
| **Controller** | `@RequiredArgsConstructor` |
| **Service** | `@RequiredArgsConstructor`, `@Slf4j` |
| **Entity** (JPA) | `@Getter`, `@Setter`, `@NoArgsConstructor`, `@Builder` |
| **DTO** | Prefira `record`; se usar classe, `@Getter`, `@Builder` |
| **Configuration** | `@RequiredArgsConstructor`, `@Slf4j` |

---

## Lombok com JPA: cuidados

Entidades JPA têm particularidades (proxies do Hibernate, carregamento *lazy*, relacionamentos bidirecionais). Algumas anotações do Lombok **causam problemas** nelas:

| Anotação | Risco em entidades |
|---|---|
| `@Data` | Gera `equals`, `hashCode` e `toString` com **todos os campos**, inclusive relacionamentos |
| `@ToString` (padrão) | Percorre relacionamentos: pode **disparar consultas** e causar `StackOverflowError` em relações bidirecionais |
| `@EqualsAndHashCode` (padrão) | Compara coleções *lazy* e muda o resultado conforme o estado do objeto |
| `@NoArgsConstructor` público | O JPA exige o construtor vazio, mas ele não precisa ser público |

### Padrão mais seguro

```java
@Entity
@Table(name = "usuarios")
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)      // exigido pelo JPA, escondido do resto do código
@AllArgsConstructor(access = AccessLevel.PRIVATE)       // usado pelo @Builder
@Builder
@ToString(onlyExplicitlyIncluded = true)                // só o que eu escolher
public class Usuario {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @ToString.Include
    private Long id;

    @ToString.Include
    private String nome;

    private String email;

    @OneToMany(mappedBy = "usuario")
    private List<Pedido> pedidos;                       // fora do toString, evita carregar a lista
}
```

> [!IMPORTANT]
> Regra prática para entidades: use `@Getter` e `@Setter`, **evite `@Data`**, escolha explicitamente os campos do `toString`, e trate `equals`/`hashCode` com cuidado (um padrão comum é comparar apenas pelo `id`).

---

## Lombok vs record

Desde o Java 16, o `record` resolve boa parte do que o Lombok fazia para classes de dados:

```java
public record UsuarioResponse(Long id, String nome, String email) {}
```

| | `record` | Lombok |
|---|---|---|
| Dependência extra | Não (é da linguagem) | Sim |
| Imutável | **Sempre** | Só com `@Value` |
| Getters | `nome()` (sem o prefixo `get`) | `getNome()` |
| Pode ser **entidade JPA** | **Não** | Sim |
| Herança | Não | Sim |
| Builder | Não | `@Builder` |
| Construtor injetado (`@RequiredArgsConstructor`) | Não se aplica | Sim |

| Situação | Melhor escolha |
|---|---|
| **DTOs** de entrada e saída | `record` |
| **Entidades JPA** | Lombok (`@Getter`, `@Setter`, `@Builder`...) |
| **Injeção de dependência** (Controller, Service) | Lombok (`@RequiredArgsConstructor`) |
| **Logs** | Lombok (`@Slf4j`) |

---

## Exemplo completo explicado linha por linha

### 1. A entidade

```java
import jakarta.persistence.*;                                   // anotações do JPA
import lombok.*;                                                // todas as anotações do Lombok

@Entity                                                         // diz ao JPA: esta classe é uma tabela
@Table(name = "usuarios")                                       // nome da tabela no banco
@Getter                                                         // gera os getters de todos os campos
@Setter                                                         // gera os setters de todos os campos
@NoArgsConstructor(access = AccessLevel.PROTECTED)              // construtor vazio exigido pelo JPA, mas protegido
@AllArgsConstructor(access = AccessLevel.PRIVATE)               // construtor completo, privado: só o builder usa
@Builder                                                        // gera Usuario.builder()...build()
@ToString(onlyExplicitlyIncluded = true)                        // toString só com os campos marcados @ToString.Include
public class Usuario {

    @Id                                                         // chave primária
    @GeneratedValue(strategy = GenerationType.IDENTITY)         // o banco gera o id
    @ToString.Include                                           // id aparece no toString
    private Long id;

    @ToString.Include                                           // nome aparece no toString
    private String nome;

    private String email;                                       // fora do toString (padrão: não incluído)
}
```

### 2. O Service

```java
import lombok.RequiredArgsConstructor;                          // gera o construtor com os campos final
import lombok.extern.slf4j.Slf4j;                               // gera a variável "log"

@Slf4j                                                          // cria: private static final Logger log = ...
@Service                                                        // o Spring gerencia esta classe
@RequiredArgsConstructor                                        // cria o construtor com repository e mapper
public class UsuarioService {

    private final UsuarioRepository repository;                // final: entra no construtor gerado
    private final UsuarioMapper mapper;                         // final: entra no construtor gerado

    public UsuarioResponse criar(UsuarioRequest request) {
        log.info("Criando usuário com e-mail {}", request.email());   // "log" veio do @Slf4j; {} evita concatenar String

        Usuario usuario = Usuario.builder()                     // builder gerado pelo @Builder
                .nome(request.nome())                           // preenche o nome
                .email(request.email())                         // preenche o e-mail
                .build();                                       // cria o objeto (id fica null até salvar)

        Usuario salvo = repository.save(usuario);               // grava no banco; volta com o id preenchido
        return mapper.toResponse(salvo);                        // converte a entidade em DTO de resposta
    }
}
```

### 3. O DTO (sem Lombok)

```java
public record UsuarioResponse(Long id, String nome, String email) {}   // record: sem necessidade de Lombok
```

---

## Armadilhas comuns

| Problema | Causa | Solução |
|---|---|---|
| Dependência vem `null` | Campo sem `final` com `@RequiredArgsConstructor` | Declarar o campo como `final` |
| IDE marca erro em `getNome()` | Lombok não configurado na IDE | Plugin + **annotation processing** ligado |
| `@Builder` não compila com `@NoArgsConstructor` | O builder precisa de um construtor com todos os campos | Adicionar `@AllArgsConstructor` |
| Valor padrão do campo some no builder | O `@Builder` ignora o valor inicial | Usar `@Builder.Default` |
| `StackOverflowError` em entidades | `@ToString`/`@EqualsAndHashCode` em relações bidirecionais | `@ToString.Exclude` ou `onlyExplicitlyIncluded = true` |
| Senha aparece nos logs | Campo incluído no `toString` | `@ToString.Exclude` |
| `@Qualifier` ignorado | A anotação não chega ao construtor gerado | `lombok.copyableAnnotations` no `lombok.config` |
| `@Builder` não funciona com herança | O builder comum não enxerga os campos da classe pai | Usar `@SuperBuilder` na pai e na filha |

### Anotações para usar com cuidado

| Anotação | Por que evitar |
|---|---|
| `@SneakyThrows` | Esconde exceções checadas: quem chama o método não sabe que ele pode falhar |
| `@Cleanup` | O `try-with-resources` do Java faz o mesmo, de forma mais clara |
| `val` / `var` do Lombok | O Java já tem `var` nativo |
| `@Accessors(fluent = true)` | Troca `getNome()` por `nome()` e pode quebrar ferramentas que esperam o padrão JavaBeans (Jackson, JPA) |

---

## Como ver o código gerado

**1. Na IDE:** no IntelliJ, a janela *Structure* lista os métodos gerados pelo Lombok.

**2. No bytecode**, com o `javap` (já vem no JDK):

```bash
javap -p target/classes/com/exemplo/Usuario.class
```

A saída lista os getters, setters e construtores presentes no `.class`.

**3. Com o `delombok`**, que reescreve o `.java` já com o código gerado:

```bash
java -jar lombok.jar delombok src/main/java -d target/delombok
```

> [!TIP]
> O `delombok` é útil para estudar o que cada anotação faz e para remover o Lombok de um projeto, caso um dia seja necessário.

---

## Boas práticas

- [x] Usar **`@RequiredArgsConstructor`** com campos `final` para injeção de dependência
- [x] Usar **`@Slf4j`** em vez de declarar o `Logger` manualmente
- [x] Em **entidades JPA**: `@Getter`, `@Setter`, `@NoArgsConstructor(access = PROTECTED)` e `@Builder`
- [x] **Não usar `@Data` em entidades** JPA
- [x] Definir `@ToString.Exclude` em senhas, tokens e relacionamentos
- [x] Usar **`record`** para DTOs em vez de classes com Lombok
- [x] Usar `@Builder.Default` quando o campo tem valor inicial
- [x] Manter o plugin do Lombok e o **annotation processing** ativos na IDE
- [x] Conferir o código gerado com `javap` ou `delombok` quando houver dúvida
- [x] Combinar com a equipe: se o projeto usa Lombok, usar de forma consistente

<details>
<summary>Exemplo do que <b>não</b> fazer</summary>

```java
@Data                                       // ❌ @Data em entidade JPA
@Entity
public class Usuario {

    @Id
    private Long id;

    private String senha;                   // ❌ vai para o toString e para os logs

    @OneToMany(mappedBy = "usuario")
    private List<Pedido> pedidos;           // ❌ toString/equals percorrem a lista (consultas e loop infinito)
}

@Service
@RequiredArgsConstructor
public class UsuarioService {

    private UsuarioRepository repository;   // ❌ sem "final": o construtor gerado não recebe este campo

    @SneakyThrows                           // ❌ esconde a exceção checada
    public void processar() { ... }
}
```

</details>

## Referências

- [Lombok: lista de funcionalidades](https://projectlombok.org/features/)
- [Lombok: configuração com Maven](https://projectlombok.org/setup/maven)
- [Lombok: configuração com Gradle](https://projectlombok.org/setup/gradle)
- [Lombok: arquivo lombok.config](https://projectlombok.org/features/configuration)
- [Plugin Lombok no IntelliJ IDEA](https://www.jetbrains.com/help/idea/lombok-plugin.html)
- [Guia de Markdown do GitHub (GFM)](https://docs.github.com/pt/get-started/writing-on-github)
