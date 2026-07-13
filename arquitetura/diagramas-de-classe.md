# Diagramas De Classe Do Backend

Este documento representa o estado atual do backend Spring Boot, com base nas entidades JPA
implementadas em `backend/src/main/java/br/ufrgs/odonto/modules`.

Escopo atual:

- `core`: usuarios, autenticacao, verificacao por e-mail e RBAC;
- `turma`: turmas, disciplinas clinicas e matriculas;
- `atividade`: atividades clinicas e feedbacks.

## Visao Geral Do Dominio

```mermaid
classDiagram
    direction LR

    class Auditable~String~ {
        +LocalDateTime createdAt
        +LocalDateTime updatedAt
        +String createdBy
        +String lastModifiedBy
    }

    class User {
        +Long id
        +String name
        +String email
        +String cardNumber
        +String passwordHash
        +Role role
        +Set~SecurityRole~ roles
        +boolean active
        +getAuthorities()
        +getPassword()
        +getUsername()
        +isEnabled()
    }

    class SecurityRole {
        +Long id
        +RoleName name
        +String description
        +Set~SecurityPermission~ permissions
    }

    class SecurityPermission {
        +Long id
        +PermissionName name
        +String description
    }

    class Turma {
        +Long id
        +DisciplinaClinica disciplina
        +String codigoTurma
        +String name
        +String semester
        +boolean active
        +Set~TurmaAluno~ matriculas
        +getAlunosAtivos()
    }

    class TurmaAluno {
        +TurmaAlunoId id
        +Turma turma
        +User aluno
        +boolean active
    }

    class Atividade {
        +Long id
        +LocalDate data
        +LocalDate dataConclusao
        +String prontuario
        +String nomePaciente
        +String observacoes
        +StatusAtividade status
        +TipoAtividade tipo
        +String tipoDescricao
        +User aluno
        +User professorOrientador
        +User professorTutor
        +Turma turma
        +Atividade atividadePai
        +boolean active
    }

    class Feedback {
        +Long id
        +String texto
        +Atividade atividade
        +User professor
    }

    Auditable~String~ <|-- User
    Auditable~String~ <|-- Turma
    Auditable~String~ <|-- Atividade
    Auditable~String~ <|-- Feedback

    User "0..*" --> "0..*" SecurityRole : roles
    SecurityRole "0..*" --> "0..*" SecurityPermission : permissions

    Turma "1" o-- "0..*" TurmaAluno : matriculas
    TurmaAluno "0..*" --> "1" User : aluno

    Atividade "0..*" --> "1" User : aluno
    Atividade "0..*" --> "1" User : professorOrientador
    Atividade "0..*" --> "0..1" User : professorTutor
    Atividade "0..*" --> "1" Turma : turma
    Atividade "0..*" --> "0..1" Atividade : atividadePai

    Feedback "0..*" --> "1" Atividade : atividade
    Feedback "0..*" --> "1" User : professor
```

## Core, Autenticacao E RBAC

```mermaid
classDiagram
    direction TB

    class UserDetails {
        <<interface>>
        +getAuthorities()
        +getPassword()
        +getUsername()
        +isEnabled()
    }

    class User {
        +Long id
        +String name
        +String email
        +String cardNumber
        +String passwordHash
        +Role role
        +Set~SecurityRole~ roles
        +boolean active
        +getAuthorities()
    }

    class SecurityRole {
        +Long id
        +RoleName name
        +String description
        +Set~SecurityPermission~ permissions
    }

    class SecurityPermission {
        +Long id
        +PermissionName name
        +String description
    }

    class EmailVerificationCode {
        +Long id
        +String email
        +String code
        +LocalDateTime expiresAt
        +boolean used
        +LocalDateTime createdAt
    }

    class Role {
        <<enumeration>>
        PROFESSOR
        ALUNO
    }

    class RoleName {
        <<enumeration>>
        ADMIN
        PROFESSOR
        ALUNO
        COORDENADOR
        MONITOR
    }

    class PermissionName {
        <<enumeration>>
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
    }

    UserDetails <|.. User
    User --> Role : legado
    User "0..*" --> "0..*" SecurityRole : user_roles
    SecurityRole --> RoleName
    SecurityRole "0..*" --> "0..*" SecurityPermission : role_permissions
    SecurityPermission --> PermissionName
    EmailVerificationCode ..> User : email corresponde a User.email
```

Regras relevantes:

- Login usa `cardNumber` como identificador (`getUsername()` retorna o numero de cartao).
- `User.role` permanece para compatibilidade com fluxos legados.
- `User.roles.permissions` gera as authorities usadas por `@PreAuthorize`.
- Resources novos devem preferir `hasAuthority(...)` em vez de `hasRole(...)`.
- `EmailVerificationCode` guarda codigos de primeiro acesso por e-mail sem FK para `User`.

## Turmas E Matriculas

```mermaid
classDiagram
    direction LR

    class Turma {
        +Long id
        +DisciplinaClinica disciplina
        +String codigoTurma
        +String name
        +String semester
        +boolean active
        +Set~TurmaAluno~ matriculas
        +getAlunosAtivos()
    }

    class DisciplinaClinica {
        <<enumeration>>
        ODO99012
        ODO99013
        ODO99014
        ODO99016
        +getCodigo()
        +getNome()
        +getLabel()
    }

    class TurmaAluno {
        +TurmaAlunoId id
        +Turma turma
        +User aluno
        +boolean active
    }

    class TurmaAlunoId {
        +Long turmaId
        +Long alunoId
    }

    Turma --> DisciplinaClinica
    Turma "1" o-- "0..*" TurmaAluno : matriculas
    TurmaAluno "0..*" --> "1" Turma : turma
    TurmaAluno "0..*" --> "1" User : aluno
    TurmaAluno --> TurmaAlunoId : chave composta
```

## Atividades E Feedbacks

```mermaid
classDiagram
    direction LR

    class Atividade {
        +Long id
        +LocalDate data
        +LocalDate dataConclusao
        +String prontuario
        +String nomePaciente
        +String observacoes
        +StatusAtividade status
        +TipoAtividade tipo
        +String tipoDescricao
        +User aluno
        +User professorOrientador
        +User professorTutor
        +Turma turma
        +Atividade atividadePai
        +boolean active
    }

    class Feedback {
        +Long id
        +String texto
        +Atividade atividade
        +User professor
    }

    class StatusAtividade {
        <<enumeration>>
        PENDENTE
        EM_ANDAMENTO
        CONCLUIDA
        ALTA
    }

    class TipoAtividade {
        <<enumeration>>
        ENTREVISTA_DECAPAGEM_EXAME
        ESTOMATOLOGIA_PATOLOGIA
        BIOPSIA
        EXAME_LABORATORIAL
        EXAME_MICROBIOLOGICO
        OUTROS
    }

    Atividade "0..*" --> "1" User : aluno
    Atividade "0..*" --> "1" User : professorOrientador
    Atividade "0..*" --> "0..1" User : professorTutor
    Atividade "0..*" --> "1" Turma : turma
    Atividade "0..*" --> "0..1" Atividade : atividadePai
    Atividade --> StatusAtividade
    Atividade --> TipoAtividade
    Feedback "0..*" --> "1" Atividade : atividade
    Feedback "0..*" --> "1" User : professor
```

## Observacoes Para Evolucao

O modelo atual pressupoe que uma `Atividade` ja possui aluno executante e turma definidos. Se o
fluxo da clinica exigir agendamentos ou demandas ainda sem aluno atribuido, recomenda-se criar
uma entidade anterior, como `DemandaClinica` ou `EncaminhamentoClinico`, em vez de tornar
`Atividade.aluno` opcional sem revisar as regras atuais.
