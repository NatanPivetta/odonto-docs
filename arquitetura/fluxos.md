# Fluxos Principais — OdontoUFRGS

---

## 1. Primeiro Acesso (Cadastro de Aluno)

<img src="../img/fluxos-primeiro-acesso.png" width="1000">

---

## 2. Autenticação

<img src="../img/fluxos-autenticacao.png" width="1000">


---

## 3. Ciclo de Vida do Status de uma Atividade

<img src="../img/fluxos-ciclo-de-vida-atv.png" width="1000">

---

## 4. Criação de Atividade pelo Aluno

<img src="../img/fluxos-criacao-atividade-aluno.png" width="1000">

---

## 5. Criação de Atividade pelo Professor


<img src="../img/fluxos-criacao-atividade-professor.png" width="1000">

---

## 6. Atualização de Status pelo Professor (Alta do Paciente)


<img src="../img/fluxos-atualizacao-status-professor.png" width="1000">

---

> **Regras de matrícula (seções 7 e 8).** Cada turma é específica de um semestre (turmas de
> semestres diferentes têm `id` diferente). Não se matricula em turma **encerrada**
> (`active = false`): para o semestre seguinte cria-se uma turma nova. Um aluno tem no máximo
> **uma matrícula ativa** por vez — ao entrar numa turma nova, as matrículas ativas anteriores
> são desativadas. A checagem de "já matriculado" considera apenas matrículas **ativas**; se o
> aluno já esteve nesta (mesma) turma aberta, a matrícula é **reativada** em vez de duplicada.

## 7. Matrícula de Aluno em Turma


<img src="../img/fluxos-matricula-professor.png" width="1000">

---

## 8. Matrícula em Lote (Bulk)


<img src="../img/fluxos-matricula-lote-professor.png" width="1000">

> **Nota sobre status HTTP.** Os fluxogramas indicam `400` para validações por consistência
> conceitual, mas a implementação atual responde **`409 Conflict`** para `BusinessException`
> (ver item 12 em [bugs-e-backlog.md](../bugs-e-backlog.md)).