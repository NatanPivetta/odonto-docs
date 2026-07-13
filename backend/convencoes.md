# Backend - Convencoes De Codigo

## Estrutura De Pacotes

```text
br.ufrgs.odonto/
├── config/                 seguranca, auditoria, excecoes, OpenAPI, seed
│   ├── auth/               JwtService, JwtAuthenticationFilter, SecurityConfig, JwtProperties
│   ├── audit/              Auditable, AuditorAwareImpl, JpaAuditingConfig
│   └── exception/          GlobalExceptionHandler
├── modules/                modulos de dominio
│   ├── core/               usuarios, auth, e-mail, roles e permissions
│   ├── turma/              turmas e matriculas
│   └── atividade/          atividades e feedbacks
│       └── {entity,repository,service,resource,mapper}
├── dto/                    DTOs de request/response por dominio
├── enums/                  Role, RoleName, PermissionName, ...
└── exception/              BusinessException, ResourceNotFoundException
```

Cada modulo segue o mesmo formato: `entity`, `repository`, `service`, `resource` e `mapper`
quando aplicavel. Ao criar um modulo novo, espelhe um modulo existente.

## Padroes

- **Resources REST:** ficam sob `resource/v1`, por exemplo `@RequestMapping("/v1/atividades")`.
- **Injecao por construtor:** usar Lombok `@RequiredArgsConstructor`; evitar `@Autowired` em campo.
- **Autorizacao:** usar `@PreAuthorize` com `hasAuthority(...)` ou `hasAnyAuthority(...)`.
- **Permissions:** devem existir no enum `PermissionName` e na tabela `permissions`.
- **Roles persistidas:** usar `RoleName` e a tabela `roles`; o campo `User.role` e legado.
- **Regras contextuais:** quando a permissao depender do recurso acessado, usar uma policy de
  seguranca dedicada em vez de espalhar condicionais no resource.
- **Mapeamento entidade-DTO:** usar MapStruct (`componentModel = spring`).
- **Validacao:** usar Bean Validation (`@Valid` e anotacoes nos DTOs).
- **DTOs:** preferir `record` para requests/responses simples.
- **Excecoes de negocio:** lancar `BusinessException` ou `ResourceNotFoundException`, tratadas
  pelo `GlobalExceptionHandler`.
- **Exclusao logica:** entidades com `active` devem ser desativadas, nao removidas fisicamente.

## Banco E Migrations

- Schema versionado com Liquibase em `db/changelog/migrations/db.changelog-x.y.z.sql`.
- O `db.changelog-master.yaml` deve incluir toda migration nova no final.
- Hibernate roda em `ddl-auto: validate`; toda mudanca de modelo exige migration.
- Cada changeset deve ter autor e id unicos (`-- changeset autor:id`).
- Nao editar changesets que ja foram aplicados em ambientes compartilhados; criar nova migration.

## Seguranca

- Autenticacao stateless por JWT. O token usa `sub = cardNumber`.
- Senhas com BCrypt.
- `cardNumber` possui 8 digitos e e a identidade canonica de login.
- Endpoints publicos: `/auth/**`, Swagger e `/actuator/health`.
- O restante exige Bearer token.
- O `User` implementa `UserDetails` e carrega authorities a partir de:
  - `ROLE_<role>` do campo legado `User.role`;
  - `ROLE_<role>` das roles persistidas;
  - permissions associadas as roles persistidas.
- Novos endpoints devem ser protegidos por permissions, nao por `hasRole`.

## Permissions Atuais

```text
USER_MANAGE
USER_VIEW
PROFESSOR_VIEW
STUDENT_VIEW
CLASS_MANAGE
CLASS_VIEW
ACTIVITY_CREATE
ACTIVITY_UPDATE_OWN
ACTIVITY_UPDATE_ANY
ACTIVITY_VIEW_OWN
ACTIVITY_VIEW_ANY
ACTIVITY_REVIEW
FEEDBACK_CREATE
FEEDBACK_VIEW_OWN
FEEDBACK_VIEW_ANY
```

## Estilo

- Codigo deve ser autoexplicativo.
- Comentarios devem explicar regras de negocio nao obvias ou decisoes de arquitetura.
- Decisoes maiores devem ser registradas na documentacao, nao apenas em comentario inline.
