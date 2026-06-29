# Infraestrutura — Runbook de operação

> **Opcional / referência.** Procedimentos para operar o ambiente **AWS** atual. Só se aplica a
> quem mantiver a infra de nuvem. Para execução local/self-hosted, ver
> [Formas de executar e implantar](../README.md#formas-de-executar-e-implantar).

Pré-requisitos: AWS CLI autenticada (região `us-east-1`) e o **Session Manager plugin**
instalado. Valores do ambiente atual: instância `odonto-dev-api`
(`i-055108cec45fdb20a`), RDS `odonto-dev-postgres.cml4geyiowe1.us-east-1.rds.amazonaws.com`,
banco/usuário `odonto`.

## Acessar a EC2 (sem SSH)

```bash
aws ssm start-session --target <EC2_INSTANCE_ID>
```

## Ver logs do container

```bash
aws ssm start-session --target <EC2_INSTANCE_ID>
# dentro da instância:
docker logs -f odonto-api
# ou via journald:
journalctl -u odonto-api -f
```

## Forçar redeploy manual

```bash
aws ssm send-command \
  --instance-ids <EC2_INSTANCE_ID> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["/opt/odonto/run.sh latest"]'
```

## Acessar o RDS

O RDS é privado; conecta-se por **port forwarding via SSM** através da EC2.

**1) Abrir o túnel** (deixe o terminal aberto):

```bash
aws ssm start-session \
  --target i-055108cec45fdb20a \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters "host=odonto-dev-postgres.cml4geyiowe1.us-east-1.rds.amazonaws.com,portNumber=5432,localPortNumber=15432"
```

**2) Conectar com psql** (outro terminal):

```bash
export PGPASSWORD=$(aws ssm get-parameter --name /odonto-dev/db/password --with-decryption --query Parameter.Value --output text)
psql -h localhost -p 15432 -d odonto -U odonto
```

PowerShell: trocar `export VAR=$(...)` por `$env:PGPASSWORD = (...)`. Se o `psql` exigir SSL:
`psql "host=localhost port=15432 dbname=odonto user=odonto sslmode=require"`.

Ferramentas gráficas (DBeaver, pgAdmin) conectam em `localhost:15432` com as mesmas credenciais
enquanto o túnel estiver aberto.

## Inspecionar / rotacionar segredos (SSM)

```bash
# ver um parâmetro
aws ssm get-parameter --name /odonto-dev/db/password --with-decryption --query Parameter.Value --output text

# rotacionar (ex.: senha do banco)
aws ssm put-parameter --name /odonto-dev/db/password --type SecureString --value "nova-senha" --overwrite
# depois: aplicar a senha no RDS (console/CLI) e disparar redeploy para o app recarregar
```

Parâmetros disponíveis: `/odonto-dev/{db/url,db/username,db/password,jwt/secret,mail/username,mail/password}`.

## ⚠️ Higiene de segredos (fazer antes de publicar os repos)

Recomendado:
1. **Rotacionar** App Password do Gmail, senha do RDS e JWT secret.
2. Adicionar ao `.gitignore`: `*.tfvars`, `*.tfstate`, `*.tfstate.*`, `.terraform/`.
3. Migrar o state para backend remoto (S3 + DynamoDB), como `providers.tf` já sugere em comentário.
4. Remover os segredos do histórico do git (ex.: `git filter-repo`) se os repos forem públicos.

Ver também o item de segurança em [bugs-e-backlog.md](../bugs-e-backlog.md).
