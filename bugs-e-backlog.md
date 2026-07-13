# Bugs conhecidos e backlog técnico

Levantamento da preparação do **code freeze**. Itens marcados **✅ corrigido** já foram
ajustados; os demais são backlog priorizado para a próxima equipe.

Severidade: 🔴 alta · 🟠 média · 🟡 baixa.

---

## 🔴 Alta

### 1. Matrícula: turma encerrada e remanejo dentro do semestre ✅ corrigido
**Onde:** `TurmaService.addAluno` / `addAlunosBulk`.

Regra (esclarecida com o demandante): cada turma é de um semestre específico (turmas de
semestres diferentes têm `id` próprio). Aluno reprovado entra na turma do **semestre seguinte**
(turma nova), nunca de volta na anterior, já **encerrada** (`turma.active = false`).

Corrigido: `validarTurmaAberta` bloqueia matrícula em turma encerrada; a duplicidade passou a
considerar só matrículas **ativas** (`matriculaAtivaExiste`); `matricularOuReativar` reativa a
linha existente em remanejo dentro de turma aberta, em vez de duplicar/lançar erro.

### 2. `docker-compose.yml` apontava o Liquibase para changelog inexistente ✅ corrigido
`SPRING_LIQUIBASE_CHANGE_LOG` usava `.xml`; o arquivo é `db.changelog-master.yaml`. Causava
falha de startup em container. Corrigido para `.yaml`.

### 3. Porta do container divergente da aplicação ✅ corrigido
App escuta em **8090**, mas compose/Dockerfile usavam **8080** → API inacessível. Alinhado p/ 8090.

### 4. `.env.example` do frontend com porta errada ✅ corrigido
Apontava `:8080/api`; o backend local roda em **8090**. Corrigido.

---

## 🟠 Média

### 5. Frontend esperava 400, backend retorna 409 ✅ corrigido
`alunos/turmas/[id]/page.tsx` tratava `400` para "já matriculado", mas `BusinessException` →
**409**. A mensagem amigável nunca aparecia. Ajustado para `409`.

### 6. Detalhe de atividade do aluno quebrava com muitas atividades ✅ corrigido
Carregava `listMinhasAtividades({ size: 200 })` e buscava por id no array. Passou a usar
`getAtividadeById(id)`. **Pendência menor (backlog):** listagem de atividades-filhas ainda usa
paginação fixa — expor filtro `atividadePaiId` em `/v1/atividades/minhas` e paginar.

### 7. Status `ALTA` não era irreversível no backend ✅ corrigido
`updateStatus` só validava a transição **para** `ALTA`. Adicionado guard: atividade em `ALTA`
não pode mudar de status.

### 8. Sem tratamento global de 401 ✅ corrigido
`lib/api.ts` agora intercepta 401: limpa o token e redireciona para `/login` (evita loop).

---

## 🟡 Baixa / Segurança

### 9. JWT secret versionado ✅ mitigado
**Onde:** `application.yml` (default de fallback) e `application-local.yml`.
Havia um JWT secret real commitado. Substituído por um **default de dev claramente inseguro**;
em qualquer ambiente real o `JWT_SECRET` deve vir de variável de ambiente (em produção, do AWS
SSM `/odonto-dev/jwt/secret`). **Pendência:** o valor antigo continua no **histórico do git** —
se quiser eliminá-lo de vez, reescrever o histórico (`git filter-repo`) e/ou rotacionar.

### 10. Segredos na pasta Terraform (tfvars/tfstate) ✅ corrigido
**Onde:** `aws terraform deploy/.../terraform/terraform.tfvars`, `terraform.tfstate(.backup)`.
Contêm App Password do Gmail, senha master do RDS e JWT secret em texto puro. A pasta
`aws terraform deploy/` está no `.gitignore` do backend (verificar que não foram commitados
antes da regra com `git ls-files`). Ações: rotacionar os segredos, garantir o gitignore, e
considerar mover o state para backend remoto (S3 + DynamoDB + KMS). Ver [runbook](infra/runbook.md#️-higiene-de-segredos-fazer-antes-de-publicar-os-repos).

### 11. Código de verificação com `java.util.Random`
`AuthService.sendVerificationCode` — trocar por `SecureRandom` (PRNG previsível).

### 12. Ausência de rate-limiting
`/auth/login` e `/auth/verify/send` — sem limite de tentativas (força bruta / spam de e-mail).

### 13. Semântica de status HTTP inconsistente
Toda `BusinessException` retorna **409**, inclusive casos de entrada inválida (ex.: "aluno sem
turma ativa"). Os fluxogramas indicam `400`. Alinhar handler ↔ documentação (usar 400/422 p/
validação e 409 só para conflito real).

### 14. Usuário administrador padrão com senha conhecida
`DataInitializer` cria `000000001 / Admin@123` em qualquer ambiente. Restringir ao perfil local
ou forçar troca de senha no primeiro login.

### 15. Select de turmas no cadastro de atividade (professor) exibe todas as turmas ativas ✅ corrigido
**Onde:** `NewActivityModal.tsx` (frontend); `TurmaRepository`, `TurmaService`, `TurmaResource` (backend).

Ao cadastrar uma atividade para um aluno, o select de turmas listava **todas as turmas ativas**
do sistema em vez das turmas em que o aluno está matriculado.

**Correção aplicada:**
- Backend: novo endpoint `GET /v1/turmas/aluno/{alunoId}` que retorna apenas turmas com matrícula ativa para o aluno.
- Frontend: select de turmas agora é reativo ao aluno selecionado — carrega via `listTurmasDoAluno(alunoId)`. Se o aluno tiver apenas uma turma ativa, é pré-selecionada automaticamente.

**Efeito colateral identificado:** alunos com duas matrículas ativas simultâneas no banco (dado
inconsistente, violação da regra de item 1) causavam 401 no cadastro de atividade pelo aluno —
ver item 17.

---

### 16. Status não inferido automaticamente quando dataConclusão é informada ✅ corrigido
**Onde:** `AtividadeService.resolverStatus` (backend); `NewActivityModal.tsx` (frontend).

Atividade criada com `dataConclusao` preenchida aceitava `status = EM_ANDAMENTO` sem ajuste.

**Correção aplicada:**
- Backend: overload `resolverStatus(data, status, dataConclusao)` — se `dataConclusao != null` e status não for `CONCLUIDA`/`ALTA`, força `CONCLUIDA`. Aplicado nos dois métodos de criação (`create` e `createAsAluno`).
- Frontend: `onChange` da data de conclusão seta `status = 'CONCLUIDA'` automaticamente.

---

### 17. Aluno com duas matrículas ativas causa 401 ao criar atividade ✅ corrigido
**Onde:** `TurmaRepository.findActiveTurmaForAluno`.

`createAsAluno` chama `findActiveTurmaForAluno`, que retorna `Optional<Turma>`. Com dois
registros ativos no banco, o JPA lançava `NonUniqueResultException` — capturado pelo Spring
Security como falha de autenticação e convertido em **401**.

**Correção aplicada:** adicionado `ORDER BY t.id DESC LIMIT 1` na query, tornando-a tolerante
a dados inconsistentes. A causa raiz (duas matrículas ativas) deve ser corrigida diretamente
no banco e está coberta pelo guard do `TurmaService` para novas matrículas (item 1).

---

## Backlog (não-bugs)

- **Testes do frontend:** sem suíte automatizada. Backend tem `AtividadeServiceTest`.
- **OAuth (Google):** login alternativo planejado, mantendo cardNumber + senha como identidade
  canônica (`tokenStorage` em `src/lib/api.ts` é o ponto de extensão).
- **Versionar o código Terraform:** ✅ concluído. Código versionado em
  [NatanPivetta/odonto-terraform](https://github.com/NatanPivetta/odonto-terraform)
  com `.gitignore` excluindo `*.tfvars`, `*.tfstate*` e `.terraform/`.
- **`src.rar`** na raiz do projeto: artefato antigo; avaliar remoção.

## Backlog funcional

- **Disciplinas clinicas e turmas estruturadas:** separar disciplina curricular
  de turma ofertada. Disciplinas iniciais: `ODO99012 - CLINICA ODONTOLOGICA I`,
  `ODO99013 - CLINICA ODONTOLOGICA II`, `ODO99014 - CLINICA ODONTOLOGICA III`,
  `ODO99016 - CLINICA ODONTOLOGICA IV`. Turma deve possuir codigo proprio,
  semestre e disciplina.
- **Escopo de turmas por professor/coordenador/admin:** professor deve visualizar
  apenas turmas vinculadas a ele; coordenador e admin devem visualizar todas as
  turmas do seu escopo. Requer revisao dos papeis atuais e dos filtros de
  listagem.

## Backlog tecnico

- **Enum de disciplinas no backend e banco:** concluido. As disciplinas clinicas
  foram padronizadas por enum/codigo (`ODO99012`, `ODO99013`, `ODO99014`,
  `ODO99016`) e `turmas` passou a possuir `codigo_turma`.
- **Base RBAC:** concluida a estrutura inicial com `roles`, `permissions`,
  `user_roles` e `role_permissions`. Os resources passaram a usar
  `hasAuthority(...)`/`hasAnyAuthority(...)`. Pendente evoluir policies
  contextuais para regras como professor vinculado a turma e aluno acessando
  apenas recursos proprios.
- **Vinculo professor-turma:** modelar tabela associativa futura
  `turma_professores` com `turma_id`, `professor_id`, `tipo_vinculo` e `active`,
  evitando campos fixos em `turmas`.
