# Backend — Convenções de código

## Estrutura de pacotes

```
br.ufrgs.odonto/
├── config/                 segurança (JWT), auditoria, exceções, OpenAPI, seed (DataInitializer)
│   ├── auth/               JwtService, JwtAuthenticationFilter, SecurityConfig, JwtProperties
│   ├── audit/              Auditable, AuditorAwareImpl, JpaAuditingConfig
│   └── exception/          GlobalExceptionHandler
├── modules/                módulos de domínio
│   ├── core/               usuários, auth, e-mail
│   ├── turma/              turmas e matrículas
│   └── atividade/          atividades e feedbacks
│       └── {entity,repository,service,resource,mapper}
├── dto/                    DTOs de request/response por domínio
├── enums/                  Role, ...
└── exception/              BusinessException, ResourceNotFoundException
```

Cada módulo segue o mesmo formato: `entity` · `repository` · `service` · `resource` ·
`mapper` (+ `dto` quando aplicável). **Ao criar um módulo novo, espelhe um existente** (ex.:
`turma`).

## Padrões

- **Resources REST** sob `resource/v1` (ex.: `@RequestMapping("/v1/atividades")`).
- **Injeção por construtor** via Lombok `@RequiredArgsConstructor`. Não usar `@Autowired` em campo.
- **Autorização** com `@PreAuthorize("hasRole('PROFESSOR'|'ALUNO')")` no resource. O usuário
  autenticado chega via `@AuthenticationPrincipal User loggedUser`.
- **Mapeamento entidade↔DTO** com **MapStruct** (`componentModel = spring`).
- **Validação** de entrada com Bean Validation (`@Valid` + anotações nos DTOs).
- **DTOs como `record`**; respostas com `default-property-inclusion: non_null`.
- **Exceções de negócio** lançam `BusinessException` (→ HTTP 409) ou
  `ResourceNotFoundException` (→ HTTP 404), tratadas no `GlobalExceptionHandler`.
- **Exclusão lógica**: entidades com flag `active` (ex.: `Atividade`, `Turma`, `TurmaAluno`)
  são desativadas, não removidas fisicamente.

## Banco e migrations

- Schema versionado com **Liquibase** em `db/changelog/migrations/db.changelog-x.y.z.sql`,
  agregados em `db.changelog-master.yaml`.
- Hibernate roda em `ddl-auto: validate` → **toda mudança de modelo exige uma migration nova**;
  o app não cria/altera tabelas sozinho.
- Cada changeset tem autor e id únicos (`-- changeset autor:id`).

## Segurança

- Autenticação stateless por **JWT** (jjwt). Token carrega `sub = cardNumber` e claim `role`.
- Senhas com **BCrypt**. O `cardNumber` (9 dígitos) é a identidade de login canônica.
- Endpoints públicos: `/auth/**`, Swagger e `/actuator/health`. O resto exige Bearer token.

## Estilo

- Código autoexplicativo; comentários reservados a regras de negócio não óbvias ou decisões
  de arquitetura (o "porquê"). Decisões maiores devem ir para estes docs, não para comentários
  inline.
