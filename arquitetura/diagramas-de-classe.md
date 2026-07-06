# Diagramas de classe do backend

Este documento representa o estado atual do backend Spring Boot, com base nas entidades JPA
implementadas em `backend/src/main/java/br/ufrgs/odonto/modules`.

Escopo atual:

- `core`: usuarios e codigos de verificacao por e-mail;
- `turma`: turmas e matriculas;
- `atividade`: atividades clinicas e feedbacks.

Os modulos futuros de agendamento, anexos, exames e demandas clinicas ainda nao existem no
backend atual e, portanto, nao aparecem nestes diagramas.

## Visao geral do dominio

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
        +boolean active
        +getAuthorities()
        +getPassword()
        +getUsername()
        +isEnabled()
    }

    class EmailVerificationCode {
        +Long id
        +String email
        +String code
        +LocalDateTime expiresAt
        +boolean used
        +LocalDateTime createdAt
    }

    class Turma {
        +Long id
        +String disciplina
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

    class TurmaAlunoId {
        +Long turmaId
        +Long alunoId
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

    class Role {
        <<enumeration>>
        PROFESSOR
        ALUNO
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

    Auditable~String~ <|-- User
    Auditable~String~ <|-- Turma
    Auditable~String~ <|-- Atividade
    Auditable~String~ <|-- Feedback

    User --> Role
    Turma "1" o-- "0..*" TurmaAluno : matriculas
    TurmaAluno --> TurmaAlunoId
    TurmaAluno "0..*" --> "1" User : aluno

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

Observacoes:

- `Auditable<String>` e uma superclasse mapeada, nao uma tabela propria.
- `User` tambem implementa `UserDetails`, usado pelo Spring Security.
- `EmailVerificationCode` guarda codigos de primeiro acesso por e-mail, mas nao possui chave
  estrangeira para `User`; a associacao e feita pelo campo `email`.
- `TipoAtividade` possui mais valores no codigo do que os exibidos no diagrama geral. A lista
  completa aparece mais abaixo.

## Core e autenticacao

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
        +boolean active
        +getAuthorities()
        +getPassword()
        +getUsername()
        +isEnabled()
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

    Auditable~String~ <|-- User
    UserDetails <|.. User
    User --> Role
    EmailVerificationCode ..> User : email corresponde a User.email
```

Regras relevantes:

- Login usa `cardNumber` como identificador (`getUsername()` retorna o numero de cartao).
- `role` define autorizacao por perfil (`PROFESSOR` ou `ALUNO`).
- `active` controla se a conta esta habilitada.
- O fluxo de primeiro acesso usa `EmailVerificationCode` com validade e flag `used`.

## Turmas e matriculas

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
        +Role role
        +boolean active
    }

    class Turma {
        +Long id
        +String disciplina
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

    class TurmaAlunoId {
        +Long turmaId
        +Long alunoId
    }

    Auditable~String~ <|-- User
    Auditable~String~ <|-- Turma

    Turma "1" o-- "0..*" TurmaAluno : matriculas
    TurmaAluno "0..*" --> "1" Turma : turma
    TurmaAluno "0..*" --> "1" User : aluno
    TurmaAluno --> TurmaAlunoId : chave composta
```

Regras relevantes:

- `TurmaAluno` e a entidade de associacao entre turma e aluno.
- A chave composta `TurmaAlunoId` contem `turmaId` e `alunoId`.
- A flag `active` em `TurmaAluno` permite manter historico e desativar matriculas antigas.
- O backend trata a regra de que um aluno deve possuir no maximo uma matricula ativa.
- `Turma.active = false` representa turma encerrada.

## Atividades e feedbacks

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
        +Role role
        +boolean active
    }

    class Turma {
        +Long id
        +String disciplina
        +String name
        +String semester
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
        AFASTAMENTO_DENTARIO
        ATF
        ORIENTACAO_HB_ROHB
        RESTAURACAO_PROVISORIA
        RESTAURACAO_RESINA_POSTERIOR
        RESTAURACAO_RESINA_ANTERIOR
        CONFECCAO_DE_GUIA
        RESTAURACAO_AMALGAMA
        ACABAMENTO_POLIMENTO
        REPARO_CONSERTO_RESTAURACAO
        INLAY_ONLAY
        FACETAS
        CLAREAMENTO_VITAL
        RAP_OHB
        RASUB
        EXAME_INTERMEDIARIO_PERIODONTAL
        CIRURGIA_PERIODONTAL
        PLACA_MIORRELAXANTE
        ENDODONTIA_MONO
        ENDODONTIA_PRE_MOLAR
        ENDODONTIA_MOLAR
        PULPOTOMIA
        CLAREAMENTO_INTERNO
        PROVISORIO_PROTESE_FIXA
        NUCLEO
        COROA
        PPR
        PT
        PPF
        PPR_PROVISORIA
        URGENCIA
        CONSERTO_PROTESE
        OUTROS
    }

    Auditable~String~ <|-- User
    Auditable~String~ <|-- Turma
    Auditable~String~ <|-- Atividade
    Auditable~String~ <|-- Feedback

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

Regras relevantes:

- `Atividade.aluno`, `Atividade.professorOrientador` e `Atividade.turma` sao obrigatorios.
- `Atividade.professorTutor` e opcional.
- `Atividade.atividadePai` permite representar atividades-filhas.
- `Feedback` sempre pertence a uma atividade e a um professor.
- O status `ALTA` representa encerramento e, conforme regra atual, deve ser irreversivel.

## Observacoes para evolucao

O modelo atual pressupoe que uma `Atividade` ja possui aluno executante e turma definidos.
Se o fluxo da clinica exigir agendamentos ou demandas ainda sem aluno atribuido, recomenda-se
criar uma entidade anterior, como `DemandaClinica` ou `EncaminhamentoClinico`, em vez de tornar
`Atividade.aluno` opcional sem revisar as regras atuais.

Essa entidade futura poderia ser criada por um sistema de agendamento e, posteriormente,
convertida em `Atividade` quando aluno, turma e professores forem definidos.
