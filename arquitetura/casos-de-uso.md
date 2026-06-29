# Casos de uso

```mermaid
flowchart LR
    aluno([👤 Aluno])
    prof([👤 Professor])
    email{{Serviço de E-mail}}

    subgraph SISTEMA["Sistema OdontoUFRGS"]
        UC_primeiro((Primeiro acesso\nauto-cadastro))
        UC_login((Login))
        UC_criarAtivAluno((Registrar atividade))
        UC_editarAtivAluno((Editar dados da\nprópria atividade))
        UC_statusAluno((Atualizar status\nda própria atividade))
        UC_minhas((Ver minhas atividades))
        UC_verFb((Ver feedbacks))

        UC_turmas((Gerenciar turmas))
        UC_matricular((Matricular alunos\nindividual / em lote))
        UC_remover((Remover alunos\nda turma))
        UC_criarAtiv((Criar / editar /\nexcluir atividade))
        UC_status((Atualizar status))
        UC_alta((Registrar alta\ndo paciente))
        UC_feedback((Dar feedback))
        UC_usuarios((Cadastrar alunos\ne professores))
        UC_filtrar((Listar e filtrar\natividades))
    end

    aluno --- UC_primeiro
    aluno --- UC_login
    aluno --- UC_criarAtivAluno
    aluno --- UC_editarAtivAluno
    aluno --- UC_statusAluno
    aluno --- UC_minhas
    aluno --- UC_verFb

    prof --- UC_login
    prof --- UC_turmas
    prof --- UC_matricular
    prof --- UC_remover
    prof --- UC_criarAtiv
    prof --- UC_status
    prof --- UC_alta
    prof --- UC_feedback
    prof --- UC_usuarios
    prof --- UC_filtrar

    UC_primeiro -. envia código .-> email
```

## Regras de fronteira

- O **aluno** só enxerga e altera as **próprias** atividades; a turma da atividade é inferida
  da sua matrícula ativa.
- **Registrar alta** é exclusivo do **professor** e só é permitido sobre atividades em status
  `CONCLUIDA`. Uma atividade em `ALTA` é terminal — seu status não pode mais ser alterado.
- O cadastro via **primeiro acesso** cria apenas usuários com papel `ALUNO`.
- **Matrícula** só é permitida em turmas **abertas** (`active = true`); turmas de semestres
  encerrados não aceitam novas matrículas.