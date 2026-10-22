# 🔐 Spring Security: configuração inicial

![Java](https://img.shields.io/badge/Java-17%2B-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.x-6DB33F?logo=springboot&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring%20Security-7.x-6DB33F?logo=springsecurity&logoColor=white)
![Uso](https://img.shields.io/badge/uso-apenas%20estudos-red)

Uma base inicial para quem está começando no Spring Security: usuários em memória, roles e autenticação com **Basic Auth**.

> [!WARNING]
> Esta configuração é **APENAS PARA ESTUDOS**. Os usuários ficam na memória e as senhas estão sem criptografia (`{noop}`).

## 📑 Sumário

- [Passo a passo](#-passo-a-passo)
- [Código completo](#-código-completo)
- [Como testar](#-como-testar)
- [UserDetails](#-userdetails)
- [InMemoryUserDetailsManager](#-inmemoryuserdetailsmanager)
- [UserDetailsService](#-userdetailsservice)
- [PasswordEncoder](#-passwordencoder)
- [Restrição de rotas](#-restrição-de-rotas)
- [Segurança por método](#-segurança-por-método)
- [Indo para produção](#-indo-para-produção)
- [Referências](#-referências)

## 🪜 Passo a passo

1. Crie uma pasta (pacote) chamada `security`.
2. Dentro dela, crie a classe `DemoSecurityConfig`.
3. Anote a classe com `@Configuration`.
4. Crie um `@Bean` que retorna um `InMemoryUserDetailsManager` com os usuários.
5. Crie um `@Bean` do tipo `SecurityFilterChain` para definir **quais roles acessam quais rotas**.
6. *(Opcional)* Habilite a [segurança por método](#-segurança-por-método) com `@EnableMethodSecurity` para proteger métodos com `@PreAuthorize`.

> [!IMPORTANT]
> Os usuários (passo 4) dizem **quem** pode entrar. O `SecurityFilterChain` (passo 5) diz **o que cada um pode acessar**. Sem ele, qualquer usuário autenticado acessa qualquer rota, e as roles não servem de nada.

## 💻 Código completo

`security/DemoSecurityConfig.java`

```java
@Configuration
public class DemoSecurityConfig {

    @Bean
    public InMemoryUserDetailsManager userDetailsManager() {

        UserDetails gabriel = User.builder()
                .username("Gabriel")
                .password("{noop}test123")
                .roles("EMPLOYEE")
                .build();

        UserDetails vitoria = User.builder()
                .username("Vitoria")
                .password("{noop}test123")
                .roles("EMPLOYEE", "MANAGER")
                .build();

        UserDetails grasi = User.builder()
                .username("Grasi")
                .password("{noop}test123")
                .roles("EMPLOYEE", "MANAGER", "ADMIN")
                .build();

        return new InMemoryUserDetailsManager(gabriel, vitoria, grasi);
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

        http.authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.GET, "/employees/**").hasRole("EMPLOYEE")
                .requestMatchers("/managers/**").hasRole("MANAGER")
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
        );

        http.httpBasic(Customizer.withDefaults());

        http.csrf(csrf -> csrf.disable());

        return http.build();
    }
}
```

### Quem acessa o quê

| Usuário | Roles | `/employees` | `/managers` | `/admin` |
|---|---|:---:|:---:|:---:|
| Gabriel | EMPLOYEE | ✅ | ❌ | ❌ |
| Vitoria | EMPLOYEE, MANAGER | ✅ | ✅ | ❌ |
| Grasi | EMPLOYEE, MANAGER, ADMIN | ✅ | ✅ | ✅ |

> [!NOTE]
> Ao usar `.roles("EMPLOYEE")`, o Spring guarda internamente a authority `ROLE_EMPLOYEE`. Por isso `hasRole("EMPLOYEE")` funciona sem escrever o prefixo.

## 🧪 Como testar

**Postman**

1. Abra a requisição e vá na aba **Authorization**.
2. Em *Auth Type*, escolha **Basic Auth**.
3. Preencha *Username* e *Password* com um dos usuários acima (ex.: `Vitoria` / `test123`).

**Swagger**

1. Clique no botão **Authorize**.
2. Informe o *username* e o *password* do usuário.

**Resultados esperados**

| Código | Significado |
|---|---|
| `200` | Autenticado e com a role necessária |
| `401` | Sem credenciais ou credenciais inválidas |
| `403` | Autenticado, mas **sem a role** exigida pela rota |

```mermaid
sequenceDiagram
    participant C as Cliente (Postman/Swagger)
    participant F as Spring Security
    participant M as InMemoryUserDetailsManager

    C->>F: Requisição + Basic Auth
    F->>M: loadUserByUsername("Vitoria")
    M-->>F: UserDetails (senha + roles)
    F->>F: Confere a senha
    F->>F: Confere a role da rota
    F-->>C: 200 / 401 / 403
```

## 👤 UserDetails

É a **interface que representa o usuário** dentro do Spring Security. O framework não conhece a sua classe de usuário, ele só conhece o `UserDetails`.

| Método | Descrição |
|---|---|
| `getUsername()` | Login do usuário |
| `getPassword()` | Senha (no formato esperado pelo `PasswordEncoder`) |
| `getAuthorities()` | Roles e permissões |
| `isEnabled()` | Conta ativa? |
| `isAccountNonLocked()` | Conta não bloqueada? |
| `isAccountNonExpired()` | Conta não expirada? |
| `isCredentialsNonExpired()` | Senha não expirada? |

No exemplo, usamos o `User.builder()`, uma implementação pronta de `UserDetails` que monta o usuário de forma simples:

```java
UserDetails gabriel = User.builder()
        .username("Gabriel")
        .password("{noop}test123")
        .roles("EMPLOYEE")
        .build();
```

> [!TIP]
> Em projetos reais, é comum fazer a própria entidade `Usuario` implementar `UserDetails`, ou converter a entidade em um `UserDetails` dentro do `UserDetailsService`.

## 🧠 InMemoryUserDetailsManager

É uma implementação pronta que **guarda os usuários na memória da aplicação**, sem precisar de banco de dados. Você entrega os `UserDetails` no construtor e ele cuida da busca durante o login:

```java
return new InMemoryUserDetailsManager(gabriel, vitoria, grasi);
```

Ele também permite gerenciar usuários em tempo de execução (`createUser`, `updateUser`, `deleteUser`, `userExists`, `changePassword`), mas tudo é perdido quando a aplicação reinicia.

| Vantagens | Limitações |
|---|---|
| Zero configuração de banco | Usuários somem ao reiniciar |
| Ótimo para estudo e testes | Não escala para vários usuários |
| Fácil de entender | Senhas e usuários ficam no código |

## 🔎 UserDetailsService

Interface com **um único método**, usada pelo Spring Security para buscar o usuário durante a autenticação:

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

O `InMemoryUserDetailsManager` **é** um `UserDetailsService` (ele implementa `UserDetailsManager`, que estende `UserDetailsService`). Por isso o Spring o usa automaticamente quando ele é declarado como `@Bean`.

| Implementação | Onde busca os usuários |
|---|---|
| `InMemoryUserDetailsManager` | Memória |
| `JdbcUserDetailsManager` | Banco, via JDBC (tabelas padrão) |
| Sua própria classe | Banco com JPA, API externa, etc. |

<details>
<summary>Exemplo: UserDetailsService próprio com JPA</summary>

```java
@Service
public class UsuarioDetailsService implements UserDetailsService {

    private final UsuarioRepository repository;

    public UsuarioDetailsService(UsuarioRepository repository) {
        this.repository = repository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) {
        Usuario usuario = repository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("Usuário não encontrado"));

        return User.withUsername(usuario.getUsername())
                .password(usuario.getPassword())
                .roles(usuario.getRole())
                .build();
    }
}
```

</details>

## 🔑 PasswordEncoder

Interface responsável por **codificar (hash)** e **verificar** senhas:

```java
String encode(CharSequence senhaPura);
boolean matches(CharSequence senhaPura, String senhaCodificada);
```

### O que significa o `{noop}`?

Sem um bean de `PasswordEncoder`, o Spring usa o `DelegatingPasswordEncoder`, que lê o **prefixo** da senha para saber qual algoritmo aplicar:

| Prefixo | Algoritmo |
|---|---|
| `{noop}` | Nenhum (texto puro) ⚠️ |
| `{bcrypt}` | BCrypt |
| `{argon2}` | Argon2 |

Neste exemplo, `{noop}test123` significa "senha em texto puro: `test123`". Sem esse prefixo, ocorre o erro:

```text
java.lang.IllegalArgumentException: There is no PasswordEncoder mapped for the id "null"
```

### Versão com BCrypt

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

@Bean
public InMemoryUserDetailsManager userDetailsManager(PasswordEncoder encoder) {

    UserDetails gabriel = User.builder()
            .username("Gabriel")
            .password(encoder.encode("test123"))
            .roles("EMPLOYEE")
            .build();

    return new InMemoryUserDetailsManager(gabriel);
}
```

> [!TIP]
> O BCrypt gera um hash **diferente a cada chamada** (salt aleatório). Por isso a verificação é sempre feita com `matches()`, nunca com `equals()`.

## 🚦 Restrição de rotas

Agora veremos como **restringir rotas** para que apenas usuários com a **role** correta possam acessar determinados recursos.

### Sintaxe

Para liberar o acesso a **uma única role**, usamos o `hasRole`:

```java
requestMatchers(HttpMethod.MÉTODO, "/url")
    .hasRole("ROLE_AUTORIZADA")
```

Para liberar para **várias roles**, usamos o `hasAnyRole`:

```java
requestMatchers(HttpMethod.MÉTODO, "/url")
    .hasAnyRole("ROLE_1", "ROLE_2", "ROLE_3")
```

| Parte | O que significa |
|---|---|
| `HttpMethod.MÉTODO` | Método HTTP da requisição (`GET`, `POST`, `PUT`, `DELETE`...). É opcional: sem ele, a regra vale para **todos** os métodos |
| `"/url"` | Caminho da requisição, que pode usar `*` e `**` |
| `hasRole(...)` | Exige **uma** role específica |
| `hasAnyRole(...)` | Exige **pelo menos uma** das roles listadas |

> [!NOTE]
> Escreva a role **sem** o prefixo `ROLE_`. O `hasRole("MANAGER")` já procura pela authority `ROLE_MANAGER`.

### Exemplo

```java
http.authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.GET, "/api/employee/**").hasRole("EMPLOYEE")    // listar e buscar por id
        .requestMatchers(HttpMethod.POST, "/api/employee/**").hasRole("MANAGER")    // criar
        .requestMatchers(HttpMethod.PUT, "/api/employee/**").hasRole("MANAGER")     // atualizar
        .requestMatchers(HttpMethod.DELETE, "/api/employee/**").hasRole("ADMIN")    // deletar
        .anyRequest().authenticated()                                               // qualquer outra rota exige login
);
```

| Método | Rota | Quem acessa |
|---|---|---|
| `GET` | `/api/employee/**` | EMPLOYEE |
| `POST` | `/api/employee/**` | MANAGER |
| `PUT` | `/api/employee/**` | MANAGER |
| `DELETE` | `/api/employee/**` | ADMIN |
| qualquer outro | outras rotas | qualquer usuário autenticado |

> [!IMPORTANT]
> **A ordem importa.** O Spring aplica a **primeira** regra que casar com a requisição. Coloque as regras mais específicas primeiro e deixe o `anyRequest()` sempre por último.

### 🌟 O que são os asteriscos (`*`)?

Eles definem **quanto do caminho** a regra cobre:

| Padrão | Casa com | Não casa com |
|---|---|---|
| `"/api/employee"` | `/api/employee` | `/api/employee/1`, `/api/employee/1/orders` |
| `"/api/employee/**"` | `/api/employee`, `/api/employee/1`, `/api/employee/1/orders` | `/api/employees` |

- **Sem `/**`**: o padrão casa **apenas com aquele caminho exato**. É útil para proteger um endpoint específico, como o `GET /api/employee` que lista todos.
- **Com `/**`**: o padrão casa com o caminho **e com tudo que vem depois**, em qualquer profundidade. Inclui o próprio `/api/employee`, porque o `**` aceita zero ou mais segmentos.

#### E o `*` simples?

Um asterisco cobre **somente um nível**:

| Padrão | `/api/employee` | `/api/employee/1` | `/api/employee/1/orders` |
|:---|:---:|:---:|:---:|
| `/api/employee` | ✅ | ❌ | ❌ |
| `/api/employee/*` | ❌ | ✅ | ❌ |
| `/api/employee/**` | ✅ | ✅ | ✅ |

<details>
<summary>Quando separar <code>/api/employee</code> e <code>/api/employee/**</code>?</summary>

Só quando o **caminho base** e os **subcaminhos** precisam de **regras diferentes**. Por exemplo, listar todos exige mais permissão do que buscar um por id:

```java
http.authorizeHttpRequests(auth -> auth
        .requestMatchers(HttpMethod.GET, "/api/employee").hasRole("MANAGER")       // listar todos: só gerente
        .requestMatchers(HttpMethod.GET, "/api/employee/**").hasRole("EMPLOYEE")   // buscar por id: funcionário
        .anyRequest().authenticated()
);
```

O matcher exato precisa vir **antes** do `/**`. Se a ordem for invertida, o `/**` casa primeiro com `/api/employee` e a regra do gerente nunca é aplicada.

</details>

### Outro exemplo: rotas públicas

```java
http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/auth/**", "/actuator/health").permitAll()
        .anyRequest().authenticated()                                               // qualquer outra rota exige login
);
```

Aqui o `requestMatchers` é usado **sem informar o método HTTP** (`POST`, `GET`, `PUT`, `DELETE`), então a regra vale para **todos os métodos** nesses caminhos.
Também podemos passar **vários caminhos** para a mesma permissão no mesmo `requestMatchers`, como fizemos acima com `"/api/auth/**"` e `"/actuator/health"`.

O `.permitAll()` deixa passar **qualquer um, mesmo que não esteja autenticado**.

### ⛓️ SecurityFilterChain

Todas essas regras ficam dentro do **`SecurityFilterChain`**, a cadeia de filtros que define as regras de autorização dos endpoints. Ele é explicado com mais detalhes no [SECURITY-RESUMO.md](SECURITY-RESUMO.md).

Exemplo completo, no mesmo arquivo `DemoSecurityConfig`:

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

    http.authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/employees/**").hasRole("EMPLOYEE")
            .requestMatchers("/managers/**").hasRole("MANAGER")
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
    );

    // Permite usar o HTTP Basic Authentication no Swagger e no Postman
    http.httpBasic(Customizer.withDefaults());

    // Desativado apenas para facilitar testes de POST/PUT/DELETE em projetos de estudo
    http.csrf(csrf -> csrf.disable());

    return http.build();
}
```

| Linha | Função |
|---|---|
| `authorizeHttpRequests` | Define quem pode acessar cada rota |
| `httpBasic` | Ativa a autenticação **Basic Auth** |
| `csrf.disable()` | Desativa a proteção CSRF (somente para estudo) |
| `http.build()` | Constrói e devolve a cadeia de filtros |

### 🚨 Respostas esperadas

| Código | Significado |
|:---:|---|
| `200` | Autenticado e com a role necessária |
| `401` | Sem credenciais ou credenciais inválidas |
| `403` | Autenticado, mas **sem a role** exigida pela rota |

## 🔒 Segurança por método

O `requestMatchers` protege por **URL**. Já a **segurança por método** protege o **próprio método Java** (no controller ou no service), com regras mais finas, como "só o dono do registro pode ver" ou "só pode retornar dados da própria equipe".

| | `requestMatchers` (URL) | Anotações (método) |
|---|---|---|
| Onde fica | `SecurityFilterChain` | Em cima do método |
| Enxerga | URL, método HTTP e role | Parâmetros, retorno e usuário logado |
| Bom para | Regras gerais por rota | Regras detalhadas e por dono do recurso |

> [!TIP]
> Os dois se combinam: use o `requestMatchers` para as regras gerais e as anotações para o filtro fino por cima.

### 🪜 Passo a passo

**1. Habilite a segurança por método** na `DemoSecurityConfig`:

```java
@Configuration
@EnableMethodSecurity
public class DemoSecurityConfig {
    // ...
}
```

> [!NOTE]
> O `@EnableMethodSecurity` substitui o antigo `@EnableGlobalMethodSecurity`, que não deve mais ser usado.

**2. Anote o método** que você quer proteger:

```java
@RestController
@RequestMapping("/api/employee")
public class EmployeeController {

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void delete(@PathVariable Long id) {
        // só ADMIN executa este método
    }
}
```

**3. Teste com usuários diferentes.** Quem não tem a role recebe **403**.

| Usuário | Resultado em `DELETE /api/employee/1` |
|---|---|
| Gabriel (EMPLOYEE) | `403` |
| Vitoria (MANAGER) | `403` |
| Grasi (ADMIN) | `200` |

### 🏷️ Anotações disponíveis

| Anotação | Quando executa | Para que serve | Precisa habilitar? |
|---|---|---|---|
| `@PreAuthorize` | **Antes** do método | Libera ou bloqueia a execução (a mais usada) | Já vem ativa |
| `@PostAuthorize` | **Depois** do método | Decide com base no **retorno** (`returnObject`) | Já vem ativa |
| `@PreFilter` | **Antes** do método | Filtra a **coleção recebida** como parâmetro | Já vem ativa |
| `@PostFilter` | **Depois** do método | Filtra a **coleção retornada** | Já vem ativa |
| `@Secured` | **Antes** do método | Só roles, com o prefixo: `@Secured("ROLE_ADMIN")` | `securedEnabled = true` |
| `@RolesAllowed` | **Antes** do método | Padrão JSR-250, só roles: `@RolesAllowed("ADMIN")` | `jsr250Enabled = true` |

Para ativar as duas últimas:

```java
@EnableMethodSecurity(securedEnabled = true, jsr250Enabled = true)
```

> [!TIP]
> Para estudo, concentre-se em **`@PreAuthorize`**. Ela aceita expressões completas e substitui `@Secured` e `@RolesAllowed` na maioria dos casos.

### 🧮 Expressões (SpEL) mais usadas

Elas ficam dentro da anotação, entre aspas, e usam **aspas simples** para os textos:

| Expressão | Significado |
|---|---|
| `hasRole('ADMIN')` | Tem a role ADMIN |
| `hasAnyRole('MANAGER', 'ADMIN')` | Tem pelo menos uma das roles |
| `hasAuthority('ROLE_ADMIN')` | Tem a authority exata (com prefixo) |
| `isAuthenticated()` | Está logado |
| `permitAll()` / `denyAll()` | Libera todos / bloqueia todos |
| `authentication.name` | Username do usuário logado |
| `#id` | Valor do parâmetro `id` do método |
| `returnObject` | Objeto retornado (só no `@PostAuthorize` e `@PostFilter`) |
| `filterObject` | Item atual da coleção (só no `@PreFilter` e `@PostFilter`) |
| `@bean.metodo(...)` | Chama um método de um bean Spring |

Dá para combinar com `and`, `or` e `!`:

```java
@PreAuthorize("hasRole('MANAGER') or hasRole('ADMIN')")
```

### 💡 Exemplos de cada anotação

**`@PostAuthorize`**: busca primeiro e confere o dono do resultado depois.

```java
@GetMapping("/{id}")
@PostAuthorize("returnObject.username == authentication.name or hasRole('ADMIN')")
public Employee findById(@PathVariable Long id) {
    return repository.findById(id).orElseThrow();
}
```

**`@PostFilter`**: devolve só os itens que o usuário pode ver.

```java
@GetMapping
@PostFilter("filterObject.username == authentication.name or hasRole('MANAGER')")
public List<Employee> findAll() {
    return repository.findAll();
}
```

**`@PreFilter`**: descarta, antes de executar, os itens que o usuário não pode enviar.

```java
@PreFilter("filterObject.username == authentication.name")
public void saveAll(List<Employee> employees) {
    repository.saveAll(employees);
}
```

**`@Secured` e `@RolesAllowed`**:

```java
@Secured("ROLE_ADMIN")          // com o prefixo ROLE_
public void metodoA() { }

@RolesAllowed("ADMIN")          // sem o prefixo
public void metodoB() { }
```

> [!WARNING]
> `@PostFilter` carrega **todos** os registros do banco e filtra na memória. Em tabelas grandes, prefira filtrar na própria consulta (ex.: `findByUsername`).

### 👥 Cada usuário vê apenas o seu registro

**1. Ligue o usuário logado ao registro** guardando o `username` na entidade:

```java
@Entity
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String username; // mesmo valor usado no login (ex.: "Gabriel")

    // getters e setters
}
```

**2. Crie um componente que confere o dono:**

```java
@Component("employeeSecurity")
public class EmployeeSecurity {

    private final EmployeeRepository repository;

    public EmployeeSecurity(EmployeeRepository repository) {
        this.repository = repository;
    }

    public boolean isOwner(Long id, String username) {
        return repository.findById(id)
                .map(e -> e.getUsername().equals(username))
                .orElse(false);
    }
}
```

**3. Use no método:**

```java
@GetMapping("/{id}")
@PreAuthorize("hasAnyRole('MANAGER', 'ADMIN') or @employeeSecurity.isOwner(#id, authentication.name)")
public Employee findById(@PathVariable Long id) {
    return repository.findById(id).orElseThrow();
}
```

Resultado: o funcionário só vê o **próprio** registro, enquanto gerentes e admins veem todos. Qualquer outra tentativa retorna **403**.

### ⚠️ Cuidados

> [!WARNING]
> - As anotações só funcionam em **beans do Spring** (`@Controller`, `@Service`, `@Component`...), chamados **de fora** da classe. Um `this.metodo()` dentro da mesma classe **ignora** a anotação.
> - Anote métodos **públicos**. Métodos `private` não são protegidos.
> - Se você tiver um `@ExceptionHandler(Exception.class)` genérico, ele pode capturar o `AccessDeniedException` e devolver **500** no lugar do **403**. Trate essa exceção separadamente.

## 🚀 Indo para produção

- [x] Entender usuários em memória, roles e restrição de rotas (este guia)
- [x] Conhecer a segurança por método (`@PreAuthorize` e cia.)
- [ ] Trocar `{noop}` por `BCryptPasswordEncoder`
- [ ] Buscar usuários no banco com um `UserDetailsService` próprio
- [ ] Reativar o CSRF ou adotar autenticação stateless (JWT / OAuth2)
- [ ] Nunca versionar senhas reais no repositório

## 📚 Referências

- [Documentação do Spring Security](https://docs.spring.io/spring-security/reference/)
- [Password Storage](https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html)
- [Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)
- [spring-projects/spring-security no GitHub](https://github.com/spring-projects/spring-security)
