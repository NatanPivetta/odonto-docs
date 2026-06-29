# DER — Modelo de dados

```mermaid
erDiagram
    USERS ||--o{ TURMA_ALUNOS : "matriculado em"
    TURMAS ||--o{ TURMA_ALUNOS : "possui"
    USERS ||--o{ ATIVIDADES : "aluno"
    USERS ||--o{ ATIVIDADES : "prof. orientador"
    USERS ||--o{ ATIVIDADES : "prof. tutor (opc.)"
    TURMAS ||--o{ ATIVIDADES : "contém"
    ATIVIDADES ||--o{ ATIVIDADES : "atividade pai/filha"
    ATIVIDADES ||--o{ FEEDBACKS : "recebe"
    USERS ||--o{ FEEDBACKS : "autor (professor)"

    USERS {
        bigint    id PK
        varchar   name
        varchar   email UK
        varchar   card_number UK "9 dígitos — login"
        varchar   password_hash "BCrypt"
        varchar   role "ALUNO | PROFESSOR"
        boolean   active
        timestamp created_at
        timestamp updated_at
    }

    TURMAS {
        bigint    id PK
        varchar   disciplina
        varchar   name
        varchar   semester
        boolean   active
        timestamp created_at
        timestamp updated_at
    }

    TURMA_ALUNOS {
        bigint  turma_id PK,FK
        bigint  aluno_id PK,FK
        boolean active "matrícula ativa?"
    }

    ATIVIDADES {
        bigint    id PK
        date      data
        date      data_conclusao "nullable"
        varchar   prontuario
        varchar   nome_paciente "nullable"
        text      observacoes "nullable"
        varchar   status "PENDENTE|EM_ANDAMENTO|CONCLUIDA|ALTA"
        varchar   tipo "enum TipoAtividade"
        varchar   tipo_descricao "p/ tipo=OUTROS"
        bigint    aluno_id FK
        bigint    professor_orientador_id FK
        bigint    professor_tutor_id FK "nullable"
        bigint    turma_id FK
        bigint    atividade_pai_id FK "nullable"
        boolean   active
        timestamp created_at
        timestamp updated_at
    }

    FEEDBACKS {
        bigint    id PK
        text      texto
        bigint    atividade_id FK
        bigint    professor_id FK
        timestamp created_at
        timestamp updated_at
    }

    EMAIL_VERIFICATION_CODES {
        bigint    id PK
        varchar   email
        varchar   code "6 dígitos"
        timestamp expires_at "TTL 15 min"
        boolean   used
        timestamp created_at
    }
```

> `EMAIL_VERIFICATION_CODES` não tem FK para `USERS` por design: o código é gerado **antes** de
> a conta existir (fluxo de primeiro acesso), usando o e-mail como chave lógica.

## Observações de modelagem

- `TURMA_ALUNOS` tem PK composta `(turma_id, aluno_id)` → cada par aluno/turma existe no máximo
  uma vez. A flag `active` representa o histórico de matrícula (um aluno migra de uma turma a
  outra ao longo dos semestres; só uma matrícula fica ativa por vez).
- Exclusões de `ATIVIDADES` e `TURMAS` são **lógicas** (`active = false`), não físicas.
- Campos de auditoria (`created_at`, `updated_at`, `created_by`, `last_modified_by`) vêm da
  superclasse `Auditable` (JPA Auditing).

## Migrations (Liquibase)

O schema é versionado em `backend/src/main/resources/db/changelog/migrations/`:

| Arquivo                    | Conteúdo                                                        |
|----------------------------|----------------------------------------------------------------|
| `db.changelog-1.0.0.sql`   | `users`, `turmas`, `turma_alunos` + índices                    |
| `db.changelog-1.1.0.sql`   | `atividades` + índices + coluna `active`                        |
| `db.changelog-1.2.0.sql`   | `feedbacks`; remove `feedback_privado` de `atividades`          |
| `db.changelog-1.3.0.sql`   | colunas `tipo` e `tipo_descricao` em `atividades`              |
| `db.changelog-1.4.0.sql`   | `email_verification_codes`                                      |
| `db.changelog-1.5.0.sql`   | coluna `active` em `turma_alunos`                              |

Toda mudança de modelo exige uma migration nova (o Hibernate roda em `ddl-auto: validate`).