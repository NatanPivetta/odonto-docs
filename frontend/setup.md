# Frontend — Setup e Quick Start

Interface web em **Next.js 16 (App Router) · React 19 · TypeScript · Tailwind CSS · Radix UI**.
Deploy na **Vercel** (auto-deploy a cada push).

## Pré-requisitos

| Ferramenta | Versão            |
|------------|-------------------|
| Node.js    | 20 LTS ou superior |
| npm        | 10+               |

## Quick start

```bash
cp .env.example .env.local   # confira NEXT_PUBLIC_API_URL
npm install
npm run dev                  # http://localhost:3000
```

Suba o backend antes (ver [backend/setup.md](../backend/setup.md)).

## Variáveis de ambiente

| Variável               | Descrição                                                              |
|------------------------|------------------------------------------------------------------------|
| `NEXT_PUBLIC_API_URL`  | URL base da API, **incluindo `/api`**.                                 |

- Local: `http://localhost:8090/api`
- Produção (Vercel): `http://<EIP-da-EC2>/api` (ver [infra/runbook.md](../infra/runbook.md))

> **Mixed content:** o backend roda em HTTP e a Vercel serve HTTPS — o browser pode bloquear a
> chamada. Para homologação/produção, colocar TLS na frente do backend (Caddy/Nginx +
> Let's Encrypt na EC2) ou um domínio com certificado.

## Organização

```
src/
├── app/          rotas (App Router):
│                   (auth)/ login · primeiro-acesso
│                   (app)/  dashboard · atividades · alunos/turmas · administracao/professores
├── components/   ui/ (Button, Input, Badge, modais, ComboboxSelect…) e layout/ (Sidebar, BottomNav)
├── config/       navigation.ts
├── hooks/        useTheme, useSidebar
├── lib/
│   ├── api.ts            cliente HTTP (fetch + token + tratamento de 401)
│   ├── auth.tsx          AuthProvider / useAuth (contexto de sessão)
│   └── services/         um arquivo por domínio (atividades, turmas, users, professores, feedbacks, auth)
└── types/        tipos compartilhados (Role, enums, DTOs, TIPO_ATIVIDADE_OPTIONS)
```

### Pontos de atenção

- **Token JWT** guardado em `localStorage` via `tokenStorage` (`lib/api.ts`) — desenhado como
  ponto de extensão para futura troca por cookies/OAuth.
- **401 global:** `api.ts` intercepta respostas 401, limpa a sessão e redireciona para `/login`.
- **Guarda de rotas:** o layout `(app)/layout.tsx` redireciona para `/login` se não houver
  usuário autenticado.

## Scripts

| Comando         | Descrição                    |
|-----------------|------------------------------|
| `npm run dev`   | Servidor de desenvolvimento. |
| `npm run build` | Build de produção.           |
| `npm start`     | Serve o build de produção.   |
| `npm run lint`  | ESLint.                      |