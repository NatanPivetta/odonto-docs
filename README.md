# OdontoUFRGS — Kit de Transição (Documentação)

Documentação central do **Sistema de Gerenciamento de Atividades Clínicas da Faculdade de
Odontologia da UFRGS**. Este repositório é o **Guia de Sobrevivência** do projeto: reúne, num
único lugar, tudo que uma nova equipe precisa para **entender, instalar, operar e evoluir** o
sistema — backend, frontend e infraestrutura (Terraform/AWS).

> Os repositórios de código apontam para cá:
> - **Backend** (Spring Boot): [NatanPivetta/odontoBackend](https://github.com/NatanPivetta/odontoBackend)
> - **Frontend** (Next.js): [NatanPivetta/odonto](https://github.com/NatanPivetta/odonto)
> - **Infra** (Terraform/AWS) — opcional: [NatanPivetta/odonto-terraform](https://github.com/NatanPivetta/odonto-terraform)

---

## O que é o sistema

Professores gerenciam turmas, matrículas e atividades clínicas; alunos registram e acompanham
seus próprios procedimentos e recebem feedback. Login por **número de cartão (9 dígitos) +
senha**; alunos podem se auto-cadastrar via fluxo de **primeiro acesso** (código por e-mail).

| Módulo      | Stack                                          | Porta (local) | Deploy            |
|-------------|------------------------------------------------|---------------|-------------------|
| **Backend** | Java 21 · Spring Boot 3.4 · PostgreSQL · Maven | `8090`        | AWS EC2 + RDS     |
| **Frontend**| Next.js 16 · React 19 · TypeScript · Tailwind  | `3000`        | Vercel            |
| **Infra** *(opcional)* | Terraform · AWS (VPC, EC2, RDS, ECR, SSM, IAM) | — | `terraform apply` — ver [NatanPivetta/odonto-terraform](https://github.com/NatanPivetta/odonto-terraform) |

---

## Índice

### Arquitetura
- [Visão geral](arquitetura/visao-geral.md) — módulos, stack, comunicação, modelo de domínio.
- [DER (banco de dados)](arquitetura/der.md) — entidades, relacionamentos e tabelas.
- [Casos de uso](arquitetura/casos-de-uso.md) — atores e fronteiras do sistema.
- [Fluxos de negócio](arquitetura/fluxos.md) — sequência, estados e fluxogramas.

### Backend
- [Setup e quick start](backend/setup.md) — pré-requisitos, execução local, variáveis de ambiente.
- [Convenções de código](backend/convencoes.md) — estrutura de módulos, padrões, migrations.

### Frontend
- [Setup e quick start](frontend/setup.md) — execução local, variáveis, organização.

### Infraestrutura (Terraform / AWS) — **opcional**
> A nuvem (AWS + Vercel) é a implantação **de referência** usada até aqui, mas **não é
> obrigatória**. A aplicação roda inteira localmente ou em qualquer servidor com Docker — ver
> [Formas de executar e implantar](#formas-de-executar-e-implantar). A próxima equipe pode:
> herdar a infra AWS atual, recriá-la com o Terraform incluído, ou ignorá-la e hospedar onde
> preferir.

- [Arquitetura AWS](infra/arquitetura-aws.md) — VPC, EC2, RDS, ECR, SSM, Free Tier.
- [Deploy e CI/CD](infra/deploy-ci-cd.md) — pipeline GitHub Actions, OIDC, provisionamento.
- [Runbook de operação](infra/runbook.md) — logs, redeploy, acesso ao RDS, rotação de segredos.

### Qualidade
- [Bugs conhecidos e backlog](bugs-e-backlog.md) — o que foi corrigido e o que falta.

---

## Formas de executar e implantar

A nuvem é **opcional**. Há três caminhos, do mais simples ao mais completo:

| Cenário | Como | Quando usar |
|---|---|---|
| **1. Local (dev)** | `mvnw spring-boot:run` + `npm run dev` + Postgres em Docker | Desenvolvimento e avaliação. **Não precisa de AWS nem Vercel.** |
| **2. Self-hosted (Docker)** | `docker compose up` no backend (+ servir o build do frontend) em qualquer VM/servidor | Subir num servidor próprio, on-premise ou outro provedor. |
| **3. Nuvem de referência** | AWS (EC2 + RDS via Terraform) + Vercel (frontend) | Reaproveitar a infra atual já provisionada. Ver seção [Infraestrutura](#infraestrutura-terraform--aws--opcional). |

Os cenários 1 e 2 cobrem 100% do sistema sem depender de nenhum serviço gerenciado.

## Quick start (resumo)

```bash
# 1. Banco (Docker)
cd backend && cp .env.example .env && docker compose up -d postgres

# 2. Backend  → http://localhost:8090/api  (Swagger em /api/swagger-ui.html)
cd backend && ./mvnw spring-boot:run

# 3. Frontend → http://localhost:3000
cd odontoufrgs && cp .env.example .env.local && npm install && npm run dev
```

Detalhes em [backend/setup.md](backend/setup.md) e [frontend/setup.md](frontend/setup.md).
Usuário admin inicial: cartão `000000001`, senha `Admin@123` (**troque após o 1º login**).

---

## Status de transição

- **MVP concluído:** autenticação (cartão+senha e primeiro acesso), turmas, matrícula
  individual e em lote, atividades (CRUD, tipos, status, alta, atividades-filhas), feedbacks,
  filtros.
- **Em produção:** backend em AWS (EC2 + RDS), frontend na Vercel, deploy contínuo via GitHub
  Actions.
- **Backlog priorizado:** ver [bugs-e-backlog.md](bugs-e-backlog.md).