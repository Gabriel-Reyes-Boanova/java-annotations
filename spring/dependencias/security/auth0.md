## Spring Security: JWT, Auth0 e outros meios de autenticar requisições

![Java](https://img.shields.io/badge/Java-17%2B-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.x-6DB33F?logo=springboot&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring%20Security-7.x-6DB33F?logo=springsecurity&logoColor=white)
![JWT](https://img.shields.io/badge/JWT-RFC%207519-000000?logo=jsonwebtokens&logoColor=white)
![Auth0](https://img.shields.io/badge/Auth0-java--jwt%20%7C%20OAuth2-EB5424?logo=auth0&logoColor=white)

Continuação do guia de `UserDetails` e `UserDetailsService`. Lá, o login usava Basic Auth: o cliente mandava e-mail e senha em **toda** requisição. Aqui damos o passo seguinte: autenticar **uma vez** e usar um **token** nas próximas requisições. Vamos ver o **JWT**, a **biblioteca da Auth0 (`java-jwt`)**, o **JJWT**, o **suporte nativo do Spring (Resource Server)**, a **Auth0 como provedor de identidade** e **outros meios** de autenticar requisições, sempre com foco em **como isso é usado hoje no mercado** e em **como se junta com as outras bibliotecas do Spring**.

> [!IMPORTANT]
> Este guia mostra **vários caminhos** para o mesmo objetivo. Não é para usar todos ao mesmo tempo: escolha um conforme o seu projeto (veja [Caminhos para usar JWT no Spring](#caminhos-para-usar-jwt-no-spring) e [Como isso é usado no mercado](#como-isso-é-usado-no-mercado)). Os exemplos pressupõem o modelo de usuário do guia anterior (`Usuario`, `Perfil`, `UsuarioAutenticado`, `UsuarioDetailsService`).

## 📑 Sumário

- [Do UserDetailsService ao token](#do-userdetailsservice-ao-token)
- [JWT em profundidade](#jwt-em-profundidade)
  - [Estrutura e assinatura](#estrutura-e-assinatura)
  - [Access token e refresh token](#access-token-e-refresh-token)
  - [O que validar em um token](#o-que-validar-em-um-token)
  - [Onde guardar o token no cliente](#onde-guardar-o-token-no-cliente)
- [Caminhos para usar JWT no Spring](#caminhos-para-usar-jwt-no-spring)
- [Outros meios de autenticar requisições](#outros-meios-de-autenticar-requisições)
  - [API Key (exemplo completo)](#api-key-exemplo-completo)
  - [Token opaco com introspecção](#token-opaco-com-introspecção)
- [A biblioteca Auth0 (java-jwt)](#a-biblioteca-auth0-java-jwt)
  - [Instalação](#instalação)
  - [Exemplo simples, sem Spring](#exemplo-simples-sem-spring)
  - [Exemplo completo com Spring Security](#exemplo-completo-com-spring-security)
  - [Carregar do banco ou usar só as claims](#carregar-do-banco-ou-usar-só-as-claims)
- [JJWT: a outra biblioteca popular](#jjwt-a-outra-biblioteca-popular)
- [JWT nativo do Spring: Resource Server](#jwt-nativo-do-spring-resource-server)
  - [Com chave simétrica (HS256)](#com-chave-simétrica-hs256)
  - [Com chaves RSA (RS256)](#com-chaves-rsa-rs256)
  - [Juntando com o UserDetailsService](#juntando-com-o-userdetailsservice)
- [Auth0 como provedor de identidade](#auth0-como-provedor-de-identidade)
  - [OAuth 2.0 e OpenID Connect em poucas linhas](#oauth-20-e-openid-connect-em-poucas-linhas)
  - [Configurando a API Spring](#configurando-a-api-spring)
  - [Permissões e roles](#permissões-e-roles)
  - [Ligando o usuário do provedor ao seu banco](#ligando-o-usuário-do-provedor-ao-seu-banco)
  - [Outros provedores](#outros-provedores)
  - [Rodando o seu próprio servidor de autorização](#rodando-o-seu-próprio-servidor-de-autorização)
- [Fazendo requisições com tokens](#fazendo-requisições-com-tokens)
  - [curl, arquivo .http e front-end](#curl-arquivo-http-e-front-end)
  - [De uma API para outra (RestClient)](#de-uma-api-para-outra-restclient)
- [Como isso é usado no mercado](#como-isso-é-usado-no-mercado)
- [Integração com outras bibliotecas do Spring](#integração-com-outras-bibliotecas-do-spring)
- [Segurança do JWT](#segurança-do-jwt)
- [Testando](#testando)
- [Erros comuns](#erros-comuns)
- [Cuidados importantes](#cuidados-importantes)
- [Indo para produção](#indo-para-produção)
- [Referências](#-referências)

## Do UserDetailsService ao token

No guia anterior, o `UserDetailsService` era chamado **a cada requisição**, porque o Basic Auth reenvia o e-mail e a senha toda vez (e o `BCrypt` é lento de propósito). Com um token, o fluxo muda: o login acontece **uma vez**, e as demais requisições carregam apenas o token.

| Etapa | Guia | Como o cliente se identifica | Peça principal |
|---|---|---|---|
| 1 | Configuração inicial | Basic Auth | `InMemoryUserDetailsManager` |
| 2 | Autenticação com JDBC | Basic Auth | `JdbcUserDetailsManager` |
| 3 | `UserDetails` e `UserDetailsService` | Basic Auth | `UserDetailsService` próprio |
| **4** | **Este guia** | **Token (`Authorization: Bearer ...`)** | **JWT, `java-jwt`, Resource Server, Auth0** |

```mermaid
flowchart LR
    A["Basic Auth<br/>e-mail + senha<br/>em TODA requisição"] -->|"evolui para"| B["Login uma vez<br/>POST /api/auth/login"]
    B --> C["Servidor devolve<br/>access token + refresh token"]
    C --> D["Próximas requisições<br/>Authorization: Bearer token"]
```

> [!NOTE]
> O que você já construiu **continua valendo**: o `UsuarioDetailsService`, o `PasswordEncoder` e o `UsuarioAutenticado` seguem sendo usados **no login**. O token apenas substitui o reenvio da senha nas requisições seguintes.

## JWT em profundidade

### Estrutura e assinatura

Um JWT (*JSON Web Token*, definido na RFC 7519) tem **três partes** em Base64URL, separadas por ponto:

```text
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbmFAZW1haWwuY29tIn0.k3Qx...
└───── header ─────┘ └────────── payload ───────────┘ └ assinatura ┘
```

| Parte | Conteúdo | Exemplo |
|---|---|---|
| **Header** | Algoritmo de assinatura (`alg`), tipo (`typ`) e, às vezes, o identificador da chave (`kid`) | `{"alg":"RS256","typ":"JWT","kid":"chave-1"}` |
| **Payload** | As *claims*: dados sobre o usuário e o token | `{"sub":"ana@email.com","exp":1767226500}` |
| **Assinatura** | Calculada com a chave sobre `header.payload`. Prova que o token **não foi alterado** | Bytes em Base64URL |

> [!WARNING]
> O payload é **codificado, e não criptografado**: qualquer pessoa com o token consegue **ler** o conteúdo (cole um token em [jwt.io](https://jwt.io) e veja). **Nunca coloque senha, CPF, cartão ou dados sensíveis em um JWT.** A assinatura impede **alterar**, e não **ler**. (Existe o JWE, um JWT criptografado, mas é raro em APIs comuns.)

#### Claims padrão (registradas)

| Claim | Nome | Significado |
|---|---|---|
| `iss` | *issuer* | Quem emitiu o token |
| `sub` | *subject* | De quem é o token (e-mail, id) |
| `aud` | *audience* | Para quem o token se destina (qual API) |
| `exp` | *expiration* | Quando **expira** (segundos desde 1970) |
| `nbf` | *not before* | Só vale **depois** desta data |
| `iat` | *issued at* | Quando foi emitido |
| `jti` | *JWT ID* | Identificador único do token (útil para revogação) |

Além delas, você cria as suas (`uid`, `roles`, `tenant`...). Em tokens de provedores OAuth2 aparecem também `scope`/`scp` e `permissions`.

#### Algoritmos de assinatura

| Tipo | Algoritmos | Como funciona | Quando usar |
|---|---|---|---|
| **Simétrico** (HMAC) | `HS256`, `HS384`, `HS512` | **Uma chave secreta** assina e valida | Um único sistema emite **e** valida |
| **Assimétrico** (RSA) | `RS256`, `RS384`, `RS512` | **Chave privada** assina; **chave pública** valida | Quem emite é diferente de quem valida (provedores, microsserviços) |
| **Assimétrico** (curva elíptica) | `ES256`, `ES384`, `ES512` | Igual ao RSA, com chaves menores | Mesmo uso do RSA, quando há suporte |

```mermaid
flowchart LR
    subgraph H["HS256 (simétrico)"]
        A1["Servidor<br/>chave secreta"] -->|"assina e valida"| A2["Mesma chave<br/>nos dois lados"]
    end
    subgraph R["RS256 (assimétrico)"]
        B1["Emissor<br/>chave PRIVADA"] -->|"assina"| B2["Token"]
        B2 -->|"validam"| B3["APIs<br/>chave PÚBLICA"]
    end
```

> [!TIP]
> Com **HS256**, **todo** serviço que valida também **consegue emitir** tokens (tem a mesma chave). Com **RS256**, os serviços só têm a chave **pública**: podem validar, mas não emitir. Por isso o mercado usa RS256 quando há mais de um serviço ou um provedor de identidade (Auth0, Keycloak...).

Os provedores publicam as chaves públicas em uma URL chamada **JWKS** (*JSON Web Key Set*, como `https://seu-dominio/.well-known/jwks.json`). Cada chave tem um `kid`, e o token indica qual usar. Isso permite **trocar chaves** sem derrubar o sistema.

### Access token e refresh token

| | **Access token** | **Refresh token** |
|---|---|---|
| Para que serve | Acessar a API | Obter um **novo** access token |
| Formato | Normalmente um **JWT** | Normalmente um **texto aleatório opaco** |
| Vida | **Curta** (minutos) | **Longa** (dias) |
| Enviado para | Toda requisição à API | **Só** ao endpoint de renovação |
| Guardado no servidor? | Não (é validado pela assinatura) | **Sim** (o hash), para poder revogar |
| Se roubado | Vale só até expirar | Pior: renova o acesso. Por isso há **rotação** e revogação |

```mermaid
sequenceDiagram
    participant C as Cliente
    participant API as API

    C->>API: POST /login (e-mail + senha)
    API-->>C: access token (15 min) + refresh token (7 dias)
    C->>API: GET /pedidos (Bearer access token)
    API-->>C: 200 OK
    Note over C,API: 15 minutos depois, o access token expira
    C->>API: GET /pedidos (Bearer access token expirado)
    API-->>C: 401 Unauthorized
    C->>API: POST /refresh (refresh token)
    API-->>C: NOVO access token + NOVO refresh token (o antigo é revogado)
    C->>API: GET /pedidos (Bearer novo access token)
    API-->>C: 200 OK
```

### O que validar em um token

Receber um token **não basta**: é preciso conferir várias coisas.

| Verificação | Por quê |
|---|---|
| **Assinatura** | Garante que o token não foi forjado nem alterado |
| **Algoritmo esperado** | Evita ataques de troca de algoritmo (veja [Segurança do JWT](#segurança-do-jwt)) |
| **`exp`** (expiração) | Token vencido não vale |
| **`nbf`** | Token que ainda não começou a valer |
| **`iss`** (emissor) | Só aceite tokens de quem você confia |
| **`aud`** (audiência) | O token foi feito **para a sua API**, e não para outra |
| **Usuário ainda ativo** (opcional) | O token vale, mas a conta pode ter sido desativada depois |

### Onde guardar o token no cliente

| Lugar | Vantagem | Risco |
|---|---|---|
| **Memória** (variável do aplicativo) | Não persiste: mais seguro contra roubo posterior | Some ao recarregar a página (precisa de refresh) |
| **`localStorage` / `sessionStorage`** | Simples | Qualquer JavaScript injetado (**XSS**) lê o token |
| **Cookie `HttpOnly` + `Secure` + `SameSite`** | O JavaScript **não consegue ler** o cookie | Volta a exigir proteção **CSRF** |
| **BFF** (o servidor do front guarda o token) | O navegador nunca vê o token | Mais uma peça na arquitetura |

> [!NOTE]
> Não há resposta perfeita. Em aplicações reais, é comum guardar o **access token em memória** e o **refresh token em cookie `HttpOnly`**, ou usar o padrão **BFF** (veja [Como isso é usado no mercado](#como-isso-é-usado-no-mercado)).

## Caminhos para usar JWT no Spring

Existem várias formas de trabalhar com JWT em uma aplicação Spring. Elas diferem em **quem emite o token**, **quem o valida** e **quanta coisa o Spring faz por você**.

| Caminho | Quem emite | Quem valida | Dependência | Quando faz sentido |
|---|---|---|---|---|
| **A. `java-jwt` (Auth0)** com filtro próprio | Sua API | Seu filtro | `com.auth0:java-jwt` | Projetos simples, estudo, controle total |
| **B. JJWT** com filtro próprio | Sua API | Seu filtro | `io.jsonwebtoken:jjwt-*` | Igual ao A, com outra API fluente |
| **C. Resource Server nativo** (Nimbus) | Sua API (`JwtEncoder`) | **Spring Security** (`JwtDecoder`) | Starter OAuth2 Resource Server | Token próprio, com **menos código** e padrões do Spring |
| **D. Provedor externo** (Auth0, Keycloak, Cognito, Okta, Entra ID) | **O provedor** | **Spring Security** (Resource Server) | Starter OAuth2 Resource Server | **Aplicações novas em ambiente profissional** |
| **E. Servidor de autorização próprio** (Spring Authorization Server) | Você, como provedor | Resource Servers | Spring Authorization Server | Quando precisa ser o provedor de identidade |

```mermaid
flowchart TD
    A{"Quem vai emitir<br/>os tokens?"} -->|"Um provedor externo<br/>(Auth0, Keycloak...)"| D["Caminho D<br/>Resource Server + issuer-uri"]
    A -->|"A minha própria API"| B{"Quero escrever<br/>o filtro de validação?"}
    A -->|"Eu serei o provedor<br/>de identidade"| E["Caminho E<br/>Spring Authorization Server"]
    B -->|"Não, prefiro o padrão do Spring"| C["Caminho C<br/>Resource Server nativo"]
    B -->|"Sim, quero controle total"| F{"Qual biblioteca?"}
    F --> A1["Caminho A<br/>java-jwt (Auth0)"]
    F --> B1["Caminho B<br/>JJWT"]
```

> [!TIP]
> Em estudo, o **caminho A** ensina o que acontece por baixo. Em projetos profissionais, os caminhos **C** e **D** são os mais comuns, porque delegam ao Spring Security a validação (a parte onde mais se erra). Este guia mostra todos, na ordem A, B, C, D, E.

## Outros meios de autenticar requisições

O JWT é só **uma** das formas de o cliente provar quem é. Conheça as outras:

| Meio | Como o cliente envia | Como funciona | Onde é usado |
|---|---|---|---|
| **Basic Auth** | `Authorization: Basic base64(usuario:senha)` | Reenvia a senha em toda requisição | Estudo, ferramentas internas, testes (**só com HTTPS**) |
| **Sessão + cookie** | `Cookie: JSESSIONID=...` | O servidor guarda a sessão | Sites com páginas no servidor (Thymeleaf), sistemas internos |
| **API Key** | `X-API-Key: chave` | Chave fixa que identifica um **sistema** | Integrações entre sistemas, APIs públicas com plano |
| **Bearer JWT** | `Authorization: Bearer eyJ...` | Token assinado, validado sem consultar o banco | APIs REST, SPAs, aplicativos móveis |
| **Bearer opaco** | `Authorization: Bearer abc123` | Texto aleatório, validado **perguntando** ao emissor | Quando é preciso revogar na hora |
| **OAuth 2.0 / OIDC** | Bearer token vindo de um provedor | Delegação de login a um provedor de identidade | Praticamente todo sistema profissional novo |
| **Client credentials** | Token obtido com `client_id` + `client_secret` | Sistema autentica **como sistema** (sem usuário) | Comunicação entre serviços (*machine to machine*) |
| **mTLS** | Certificado do cliente | Cliente e servidor se autenticam por certificados | Bancos, ambientes de alta segurança |
| **Assinatura HMAC** | Cabeçalho com a assinatura do corpo | O servidor refaz a assinatura e compara | *Webhooks* (Stripe, GitHub, pagamentos) |

> [!NOTE]
> As opções se combinam: um mesmo sistema pode aceitar **JWT** de usuários, **API Key** de parceiros e **client credentials** entre os seus serviços, cada um em uma `SecurityFilterChain` ou em rotas diferentes.

### API Key (exemplo completo)

Útil para identificar **sistemas parceiros** (e não pessoas). Um filtro lê o cabeçalho e, se a chave for válida, autentica o sistema.

`security/apikey/ApiKeyAuthenticationFilter.java`:

```java
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {

    private static final String CABECALHO = "X-API-Key";

    private final ApiKeyService apiKeyService;

    public ApiKeyAuthenticationFilter(ApiKeyService apiKeyService) {
        this.apiKeyService = apiKeyService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {

        String chave = request.getHeader(CABECALHO);

        if (chave != null) {
            apiKeyService.buscarParceiro(chave).ifPresent(parceiro -> {          // chave válida?
                var autenticacao = UsernamePasswordAuthenticationToken.authenticated(
                        parceiro.nome(),                                          // principal: o nome do sistema
                        null,
                        List.of(new SimpleGrantedAuthority("ROLE_PARCEIRO")));    // a "role" do sistema parceiro
                SecurityContextHolder.getContext().setAuthentication(autenticacao);
            });
        }

        chain.doFilter(request, response);                                        // segue: sem chave, o Spring responde 401
    }
}
```

`security/apikey/ApiKeyService.java` (as chaves ficam em **hash**, como as senhas):

```java
@Service
public class ApiKeyService {

    public record Parceiro(String nome) { }

    private final ApiKeyRepository repository;

    public ApiKeyService(ApiKeyRepository repository) {
        this.repository = repository;
    }

    public Optional<Parceiro> buscarParceiro(String chaveEmTextoPuro) {
        return repository.findByChaveHashAndAtivaTrue(sha256(chaveEmTextoPuro))   // compara o HASH, e não a chave
                .map(k -> new Parceiro(k.getNome()));
    }

    // Chaves de API são aleatórias e longas: um SHA-256 simples basta (BCrypt seria lento demais)
    static String sha256(String valor) {
        try {
            byte[] hash = MessageDigest.getInstance("SHA-256").digest(valor.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

Registro na configuração, em uma cadeia **só para as rotas dos parceiros**:

```java
@Bean
@Order(1)                                                          // avaliada ANTES da cadeia principal
SecurityFilterChain parceirosChain(HttpSecurity http, ApiKeyService apiKeyService) throws Exception {
    http
        .securityMatcher("/api/parceiros/**")                      // esta cadeia só vale para estas rotas
        .csrf(csrf -> csrf.disable())
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth.anyRequest().hasRole("PARCEIRO"))
        .addFilterBefore(new ApiKeyAuthenticationFilter(apiKeyService), AuthorizationFilter.class);
    return http.build();
}
```

| Ponto | Detalhe |
|---|---|
| `securityMatcher(...)` | Permite ter **várias cadeias**, cada uma com seu meio de autenticação |
| `@Order(1)` | Define a ordem: a primeira cadeia que combinar com a rota é usada |
| Chave em **hash** | Se o banco vazar, as chaves originais não aparecem |
| Mostrar a chave **uma vez** | Ao criar, exiba a chave completa só nesse momento |

### Token opaco com introspecção

Em vez de um JWT (que carrega os dados), o token é um **texto aleatório** e a API **pergunta ao emissor** se ele é válido (*introspection*, RFC 7662). A vantagem é poder **revogar na hora**. O custo é uma chamada de rede por validação (normalmente com cache).

```properties
spring.security.oauth2.resourceserver.opaquetoken.introspection-uri=https://auth.exemplo.com/oauth2/introspect
spring.security.oauth2.resourceserver.opaquetoken.client-id=${INTROSPECTION_CLIENT_ID}
spring.security.oauth2.resourceserver.opaquetoken.client-secret=${INTROSPECTION_CLIENT_SECRET}
```

```java
http.oauth2ResourceServer(oauth2 -> oauth2.opaqueToken(Customizer.withDefaults()));
```

| | **JWT** | **Token opaco** |
|---|---|---|
| Validação | Local, pela assinatura | Chamada ao emissor |
| Revogação imediata | Difícil | **Fácil** |
| Dados no token | Sim (claims) | Não |
| Desempenho | **Melhor** (sem rede) | Depende do emissor e do cache |

## A biblioteca Auth0 (java-jwt)

A **Auth0** (empresa de identidade) mantém uma biblioteca de código aberto chamada **`java-jwt`** (`com.auth0:java-jwt`), usada para **criar, assinar, decodificar e verificar** JWTs em aplicações Java. Ela **não depende do Spring**: o Spring só entra quando você escreve o filtro que a usa.

> [!NOTE]
> Não confunda: **`java-jwt`** é uma **biblioteca** (código que você coloca no seu projeto). **Auth0** como produto é uma **plataforma de identidade** (um provedor que cuida do login), que veremos em [Auth0 como provedor de identidade](#auth0-como-provedor-de-identidade). Você pode usar a biblioteca **sem** usar a plataforma, e vice-versa.

### Instalação

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>4.5.2</version>   <!-- use a versão estável mais recente -->
</dependency>
```

| Característica | Detalhe |
|---|---|
| Algoritmos | HMAC (`HS256`, `HS384`, `HS512`), RSA (`RS256`, `RS384`, `RS512`) e ECDSA (`ES256`, `ES384`, `ES512`) |
| Uso | **Servidor** (aplicações JVM de back-end) |
| Dependências | Jackson (usado internamente para o JSON) |
| Versões do Java | O projeto declara suporte às versões LTS (8, 11, 17) |
| Classes principais | `JWT`, `Algorithm`, `JWTVerifier`, `DecodedJWT` |

### Exemplo simples, sem Spring

Para entender a biblioteca antes de juntar com o Spring:

```java
import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTVerificationException;
import com.auth0.jwt.exceptions.TokenExpiredException;
import com.auth0.jwt.interfaces.DecodedJWT;

import java.time.Instant;
import java.util.UUID;

public class ExemploJavaJwt {

    public static void main(String[] args) {

        // 1) Algoritmo + chave secreta (use uma chave longa e aleatória, vinda de variável de ambiente)
        Algorithm algoritmo = Algorithm.HMAC256("troque-por-uma-chave-aleatoria-com-mais-de-32-caracteres");

        // 2) CRIAR e assinar o token
        Instant agora = Instant.now();
        String token = JWT.create()
                .withIssuer("minha-api")                              // iss
                .withSubject("ana@email.com")                         // sub
                .withJWTId(UUID.randomUUID().toString())              // jti
                .withIssuedAt(agora)                                  // iat
                .withExpiresAt(agora.plusSeconds(900))                // exp: 15 minutos
                .withClaim("uid", 42L)                                // claim própria (número)
                .withArrayClaim("roles", new String[] {"ROLE_USER"})  // claim própria (lista)
                .sign(algoritmo);                                     // assina

        System.out.println(token);

        // 3) VERIFICAR o token (crie o verificador UMA vez e reutilize)
        JWTVerifier verificador = JWT.require(algoritmo)
                .withIssuer("minha-api")                              // exige este emissor
                .build();

        try {
            DecodedJWT jwt = verificador.verify(token);               // confere assinatura, exp, iss...

            System.out.println(jwt.getSubject());                     // ana@email.com
            System.out.println(jwt.getClaim("uid").asLong());         // 42
            System.out.println(jwt.getClaim("roles").asList(String.class));   // [ROLE_USER]

        } catch (TokenExpiredException e) {
            System.out.println("Token expirado");
        } catch (JWTVerificationException e) {
            System.out.println("Token inválido: " + e.getMessage());  // assinatura errada, emissor errado, malformado...
        }
    }
}
```

| Trecho | O que faz |
|---|---|
| `Algorithm.HMAC256(segredo)` | Define o algoritmo e a chave. Para RSA: `Algorithm.RSA256(publicKey, privateKey)` |
| `JWT.create()` | Começa a montar o token |
| `.withIssuer(...)`, `.withSubject(...)`, `.withExpiresAt(...)` | Preenchem as claims padrão |
| `.withClaim(nome, valor)` | Claim própria (texto, número, booleano, data...) |
| `.withArrayClaim(nome, itens)` | Claim própria em forma de lista |
| `.sign(algoritmo)` | Assina e devolve o texto do token |
| `JWT.require(algoritmo)` | Começa a montar o **verificador** |
| `.withIssuer(...)` / `.withAudience(...)` | Exigem esses valores no token |
| `.acceptLeeway(segundos)` | Tolera diferença de relógio entre servidores |
| `.build()` | Cria o `JWTVerifier` (**reutilize**: é seguro entre threads) |
| `verificador.verify(token)` | Confere tudo e devolve o `DecodedJWT`, ou **lança exceção** |

Exceções mais comuns (todas filhas de `JWTVerificationException`):

| Exceção | Quando acontece |
|---|---|
| `TokenExpiredException` | O `exp` já passou |
| `SignatureVerificationException` | A assinatura não confere (token alterado ou chave errada) |
| `InvalidClaimException` | Uma claim exigida (`iss`, `aud`...) não bate |
| `JWTDecodeException` | O texto não é um JWT bem formado |

> [!WARNING]
> Existe o método `JWT.decode(token)`, que **apenas lê** o conteúdo **sem verificar a assinatura**. Use-o só para inspecionar (por exemplo, para ler o `kid` do header). **Nunca autentique alguém com `decode`**: qualquer pessoa pode fabricar um token que o `decode` aceita.

### Exemplo completo com Spring Security

Agora, a **API própria** emitindo e validando tokens com a `java-jwt`. Estrutura de arquivos (somente o que é novo ou muda em relação ao guia anterior):

```text
src/main/java/br/com/exemplo/demo/
├── config/
│   └── JwtProperties.java
├── controller/
│   ├── AuthController.java
│   └── MeController.java
├── dto/
│   ├── LoginRequest.java
│   ├── RefreshRequest.java
│   └── TokenResponse.java
├── entities/
│   └── RefreshToken.java
├── exception/
│   ├── TokenInvalidoException.java
│   └── GlobalExceptionHandler.java      (novo handler de 401)
├── repository/
│   └── RefreshTokenRepository.java
├── security/
│   ├── SecurityConfig.java              (agora com o filtro JWT)
│   └── jwt/
│       ├── JwtAuthenticationFilter.java
│       └── JwtService.java
└── service/
    └── AuthService.java
```

#### 1. Configuração (`application.properties` e `JwtProperties`)

```properties
# gere com:  openssl rand -base64 48
app.jwt.secret=${JWT_SECRET}
app.jwt.issuer=minha-api
app.jwt.access-ttl=15m
app.jwt.refresh-ttl=7d
```

```java
@ConfigurationProperties(prefix = "app.jwt")
@Validated
public record JwtProperties(
        @NotBlank @Size(min = 32) String secret,        // segredo com pelo menos 32 caracteres
        @NotBlank String issuer,
        @NotNull Duration accessTtl,                    // "15m" vira Duration automaticamente
        @NotNull Duration refreshTtl                    // "7d" vira Duration
) { }
```

Registre na classe principal com `@ConfigurationPropertiesScan` (veja o guia de arquivos do projeto sobre `@ConfigurationProperties`).

> [!IMPORTANT]
> O segredo **nunca** vai no código nem no GitHub. Ele vem de uma variável de ambiente (`JWT_SECRET`), como o `.env` do guia de arquivos importantes. Quem tem esse segredo **emite tokens válidos** para qualquer usuário, inclusive administradores.

#### 2. `JwtService`: criar e validar tokens

`security/jwt/JwtService.java`:

```java
@Service
public class JwtService {

    private final JwtProperties props;
    private final Algorithm algoritmo;
    private final JWTVerifier verificador;

    public JwtService(JwtProperties props) {
        this.props = props;
        this.algoritmo = Algorithm.HMAC256(props.secret());
        this.verificador = JWT.require(algoritmo)               // criado UMA vez
                .withIssuer(props.issuer())
                .build();
    }

    public String gerarAccessToken(UsuarioAutenticado usuario) {
        Instant agora = Instant.now();

        String[] roles = usuario.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toArray(String[]::new);

        return JWT.create()
                .withIssuer(props.issuer())
                .withSubject(usuario.getUsername())             // o e-mail
                .withJWTId(UUID.randomUUID().toString())
                .withIssuedAt(agora)
                .withExpiresAt(agora.plus(props.accessTtl()))
                .withClaim("uid", usuario.getId())              // útil para regras de "dono do recurso"
                .withArrayClaim("roles", roles)
                .sign(algoritmo);
    }

    /** Valida assinatura, emissor e expiração. Lança JWTVerificationException se algo estiver errado. */
    public DecodedJWT validar(String token) {
        return verificador.verify(token);
    }
}
```

#### 3. `JwtAuthenticationFilter`: validar o token em cada requisição

O filtro lê o cabeçalho, valida o token e coloca o usuário no `SecurityContext`. Se algo falhar, ele **não responde erro**: apenas segue **sem autenticar**, e o Spring Security devolve `401` na frente do `AuthorizationFilter`.

`security/jwt/JwtAuthenticationFilter.java`:

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final String PREFIXO = "Bearer ";

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtService jwtService, UserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {

        String cabecalho = request.getHeader(HttpHeaders.AUTHORIZATION);

        if (cabecalho != null && cabecalho.startsWith(PREFIXO)
                && SecurityContextHolder.getContext().getAuthentication() == null) {

            String token = cabecalho.substring(PREFIXO.length());

            try {
                DecodedJWT jwt = jwtService.validar(token);                           // 1) assinatura, emissor, exp

                UserDetails usuario = userDetailsService.loadUserByUsername(jwt.getSubject());   // 2) usuário atual no banco

                if (usuario.isEnabled() && usuario.isAccountNonLocked()) {            // 3) conta ainda ativa?
                    var autenticacao = UsernamePasswordAuthenticationToken.authenticated(
                            usuario, null, usuario.getAuthorities());
                    autenticacao.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                    SecurityContext contexto = SecurityContextHolder.createEmptyContext();
                    contexto.setAuthentication(autenticacao);
                    SecurityContextHolder.setContext(contexto);                       // 4) "quem está logado" nesta requisição
                }

            } catch (JWTVerificationException | UsernameNotFoundException ex) {
                SecurityContextHolder.clearContext();                                 // token ruim: segue sem autenticação
            }
        }

        chain.doFilter(request, response);
    }
}
```

| Linha | O que faz |
|---|---|
| `extends OncePerRequestFilter` | Garante que o filtro rode **uma vez** por requisição |
| `startsWith("Bearer ")` | Só processa o esquema `Bearer` |
| `jwtService.validar(token)` | Confere assinatura, emissor e expiração |
| `loadUserByUsername(jwt.getSubject())` | Recarrega o usuário do banco, com as permissões **atuais** |
| `isEnabled()` / `isAccountNonLocked()` | Uma conta desativada **perde o acesso na hora**, mesmo com token válido |
| `UsernamePasswordAuthenticationToken.authenticated(...)` | Cria o `Authentication` **já autenticado** |
| `SecurityContextHolder.setContext(...)` | Disponibiliza o usuário para o resto da requisição (`@AuthenticationPrincipal`, `@PreAuthorize`) |
| `catch (...)` | Token inválido: **não** lança erro, e o Spring devolve `401` |

> [!WARNING]
> **Não anote o filtro com `@Component`.** O Spring Boot registra todo filtro que é bean no contêiner de servlets, e ele rodaria **duas vezes** (uma fora e outra dentro da cadeia do Security). Por isso criamos o filtro com `new` na configuração, como abaixo.

#### 4. `SecurityConfig` com o filtro

`security/SecurityConfig.java`:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http,
                                    JwtService jwtService,
                                    UsuarioDetailsService userDetailsService) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                                    // API stateless, sem cookies de autenticação
            .cors(Customizer.withDefaults())
            .sessionManagement(session -> session
                    .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/auth/**").permitAll()             // register, login, refresh, logout
                    .requestMatchers("/actuator/health").permitAll()
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated())
            .exceptionHandling(erros -> erros
                    .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))   // 401
                    .accessDeniedHandler((req, res, ex) -> res.setStatus(HttpServletResponse.SC_FORBIDDEN)))   // 403
            .addFilterBefore(new JwtAuthenticationFilter(jwtService, userDetailsService),
                    UsernamePasswordAuthenticationFilter.class);             // o nosso filtro entra ANTES do filtro de login por formulário

        return http.build();
    }
}
```

| Mudança em relação ao guia anterior | Por quê |
|---|---|
| Sai o `httpBasic(...)` | A autenticação agora é por **token** |
| `addFilterBefore(...)` | Coloca o filtro JWT na cadeia, na posição certa |
| `/api/auth/**` liberado | Cadastro, login e renovação precisam ser públicos |
| `exceptionHandling(...)` | Define `401` (sem autenticação) e `403` (sem permissão) |

#### 5. Refresh token: entidade e repository

O **refresh token** é um texto aleatório. No banco, guardamos **apenas o hash** (assim como as senhas), para que um vazamento do banco não entregue tokens utilizáveis.

```sql
CREATE TABLE refresh_tokens (
    id          BIGSERIAL PRIMARY KEY,
    token_hash  VARCHAR(64) NOT NULL UNIQUE,                  -- SHA-256 em hexadecimal
    usuario_id  BIGINT NOT NULL REFERENCES usuarios (id),
    expira_em   TIMESTAMP NOT NULL,
    revogado    BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_refresh_usuario ON refresh_tokens (usuario_id);
```

`entities/RefreshToken.java`:

```java
@Entity
@Table(name = "refresh_tokens")
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "token_hash", nullable = false, unique = true, length = 64)
    private String tokenHash;                                    // o hash, e NUNCA o token original

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "usuario_id", nullable = false)
    private Usuario usuario;

    @Column(name = "expira_em", nullable = false)
    private Instant expiraEm;

    @Column(nullable = false)
    private boolean revogado = false;

    protected RefreshToken() { }

    public RefreshToken(String tokenHash, Usuario usuario, Instant expiraEm) {
        this.tokenHash = tokenHash;
        this.usuario = usuario;
        this.expiraEm = expiraEm;
    }

    public boolean estaExpirado() { return expiraEm.isBefore(Instant.now()); }
    public void revogar()         { this.revogado = true; }

    public boolean isRevogado()   { return revogado; }
    public Usuario getUsuario()   { return usuario; }
}
```

`repository/RefreshTokenRepository.java`:

```java
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {

    Optional<RefreshToken> findByTokenHash(String tokenHash);

    @Modifying
    @Query("update RefreshToken r set r.revogado = true where r.usuario.id = :usuarioId and r.revogado = false")
    int revogarTodosDoUsuario(@Param("usuarioId") Long usuarioId);

    long deleteByExpiraEmBefore(Instant limite);                 // limpeza periódica
}
```

#### 6. `AuthService` e `AuthController`: login, renovação e logout

DTOs:

```java
public record LoginRequest(@NotBlank @Email String email, @NotBlank String senha) { }

public record RefreshRequest(@NotBlank String refreshToken) { }

public record TokenResponse(String accessToken, String refreshToken, String tipo, long expiraEmSegundos) { }
```

`service/AuthService.java`:

```java
@Service
public class AuthService {

    private final AuthenticationManager authenticationManager;
    private final UsuarioDetailsService userDetailsService;
    private final UsuarioRepository usuarioRepository;
    private final RefreshTokenRepository refreshTokenRepository;
    private final JwtService jwtService;
    private final JwtProperties props;

    public AuthService(AuthenticationManager authenticationManager,
                       UsuarioDetailsService userDetailsService,
                       UsuarioRepository usuarioRepository,
                       RefreshTokenRepository refreshTokenRepository,
                       JwtService jwtService,
                       JwtProperties props) {
        this.authenticationManager = authenticationManager;
        this.userDetailsService = userDetailsService;
        this.usuarioRepository = usuarioRepository;
        this.refreshTokenRepository = refreshTokenRepository;
        this.jwtService = jwtService;
        this.props = props;
    }

    /** LOGIN: o UserDetailsService e o PasswordEncoder entram em ação aqui, e só aqui. */
    @Transactional
    public TokenResponse login(LoginRequest request) {
        Authentication autenticado = authenticationManager.authenticate(
                UsernamePasswordAuthenticationToken.unauthenticated(
                        request.email().trim().toLowerCase(), request.senha()));   // lança AuthenticationException se falhar

        return emitirTokens((UsuarioAutenticado) autenticado.getPrincipal());
    }

    /** RENOVAÇÃO: troca um refresh token válido por um par novo (rotação). */
    @Transactional(noRollbackFor = TokenInvalidoException.class)    // a revogação em massa NÃO pode ser desfeita pela exceção
    public TokenResponse renovar(String refreshToken) {
        RefreshToken atual = refreshTokenRepository.findByTokenHash(hash(refreshToken))
                .orElseThrow(TokenInvalidoException::new);

        if (atual.isRevogado()) {
            // Token já usado sendo apresentado de novo: possível roubo. Revoga TODOS os tokens do usuário.
            refreshTokenRepository.revogarTodosDoUsuario(atual.getUsuario().getId());
            throw new TokenInvalidoException();
        }
        if (atual.estaExpirado()) {
            throw new TokenInvalidoException();
        }

        atual.revogar();                                            // o token antigo nunca mais vale

        try {
            UserDetails usuario = userDetailsService.loadUserByUsername(atual.getUsuario().getEmail());
            if (!usuario.isEnabled() || !usuario.isAccountNonLocked()) {
                throw new TokenInvalidoException();                 // conta desativada: não renova
            }
            return emitirTokens((UsuarioAutenticado) usuario);
        } catch (UsernameNotFoundException e) {
            throw new TokenInvalidoException();
        }
    }

    @Transactional
    public void logout(String refreshToken) {
        refreshTokenRepository.findByTokenHash(hash(refreshToken))
                .ifPresent(RefreshToken::revogar);                  // idempotente: se não existe, nada a fazer
    }

    private TokenResponse emitirTokens(UsuarioAutenticado usuario) {
        String accessToken = jwtService.gerarAccessToken(usuario);

        String refreshOpaco = gerarTokenOpaco();
        Usuario dono = usuarioRepository.getReferenceById(usuario.getId());
        refreshTokenRepository.save(new RefreshToken(
                hash(refreshOpaco), dono, Instant.now().plus(props.refreshTtl())));

        return new TokenResponse(accessToken, refreshOpaco, "Bearer", props.accessTtl().toSeconds());
    }

    /** 256 bits aleatórios, em Base64URL: impossível de adivinhar. */
    private static String gerarTokenOpaco() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }

    private static String hash(String valor) {
        try {
            byte[] digest = MessageDigest.getInstance("SHA-256").digest(valor.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(digest);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

`controller/AuthController.java` (o `register` já existe no guia anterior):

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @PostMapping("/login")
    public TokenResponse login(@Valid @RequestBody LoginRequest request) {
        return authService.login(request);
    }

    @PostMapping("/refresh")
    public TokenResponse refresh(@Valid @RequestBody RefreshRequest request) {
        return authService.renovar(request.refreshToken());
    }

    @PostMapping("/logout")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void logout(@Valid @RequestBody RefreshRequest request) {
        authService.logout(request.refreshToken());
    }
}
```

`controller/MeController.java` (uma rota **protegida**, para testar):

```java
@RestController
public class MeController {

    @GetMapping("/api/me")
    public Map<String, Object> eu(@AuthenticationPrincipal UsuarioAutenticado logado) {
        return Map.of("id", logado.getId(), "nome", logado.getNome(), "email", logado.getUsername());
    }
}
```

`exception/TokenInvalidoException.java` e o tratamento do `401`:

```java
public class TokenInvalidoException extends RuntimeException {
    public TokenInvalidoException() { super("Token inválido ou expirado"); }
}
```

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Mesma resposta genérica para qualquer falha de autenticação: não revela se o e-mail existe
    @ExceptionHandler({AuthenticationException.class, TokenInvalidoException.class})
    public ProblemDetail naoAutenticado(Exception ex) {
        ProblemDetail problema = ProblemDetail.forStatusAndDetail(HttpStatus.UNAUTHORIZED, "Credenciais inválidas");
        problema.setTitle("Não autenticado");
        return problema;
    }
}
```

> [!NOTE]
> Por que o `AuthenticationManager` aqui? `authenticationManager.authenticate(...)` **reaproveita todo o fluxo do guia anterior**: `DaoAuthenticationProvider` → `UsuarioDetailsService` → `PasswordEncoder.matches` → flags da conta (`isEnabled`, `isAccountNonLocked`...). Você **não** reimplementa a conferência de senha.

Limpeza periódica de tokens expirados (integração com o **Spring Scheduling**):

```java
@Component
public class LimpezaRefreshTokens {

    private final RefreshTokenRepository repository;

    public LimpezaRefreshTokens(RefreshTokenRepository repository) {
        this.repository = repository;
    }

    @Scheduled(cron = "0 0 3 * * *")                       // todo dia, às 3h (precisa de @EnableScheduling)
    @Transactional
    public void limparExpirados() {
        repository.deleteByExpiraEmBefore(Instant.now());
    }
}
```

### Carregar do banco ou usar só as claims

O filtro acima **consulta o banco em toda requisição** (via `loadUserByUsername`). Há uma alternativa que **não consulta**, montando o usuário só com o que está no token:

```java
DecodedJWT jwt = jwtService.validar(token);

List<GrantedAuthority> authorities = jwt.getClaim("roles").asList(String.class).stream()
        .<GrantedAuthority>map(SimpleGrantedAuthority::new)
        .toList();

PrincipalJwt principal = new PrincipalJwt(jwt.getClaim("uid").asLong(), jwt.getSubject());

var autenticacao = UsernamePasswordAuthenticationToken.authenticated(principal, null, authorities);
```

```java
public record PrincipalJwt(Long id, String email) { }      // usado com @AuthenticationPrincipal PrincipalJwt
```

| | **Carregar do banco** (filtro principal) | **Só as claims** |
|---|---|---|
| Consulta ao banco por requisição | Sim | **Não** |
| Permissões | **Sempre atuais** | As do momento da emissão (até expirar) |
| Conta desativada ou bloqueada | **Barra na hora** | Continua valendo até o token expirar |
| Desempenho | Um `SELECT` a mais | **Mais rápido** |
| `principal` | `UsuarioAutenticado` | `PrincipalJwt` (ou `Jwt`) |
| Indicado para | Sistemas pequenos e médios, onde segurança vale mais que milissegundos | Alto volume; compense com **access token de vida curta** |

> [!TIP]
> Um meio-termo comum: usar as claims para a maioria das requisições e **cache** (Spring Cache) do usuário carregado, com expiração curta. Assim, o banco é consultado uma vez por usuário a cada poucos minutos.

## JJWT: a outra biblioteca popular

O **JJWT** (`io.jsonwebtoken`) faz o mesmo que a `java-jwt`, com outra API. É muito encontrado em tutoriais e projetos existentes. Ele é dividido em **três artefatos**:

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>              <!-- confira a versão estável mais recente -->
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

O mesmo `JwtService`, agora com o JJWT:

```java
@Service
public class JwtServiceJjwt {

    private final JwtProperties props;
    private final SecretKey chave;

    public JwtServiceJjwt(JwtProperties props) {
        this.props = props;
        // aqui o segredo precisa estar em Base64 e ter pelo menos 256 bits (ex.: openssl rand -base64 48)
        this.chave = Keys.hmacShaKeyFor(Decoders.BASE64.decode(props.secret()));
    }

    public String gerarAccessToken(UsuarioAutenticado usuario) {
        Instant agora = Instant.now();

        List<String> roles = usuario.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList();

        return Jwts.builder()
                .issuer(props.issuer())
                .subject(usuario.getUsername())
                .id(UUID.randomUUID().toString())
                .issuedAt(Date.from(agora))
                .expiration(Date.from(agora.plus(props.accessTtl())))
                .claim("uid", usuario.getId())
                .claim("roles", roles)
                .signWith(chave)                                   // o algoritmo (HS256/384/512) é escolhido pelo tamanho da chave
                .compact();
    }

    public Claims validar(String token) {
        return Jwts.parser()
                .verifyWith(chave)                                 // exige que o token tenha sido assinado com esta chave
                .requireIssuer(props.issuer())
                .build()
                .parseSignedClaims(token)                          // lança JwtException se algo estiver errado
                .getPayload();
    }
}
```

| `java-jwt` (Auth0) | JJWT |
|---|---|
| `JWT.create()...sign(algoritmo)` | `Jwts.builder()...signWith(chave).compact()` |
| `JWT.require(alg).build().verify(token)` | `Jwts.parser().verifyWith(chave).build().parseSignedClaims(token)` |
| `JWTVerificationException` | `JwtException` (e `ExpiredJwtException`, `MalformedJwtException`...) |
| `HMAC256("texto")` aceita um texto qualquer | `Keys.hmacShaKeyFor(bytes)` **exige** 256 bits ou mais (senão lança `WeakKeyException`) |
| 1 dependência | 3 artefatos (`api`, `impl`, `jackson`) |
| Suporte a JWE (criptografia) | Sim |

> [!NOTE]
> O resto do projeto (filtro, `SecurityConfig`, `AuthService`) é **idêntico**: só o `JwtService` muda. Isso mostra uma boa prática: **isole a biblioteca de JWT em uma classe**, para poder trocá-la.

## JWT nativo do Spring: Resource Server

O Spring Security tem suporte próprio a JWT, baseado na biblioteca **Nimbus JOSE + JWT**, que é a mesma usada por baixo. É o **OAuth2 Resource Server**: você declara um `JwtDecoder` e o Spring cuida do filtro (`BearerTokenAuthenticationFilter`), da validação, do `401` com o cabeçalho `WWW-Authenticate: Bearer` e da criação do `Authentication`.

> [!NOTE]
> "Resource Server" é o nome OAuth2 para "a sua API, que protege recursos e exige um token". Ele funciona **mesmo sem** um provedor externo, com tokens emitidos por você.

| Dependência | Versão do Spring Boot |
|---|---|
| `spring-boot-starter-oauth2-resource-server` | **3.x** |
| `spring-boot-starter-security-oauth2-resource-server` | **4.x** (o nome mudou com a modularização dos starters) |

> [!WARNING]
> Os nomes dos starters mudam entre versões do Spring Boot. Confira o nome exato na documentação da versão que você usa.

O que você **não** escreve mais, em comparação com o caminho A:

| No caminho A (`java-jwt`) você escrevia | No caminho C (nativo) |
|---|---|
| `JwtAuthenticationFilter` | **O Spring já tem** (`BearerTokenAuthenticationFilter`) |
| Validação de assinatura e `exp` | `JwtDecoder` |
| Entry point de `401` | **Já incluso** |
| Conversão das roles | `JwtAuthenticationConverter` (configurável) |

### Com chave simétrica (HS256)

Arquivos novos: `JwtBeansConfig`, `TokenService`. O `SecurityConfig` muda. (`JwtProperties`, `AuthService`, `AuthController` e o refresh token são os mesmos do caminho A.)

`config/JwtBeansConfig.java`:

```java
@Configuration
public class JwtBeansConfig {

    @Bean
    SecretKey jwtSecretKey(JwtProperties props) {
        byte[] bytes = Base64.getDecoder().decode(props.secret());        // segredo em Base64, com 256 bits ou mais
        return new SecretKeySpec(bytes, "HmacSHA256");
    }

    @Bean
    JwtEncoder jwtEncoder(SecretKey jwtSecretKey) {
        return new NimbusJwtEncoder(new ImmutableSecret<>(jwtSecretKey)); // quem EMITE tokens
    }

    @Bean
    JwtDecoder jwtDecoder(SecretKey jwtSecretKey, JwtProperties props) {
        NimbusJwtDecoder decoder = NimbusJwtDecoder
                .withSecretKey(jwtSecretKey)
                .macAlgorithm(MacAlgorithm.HS256)                         // só aceita HS256
                .build();
        decoder.setJwtValidator(JwtValidators.createDefaultWithIssuer(props.issuer()));   // exp, nbf e emissor
        return decoder;                                                   // quem VALIDA tokens
    }

    @Bean
    JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authorities = new JwtGrantedAuthoritiesConverter();
        authorities.setAuthoritiesClaimName("roles");                     // lê a claim "roles"
        authorities.setAuthorityPrefix("");                               // já vêm como ROLE_USER, sem acrescentar "SCOPE_"

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authorities);
        return converter;
    }
}
```

`security/jwt/TokenService.java`:

```java
@Service
public class TokenService {

    private final JwtEncoder jwtEncoder;
    private final JwtProperties props;

    public TokenService(JwtEncoder jwtEncoder, JwtProperties props) {
        this.jwtEncoder = jwtEncoder;
        this.props = props;
    }

    public String gerarAccessToken(UsuarioAutenticado usuario) {
        Instant agora = Instant.now();

        List<String> roles = usuario.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList();

        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer(props.issuer())
                .subject(usuario.getUsername())
                .id(UUID.randomUUID().toString())
                .issuedAt(agora)
                .expiresAt(agora.plus(props.accessTtl()))
                .claim("uid", usuario.getId())
                .claim("roles", roles)
                .build();

        JwsHeader header = JwsHeader.with(MacAlgorithm.HS256).build();    // algoritmo no header

        return jwtEncoder.encode(JwtEncoderParameters.from(header, claims)).getTokenValue();
    }
}
```

`SecurityConfig` (o filtro `JwtAuthenticationFilter` **sai**; entra `oauth2ResourceServer`):

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http,
                                JwtAuthenticationConverter jwtAuthenticationConverter) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .cors(Customizer.withDefaults())
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter)));   // valida o Bearer token

    return http.build();
}
```

No `AuthService`, basta trocar `jwtService.gerarAccessToken(usuario)` por `tokenService.gerarAccessToken(usuario)`.

Lendo o usuário no Controller, agora o `principal` é um **`Jwt`**:

```java
@GetMapping("/api/me")
public Map<String, Object> eu(@AuthenticationPrincipal Jwt jwt) {
    return Map.of(
            "email", jwt.getSubject(),
            "id", jwt.getClaim("uid"),
            "roles", jwt.getClaim("roles"));
}
```

> [!IMPORTANT]
> O segredo HS256 do Nimbus precisa ter **pelo menos 256 bits (32 bytes)**. Gere com `openssl rand -base64 48` e guarde em variável de ambiente.

### Com chaves RSA (RS256)

Com RSA, a API **assina com a chave privada** e **valida com a pública**. É o formato usado quando há mais de um serviço: os outros só precisam da chave pública.

Gerando as chaves (a privada precisa estar no formato **PKCS#8**):

```bash
openssl genrsa -out keypair.pem 2048
openssl rsa -in keypair.pem -pubout -out public.pem
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in keypair.pem -out private.pem
```

> [!WARNING]
> Guarde `private.pem` **fora do repositório** (adicione `*.pem` ao `.gitignore`). Em produção, entregue a chave por um cofre de segredos ou volume, e não dentro do `.jar`.

```properties
app.jwt.public-key=classpath:certs/public.pem
app.jwt.private-key=file:${JWT_PRIVATE_KEY_PATH}
```

```java
@ConfigurationProperties(prefix = "app.jwt")
public record JwtRsaProperties(RSAPublicKey publicKey, RSAPrivateKey privateKey) { }
```

> [!NOTE]
> O Spring Security converte arquivos PEM em `RSAPublicKey` e `RSAPrivateKey` nas propriedades. Se na sua versão a conversão não for automática, registre os conversores do Spring Security ou leia o arquivo manualmente.

```java
@Configuration
@EnableConfigurationProperties(JwtRsaProperties.class)
public class JwtRsaBeansConfig {

    @Bean
    JwtEncoder jwtEncoder(JwtRsaProperties keys) {
        RSAKey jwk = new RSAKey.Builder(keys.publicKey())
                .privateKey(keys.privateKey())
                .keyID("minha-chave-1")                                    // o "kid" que vai no header
                .build();
        JWKSource<SecurityContext> fonte = new ImmutableJWKSet<>(new JWKSet(jwk));
        return new NimbusJwtEncoder(fonte);                                // usa RS256 por padrão para chaves RSA
    }

    @Bean
    JwtDecoder jwtDecoder(JwtRsaProperties keys) {
        return NimbusJwtDecoder.withPublicKey(keys.publicKey()).build();   // valida com a chave PÚBLICA
    }
}
```

Um **segundo serviço** (que só valida) precisa apenas da chave pública. Com o Spring Boot basta uma propriedade, sem escrever bean:

```properties
spring.security.oauth2.resourceserver.jwt.public-key-location=classpath:certs/public.pem
```

### Juntando com o UserDetailsService

No Resource Server, o `principal` é um `Jwt`. Para ter de volta o seu `UsuarioAutenticado` (e as permissões **atuais** do banco), escreva um conversor que usa o `UserDetailsService`:

```java
@Component
public class UsuarioJwtConverter implements Converter<Jwt, AbstractAuthenticationToken> {

    private final UserDetailsService userDetailsService;

    public UsuarioJwtConverter(UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }

    @Override
    public AbstractAuthenticationToken convert(Jwt jwt) {
        UserDetails usuario = userDetailsService.loadUserByUsername(jwt.getSubject());   // o token já foi validado pelo Spring
        return UsernamePasswordAuthenticationToken.authenticated(usuario, jwt, usuario.getAuthorities());
    }
}
```

```java
.oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(usuarioJwtConverter)))
```

| Opção | `principal` | Banco por requisição | Quando usar |
|---|---|---|---|
| Conversor padrão (`JwtAuthenticationConverter`) | `Jwt` | Não | Alto volume, permissões vindas do token |
| `UsuarioJwtConverter` (acima) | `UsuarioAutenticado` | Sim | Quando precisa do usuário completo e de permissões sempre atuais |

## Auth0 como provedor de identidade

Até aqui, **a sua API** emitia os tokens: ela guardava as senhas, cuidava do login, da renovação, da recuperação de senha e do bloqueio de contas. Em muitas empresas, isso é **terceirizado** para um **provedor de identidade** (IdP), como a **Auth0**. A sua API passa a **só validar** tokens.

### OAuth 2.0 e OpenID Connect em poucas linhas

| Termo | O que é |
|---|---|
| **OAuth 2.0** | Protocolo de **autorização**: como um aplicativo obtém um token para acessar uma API em nome de alguém |
| **OpenID Connect (OIDC)** | Camada sobre o OAuth 2.0 para **autenticação** (quem é o usuário). Acrescenta o **ID token** |
| **Resource Owner** | O usuário |
| **Client** | O aplicativo (SPA, app móvel, outro serviço) |
| **Authorization Server** | Quem faz o login e emite os tokens (Auth0, Keycloak...) |
| **Resource Server** | A sua API |

Os tokens, para não confundir:

| Token | Para quem | Para que serve |
|---|---|---|
| **ID token** | O **aplicativo** (front-end) | Dizer **quem é o usuário** (nome, e-mail). **Não** vai para a API |
| **Access token** | A **API** | Autorizar a chamada. É o que o `Authorization: Bearer` carrega |
| **Refresh token** | O provedor | Obter novos access tokens |

Os fluxos (*grant types*) usados hoje:

| Fluxo | Quando usar |
|---|---|
| **Authorization Code + PKCE** | **O padrão** para SPAs, aplicativos móveis e aplicações web |
| **Client Credentials** | Sistema chamando sistema, sem usuário (*machine to machine*) |
| **Refresh Token** | Renovar o access token |
| **Device Code** | Dispositivos sem teclado (TVs) |
| ~~Password (ROPC)~~ | **Evitar.** Foi retirado das práticas atuais (o Spring Security 7 removeu o suporte do lado cliente) |

```mermaid
sequenceDiagram
    participant U as Usuário
    participant S as SPA (front-end)
    participant A as Auth0 (servidor de autorização)
    participant API as Sua API Spring (Resource Server)

    U->>S: clica em "Entrar"
    S->>A: redireciona para a tela de login (Authorization Code + PKCE)
    U->>A: informa e-mail e senha (ou login social, MFA...)
    A-->>S: código de autorização
    S->>A: troca o código (+ code_verifier) por tokens
    A-->>S: access token (JWT) + ID token (+ refresh token)
    S->>API: GET /api/pedidos (Authorization: Bearer access token)
    API->>A: busca as chaves públicas (JWKS), com cache
    API-->>S: 200 OK
```

> [!NOTE]
> Repare que o `UserDetailsService`, o `PasswordEncoder` e a tabela de senhas **não aparecem**: o login acontece inteiramente na Auth0. A sua API só valida o token, e o Spring faz isso sozinho.

### Configurando a API Spring

No painel da Auth0, você cria uma **API** (com um **Identifier**, que será a *audience*) e anota o seu **domínio** (tenant). No Spring:

```properties
auth0.domain=${AUTH0_DOMAIN}                       # ex.: meu-tenant.us.auth0.com (sem https://)
auth0.audience=${AUTH0_AUDIENCE}                   # o Identifier da API, ex.: https://api.meusistema.com

# O Spring descobre as chaves públicas e o emissor por aqui. A barra final é OBRIGATÓRIA
spring.security.oauth2.resourceserver.jwt.issuer-uri=https://${auth0.domain}/
```

> [!WARNING]
> Em arquivos `.properties`, comentário só em linha separada: os comentários acima estão no fim da linha apenas para ilustrar. Na prática, coloque-os **acima** de cada propriedade, senão viram parte do valor.

A forma **mais simples** (o Spring Boot valida a *audience* para você):

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=https://${auth0.domain}/
spring.security.oauth2.resourceserver.jwt.audiences=${auth0.audience}
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class Auth0SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/public/**").permitAll()
                    .requestMatchers("/api/admin/**").hasAuthority("admin:tudo")
                    .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));

        return http.build();
    }
}
```

Isso é **tudo** o que a API precisa. O Spring Boot cria o `JwtDecoder` a partir do `issuer-uri`, baixa as chaves públicas (JWKS) e valida assinatura, `exp` e emissor.

#### A forma manual (controle total da validação)

Quando é preciso validar a *audience* por código (ou combinar validações), declare o `JwtDecoder` e um validador próprio:

```java
class AudienceValidator implements OAuth2TokenValidator<Jwt> {

    private final String audience;

    AudienceValidator(String audience) {
        this.audience = audience;
    }

    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        if (jwt.getAudience() != null && jwt.getAudience().contains(audience)) {
            return OAuth2TokenValidatorResult.success();
        }
        return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("invalid_token", "O token não foi emitido para esta API", null));
    }
}
```

```java
@Bean
JwtDecoder jwtDecoder(@Value("${auth0.audience}") String audience,
                      @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}") String issuer) {

    NimbusJwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuer);    // baixa a configuração e as chaves do emissor

    OAuth2TokenValidator<Jwt> comEmissor = JwtValidators.createDefaultWithIssuer(issuer);   // exp, nbf e iss
    OAuth2TokenValidator<Jwt> comAudience = new DelegatingOAuth2TokenValidator<>(comEmissor, new AudienceValidator(audience));

    decoder.setJwtValidator(comAudience);
    return decoder;
}
```

> [!IMPORTANT]
> **Valide sempre a *audience*.** Sem isso, um token emitido pela Auth0 para **outra** API do mesmo provedor seria aceito pela sua.

> [!NOTE]
> A Auth0 também oferece um SDK próprio para APIs Spring Boot (o quickstart atual, no momento da pesquisa, estava em beta), que reduz ainda mais a configuração. O caminho acima, com o suporte **nativo** do Spring, funciona com qualquer provedor, e é o que vale aprender primeiro.

### Permissões e roles

O Spring converte, por padrão, a claim `scope` em authorities com o prefixo `SCOPE_`. Na Auth0, há duas formas de levar permissões ao token:

| Origem | Claim no token | Authority no Spring (padrão) | Como autorizar |
|---|---|---|---|
| **Scopes** (permissões pedidas pelo cliente) | `scope: "read:pedidos write:pedidos"` | `SCOPE_read:pedidos` | `hasAuthority("SCOPE_read:pedidos")` |
| **RBAC** (permissões atribuídas ao usuário, com a opção *Add Permissions in the Access Token* ligada) | `permissions: ["read:pedidos"]` | **Nenhuma** (precisa de conversor) | Após o conversor: `hasAuthority("read:pedidos")` |
| **Roles** (via *Action*, em uma claim personalizada) | `https://meusistema.com/roles: ["admin"]` | **Nenhuma** (precisa de conversor) | Após o conversor: `hasRole("ADMIN")` |

Conversor para a claim `permissions`:

```java
@Bean
JwtAuthenticationConverter permissionsConverter() {
    JwtGrantedAuthoritiesConverter authorities = new JwtGrantedAuthoritiesConverter();
    authorities.setAuthoritiesClaimName("permissions");     // a claim do RBAC da Auth0
    authorities.setAuthorityPrefix("");                     // sem prefixo: "read:pedidos"

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(authorities);
    return converter;
}
```

```java
.oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.jwtAuthenticationConverter(permissionsConverter())))
```

```java
@RestController
@RequestMapping("/api/pedidos")
public class PedidoController {

    @GetMapping
    @PreAuthorize("hasAuthority('read:pedidos')")            // permissão do RBAC da Auth0
    public List<PedidoResponse> listar() { ... }

    @PostMapping
    @PreAuthorize("hasAuthority('write:pedidos')")
    public PedidoResponse criar(@Valid @RequestBody PedidoRequest request,
                                @AuthenticationPrincipal Jwt jwt) {
        String donoId = jwt.getSubject();                    // "auth0|65a1b2c3..." identifica o usuário na Auth0
        ...
    }
}
```

> [!NOTE]
> O `@PreAuthorize` funciona **exatamente igual** a antes: o que mudou foi só **de onde** as authorities vêm (do token, e não do banco).

### Ligando o usuário do provedor ao seu banco

Mesmo com o login na Auth0, a **sua aplicação** continua tendo dados próprios (pedidos, perfil, preferências) ligados a um usuário. A ligação é feita pelo **`sub`**, o identificador único do usuário no provedor.

```sql
ALTER TABLE usuarios ADD COLUMN provedor_id VARCHAR(100) UNIQUE;   -- guarda o "sub" (ex.: auth0|65a1b2...)
```

```java
@Service
public class UsuarioProvisionamentoService {

    private final UsuarioRepository repository;

    public UsuarioProvisionamentoService(UsuarioRepository repository) {
        this.repository = repository;
    }

    /** Encontra o usuário local do "sub" ou cria um no primeiro acesso (provisionamento just-in-time). */
    @Transactional
    public Usuario obterOuCriar(Jwt jwt) {
        return repository.findByProvedorId(jwt.getSubject())
                .orElseGet(() -> repository.save(Usuario.doProvedor(jwt.getSubject())));
    }
}
```

```java
public interface UsuarioRepository extends JpaRepository<Usuario, Long> {
    Optional<Usuario> findByProvedorId(String provedorId);
}
```

Usando no Controller, ou em um conversor, para ter o **usuário local** disponível:

```java
@GetMapping("/api/me")
public UsuarioResponse eu(@AuthenticationPrincipal Jwt jwt) {
    Usuario usuario = provisionamentoService.obterOuCriar(jwt);       // dados próprios + identidade do provedor
    return UsuarioResponse.de(usuario);
}
```

| Onde fica | Quem é o dono |
|---|---|
| E-mail, senha, MFA, login social, recuperação de senha | **O provedor** (Auth0) |
| Pedidos, preferências, regras de negócio, relacionamentos | **O seu banco** (ligados ao `sub`) |

> [!NOTE]
> Por padrão, o access token **não traz e-mail nem nome**: eles estão no ID token e no endpoint `/userinfo`. Se a API precisar deles, adicione-os ao access token como **claims personalizadas** (na Auth0, por uma *Action*) ou consulte o `/userinfo` uma única vez, no provisionamento.

### Outros provedores

A configuração no Spring é **a mesma** em todos: muda o `issuer-uri` e o formato das claims de permissões.

| Provedor | `issuer-uri` (exemplo) | Onde ficam as permissões |
|---|---|---|
| **Auth0** | `https://{tenant}.auth0.com/` (com a barra final) | `scope`, `permissions` |
| **Keycloak** (código aberto, instalável) | `https://{host}/realms/{realm}` | `realm_access.roles`, `resource_access.{cliente}.roles` (precisa de conversor) |
| **Okta** | `https://{org}.okta.com/oauth2/default` | `scp`, `groups` |
| **Amazon Cognito** | `https://cognito-idp.{regiao}.amazonaws.com/{poolId}` | `cognito:groups`, `scope` |
| **Microsoft Entra ID** (Azure AD) | `https://login.microsoftonline.com/{tenant}/v2.0` | `roles`, `scp` |

> [!TIP]
> Para **estudar sem criar conta**, o **Keycloak** roda local, em Docker, e imita o que os provedores comerciais fazem. A configuração da sua API Spring é praticamente a mesma. Confira sempre na documentação do provedor o formato exato do `issuer-uri` e das claims.

### Rodando o seu próprio servidor de autorização

Se você **precisa ser** o provedor de identidade (por exemplo, para vários sistemas da mesma empresa), o Spring oferece o **Spring Authorization Server**. A partir do Spring Security 7, ele passou a fazer **parte do próprio Spring Security**, com a mesma numeração de versão. Ele implementa OAuth 2.1 e OpenID Connect, e a sua API continua sendo um **Resource Server**, exatamente como mostrado acima.

| Você é... | Usa |
|---|---|
| Só a **API** que valida tokens | `spring-boot-starter-...-oauth2-resource-server` |
| O **aplicativo** que faz login via provedor (site com login social) | `spring-boot-starter-...-oauth2-client` |
| O **provedor** que emite tokens | Spring Authorization Server |

## Fazendo requisições com tokens

### curl, arquivo .http e front-end

**Fluxo completo com `curl`** (API própria, caminhos A, B ou C):

```bash
# 1) Cadastro
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"nome":"Ana","email":"ana@email.com","senha":"Senha@12345"}'

# 2) Login: devolve accessToken e refreshToken
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ana@email.com","senha":"Senha@12345"}'

# 3) Rota protegida com o access token
curl http://localhost:8080/api/me \
  -H "Authorization: Bearer COLE_O_ACCESS_TOKEN_AQUI"

# 4) Renovação (quando o access token expirar)
curl -X POST http://localhost:8080/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"COLE_O_REFRESH_TOKEN_AQUI"}'

# 5) Logout
curl -X POST http://localhost:8080/api/auth/logout \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"COLE_O_REFRESH_TOKEN_AQUI"}'
```

**Arquivo `.http`** (IntelliJ ou VS Code), guardando o token automaticamente:

```http
### Login: o script guarda o token em uma variável
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{
  "email": "ana@email.com",
  "senha": "Senha@12345"
}

> {%
    client.global.set("accessToken", response.body.accessToken);
    client.global.set("refreshToken", response.body.refreshToken);
%}

### Rota protegida: usa a variável
GET http://localhost:8080/api/me
Authorization: Bearer {{accessToken}}

### Renovação
POST http://localhost:8080/api/auth/refresh
Content-Type: application/json

{
  "refreshToken": "{{refreshToken}}"
}
```

**Front-end (JavaScript)**, com renovação automática quando vier `401`:

```javascript
let accessToken = null;           // em memória (veja "Onde guardar o token no cliente")
let refreshToken = null;

async function api(caminho, opcoes = {}) {
  const requisicao = () => fetch(`http://localhost:8080${caminho}`, {
    ...opcoes,
    headers: { ...opcoes.headers, Authorization: `Bearer ${accessToken}` },
  });

  let resposta = await requisicao();

  if (resposta.status === 401 && refreshToken) {            // access token expirou: tenta renovar UMA vez
    const r = await fetch("http://localhost:8080/api/auth/refresh", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ refreshToken }),
    });

    if (r.ok) {
      ({ accessToken, refreshToken } = await r.json());     // guarda o par NOVO (o antigo foi revogado)
      resposta = await requisicao();                        // repete a requisição original
    }
  }
  return resposta;
}
```

### De uma API para outra (RestClient)

Quando a sua API chama **outra** API, há dois cenários comuns.

**1) Repassar o token do usuário** (a API de destino precisa saber quem é o usuário):

```java
@Bean
RestClient pedidosClient(RestClient.Builder builder) {
    return builder
            .baseUrl("https://pedidos.interno.exemplo.com")
            .requestInterceptor((request, body, execucao) -> {
                Authentication auth = SecurityContextHolder.getContext().getAuthentication();
                if (auth instanceof JwtAuthenticationToken jwtAuth) {                  // só no caminho C ou D
                    request.getHeaders().setBearerAuth(jwtAuth.getToken().getTokenValue());   // repassa o MESMO token
                }
                return execucao.execute(request, body);
            })
            .build();
}
```

**2) A sua API agindo como sistema** (*client credentials*): ela obtém um token **próprio**, com `client_id` e `client_secret`, do provedor. O módulo **OAuth2 Client** do Spring cuida de pedir, guardar e renovar esse token.

```properties
spring.security.oauth2.client.registration.parceiro.client-id=${PARCEIRO_CLIENT_ID}
spring.security.oauth2.client.registration.parceiro.client-secret=${PARCEIRO_CLIENT_SECRET}
spring.security.oauth2.client.registration.parceiro.authorization-grant-type=client_credentials
spring.security.oauth2.client.registration.parceiro.scope=read:pedidos
spring.security.oauth2.client.provider.parceiro.token-uri=https://auth.exemplo.com/oauth/token
```

```java
@Bean
RestClient parceiroClient(RestClient.Builder builder, OAuth2AuthorizedClientManager manager) {
    OAuth2ClientHttpRequestInterceptor interceptor = new OAuth2ClientHttpRequestInterceptor(manager);
    interceptor.setClientRegistrationIdResolver(request -> "parceiro");     // qual registro usar

    return builder
            .baseUrl("https://api.parceiro.com")
            .requestInterceptor(interceptor)                                // acrescenta o Bearer automaticamente
            .build();
}
```

> [!NOTE]
> O suporte do Spring Security ao `RestClient` e às **HTTP Service Clients** (interfaces com `@HttpExchange`) cresceu nas versões recentes, e o Spring Security 7 adicionou suporte OAuth2 para essas interfaces. Os nomes das classes podem variar entre versões: confira a documentação da que você usa. Este exemplo retoma o guia de **consumo de APIs externas**.

## Como isso é usado no mercado

As combinações mais comuns em projetos reais, da mais simples à mais completa:

| Padrão | Como funciona | Onde aparece | Cuidados |
|---|---|---|---|
| **1. Monolito com sessão e cookie** | Login por formulário, sessão no servidor (às vezes compartilhada com Spring Session e Redis) | Sistemas internos, sites com páginas no servidor | CSRF ligado; escalar exige sessão compartilhada |
| **2. API própria com JWT próprio** | A API emite e valida (caminhos A, B ou C) | Projetos acadêmicos, startups, produtos pequenos | Você assume a segurança do login, da renovação e das senhas |
| **3. SPA ou app + provedor de identidade + API Resource Server** | Login no provedor (Auth0, Keycloak...); a API só valida | **Aplicações novas em ambiente profissional** | Validar `iss` e `aud`; mapear usuário pelo `sub` |
| **4. Microsserviços + API Gateway + provedor** | O gateway (Spring Cloud Gateway) valida ou repassa; cada serviço é Resource Server | Sistemas grandes, várias equipes | Propagação do token; `aud` por serviço |
| **5. BFF (*Backend for Frontend*)** | O servidor do front guarda os tokens e o navegador só vê um cookie de sessão | Aplicações com exigência alta de segurança | Mais uma peça para manter |

```mermaid
flowchart LR
    C["Cliente<br/>SPA / App móvel"] -->|"1. login (OIDC)"| IDP["Provedor de identidade<br/>Auth0 / Keycloak / Cognito"]
    IDP -->|"2. access token"| C
    C -->|"3. Bearer token"| GW["API Gateway<br/>Spring Cloud Gateway"]
    GW -->|"4. Bearer token"| S1["Serviço de pedidos<br/>(Resource Server)"]
    GW -->|"4. Bearer token"| S2["Serviço de produtos<br/>(Resource Server)"]
    S1 -.->|"valida com JWKS"| IDP
    S2 -.->|"valida com JWKS"| IDP
```

Tendências atuais:

| Tendência | O que significa |
|---|---|
| **Authorization Code + PKCE** em todo lugar | O fluxo padrão para qualquer cliente, inclusive SPAs |
| **Delegar identidade a um provedor** | Em vez de guardar senhas, usa Auth0, Keycloak, Cognito, Entra ID... |
| **Access tokens de vida curta + refresh com rotação** | Reduz o estrago de um token roubado |
| **Tokens fora do `localStorage`** | Memória, cookies `HttpOnly` ou BFF |
| **MFA e passkeys (WebAuthn)** | Login sem senha ou com segundo fator. O Spring Security 7 trouxe suporte a **autenticação multifator** |
| **Sem *password grant*** | O fluxo foi retirado das recomendações atuais |
| **Tokens "vinculados" (DPoP, mTLS)** | Evitam que um token roubado seja usado por outro cliente |

Variantes que você vai encontrar:

| Variante | O que é |
|---|---|
| **Token opaco** | Texto aleatório validado por introspecção (revogação imediata) |
| **PASETO** | Formato de token alternativo ao JWT, com menos "armadilhas" de algoritmo (uso bem menos comum) |
| **JWT em cookie `HttpOnly`** | O JWT viaja em cookie em vez de cabeçalho (exige CSRF) |
| **Sessão com Redis (Spring Session)** | Sessão compartilhada entre servidores, sem JWT |
| **Gateway como único validador** | Só o gateway valida; os serviços internos confiam na rede (mais simples, menos defesa em profundidade) |

> [!NOTE]
> **Não existe uma escolha "certa" universal.** Para **aprender**, comece pelo caminho A ou C (você enxerga tudo). Para **um sistema novo com usuários reais**, o mais comum é o padrão 3: delegar o login a um provedor e deixar a API como Resource Server.

## Integração com outras bibliotecas do Spring

O JWT não vive sozinho: ele se encaixa nas demais peças do ecossistema.

| Biblioteca do Spring | Como se junta com JWT e autenticação |
|---|---|
| **Spring Web (MVC)** | `@AuthenticationPrincipal`, `@RequestHeader("Authorization")`, `ProblemDetail` para os erros `401` e `403` |
| **Spring Validation** | `@Valid` nos DTOs de login e cadastro (`@Email`, `@Size`) |
| **Spring Data JPA** | Usuários, perfis, refresh tokens e **auditoria** (`@CreatedBy` com o usuário do token) |
| **Spring Security Method Security** | `@PreAuthorize("hasAuthority('read:pedidos')")` funciona igual com qualquer origem de authorities |
| **Spring Boot Actuator** | Proteger `/actuator/**` (liberar só `/actuator/health`) |
| **Spring Cache / Redis** | Cache do usuário carregado, **lista de tokens revogados** (`jti`), limite de tentativas |
| **Spring Scheduling** | Limpar refresh tokens expirados |
| **Spring Cloud Gateway** | Validar o token na borda e repassar aos serviços |
| **Spring Authorization Server** | Ser o provedor de identidade |
| **Spring OAuth2 Client** | Obter tokens para chamar outras APIs (*client credentials*) |
| **Spring Security Test** | `@WithMockUser` e `jwt()` nos testes |
| **springdoc-openapi (Swagger)** | Botão **Authorize** com o Bearer token |

Alguns exemplos:

**Auditoria com o usuário do token** (JPA + Security), igual ao guia de entidades:

```java
@Bean
AuditorAware<String> auditorProvider() {
    return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(Authentication::isAuthenticated)
            .filter(auth -> !(auth instanceof AnonymousAuthenticationToken))
            .map(Authentication::getName);          // com JWT, o name é a claim "sub"
}
```

**Lista de tokens revogados com Redis** (logout imediato mesmo com JWT):

```java
@Service
public class TokenBlocklist {

    private final StringRedisTemplate redis;

    public TokenBlocklist(StringRedisTemplate redis) {
        this.redis = redis;
    }

    public void revogar(String jti, Instant expiraEm) {
        Duration restante = Duration.between(Instant.now(), expiraEm);
        if (!restante.isNegative()) {
            redis.opsForValue().set("jwt:revogado:" + jti, "1", restante);   // some sozinho quando o token expiraria
        }
    }

    public boolean estaRevogado(String jti) {
        return Boolean.TRUE.equals(redis.hasKey("jwt:revogado:" + jti));
    }
}
```

No `JwtAuthenticationFilter`, logo depois de validar: `if (blocklist.estaRevogado(jwt.getId())) { ... segue sem autenticar ... }`.

**Swagger com botão Authorize** (springdoc-openapi):

```java
@Configuration
@SecurityScheme(
        name = "bearerAuth",
        type = SecuritySchemeType.HTTP,
        scheme = "bearer",
        bearerFormat = "JWT")
public class OpenApiConfig { }
```

```java
@RestController
@SecurityRequirement(name = "bearerAuth")                 // as rotas deste controller pedem o token no Swagger
@RequestMapping("/api/pedidos")
public class PedidoController { ... }
```

> [!NOTE]
> A versão do **springdoc-openapi** precisa ser compatível com a do seu Spring Boot. Confira a tabela de compatibilidade no site do projeto.

## Segurança do JWT

| Risco | O que acontece | Como evitar |
|---|---|---|
| **`alg: none`** | Token sem assinatura aceito por bibliotecas mal configuradas | Fixe o **algoritmo esperado** (`Algorithm.HMAC256`, `macAlgorithm(HS256)`) e nunca aceite o que vem no header |
| **Troca de algoritmo** (RS256 → HS256, usando a chave pública como segredo) | Atacante forja tokens | Mesma regra: o algoritmo vem da **sua configuração**, não do token |
| **Segredo fraco** | Força bruta descobre o segredo e permite forjar tokens | 256 bits ou mais, aleatório (`openssl rand -base64 48`) |
| **Sem `exp`** | Token vale para sempre | Sempre defina e valide a expiração |
| **Sem validar `iss` e `aud`** | Aceita token de outro emissor ou de outra API | Valide as duas |
| **Dados sensíveis no payload** | Qualquer um lê | Só o necessário: `sub`, `roles`, `uid` |
| **Token em `localStorage`** | XSS rouba o token | Memória, cookie `HttpOnly` ou BFF; CSP no front-end |
| **Token na URL** (`?token=...`) | Vaza em logs, histórico e `Referer` | Sempre no cabeçalho `Authorization` |
| **Refresh token reaproveitado** | Roubo passa despercebido | **Rotação** e detecção de reutilização (como no `AuthService`) |
| **Impossível revogar** | Usuário desativado continua com acesso | Vida curta, recarregar o usuário, lista de revogados por `jti`, versão do token no usuário |
| **Segredo no repositório** | Qualquer pessoa emite tokens | Variável de ambiente ou cofre de segredos |
| **Sem HTTPS** | Token interceptado na rede | HTTPS sempre |

Estratégias de **revogação**:

| Estratégia | Como | Custo |
|---|---|---|
| **Vida curta** (5 a 15 min) | O token expira rápido | Nenhum extra (exige refresh) |
| **Recarregar o usuário** | O filtro consulta o banco e checa `isEnabled()` | Um `SELECT` por requisição |
| **Lista de revogados** (`jti` no Redis) | Consulta rápida a cada requisição | Redis |
| **Versão do token** | Campo `tokenVersion` no usuário e na claim; incrementar invalida todos | Um campo, e checagem na requisição |
| **Revogar o refresh token** | Impede novas renovações | Já incluso no `AuthService` |

## Testando

### Teste unitário do `JwtService` (`java-jwt`)

```java
class JwtServiceTest {

    private JwtService criarServico(Duration ttl) {
        JwtProperties props = new JwtProperties(
                "uma-chave-de-teste-com-mais-de-32-caracteres!", "minha-api", ttl, Duration.ofDays(7));
        return new JwtService(props);
    }

    private UsuarioAutenticado usuario() {
        Usuario u = new Usuario("Ana", "ana@email.com", "hash");
        // define o id e um perfil de teste (omitido)
        return new UsuarioAutenticado(u);
    }

    @Test
    void deveGerarEValidarToken() {
        JwtService servico = criarServico(Duration.ofMinutes(15));

        String token = servico.gerarAccessToken(usuario());
        DecodedJWT jwt = servico.validar(token);

        assertEquals("ana@email.com", jwt.getSubject());
        assertEquals("minha-api", jwt.getIssuer());
    }

    @Test
    void deveRejeitarTokenExpirado() {
        JwtService servico = criarServico(Duration.ofSeconds(-1));       // já nasce expirado

        String token = servico.gerarAccessToken(usuario());

        assertThrows(TokenExpiredException.class, () -> servico.validar(token));
    }

    @Test
    void deveRejeitarTokenAdulterado() {
        JwtService servico = criarServico(Duration.ofMinutes(15));
        String token = servico.gerarAccessToken(usuario());

        String adulterado = token.substring(0, token.length() - 2) + "xx";   // estraga a assinatura

        assertThrows(JWTVerificationException.class, () -> servico.validar(adulterado));
    }
}
```

### Teste da API com `jwt()` (Resource Server)

Para o Resource Server, o `spring-security-test` oferece o `jwt()`, que **simula um token válido** sem precisar assinar nada:

```java
@WebMvcTest(PedidoController.class)
@Import(Auth0SecurityConfig.class)
class PedidoControllerSecurityTest {

    @Autowired
    MockMvc mockMvc;

    @MockitoBean
    JwtDecoder jwtDecoder;                                   // o decoder real não é usado, mas o bean precisa existir

    @Test
    void semTokenDeveRetornar401() throws Exception {
        mockMvc.perform(get("/api/pedidos"))
               .andExpect(status().isUnauthorized());
    }

    @Test
    void comPermissaoDeveRetornar200() throws Exception {
        mockMvc.perform(get("/api/pedidos")
                        .with(jwt().authorities(new SimpleGrantedAuthority("read:pedidos"))))
               .andExpect(status().isOk());
    }

    @Test
    void semPermissaoDeveRetornar403() throws Exception {
        mockMvc.perform(get("/api/pedidos")
                        .with(jwt().authorities(new SimpleGrantedAuthority("outra:permissao"))))
               .andExpect(status().isForbidden());
    }
}
```

> [!NOTE]
> Imports omitidos de propósito. Os pacotes de algumas classes de teste (como `@WebMvcTest`) mudaram entre versões do Spring Boot, então confira os imports na versão que você usa.

## Erros comuns

| Problema | Sintoma | Causa e solução |
|---|---|---|
| **`401` com token válido** | Rota protegida sempre recusa | Faltou o prefixo `Bearer ` (com espaço) no cabeçalho |
| **Filtro executa duas vezes** | Logs duplicados, comportamento estranho | O filtro foi anotado com `@Component`. Crie com `new` na configuração |
| **`401` logo após emitir o token** | Token "já expirado" | Relógios diferentes (servidor e cliente) ou TTL negativo. Use `acceptLeeway` (java-jwt) ou `clockSkew` (Spring) |
| **`WeakKeyException` (JJWT) ou erro de tamanho de chave** | Falha ao iniciar | O segredo HS256 precisa ter **256 bits ou mais** |
| **`403` para quem deveria acessar** | Role não reconhecida | `JwtGrantedAuthoritiesConverter` lê `scope` com prefixo `SCOPE_` por padrão. Configure `setAuthoritiesClaimName` e `setAuthorityPrefix` |
| **`ClassCastException` com `@AuthenticationPrincipal`** | `UsuarioAutenticado` esperado, mas veio `Jwt` | No Resource Server o `principal` é `Jwt`, a não ser que use um conversor próprio |
| **Auth0: `401` com token válido** | Rejeitado pela API | Falta a barra final no `issuer-uri`, ou a *audience* não confere com o **Identifier** da API |
| **Auth0: token não traz `permissions`** | Authorities vazias | Ative **RBAC** e **Add Permissions in the Access Token** na API da Auth0 |
| **`401` no navegador, funciona no Postman** | Erro de CORS disfarçado | Habilite `http.cors(...)`; a requisição de pré-verificação (`OPTIONS`) não leva token |
| **Token funciona depois de desativar a conta** | Acesso continua | Token ainda válido: reduza o TTL ou recarregue o usuário no filtro |
| **Refresh token "para de funcionar"** | `401` na renovação | Ele é de uso único: guarde o **novo** par devolvido a cada renovação |
| **Renovação revoga tudo** | Usuário é deslogado de todos os aparelhos | Reutilização de um refresh token antigo foi detectada (comportamento esperado) |
| **Segredo no repositório** | Qualquer um forja tokens | Variável de ambiente; troque o segredo e invalide os tokens |
| **`@PreAuthorize` ignorado** | Qualquer um executa | Falta `@EnableMethodSecurity` |

## Cuidados importantes

- [ ] Guarde o **segredo/chave privada** em variável de ambiente ou cofre, nunca no código ou no GitHub.
- [ ] Use segredos HS256 com **256 bits ou mais** e, havendo vários serviços, prefira **RS256**.
- [ ] Defina e valide **`exp`**, **`iss`** e **`aud`** em todo token.
- [ ] Fixe o **algoritmo** na configuração: nunca confie no `alg` do header.
- [ ] **Access token de vida curta** (minutos) e **refresh token com rotação**.
- [ ] Guarde o refresh token **em hash**, e nunca em texto puro.
- [ ] Não coloque **dados sensíveis** no payload.
- [ ] Nunca use `JWT.decode` (sem verificação) para autenticar.
- [ ] Envie o token **somente no cabeçalho** `Authorization`, e nunca na URL.
- [ ] Use **HTTPS** em qualquer ambiente real.
- [ ] Não anote filtros de segurança com `@Component`.
- [ ] Devolva respostas **genéricas** de falha de login ("credenciais inválidas").
- [ ] Limite tentativas de login e de renovação (*rate limiting*).
- [ ] Tenha uma estratégia de **revogação** (vida curta, `jti`, recarregar usuário).
- [ ] Isole a biblioteca de JWT em **uma classe**, para poder trocá-la.
- [ ] Não registre tokens completos em logs.

## Indo para produção

- [x] Entender a estrutura do JWT (header, payload, assinatura) e as claims padrão.
- [x] Diferenciar access token e refresh token.
- [x] Conhecer a biblioteca `java-jwt` da Auth0 e o JJWT.
- [x] Implementar login, renovação e logout com refresh token em hash.
- [x] Conhecer o Resource Server nativo do Spring (`JwtDecoder`, `JwtEncoder`).
- [x] Entender OAuth 2.0, OpenID Connect e a Auth0 como provedor de identidade.
- [x] Conhecer outros meios: API Key, token opaco, client credentials.
- [ ] Escolher o caminho do projeto (A, B, C, D ou E) conforme o cenário.
- [ ] Migrar de HS256 para **RS256** com rotação de chaves (`kid`).
- [ ] Implementar **revogação** com lista de `jti` no Redis.
- [ ] Guardar os tokens de forma segura no front-end (memória, cookie `HttpOnly` ou BFF).
- [ ] Configurar CORS para o front-end.
- [ ] Documentar a API no Swagger com o botão **Authorize**.
- [ ] Integrar um provedor de identidade (Auth0, Keycloak) em um projeto de estudo.
- [ ] Estudar MFA, passkeys (WebAuthn) e tokens vinculados (DPoP).

## 📚 Referências

- [RFC 7519: JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 8725: JWT Best Current Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [auth0/java-jwt no GitHub](https://github.com/auth0/java-jwt)
- [JJWT (jwtk/jjwt) no GitHub](https://github.com/jwtk/jjwt)
- [Spring Security: OAuth 2.0 Resource Server (JWT)](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)
- [Spring Security: OAuth 2.0 Resource Server (Opaque Token)](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/opaque-token.html)
- [Spring Security: OAuth 2.0 Client](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/index.html)
- [Spring Security: What's New in 7.0](https://docs.spring.io/spring-security/reference/whats-new.html)
- [Spring Authorization Server](https://docs.spring.io/spring-authorization-server/reference/index.html)
- [Auth0 Docs: Secure an API (Spring Boot)](https://auth0.com/docs/quickstart/backend/java-spring-security5)
- [Auth0 Blog: Spring Boot Authorization Tutorial](https://auth0.com/blog/spring-boot-authorization-tutorial-secure-an-api-java/)
- [Keycloak: documentação](https://www.keycloak.org/documentation)
- [jwt.io: Introduction e depurador de tokens](https://jwt.io/introduction)
- [OWASP: JSON Web Token Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
