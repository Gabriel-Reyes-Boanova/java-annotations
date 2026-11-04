# Swagger e OpenAPI no Spring Boot: documentação, schemas e autenticação JWT

![Java](https://img.shields.io/badge/Java-17%2B-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x%20%7C%204.x-6DB33F?logo=springboot&logoColor=white)
![springdoc](https://img.shields.io/badge/springdoc--openapi-2.8.x%20%7C%203.x-85EA2D?logo=swagger&logoColor=black)
![OpenAPI](https://img.shields.io/badge/OpenAPI-3.x-6BA539?logo=openapiinitiative&logoColor=white)

Guia prático para documentar uma API Spring Boot com **springdoc-openapi**, organizar os **Schemas** e testar endpoints protegidos usando **JWT Bearer Token** no Swagger UI.

> Os exemplos usam uma API de funcionários. Adapte os nomes e caminhos ao seu projeto.

## Sumário

- [1. Swagger, OpenAPI e springdoc](#1-swagger-openapi-e-springdoc)
- [2. Dependência](#2-dependência)
- [3. URLs do Swagger](#3-urls-do-swagger)
- [4. Como os Schemas funcionam](#4-como-os-schemas-funcionam)
- [5. DTOs documentados](#5-dtos-documentados)
- [6. Controller REST melhorado](#6-controller-rest-melhorado)
- [7. Anotações principais](#7-anotações-principais)
- [8. Configuração geral do OpenAPI](#8-configuração-geral-do-openapi)
- [9. Swagger com Spring Security e JWT](#9-swagger-com-spring-security-e-jwt)
  - [9.5. Outros esquemas de autenticação](#95-outros-esquemas-de-autenticação)
- [10. application.properties](#10-applicationproperties)
- [11. Testando no Swagger UI](#11-testando-no-swagger-ui)
- [12. Erros comuns](#12-erros-comuns)
- [13. Boas práticas para produção](#13-boas-práticas-para-produção)
- [14. Checklist e referências](#14-checklist-e-referências)

---

## 1. Swagger, OpenAPI e springdoc

| Nome | O que é |
|---|---|
| **OpenAPI** | Especificação que descreve endpoints, parâmetros, autenticação, requisições e respostas de uma API |
| **Swagger UI** | Interface web que apresenta a documentação e permite executar requisições |
| **springdoc-openapi** | Biblioteca que analisa a aplicação Spring e gera a documentação OpenAPI |
| **Schema** | Descrição da estrutura de um objeto usado na entrada ou saída de um endpoint |

```mermaid
flowchart LR
    A[Controllers e DTOs] --> B[springdoc-openapi]
    B --> C[Documento OpenAPI JSON]
    C --> D[Swagger UI]
    D --> E[Testar endpoints]
```

A biblioteca infere muitas informações pelo código, mas as anotações deixam a documentação mais útil para quem consome a API.

## 2. Dependência

Escolha a versão compatível com a versão do Spring Boot:

| Spring Boot | Linha do springdoc |
|---|---|
| 3.x | 2.x |
| 4.x | 3.x |

Confirme a versão atual e a compatibilidade no [Maven Central](https://central.sonatype.com/artifact/org.springdoc/springdoc-openapi-starter-webmvc-ui).

**Maven — exemplo para Spring Boot 4:**

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>3.0.3</version>
</dependency>
```

Se o projeto usa Spring WebFlux, utilize `springdoc-openapi-starter-webflux-ui` em vez de `webmvc-ui`.

## 3. URLs do Swagger

Com a aplicação executando na porta 8080:

| Recurso | URL |
|---|---|
| Swagger UI | `http://localhost:8080/swagger-ui.html` |
| OpenAPI JSON | `http://localhost:8080/v3/api-docs` |
| OpenAPI YAML | `http://localhost:8080/v3/api-docs.yaml` |

O endereço `/swagger-ui.html` normalmente redireciona para a interface do Swagger UI.

## 4. Como os Schemas funcionam

No Swagger, a seção **Schemas** apresenta os modelos de dados usados pela API. Pense nela como um catálogo dos formatos de JSON aceitos e devolvidos pelos endpoints.

Um cadastro pode receber:

```json
{
  "name": "Gabriel Boanova",
  "role": "Desenvolvedor Backend"
}
```

E devolver:

```json
{
  "id": 1,
  "name": "Gabriel Boanova",
  "role": "Desenvolvedor Backend"
}
```

As estruturas são diferentes. Por isso, é uma boa prática separar DTO de entrada e DTO de resposta.

### 4.1 Tipos de Schema comuns

| Schema | Finalidade |
|---|---|
| `EmployeeRequest` | Dados enviados para criar um funcionário |
| `EmployeeUpdateRequest` | Dados enviados para atualizar um funcionário |
| `EmployeeResponse` | Dados devolvidos pela API |
| `PageEmployeeResponse` | Modelo próprio para resposta paginada |
| `ErrorResponse` | Formato padronizado de erro |
| `EmployeeStatus` | Valores possíveis de um campo enumerado |

O springdoc normalmente identifica os modelos referenciados nos parâmetros e retornos. Não é necessário criar um `@Schema` separado para cada DTO. Use a anotação nos campos para adicionar descrição, exemplo, formato ou obrigatoriedade.

### 4.2 `@Schema` na prática

```java
@Schema(description = "Dados necessários para cadastrar um funcionário")
public record EmployeeRequest(

    @Schema(
        description = "Nome completo do funcionário",
        example = "Gabriel Boanova",
        requiredMode = Schema.RequiredMode.REQUIRED
    )
    @NotBlank
    @Size(max = 120)
    String name,

    @Schema(
        description = "Cargo do funcionário",
        example = "Desenvolvedor Backend",
        requiredMode = Schema.RequiredMode.REQUIRED
    )
    @NotBlank
    @Size(max = 80)
    String role
) {}
```

`@Schema` documenta. Anotações como `@NotBlank` e `@Size` validam os dados em tempo de execução, desde que a validação esteja configurada.

## 5. DTOs documentados

Os exemplos usam Java Records para reduzir código repetitivo. Também é possível usar classes tradicionais.

### 5.1 DTO de entrada — `EmployeeRequest.java`

```java
import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

@Schema(description = "Dados para cadastrar um funcionário")
public record EmployeeRequest(

    @Schema(description = "Nome completo", example = "Gabriel Silva")
    @NotBlank
    @Size(max = 120)
    String name,

    @Schema(description = "Cargo", example = "Desenvolvedor Backend")
    @NotBlank
    @Size(max = 80)
    String role
) {}
```

### 5.2 DTO de atualização — `EmployeeUpdateRequest.java`

Neste exemplo, os dois campos são obrigatórios. Para atualização parcial (PATCH), crie um DTO específico com campos opcionais e validações adequadas.

```java
import io.swagger.v3.oas.annotations.media.Schema;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

@Schema(description = "Dados para atualizar um funcionário")
public record EmployeeUpdateRequest(

    @Schema(description = "Novo nome", example = "Gabriel Reyes")
    @NotBlank
    @Size(max = 120)
    String name,

    @Schema(description = "Novo cargo", example = "Engenheiro de Software")
    @NotBlank
    @Size(max = 80)
    String role
) {}
```

### 5.3 DTO de resposta — `EmployeeResponse.java`

```java
import io.swagger.v3.oas.annotations.media.Schema;

@Schema(description = "Dados públicos de um funcionário")
public record EmployeeResponse(

    @Schema(description = "Identificador do funcionário", example = "1")
    Long id,

    @Schema(description = "Nome completo", example = "Gabriel Boanova")
    String name,

    @Schema(description = "Cargo", example = "Desenvolvedor Backend")
    String role
) {}
```

O DTO permite controlar quais dados serão disponibilizados ao consumidor da API, sem expor diretamente todos os atributos da entidade JPA.

### 5.4 Enum como Schema

```java
@Schema(description = "Situação do funcionário")
public enum EmployeeStatus {
    ACTIVE,
    INACTIVE
}
```

O springdoc consegue apresentar os valores do enum na documentação.

### 5.5 Schema de erro — `ErrorResponse.java`

```java
import io.swagger.v3.oas.annotations.media.Schema;
import java.time.OffsetDateTime;

@Schema(description = "Representação de um erro da API")
public record ErrorResponse(

    @Schema(example = "2026-11-04T10:30:00-03:00")
    OffsetDateTime timestamp,

    @Schema(example = "404")
    int status,

    @Schema(example = "Not Found")
    String error,

    @Schema(example = "Funcionário não encontrado")
    String message,

    @Schema(example = "/api/employees/99")
    String path
) {}
```

Esse DTO só será uma resposta real se a aplicação o utilizar, por exemplo, em um `@RestControllerAdvice`. Declarar o record sozinho não altera o tratamento de exceções.

## 6. Controller REST melhorado

O controller deve cuidar da comunicação HTTP: receber parâmetros, validar entradas, chamar a camada de serviço e devolver status/body adequados.

Evite colocar regras de negócio, acesso ao banco ou armazenamento em `Map` diretamente no controller. Neste exemplo, a lógica fica em um `EmployeeService`, que deve ser implementado e injetado pelo Spring.

### 6.1 `EmployeeController.java`

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.support.ServletUriComponentsBuilder;

import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/employees")
@Tag(
    name = "Funcionários",
    description = "Operações de cadastro, consulta, atualização e remoção"
)
public class EmployeeController {

    private final EmployeeService employeeService;

    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    @GetMapping
    @Operation(
        summary = "Listar funcionários",
        description = "Retorna todos os funcionários cadastrados."
    )
    @ApiResponse(responseCode = "200", description = "Lista retornada com sucesso")
    public ResponseEntity<List<EmployeeResponse>> findAll() {
        return ResponseEntity.ok(employeeService.findAll());
    }

    @GetMapping("/{id}")
    @Operation(
        summary = "Buscar funcionário por ID",
        description = "Busca um funcionário utilizando seu identificador."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Funcionário encontrado"),
        @ApiResponse(
            responseCode = "404",
            description = "Funcionário não encontrado",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    public ResponseEntity<EmployeeResponse> findById(
        @Parameter(description = "ID do funcionário", example = "1")
        @PathVariable Long id
    ) {
        return ResponseEntity.ok(employeeService.findById(id));
    }

    @PostMapping
    @Operation(
        summary = "Cadastrar funcionário",
        description = "Cria um funcionário e retorna sua representação."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Funcionário criado"),
        @ApiResponse(
            responseCode = "400",
            description = "Dados inválidos",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    public ResponseEntity<EmployeeResponse> create(
        @Valid @RequestBody EmployeeRequest request
    ) {
        EmployeeResponse created = employeeService.create(request);

        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.id())
            .toUri();

        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    @Operation(
        summary = "Atualizar funcionário",
        description = "Substitui os dados editáveis do funcionário informado."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Funcionário atualizado"),
        @ApiResponse(
            responseCode = "400",
            description = "Dados inválidos",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        ),
        @ApiResponse(
            responseCode = "404",
            description = "Funcionário não encontrado",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    public ResponseEntity<EmployeeResponse> update(
        @Parameter(description = "ID do funcionário", example = "1")
        @PathVariable Long id,
        @Valid @RequestBody EmployeeUpdateRequest request
    ) {
        return ResponseEntity.ok(employeeService.update(id, request));
    }

    @DeleteMapping("/{id}")
    @Operation(
        summary = "Remover funcionário",
        description = "Remove o funcionário informado."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "204", description = "Funcionário removido"),
        @ApiResponse(
            responseCode = "404",
            description = "Funcionário não encontrado",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        )
    })
    public ResponseEntity<Void> delete(
        @Parameter(description = "ID do funcionário", example = "1")
        @PathVariable Long id
    ) {
        employeeService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

**Atenção:** o código pressupõe que `EmployeeService` tenha os métodos `findAll`, `findById`, `create`, `update` e `delete`, com os tipos apresentados. A implementação do serviço, repository e tratamento global de exceções não está incluída. O `404` depende de o serviço lançar uma exceção que sua aplicação converta para HTTP 404.

## 7. Anotações principais

| Anotação | Onde usar | Finalidade |
|---|---|---|
| `@Tag` | Controller | Agrupa endpoints |
| `@Operation` | Método | Resume e descreve a operação |
| `@ApiResponse` | Método | Documenta uma resposta HTTP |
| `@ApiResponses` | Método | Agrupa respostas possíveis |
| `@Parameter` | Parâmetro | Descreve path/query/header parameters |
| `@Schema` | DTO, campo ou enum | Descreve modelos e propriedades |
| `@ArraySchema` | Propriedade ou conteúdo | Descreve estruturas em lista/array |
| `@Content` | Resposta | Descreve o conteúdo retornado |
| `@Hidden` | Controller ou método | Remove algo da documentação |
| `@SecurityRequirement` | Controller ou método | Indica o esquema de segurança exigido |

### 7.1 Documentar uma lista

```java
@ApiResponse(
    responseCode = "200",
    description = "Lista de funcionários",
    content = @Content(
        array = @ArraySchema(
            schema = @Schema(implementation = EmployeeResponse.class)
        )
    )
)
```

Em muitos casos, o springdoc infere a lista pelo retorno `List<EmployeeResponse>`. Use a declaração explícita quando ela melhorar a clareza ou resolver uma inferência incorreta.

### 7.2 Parâmetro de busca

```java
@GetMapping("/search")
@Operation(summary = "Buscar funcionários por cargo")
public ResponseEntity<List<EmployeeResponse>> search(
    @Parameter(description = "Cargo usado no filtro", example = "Desenvolvedor")
    @RequestParam String role
) {
    return ResponseEntity.ok(employeeService.searchByRole(role));
}
```

### 7.3 `@Schema` versus validação

```java
@Schema(description = "E-mail do usuário", example = "gabriel@email.com")
@Email
@NotBlank
String email
```

- `@Schema`: explica o campo na documentação.
- `@Email`: valida o formato do e-mail.
- `@NotBlank`: impede valor nulo, vazio ou composto apenas por espaços.

Uma descrição no Swagger não impede dados inválidos de chegarem à aplicação.

## 8. Configuração geral do OpenAPI

Crie uma classe de configuração para definir título, versão, descrição, contato, servidores e esquemas de autenticação.

### 8.1 `OpenApiConfig.java` — JWT Bearer

```java
import io.swagger.v3.oas.annotations.OpenAPIDefinition;
import io.swagger.v3.oas.annotations.info.Contact;
import io.swagger.v3.oas.annotations.info.Info;
import io.swagger.v3.oas.annotations.servers.Server;
import io.swagger.v3.oas.annotations.enums.SecuritySchemeType;
import io.swagger.v3.oas.annotations.security.SecurityRequirement;
import io.swagger.v3.oas.annotations.security.SecurityScheme;
import org.springframework.context.annotation.Configuration;

@Configuration
@OpenAPIDefinition(
    info = @Info(
        title = "API de Funcionários",
        version = "1.0.0",
        description = "API REST para gerenciamento de funcionários.",
        contact = @Contact(
            name = "Equipe de Desenvolvimento",
            email = "dev@exemplo.com"
        )
    ),
    servers = {
        @Server(url = "http://localhost:8080", description = "Ambiente local")
    },
    security = {
        @SecurityRequirement(name = "bearerAuth")
    }
)
@SecurityScheme(
    name = "bearerAuth",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT",
    description = "Informe o token JWT. Não escreva o prefixo Bearer no campo do Swagger."
)
public class OpenApiConfig {
}
```

### 8.2 Entendendo as anotações

- `@Configuration`: registra a classe como configuração do Spring.
- `@OpenAPIDefinition`: define informações gerais do documento OpenAPI.
- `@Info`: título, versão, descrição e contato.
- `@Server`: endereço base da API.
- `@SecurityScheme`: declara um mecanismo de autenticação que o Swagger UI poderá utilizar.
- `name = "bearerAuth"`: identificador usado para referenciar esse esquema.
- `type = SecuritySchemeType.HTTP`: indica autenticação HTTP.
- `scheme = "bearer"`: usa o padrão Bearer Token.
- `bearerFormat = "JWT"`: informa que o token utilizado é JWT.
- `@SecurityRequirement`: associa o esquema declarado às operações documentadas.

**Atenção ao escopo:** colocar `security = @SecurityRequirement(name = "bearerAuth")` dentro de `@OpenAPIDefinition` aplica o requisito globalmente à documentação. O Swagger passa a representar todas as operações como protegidas. Se houver endpoints públicos, prefira declarar o requisito nos controllers/métodos protegidos, ou remover o requisito das operações públicas com a anotação apropriada.

### 8.3 Declarar segurança não implementa autenticação

Essa configuração apenas informa ao Swagger como representar e enviar o token. Ela **não cria JWT, não valida assinatura, não configura o Spring Security e não protege endpoints**.

A proteção real precisa existir na configuração do Spring Security, com um mecanismo que valide o token enviado em cada requisição.

## 9. Swagger com Spring Security e JWT

Quando sua API usa JWT, o Swagger UI precisa conseguir:

1. Abrir a documentação.
2. Permitir que o usuário informe um token JWT.
3. Enviar o token no cabeçalho `Authorization` ao executar endpoints protegidos.

O cabeçalho enviado será:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

### 9.1 Liberar as rotas do Swagger

Na configuração real do Spring Security, libere as rotas da documentação. Exemplo ilustrativo para uma API stateless com JWT:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(
                "/swagger-ui.html",
                "/swagger-ui/**",
                "/v3/api-docs/**"
            ).permitAll()

            .requestMatchers(HttpMethod.GET, "/api/employees/**")
                .hasRole("EMPLOYEE")

            .requestMatchers(HttpMethod.POST, "/api/employees/**")
                .hasRole("MANAGER")

            .requestMatchers(HttpMethod.PUT, "/api/employees/**")
                .hasRole("MANAGER")

            .requestMatchers(HttpMethod.DELETE, "/api/employees/**")
                .hasRole("ADMIN")

            .anyRequest().authenticated()
        );

    // O mecanismo JWT da aplicação precisa ser configurado aqui.
    // Exemplo: oauth2ResourceServer com JwtDecoder ou filtro JWT próprio.

    return http.build();
}
```

Imports relevantes:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
```

**Esse trecho não é, sozinho, uma configuração JWT completa.** Ele define autorização por rota e estado stateless, mas a aplicação ainda precisa configurar a validação do JWT. Uma alternativa é usar o suporte Resource Server do Spring Security com `oauth2ResourceServer(oauth2 -> oauth2.jwt(...))`, fornecendo um `JwtDecoder` e a estratégia de conversão das claims em authorities. Se você utiliza um filtro JWT próprio, ele também precisa ser registrado corretamente na cadeia.

> `hasRole("EMPLOYEE")` normalmente procura a authority `ROLE_EMPLOYEE`. Confira como as roles/authorities são criadas a partir das claims do token.

### 9.2 Como o Swagger envia o token

Depois de declarar o `@SecurityScheme` e associá-lo às operações:

1. Abra o Swagger UI.
2. Clique em **Authorize**.
3. Cole o JWT no campo.
4. Clique em **Authorize** e feche a janela.
5. Execute uma operação protegida.

Como o esquema é HTTP Bearer, normalmente você deve colar **somente o token**, sem escrever `Bearer ` antes dele. O Swagger adiciona o prefixo automaticamente.

### 9.3 Segurança global ou por controller?

**Estratégia A — segurança global:**

```java
@OpenAPIDefinition(
    security = @SecurityRequirement(name = "bearerAuth")
)
```

Use quando praticamente todos os endpoints exigem autenticação. Endpoints públicos precisam de tratamento específico na documentação.

**Estratégia B — segurança somente nos controllers protegidos:**

```java
@RestController
@RequestMapping("/api/employees")
@SecurityRequirement(name = "bearerAuth")
public class EmployeeController {
    // Endpoints protegidos
}
```

Para um endpoint público, não declare o requisito nele. Essa abordagem costuma ser mais clara quando a API mistura endpoints públicos e privados.

### 9.4 Basic Auth não é Bearer JWT

| Basic Auth | Bearer JWT |
|---|---|
| Envia usuário e senha codificados em Base64 | Envia um token no cabeçalho |
| O cliente normalmente envia credenciais em cada requisição | O cliente envia o token emitido após autenticação |
| OpenAPI `scheme = "basic"` | OpenAPI `scheme = "bearer"` |
| Não é JWT | Pode transportar um JWT |

Base64 não é criptografia. Basic Auth deve ser usado sobre HTTPS. Não misture a configuração Basic Auth do exemplo antigo com Bearer JWT sem intenção.

### 9.5. Outros esquemas de autenticação

Além do **Bearer Token com JWT**, existem outros mecanismos de autenticação e autorização. Cada um resolve necessidades diferentes. É importante entender que alguns são esquemas de autenticação HTTP, enquanto outros são protocolos ou frameworks mais completos.

#### 9.5.1. Basic Authentication

O **Basic Auth** é um mecanismo simples de autenticação HTTP. O cliente envia usuário e senha no cabeçalho `Authorization`.

```http
Authorization: Basic dXN1YXJpbzpzZW5oYQ==
```

O valor depois de `Basic` é a codificação Base64 de `usuario:senha`.

**Atenção:** Base64 não é criptografia. Qualquer pessoa que consiga capturar o valor pode decodificá-lo. Por isso, Basic Auth deve ser utilizado somente com HTTPS.

Configuração no OpenAPI:

```java
@SecurityScheme(
    name = "basicAuth",
    type = SecuritySchemeType.HTTP,
    scheme = "basic"
)
```

No Swagger UI, o usuário informa as credenciais nos campos próprios de autenticação Basic.

#### 9.5.2. Bearer Authentication

**Bearer** significa, nesse contexto, que quem possui o token pode apresentá-lo para acessar um recurso protegido. O cliente envia o token no cabeçalho:

```http
Authorization: Bearer <token>
```

Bearer define **como a credencial é enviada**. O token pode ser um JWT ou um token opaco, que não revela seus dados internamente.

Configuração no OpenAPI:

```java
@SecurityScheme(
    name = "bearerAuth",
    type = SecuritySchemeType.HTTP,
    scheme = "bearer",
    bearerFormat = "JWT"
)
```

O `bearerFormat = "JWT"` é uma indicação para a documentação. Ele não obriga o token a ser JWT nem configura sua validação.

#### 9.5.3. JWT (JSON Web Token)

JWT é um formato compacto de token que pode transportar informações (claims) sobre o usuário e a autorização concedida.

Um JWT costuma ter três partes separadas por pontos:

```text
header.payload.signature
```

- **Header:** informa o tipo de token e o algoritmo utilizado.
- **Payload:** contém as claims, como identificador do usuário, permissões e expiração.
- **Signature:** permite verificar a integridade e a autenticidade do token.

A assinatura não significa que o conteúdo está criptografado. Em JWTs assinados comuns, o payload pode ser decodificado e lido. Portanto, não coloque senhas ou informações secretas nele.

Fluxo simplificado:

```mermaid
sequenceDiagram
    participant C as Cliente
    participant A as API
    C->>A: Envia usuário e senha
    A->>A: Valida as credenciais
    A-->>C: Retorna um access token
    C->>A: Requisição com Authorization: Bearer token
    A->>A: Valida assinatura, expiração e permissões
```

#### 9.5.4. API Keys (Chaves de API)

Uma **API Key** é uma string única usada para identificar o cliente ou aplicação que está fazendo a requisição (não o usuário final).

Configuração no OpenAPI (Header, Query ou Cookie):

```java
@SecurityScheme(
    name = "apiKeyAuth",
    type = SecuritySchemeType.APIKEY,
    in = SecuritySchemeIn.HEADER,
    name = "X-API-Key"
)
```

##### Como funciona no Java (Spring Boot) e na aplicação

1. **Geração e Envio:** O cliente envia a chave no cabeçalho `X-API-Key` (ou parâmetro de query) em cada requisição HTTP.
2. **Interceptação:** Um filtro (`OncePerRequestFilter`) intercepta a requisição antes de chegar ao controller.
3. **Validação:** A aplicação compara a chave enviada com um valor de configuração ou registro em banco de dados/cache (preferencialmente armazenado como hash).
4. **Autorização:** Se válida, o contexto de segurança (`SecurityContextHolder`) é populado; caso contrário, a requisição é rejeitada (`401` ou `403`).

Exemplo de implementação do filtro:

```java
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.AuthorityUtils;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.filter.OncePerRequestFilter;
import java.io.IOException;

public class ApiKeyAuthFilter extends OncePerRequestFilter {

    private static final String API_KEY_HEADER = "X-API-Key";
    private final String validApiKey;

    public ApiKeyAuthFilter(String validApiKey) {
        this.validApiKey = validApiKey;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain filterChain) throws ServletException, IOException {
        
        String requestApiKey = request.getHeader(API_KEY_HEADER);

        if (requestApiKey != null && requestApiKey.equals(validApiKey)) {
            UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
                    "API-Client", 
                    null, 
                    AuthorityUtils.createAuthorityList("ROLE_API_CONSUMER")
            );
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }
}
```

#### 9.5.5. OAuth 2.0 e OpenID Connect (OIDC)

O **OAuth 2.0** é um framework de autorização que permite a uma aplicação obter acesso limitado a contas de usuário em um serviço HTTP. O **OpenID Connect** é uma camada de identidade construída sobre o OAuth 2.0.

Configuração no OpenAPI:

```java
@SecurityScheme(
    name = "oauth2Auth",
    type = SecuritySchemeType.OAUTH2,
    flows = @OAuthFlows(
        authorizationCode = @OAuthFlow(
            authorizationUrl = "https://seu-provedor.com/oauth/authorize",
            tokenUrl = "https://seu-provedor.com/oauth/token",
            scopes = {
                @OAuthScope(name = "read", description = "Acesso de leitura"),
                @OAuthScope(name = "write", description = "Acesso de escrita")
            }
        )
    )
)
```

No Swagger UI, ao clicar em **Authorize**, a interface abre a página de login do Provedor de Identidade (ex: Keycloak, Auth0, Okta), realiza o fluxo OAuth2 e injeta automaticamente o token retornado.

#### 9.5.6. Autenticação por Sessão (Session / Cookie-based)

Tradicional em aplicações web monolíticas. Após o login, o servidor cria uma sessão e envia um cookie (`Set-Cookie`) com o ID da sessão (`JSESSIONID`). O navegador reenvia esse cookie automaticamente em cada requisição.

Configuração no OpenAPI:

```java
@SecurityScheme(
    name = "cookieAuth",
    type = SecuritySchemeType.APIKEY,
    in = SecuritySchemeIn.COOKIE,
    name = "JSESSIONID"
)
```

#### 9.5.7. mTLS (Mutual TLS)

No **mTLS**, tanto o servidor quanto o cliente apresentam certificados digitais durante o handshake TLS. A autenticação ocorre na camada de transporte (Rede/TLS), antes do processamento HTTP.

* **Suporte no Swagger UI:** Não possui campo interativo no Swagger UI, pois a autenticação é gerenciada diretamente pelo navegador/cliente HTTP via certificados de sistema instalado.

#### 9.5.8. HMAC (Hash-based Message Authentication Code)

Usado principalmente para garantir **integridade** e **autenticidade** em Webhooks (ex: Stripe, GitHub). O cliente assina o corpo da requisição com uma chave secreta e um algoritmo de hash (SHA-256) e envia a assinatura em um header customizado.

* **Suporte no Swagger UI:** Como o HMAC exige calcular um hash dinâmico a cada requisição alterada, não é testável diretamente de forma nativa na interface estática do Swagger UI sem scripts customizados.

---

## 10. application.properties

```properties
# Caminho da interface Swagger UI
springdoc.swagger-ui.path=/docs

# Caminho do documento OpenAPI JSON
springdoc.api-docs.path=/api-docs

# Ordenação dos endpoints e das tags
springdoc.swagger-ui.operations-sorter=method
springdoc.swagger-ui.tags-sorter=alpha

# Exibe o tempo de resposta
springdoc.swagger-ui.display-request-duration=true

# Documenta somente caminhos específicos
springdoc.paths-to-match=/api/**

# Limita a busca aos controllers de um pacote
springdoc.packages-to-scan=com.exemplo.controller
```

| Propriedade | Finalidade |
|---|---|
| `springdoc.swagger-ui.path` | Personaliza o caminho da interface |
| `springdoc.api-docs.path` | Personaliza o caminho do JSON |
| `springdoc.paths-to-match` | Filtra os caminhos documentados |
| `springdoc.packages-to-scan` | Filtra os pacotes analisados |
| `springdoc.swagger-ui.enabled` | Habilita/desabilita o Swagger UI |
| `springdoc.api-docs.enabled` | Habilita/desabilita o documento OpenAPI |

Se alterar os caminhos padrão, lembre-se de liberar os novos caminhos no Spring Security. Por exemplo, se a documentação JSON passar para `/api-docs`, liberar apenas `/v3/api-docs/**` não será suficiente.

## 11. Testando no Swagger UI

### 11.1 Cadastro (`POST`)

```http
POST /api/employees
```

```json
{
  "name": "Gabriel Silva",
  "role": "Desenvolvedor Backend"
}
```

Resposta esperada: `201 Created`, com o funcionário no corpo e o cabeçalho `Location` apontando para o recurso criado.

### 11.2 Consulta por ID (`GET`)

```http
GET /api/employees/1
```

- `200 OK`: funcionário encontrado.
- `404 Not Found`: funcionário não encontrado.

### 11.3 Atualização (`PUT`)

```http
PUT /api/employees/1
```

```json
{
  "name": "Gabriel Reyes",
  "role": "Engenheiro de Software"
}
```

Resposta esperada: `200 OK` com os dados atualizados.

### 11.4 Remoção (`DELETE`)

```http
DELETE /api/employees/1
```

Resposta esperada: `204 No Content`, sem corpo.

### 11.5 Validar dados inválidos

Envie um `POST` com nome vazio:

```json
{
  "name": "",
  "role": "Desenvolvedor"
}
```

Com `@Valid` e `@NotBlank`, a validação deve rejeitar a requisição. O status esperado é `400 Bad Request`, desde que o tratamento de validação esteja configurado corretamente.

### 11.6 Testar autenticação e autorização

| Cenário | Resultado típico |
|---|---|
| Sem token em rota protegida | `401 Unauthorized` |
| Token inválido ou expirado | `401 Unauthorized` |
| Token válido, mas sem permissão | `403 Forbidden` |
| Token válido com permissão necessária | Status definido pelo endpoint |
| Rota pública sem token | Resposta normal da rota |

## 12. Erros comuns

| Problema | Causa provável | O que verificar |
|---|---|---|
| Swagger UI retorna `404` | Dependência ou caminho incorreto | Versão do springdoc e URL |
| Swagger retorna `401` | Security bloqueia a documentação | Liberar UI e OpenAPI docs |
| UI abre, mas endpoint retorna `401` | Token não enviado ou inválido | Authorize, expiração e assinatura |
| Endpoint retorna `403` | Usuário sem autoridade suficiente | Roles e authorities do JWT |
| Botão Authorize não aparece | Esquema não declaredo ou associado | `@SecurityScheme` e `@SecurityRequirement` |
| Schema não aparece como esperado | DTO não está referenciado ou faltam metadados | Tipos de retorno e `@Schema` |
| Campo aparece como opcional | Falta de validação ou metadados | `@NotNull`, `@NotBlank`, `requiredMode` |
| Swagger funciona em um perfil, mas não em outro | Caminhos ou propriedades diferentes | `application-<profile>.properties` |
| Erro `NoClassDefFoundError` | Dependência incompatível | Compatibilidade Spring Boot/springdoc |

## 13. Boas práticas para produção

- **Não exponha documentação publicamente sem necessidade.** Desabilite ou proteja Swagger UI e OpenAPI em produção conforme a política do projeto.
- **Não coloque tokens reais em exemplos.** Use exemplos fictícios.
- **Não exponha entidades JPA diretamente.** Prefira DTOs para controlar os dados de entrada e saída.
- **Documente erros.** Inclua respostas como `400`, `401`, `403`, `404` e `500` quando forem possíveis.
- **Mantenha os DTOs coerentes.** DTOs de criação, atualização e resposta podem ter necessidades diferentes.
- **Não confunda documentação com segurança real.** `@SecurityScheme` não valida tokens.
- **Use HTTPS.** JWT é um bearer token: quem o possuir poderá utilizá-lo enquanto estiver válido e aceito pela API.
- **Não registre tokens em logs.** Evite expor credenciais em logs, mensagens de erro ou monitoramento.
- **Revise os exemplos.** A documentação deve corresponder ao comportamento real da API.

Para desabilitar a documentação em um perfil de produção:

```properties
springdoc.api-docs.enabled=false
springdoc.swagger-ui.enabled=false
```

Ajuste essa decisão à necessidade da equipe e à política de segurança do ambiente.

## 14. Checklist e referências

- [ ] Adicionar a dependência springdoc compatível com o Spring Boot.
- [ ] Abrir `/swagger-ui.html` e `/v3/api-docs`.
- [ ] Definir informações gerais com `@OpenAPIDefinition`.
- [ ] Documentar controllers com `@Tag` e `@Operation`.
- [ ] Documentar respostas com `@ApiResponse` e `@ApiResponses`.
- [ ] Documentar DTOs com `@Schema`.
- [ ] Separar DTOs de entrada e saída.
- [ ] Declarar `bearerAuth` com `@SecurityScheme`.
- [ ] Associar o esquema com `@SecurityRequirement`.
- [ ] Liberar as rotas da documentação no Spring Security.
- [ ] Configurar a validação real dos JWTs na aplicação.
- [ ] Testar `401`, `403` e permissões por role.
- [ ] Revisar o acesso à documentação em produção.

### Referências

- [springdoc-openapi — documentação oficial](https://springdoc.org/)
- [springdoc-openapi — GitHub](https://github.com/springdoc/springdoc-openapi)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Swagger Annotations — swagger-core](https://github.com/swagger-api/swagger-core/wiki/Swagger-2.X---Annotations)
- [Spring Security — OAuth2 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
- [Spring Security — Authorization](https://docs.spring.io/spring-security/reference/servlet/authorization/index.html)
