# Fluxos Principais — OdontoUFRGS

---

## 1. Primeiro Acesso (Cadastro de Aluno)

```mermaid
sequenceDiagram
    actor Aluno
    participant API
    participant DB
    participant Email as Serviço de E-mail

    Aluno->>API: POST /auth/verify/send { email }
    API->>DB: Busca usuário pelo e-mail
    alt E-mail não encontrado
        API-->>Aluno: 400 — e-mail não cadastrado
    else E-mail encontrado
        API->>DB: Gera código de 6 dígitos (TTL 15 min)
        API->>Email: Dispara e-mail com código
        API-->>Aluno: 204 No Content

        Aluno->>API: POST /auth/register { email, code, cardNumber, password }
        API->>DB: Valida código (não expirado, não usado)
        alt Código inválido ou expirado
            API-->>Aluno: 400 — código inválido
        else Código válido
            API->>DB: Salva hash da senha e ativa conta
            API->>DB: Marca código como usado
            API-->>Aluno: 201 — { token JWT }
        end
    end
```

---

## 2. Autenticação

```mermaid
sequenceDiagram
    actor Usuário as Aluno / Professor
    participant API
    participant DB

    Usuário->>API: POST /auth/login { cardNumber, password }
    API->>DB: Busca usuário pelo cardNumber
    alt Usuário não encontrado ou inativo
        API-->>Usuário: 401 — credenciais inválidas
    else Usuário encontrado
        API->>API: Verifica hash da senha (BCrypt)
        alt Senha incorreta
            API-->>Usuário: 401 — credenciais inválidas
        else Senha correta
            API->>API: Gera JWT (exp. 24h) com id + role
            API-->>Usuário: 200 — { token JWT }
        end
    end

    Note over Usuário,API: Todas as requisições subsequentes<br/>devem incluir Authorization: Bearer {token}
```

---

## 3. Ciclo de Vida do Status de uma Atividade

```mermaid
stateDiagram-v2
    direction LR

    [*] --> PENDENTE : criação com data futura\n(forçado pelo sistema)
    [*] --> QUALQUER : criação com data atual/passada\n(status informado pelo criador)

    QUALQUER : EM_ANDAMENTO / CONCLUIDA / PENDENTE

    PENDENTE --> EM_ANDAMENTO : Aluno ou Professor
    PENDENTE --> CONCLUIDA   : Aluno ou Professor
    EM_ANDAMENTO --> PENDENTE  : Aluno ou Professor
    EM_ANDAMENTO --> CONCLUIDA : Aluno ou Professor
    CONCLUIDA --> EM_ANDAMENTO : Aluno ou Professor
    CONCLUIDA --> ALTA         : apenas Professor\n(status deve ser CONCLUIDA)

    ALTA --> [*]

    note right of ALTA
        Alta encerra o tratamento do paciente.
        Irreversível — nenhuma transição posterior.
    end note
```

---

## 4. Criação de Atividade pelo Aluno

```mermaid
flowchart TB
    subgraph ALUNO
        direction TB
        A1([Preenche formulário\ntipo · prontuário · data\nprofessor orientador · observações])
        A2([Recebe atividade criada])
    end

    subgraph SISTEMA
        direction TB
        S1{Possui\nmatrícula ativa?}
        S2[Busca turma ativa\ndo aluno]
        S3{tipo = OUTROS\ne tipoDescricao\nvazio?}
        S4{dataConclusão\nno futuro?}
        S5{data de início\nno futuro?}
        S6[status = PENDENTE]
        S7[mantém status\ninformado]
        S8[(Salva atividade)]
        ERR1[400 — Aluno sem turma ativa]
        ERR2[400 — tipoDescricao obrigatório]
        ERR3[400 — dataConclusão não pode\nser futura]
    end

    A1 -->|POST /v1/atividades/aluno| S1
    S1 -->|Não| ERR1
    S1 -->|Sim| S2
    S2 --> S3
    S3 -->|Sim| ERR2
    S3 -->|Não| S4
    S4 -->|Sim| ERR3
    S4 -->|Não| S5
    S5 -->|Sim — força| S6
    S5 -->|Não| S7
    S6 --> S8
    S7 --> S8
    S8 --> A2
```

---

## 5. Criação de Atividade pelo Professor

```mermaid
flowchart TB
    subgraph PROFESSOR
        direction TB
        P1([Preenche formulário\nalunoId · turmaId · tipo · prontuário\ndata · professorOrientadorId · status])
        P2([Recebe atividade criada])
    end

    subgraph SISTEMA
        direction TB
        S1{Usuário informado\ntem role ALUNO?}
        S2{tipo = OUTROS\ne tipoDescricao\nvazio?}
        S3{dataConclusão\nno futuro?}
        S4{data de início\nno futuro?}
        S5[status = PENDENTE]
        S6[mantém status\ninformado]
        S7[(Salva atividade)]
        ERR1[400 — usuário não é aluno]
        ERR2[400 — tipoDescricao obrigatório]
        ERR3[400 — dataConclusão não pode\nser futura]
    end

    P1 -->|POST /v1/atividades| S1
    S1 -->|Não| ERR1
    S1 -->|Sim| S2
    S2 -->|Sim| ERR2
    S2 -->|Não| S3
    S3 -->|Sim| ERR3
    S3 -->|Não| S4
    S4 -->|Sim — força| S5
    S4 -->|Não| S6
    S5 --> S7
    S6 --> S7
    S7 --> P2
```

---

## 6. Atualização de Status pelo Professor (Alta do Paciente)

```mermaid
flowchart TB
    subgraph PROFESSOR
        direction TB
        P1([Seleciona atividade\ne novo status])
        P2([Recebe atividade atualizada])
    end

    subgraph SISTEMA
        direction TB
        S1{Atividade\nexiste e está ativa?}
        S2{Atividade\nstatus = ALTA?}
        S3{status\nsolicitado = ALTA?}
        S4{status atual\n= CONCLUIDA?}
        S5[(Atualiza status)]
        ERR1[404 — atividade não encontrada]
        ERR2[409 — atividade já em ALTA\nirreversível]
        ERR3[400 — alta exige status CONCLUIDA]
    end

    P1 -->|PATCH /v1/atividades/id/status| S1
    S1 -->|Não| ERR1
    S1 -->|Sim| S2
    S2 -->|Sim| ERR2
    S2 -->|Não| S3
    S3 -->|Não — atualiza livremente| S5
    S3 -->|Sim| S4
    S4 -->|Não| ERR3
    S4 -->|Sim| S5
    S5 --> P2
```

---

> **Regras de matrícula (seções 7 e 8).** Cada turma é específica de um semestre (turmas de
> semestres diferentes têm `id` diferente). Não se matricula em turma **encerrada**
> (`active = false`): para o semestre seguinte cria-se uma turma nova. Um aluno tem no máximo
> **uma matrícula ativa** por vez — ao entrar numa turma nova, as matrículas ativas anteriores
> são desativadas. A checagem de "já matriculado" considera apenas matrículas **ativas**; se o
> aluno já esteve nesta (mesma) turma aberta, a matrícula é **reativada** em vez de duplicada.

## 7. Matrícula de Aluno em Turma

```mermaid
flowchart TB
    subgraph PROFESSOR
        direction TB
        P1([Seleciona aluno e turma])
        P2([Visualiza turma atualizada])
    end

    subgraph SISTEMA
        direction TB
        S1{Turma está\naberta?}
        S2{Usuário informado\ntem role ALUNO?}
        S3{Aluno já está\nmatriculado\nnesta turma?}
        S4[Busca matrículas\nativas do aluno\nem outras turmas]
        S5{Existem matrículas\nativas anteriores?}
        S6[Desativa matrículas\nanteriores\nactive = false]
        S7[(Cria nova matrícula\nactive = true)]
        ERR1[400 — turma encerrada]
        ERR2[400 — usuário não é aluno]
        ERR3[409 — aluno já matriculado]
    end

    P1 -->|POST /v1/turmas/id/alunos/alunoId| S1
    S1 -->|Não| ERR1
    S1 -->|Sim| S2
    S2 -->|Não| ERR2
    S2 -->|Sim| S3
    S3 -->|Sim| ERR3
    S3 -->|Não| S4
    S4 --> S5
    S5 -->|Sim| S6
    S5 -->|Não| S7
    S6 --> S7
    S7 --> P2
```

---

## 8. Matrícula em Lote (Bulk)

```mermaid
flowchart TB
    subgraph PROFESSOR
        direction TB
        P1([Informa lista de IDs\nde alunos])
        P2([Visualiza turma atualizada])
    end

    subgraph SISTEMA
        direction TB
        S1{Turma está\naberta?}
        S2[Para cada aluno da lista]
        S3{Role = ALUNO?}
        S4{Já matriculado\nnesta turma?}
        S5[Desativa matrículas\nativas anteriores]
        S6[(Cria matrícula\nnesta turma)]
        S7{Mais alunos?}
        ERR1[400 — turma encerrada]
    end

    P1 -->|POST /v1/turmas/id/alunos/bulk| S1
    S1 -->|Não| ERR1
    S1 -->|Sim| S2
    S2 --> S3
    S3 -->|Não — ignora| S7
    S3 -->|Sim| S4
    S4 -->|Sim — ignora| S7
    S4 -->|Não| S5
    S5 --> S6
    S6 --> S7
    S7 -->|Sim| S2
    S7 -->|Não| P2
```

> **Nota sobre status HTTP.** Os fluxogramas indicam `400` para validações por consistência
> conceitual, mas a implementação atual responde **`409 Conflict`** para `BusinessException`
> (ver item 13 em [bugs-e-backlog.md](../bugs-e-backlog.md)).
