# Proposal Management — Autenticação e autorização com Spring Security

Projeto desenvolvido durante o **Bootcamp CI&T – Java com IA (DIO)**, no módulo de Spring Security.

API de gestão de propostas entre **influenciadores** e **marcas**. Influenciadores criam propostas e só enxergam as próprias; marcas enxergam todas. A segurança foi encaixada sem quebrar a separação de camadas do DDD.

## Módulos

- `auth`: usuários, login REST, configuração de segurança e papéis (`ROLE_INFLUENCER`, `ROLE_BRAND`).
- `proposal`: ciclo de vida das propostas e listagem por escopo de acesso.

## Endpoints

| Método | Endpoint          | Quem pode                 | Descrição                                   |
|--------|-------------------|---------------------------|---------------------------------------------|
| `POST` | `/api/auth/login` | público                   | Login com JSON (`username`, `password`)     |
| `GET`  | `/`               | autenticado               | Retorna o ID do usuário logado              |
| `GET`  | `/influencer`     | `INFLUENCER`              | Endpoint de teste por papel                 |
| `GET`  | `/brand`          | `BRAND`                   | Endpoint de teste por papel                 |
| `POST` | `/proposals`      | `INFLUENCER`              | Cria uma proposta                           |
| `GET`  | `/proposals`      | `INFLUENCER` ou `BRAND`   | Lista propostas (próprias ou todas)         |

Usuários criados automaticamente na primeira execução (senha `password` para todos):

| Usuário        | Papel        |
|----------------|--------------|
| `fitness_vibe` | INFLUENCER   |
| `tech_guru`    | INFLUENCER   |
| `logistics`    | BRAND        |

## Tecnologias

- Java 25 + Spring Boot 4
- Spring Security (sessão HTTP, `@PreAuthorize`, BCrypt)
- Spring Data JPA + MySQL
- Docker Compose
- Lombok

## Estrutura

```text
src/main/java/dio/proposalmanagement
├── auth/
│   ├── domain/          -> UserRole
│   └── infrastructure/  -> entidade User (UserDetails), JpaUserDetailsService, filtro de login REST, SecurityConfig
└── proposal/
    ├── domain/          -> Proposal, Owner, contratos
    ├── application/     -> casos de uso + strategies de listagem (OWN / ALL)
    └── infrastructure/  -> controller HTTP e persistência JPA
```

## O que aprendi

- **`SecurityFilterChain`**: toda a regra de quais rotas são públicas e quais exigem login fica centralizada no `SecurityConfig`.
- **Usuários no banco**: a entidade `User` implementa `UserDetails`, e o `JpaUserDetailsService` busca o usuário pelo username. Senhas guardadas com BCrypt.
- **Login REST com JSON**: o `RestUsernamePasswordAuthenticationFilter` estende o filtro padrão do Spring para ler `username`/`password` do corpo da requisição em vez de um formulário.
- **Autenticação stateful**: depois do login, a sessão HTTP (cookie `JSESSIONID`) mantém o usuário autenticado. É o comportamento padrão do Spring Security, antes de partir para tokens (JWT).
- **Autorização por método**: `@EnableMethodSecurity` + `@PreAuthorize("hasRole('INFLUENCER')")`.
- **Strategy + Factory**: o caso de uso de listagem escolhe a estratégia (`OwnStrategy` ou `AllStrategy`) pelo `AccessScope`, sem `if` espalhado pelo código.
- **Fronteira entre contextos**: o módulo `proposal` não conhece o `User` do `auth`. Ele trabalha com o conceito de `Owner`, e a tradução acontece no controller.

## Como rodar

Pré-requisito: Docker em execução (o MySQL do `compose.yml` sobe automaticamente no `bootRun`).

```bash
./gradlew bootRun
```

Login e uso da sessão:

```bash
curl -i -c cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"username":"fitness_vibe","password":"password"}' \
  http://localhost:8080/api/auth/login

curl -i -b cookies.txt \
  -H "Content-Type: application/json" \
  -d '{"title":"Campanha de verão","description":"3 posts e 2 stories"}' \
  http://localhost:8080/proposals

curl -b cookies.txt http://localhost:8080/proposals
```

> O teste `contextLoads` sobe o contexto completo do Spring e precisa de um MySQL configurado. O suporte a Docker Compose (`developmentOnly`) só entra em ação no `bootRun`.
