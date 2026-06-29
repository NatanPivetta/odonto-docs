# Backend — Setup e Quick Start

API REST em **Java 21 · Spring Boot 3.4 · PostgreSQL**, build com Maven (wrapper).
Repositório de código: `NatanPivetta/odontoBackend`.

## Pré-requisitos

| Ferramenta           | Versão            | Observação                                  |
|----------------------|-------------------|---------------------------------------------|
| JDK                  | 21                | Exigido (`pom.xml` → `java.version`).        |
| Docker + Compose     | recente           | Caminho recomendado para o PostgreSQL.       |
| PostgreSQL           | 16                | Só se optar por não usar Docker.             |
| Maven                | —                 | Não precisa instalar: use `./mvnw`.          |

## Quick start

```bash
# 1. Banco via Docker
cp .env.example .env          # ajuste se necessário
docker compose up -d postgres # PostgreSQL em localhost:5432 (odonto/odonto/odonto)

# 2. Variáveis mínimas (ou via .env / gerenciador)
export JWT_SECRET="$(openssl rand -base64 64)"
export MAIL_USERNAME="seu-email@gmail.com"   # necessário p/ primeiro acesso
export MAIL_PASSWORD="sua-senha-de-app"

# 3. Subir a API
./mvnw spring-boot:run        # Windows: mvnw.cmd spring-boot:run
```

- API: `http://localhost:8090/api`
- Swagger UI: `http://localhost:8090/api/swagger-ui.html`
- As tabelas são criadas automaticamente pelo **Liquibase** no startup.

### Usuário administrador inicial

Criado na primeira subida (`config/DataInitializer.java`):

```
Cartão : 000000001
Senha  : Admin@123
```

> ⚠️ **Troque a senha após o primeiro login.** As credenciais são logadas no console apenas
> como conveniência de desenvolvimento.

## Variáveis de ambiente

Arquivo de referência: `backend/.env.example`. Lidas por `application.yml` (execução com
`./mvnw`) e por `docker-compose.yml`.

| Variável                          | Descrição                                                         |
|-----------------------------------|------------------------------------------------------------------|
| `SPRING_DATASOURCE_URL`           | JDBC URL (`localhost` local; `postgres` no compose).             |
| `SPRING_DATASOURCE_USERNAME`      | Usuário do banco.                                                |
| `SPRING_DATASOURCE_PASSWORD`      | Senha do banco.                                                  |
| `POSTGRES_DB/USER/PASSWORD`       | Usados pelo docker-compose para criar o container do Postgres.   |
| `JWT_SECRET`                      | Segredo de assinatura JWT (HS512, ≥ 64 bytes). **Obrigatório em prod.** |
| `JWT_EXPIRATION_MS`               | Validade do token em ms (padrão 24h).                           |
| `CORS_ALLOWED_ORIGINS`            | Origens autorizadas do frontend (CSV).                          |
| `MAIL_USERNAME` / `MAIL_PASSWORD` | Credenciais SMTP (Gmail "senha de app") p/ primeiro acesso.     |
| `SPRING_PROFILES_ACTIVE`          | Perfil ativo. Existe apenas `local`; deixe vazio para o base.    |

## Execução via Docker Compose (Postgres + API)

```bash
cp .env.example .env
docker compose up -d --build      # sobe postgres + api (porta 8090)
# opcional: docker compose --profile tools up -d pgadmin  → http://localhost:5050
```

## Testes

```bash
./mvnw test     # sobe um PostgreSQL real via Testcontainers (requer Docker)
```

## Acesso ao banco em produção (RDS)

O RDS é privado; o acesso é feito por túnel SSM através da EC2. Ver
[infra/runbook.md](../infra/runbook.md#acessar-o-rds).

Próximo: [convenções de código](convencoes.md).