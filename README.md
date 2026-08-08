# CryptoPay

Plataforma de carteira cripto **não custodial**: adicione um endereço
público (QR Code, texto ou WalletConnect), consulte saldo direto na
blockchain, veja o valor estimado em reais e, quando um parceiro de
conversão e um PSP Pix estiverem configurados, venda para BRL e receba
via Pix.

Web • PWA • Android (Capacitor) — monorepo **npm**, **sem Docker**,
pronto para rodar localmente com `npm install && npm run dev` e para
hospedar (Netlify + qualquer host Node para a API).

> **Este é o estado real do projeto.** O código é funcional e testado
> de ponta a ponta (auth, carteiras, blockchain, cotação, máquina de
> estados de venda, painel admin), mas **não está pronto para operar
> financeiramente em produção** até que: (1) um parceiro Off-Ramp real
> seja integrado, (2) um PSP Pix real seja integrado, (3) um provedor de
> KYC/AML real seja integrado, e (4) compliance/jurídico validem o
> modelo de negócio no Brasil. Ver [`docs/security.md`](./docs/security.md)
> e a seção Compliance abaixo.

## Por que não custodial

O sistema **nunca** pede ou armazena seed phrase / chave privada.
Endereços são usados **somente para consulta** de saldo/histórico.
Autorização de operações acontece na própria carteira do usuário
(WalletConnect). Ver [`docs/security.md`](./docs/security.md).

## Stack

| Camada | Tecnologias |
|---|---|
| Frontend | Next.js 14, React 18, TypeScript, Tailwind CSS, TanStack Query, Zustand, React Hook Form + Zod, PWA (`@ducanh2912/next-pwa`) |
| Backend | NestJS 10, TypeScript, Prisma ORM, PostgreSQL, JWT (access+refresh), Swagger/OpenAPI |
| Blockchain | BlockCypher (Bitcoin), Alchemy (Ethereum/EVM), RPC genérico, `ethers`, `bitcoin-address-validation` |
| Carteira | WalletConnect v2 (`@walletconnect/sign-client`) |
| Cotação | CoinGecko (API pública) |
| QR Code | `html5-qrcode` (leitura), `qrcode.react` (exibição) |
| Mobile | Capacitor (wrapper Android em torno do PWA) |
| Testes | Vitest (pacotes/web), Jest (API), Playwright (E2E) |
| CI | GitHub Actions (lint, typecheck, testes, build, E2E) |
| Hospedagem | Netlify (frontend) + qualquer host Node — Render/Railway/Fly.io/VPS (backend) |

Sem Docker: o backend fala diretamente com PostgreSQL/Redis via
`DATABASE_URL`/`REDIS_URL` — local (instalado na máquina) ou gerenciado
(Neon/Supabase/Railway/Upstash).

## Estrutura do monorepo

```
cryptopay/
├── apps/
│   ├── web/           # Next.js — Web/PWA
│   ├── api/            # NestJS — API REST
│   └── mobile/          # Capacitor — wrapper Android do PWA
│
├── packages/
│   ├── types/          # Tipos/contratos compartilhados
│   ├── config/          # Carregamento/validação (Zod) de env vars do backend
│   ├── database/        # Schema Prisma, client, migrations, seed
│   ├── blockchain/       # Adapters de blockchain (BlockCypher/Alchemy/RPC)
│   ├── wallet/           # Parsing de QR Code/URI de carteira
│   ├── pricing/          # Pricing Service (CoinGecko) + cálculo de taxas
│   ├── payments/         # Off-Ramp Provider (interface — sem parceiro real)
│   ├── pix/              # Pix Provider (interface — sem PSP real)
│   ├── kyc/              # KYC Provider (interface — sem provedor real)
│   └── ui/               # Design system compartilhado (Tailwind + CVA)
│
├── docs/                # Documentação técnica (ver índice abaixo)
├── tests/e2e/            # Testes Playwright (fluxo completo)
├── scripts/              # Scripts utilitários
├── .github/workflows/     # CI (lint/test/build) e deploy
│
├── .env.example
├── package.json          # Workspaces npm — orquestra tudo
└── netlify.toml
```

Cada pacote tem sua própria explicação no topo do arquivo principal e
em [`docs/architecture.md`](./docs/architecture.md).

## Requisitos

- Node.js ≥ 20 e npm ≥ 10
- PostgreSQL 14+ (local ou gerenciado)
- Redis (opcional em desenvolvimento — recomendado em produção)

Sem Docker necessário. Instalar Postgres localmente:

```bash
# macOS (Homebrew)
brew install postgresql@16 && brew services start postgresql@16

# Ubuntu/Debian
sudo apt install postgresql redis-server
```

Ou use um banco gerenciado gratuito para desenvolvimento (Neon,
Supabase, Railway) e pule a instalação local — só precisa da
`DATABASE_URL`.

## Início rápido (100% local, sem Docker)

```bash
git clone <repo>
cd cryptopay

cp .env.example .env
# edite .env: DATABASE_URL, JWT_ACCESS_SECRET, JWT_REFRESH_SECRET
# (gere segredos com: openssl rand -hex 48)

npm install          # instala tudo e builda os packages/* automaticamente
npm run db:migrate   # cria as tabelas no Postgres configurado em .env
npm run db:seed      # cria um usuário admin e o catálogo de integrações

npm run dev          # sobe API (porta 3001) e Web (porta 3000) juntos
```

| Serviço | URL |
|---|---|
| Frontend | http://localhost:3000 |
| API | http://localhost:3001/api/v1 |
| Swagger | http://localhost:3001/docs |
| Prisma Studio | `npm run db:studio` |

Login do admin criado pelo seed: `admin@cryptopay.local` — senha
definida por `SEED_ADMIN_PASSWORD` no `.env` (padrão de
desenvolvimento: `TrocarEstaSenha123!`, **troque em qualquer ambiente
compartilhado**).

Guia completo (passo a passo com troubleshooting):
[`docs/setup.md`](./docs/setup.md).

## Scripts principais (raiz)

| Script | O que faz |
|---|---|
| `npm run dev` | Builda os packages e sobe API + Web em modo desenvolvimento |
| `npm run build` | Build de produção de todos os packages e apps |
| `npm run start` | Roda API + Web já buildados (produção local) |
| `npm run lint` / `npm run typecheck` | Qualidade de código em todos os workspaces |
| `npm run test` | Testes unitários (Vitest/Jest) em todos os workspaces |
| `npm run test:e2e` | Testes E2E (Playwright) — exige API+Web rodando |
| `npm run db:migrate` | Cria/aplica uma migration Prisma (dev) |
| `npm run db:migrate:deploy` | Aplica migrations existentes (produção) |
| `npm run db:seed` | Popula usuário admin + catálogo de integrações |
| `npm run db:studio` | Abre o Prisma Studio |

## Documentação

- [`docs/architecture.md`](./docs/architecture.md) — arquitetura, fluxo de dados, diagrama
- [`docs/setup.md`](./docs/setup.md) — passo a passo de instalação local
- [`docs/database.md`](./docs/database.md) — schema, migrations, seed
- [`docs/api.md`](./docs/api.md) — endpoints REST
- [`docs/blockchain.md`](./docs/blockchain.md) — provedores de blockchain
- [`docs/wallet.md`](./docs/wallet.md) — carteiras, QR Code, WalletConnect
- [`docs/payments.md`](./docs/payments.md) — conversão cripto → BRL (Off-Ramp)
- [`docs/pix.md`](./docs/pix.md) — integração Pix
- [`docs/kyc.md`](./docs/kyc.md) — KYC/AML
- [`docs/security.md`](./docs/security.md) — modelo de segurança
- [`docs/deployment.md`](./docs/deployment.md) — GitHub, Netlify, backend, domínio
- [`docs/troubleshooting.md`](./docs/troubleshooting.md) — problemas comuns

## O que está pronto vs. o que precisa de um parceiro real

| Módulo | Status |
|---|---|
| Cadastro, login, sessão (JWT + refresh), RBAC | ✅ Funcional |
| Carteiras (QR Code, manual, WalletConnect) | ✅ Funcional |
| Consulta de saldo/tokens/transações (Bitcoin, Ethereum) | ✅ Funcional (usa BlockCypher/Alchemy reais) |
| Cotação em BRL | ✅ Funcional (usa CoinGecko real) |
| Cálculo de taxas/spread, Quote com expiração | ✅ Funcional |
| Máquina de estados da venda (CREATED → ... → COMPLETED) | ✅ Funcional |
| Painel administrativo (dashboard, usuários, auditoria, webhooks, integrações) | ✅ Funcional |
| Conversão cripto → BRL (Off-Ramp) | ⛔ Requer parceiro real — ver `docs/payments.md` |
| Pagamento Pix | ⛔ Requer PSP real — ver `docs/pix.md` |
| KYC/AML | ⛔ Requer provedor real — ver `docs/kyc.md` |
| Notificações por e-mail | ⛔ Requer SMTP configurado |

Nada nessas três últimas linhas é simulado: os endpoints existem, a
arquitetura (interfaces `OffRampProvider`/`PixProvider`/`KycProvider`)
está pronta para receber um adapter real, mas sem credenciais de um
parceiro autorizado eles retornam um erro claro em vez de fingir
sucesso — como a especificação original exige.

## Compliance

Antes de operar comercialmente no Brasil, validar com jurídico/compliance:
legislação de ativos virtuais, prevenção à lavagem de dinheiro,
regras do Pix, obrigações fiscais, contratos com parceiros e licenças
necessárias. Este projeto não afirma estar autorizado a operar
financeiramente.

## Licença

Ver [`LICENSE`](./LICENSE).
