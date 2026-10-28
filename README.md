# claude-skills-curadas

Seleção de skills de [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills) (v0.4.20, MIT, ver `LICENSE`), copiadas em 20/10/2026.

| Skill | Origem | Para quê |
|-------|--------|----------|
| `the-fool` | intacta | Questionar plano/arquitetura antes de começar |
| `spec-miner` | intacta | Extrair especificação de código legado |
| `security-reviewer` | intacta | Revisão de segurança, SAST, segredos |
| `secure-code-guardian` | + `references/fullstack-security-checklist.md` (vinda da `fullstack-guardian`) | Escrever código seguro (auth, validação, OWASP) |
| `postgres-pro` | **atualizada** | PostgreSQL 18 (19 a caminho): consultas lentas, índices, migrações sem travar o banco, JSONB/SQL-JSON, pgvector 0.8, pooling, upgrades, hospedagem (Neon, Render, Railway, Supabase) |
| `playwright-expert` | **atualizada** | Playwright 1.64: setup project de login, dados por teste, aria snapshots, clock, CI com shards, test agents/MCP |
| `nestjs-expert` | **reescrita** | NestJS 12 + Prisma 7 (adapter, migrações, transações, filtro de erros), class-validator ou Zod (Standard Schema), JWT com refresh rotativo, testes com banco real, deploy no Render e no Railway |
| `nextjs-developer` | **reescrita (v2.0.0)** | Next.js 16.4 + deploy Vercel e Cloudflare (OpenNext) |
| `react-modern` | **nova** | React 19.3 sem framework: hooks, effects, React Compiler, forms, state |
| `react-native` | **nova** | Expo SDK 57/58 ou RN puro, brownfield, monorepo com Next; módulos nativos (Expo Modules/Turbo/Nitro), push, offline, Sentry, i18n, estilização |

Ajustes nas intactas: removidas referências `related-skills` e links para skills que não vieram.

## Como usar

Todas as skills, em todos os projetos:

```bash
mkdir -p ~/.claude/skills
ln -s ~/Projects/github/claude-skills-curadas/skills/* ~/.claude/skills/
```

Só em um projeto (o repositório do projeto passa a carregar a skill):

```bash
mkdir -p .claude/skills
cp -r ~/Projects/github/claude-skills-curadas/skills/nextjs-developer .claude/skills/
```

O link simbólico mantém uma única fonte. A cópia congela a versão dentro do projeto.

Combinações por tipo de projeto: Next.js → `nextjs-developer` + `react-modern`. Vite/React → `react-modern`. React Native (Expo ou puro) → `react-native` + `react-modern`. Se usar estas, desative `frontend:react-native` nos projetos para não competirem.

## Onde fica cada deploy

| Parte | Plataformas | Skill → referência |
|-------|-------------|--------------------|
| Next.js | Vercel, Cloudflare (OpenNext) | `nextjs-developer` → `deploy-vercel.md`, `deploy-cloudflare.md` |
| API NestJS | Render, Railway | `nestjs-expert` → `deploy-render-railway.md` |
| PostgreSQL | Neon (padrão), Render, Railway, Supabase | `postgres-pro` → `hosting.md` |
| App mobile | EAS (Expo), lojas | `react-native` → `release-eas.md`, `bare-cli.md` |

React com Vite (SPA sem Next) ainda não tem guia de deploy.

## Manutenção: revisar a cada 3 meses

Próximas revisões: **20/01/2027**, 20/04/2027, 20/07/2027, 20/10/2027.

Cada `SKILL.md` atualizada tem `metadata.baseline` com as versões e a data em que foi conferida. Na revisão:

1. Ver as versões atuais (`npm view <pacote> version` e as release notes) e comparar com o `baseline` de cada skill.
2. Atualizar o que mudou, ajustar o `baseline` e subir `metadata.version`.
3. Rodar o validador: `python3 ../claude-skills/scripts/validate-skills.py --skills-dir skills`. Os erros de `related-skills` e `references/` ausente são convenção do repositório original e podem ser ignorados.

Pode pedir ao Claude: *"revise as skills de claude-skills-curadas contra as versões atuais"*.

| Skill | Conferir | Já marcado para a próxima revisão |
|-------|----------|-----------------------------------|
| `nextjs-developer` | Next.js, `@opennextjs/cloudflare`, wrangler, Vercel | Bugs do `proxy.ts` no OpenNext (#1373, #1400) |
| `react-modern` | React, React Compiler, `eslint-plugin-react-hooks` | — |
| `react-native` | Expo SDK, React Native, React Navigation, NativeWind | React Navigation 8 e NativeWind v5 estáveis? |
| `nestjs-expert` | `@nestjs/core`, `prisma`/`@prisma/client`, Render, Railway | **Prisma 8** (hoje RC, API nova); pre-deploy no IaC do Railway |
| `postgres-pro` | PostgreSQL, pgvector, PgBouncer, Neon | **PostgreSQL 19** (GA previsto para 19/11/2026) |
| `playwright-expert` | `@playwright/test` | — |
| `the-fool`, `spec-miner`, `security-reviewer`, `secure-code-guardian` | Só o OWASP Top 10 (muda raramente) | — |

Para comparar com o repositório original: `cd ../claude-skills && git pull` e ver o que mudou nas skills que vieram intactas.
