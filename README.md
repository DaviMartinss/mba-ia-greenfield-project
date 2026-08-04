# StreamTube — Plataforma de Compartilhamento de Vídeos

Projeto da disciplina **Desenvolvimento de Aplicações de IA** do MBA de Engenharia de Software com IA da [Full Cycle](https://fullcycle.com.br).

Este é um projeto greenfield desenvolvido para demonstrar como construir uma aplicação do zero utilizando IA de forma adequada no processo de desenvolvimento.

## Professor

<a href="https://github.com/argentinaluiz">
    <img src="https://avatars.githubusercontent.com/u/4926329?v=4?s=100" width="100px;" alt=""/>
    <br />
    <sub>
        <b>Luiz Carlos</b>
    </sub>
</a>

---

## Quadro Branco

- [Quadro Branco](./whiteboard.png)

---

## 🎨 Design System (Figma)

- [FC Tube.fig](./FC%20Tube.fig) — arquivo-fonte do **design system** do projeto no Figma.

Contém os fundamentos visuais do StreamTube — tokens (cores, tipografia, espaçamento, raios), componentes e as telas da plataforma. É a referência de design para a implementação do frontend: os componentes em `next-frontend/components/ui` (shadcn) e os tokens em `next-frontend/app/globals.css` derivam deste arquivo. Abra-o no Figma (`Arquivo → Importar`) para consultar especificações e estados visuais.

---

## 📋 Pré-requisitos

- Docker e Docker Compose
- Node.js v25+ (para rodar os testes E2E do Playwright no host)
- npm

## 🏗️ Arquitetura

O projeto é um monorepo baseado em containers Docker. Cada subprojeto sobe sua própria stack via `docker compose`.

- **Frontend** (Next.js 16, App Router + React Server Components) — interface da plataforma. Segue o **modelo BFF**: o navegador nunca chama a API NestJS diretamente; todo tráfego passa por Route Handlers same-origin em `app/api/**`, que fazem proxy server-side para a API.
- **API** (NestJS 11) — regras de negócio, autenticação (JWT + refresh token rotation), upload de vídeos (URLs presignadas), envio de e-mails e acesso ao banco.
- **Database** (PostgreSQL 17) — usuários, canais, tokens de autenticação e vídeos.
- **Email Service** (Mailpit) — captura os e-mails transacionais (confirmação de conta e recuperação de senha) em uma UI local.
- **Video Worker** (FFmpeg) — container dedicado que consome jobs da fila, extrai metadados (`ffprobe`) e gera thumbnail (`ffmpeg`) para cada vídeo enviado.
- **Object Storage** (MinIO, compatível com S3) — armazena os arquivos de vídeo originais e as thumbnails geradas.
- **Message Queue** (BullMQ + Redis) — fila de processamento assíncrono de vídeos; desacopla o upload (síncrono, rápido) do processamento (assíncrono, pesado).

O diagrama de arquitetura completo (C4) está em `docs/diagrams/software-arch.mermaid`.

## 🚀 Como rodar

Os dois subprojetos têm stacks Docker **separadas**. Suba primeiro o backend, rode as migrations e depois o frontend.

### 1. Backend (NestJS + PostgreSQL + Mailpit + MinIO + Redis + Video Worker)

```bash
cd nestjs-project

# Sobe API, banco, Mailpit, MinIO, Redis e o worker de vídeo
docker compose up -d

# Instala dependências (apenas na primeira vez)
docker compose exec nestjs-api npm install

# Cria o schema do banco (obrigatório — synchronize está desabilitado)
docker compose exec nestjs-api npm run migration:run

# Sobe o servidor de desenvolvimento em watch mode
docker compose exec -d nestjs-api npm run start:dev

# Sobe o worker de processamento de vídeo em watch mode
docker compose exec -d video-worker npm run start:worker:dev
```

> Os containers `nestjs-api` e `video-worker` ficam de pé aguardando (`tail -f /dev/null`) até você iniciar o processo Node explicitamente com `docker compose exec` — isso permite reiniciar a aplicação sem recriar o container, mantendo `node_modules`/cache intactos entre reinícios.

Serviços disponíveis:

| Serviço | URL / Porta |
|---------|-------------|
| API NestJS | http://localhost:3000 |
| PostgreSQL | `localhost:5432` (db/user/senha: `streamtube`) |
| Mailpit (UI de e-mails) | http://localhost:8025 |
| MinIO (console) | http://localhost:9001 (API S3 em `:9000`) |
| Redis | `localhost:6379` |
| Swagger (opcional) | http://localhost:3000/api/docs — habilite com `SWAGGER_ENABLED=true` |

### 2. Frontend (Next.js)

```bash
cd next-frontend

# Garanta que o .env.local existe (veja .env.example)
# API_URL aponta para o backend; SESSION_PASSWORD protege a sessão (iron-session)

docker compose up -d
docker compose exec next-frontend npm install        # apenas na primeira vez
docker compose exec -d next-frontend npm run dev
```

A aplicação ficará disponível em **http://localhost:3001**.

> As stacks são separadas, então o frontend acessa o backend via `host.docker.internal:3000` (configurado em `next-frontend/.env.local` e no `extra_hosts` do compose).

> **Fase 03 (upload de vídeos) é escopo de backend.** A interface de upload/reprodução no frontend ainda não foi implementada — os endpoints abaixo são consumidos via API diretamente (Swagger ou `curl`).

## 🧪 Testes

### Backend (Jest)

```bash
cd nestjs-project
docker compose exec nestjs-api npm test               # unitários + integração
docker compose exec nestjs-api npm run test:e2e       # end-to-end (HTTP via supertest)
docker compose exec nestjs-api npm run test:cov       # cobertura
```

Sufixos: `*.spec.ts` (unitário), `*.integration-spec.ts` (integração com banco real), `*.e2e-spec.ts` (end-to-end). Testes de integração/e2e rodam com `--runInBand` (execução sequencial, necessária porque as suítes compartilham o mesmo Postgres real).

Resultado da suíte completa, incluindo a Fase 03 (auth + canais + vídeos + fila + worker):

| Comando | Suítes | Testes | Cobertura |
|---|---|---|---|
| `npm test` | 38/38 ✅ | 202/202 ✅ | — |
| `npm run test:e2e` | 8/8 ✅ | 71/71 ✅ | — |
| `npm run test:cov` | 38/38 ✅ | 202/202 ✅ | ~80% statements / ~74% branches |

### Frontend (Vitest + Playwright)

```bash
cd next-frontend
docker compose exec next-frontend npm test            # unitários + integração (Vitest + MSW)
npx playwright test                                   # end-to-end (no host, com dev server em MSW_ENABLED=true)
```

Sufixos: `*.test.ts(x)` (unitário), `*.integration.test.ts(x)` (Route Handlers com MSW), `*.e2e-spec.ts` (Playwright). MSW intercepta as chamadas à API NestJS — os testes nunca batem no backend real.

## ✅ Funcionalidades implementadas

**Fase 01 — Configuração base**, **Fase 02 — Autenticação** e **Fase 03 — Upload e Processamento de Vídeos** estão concluídas (backend). Interfaces de vídeo no frontend ficam para a Fase 04+.

### Autenticação (Fase 02)

Fluxo completo de **cadastro → confirmação por e-mail → login → recuperação de senha**, com canal criado automaticamente para cada usuário (a partir do prefixo do e-mail).

Endpoints da API (`nestjs-project`):

| Método & Rota | Descrição |
|---------------|-----------|
| `POST /auth/register` | Cadastro de usuário (cria usuário + canal) |
| `GET /auth/confirm-email?token=` | Confirmação de conta via link do e-mail |
| `POST /auth/resend-confirmation` | Reenvio do e-mail de confirmação |
| `POST /auth/login` | Login (retorna access + refresh token) |
| `POST /auth/refresh` | Rotação de refresh token (com family + grace period) |
| `POST /auth/logout` | Revoga os refresh tokens da sessão |
| `POST /auth/forgot-password` | Solicita e-mail de recuperação de senha |
| `POST /auth/reset-password` | Redefine a senha via token |
| `GET /auth/me` | Dados do usuário autenticado (protegido por JWT) |

Telas e Route Handlers BFF (`next-frontend`):

- `/(auth)/signup`, `/(auth)/login`, `/(auth)/forgot-password` — formulários com React Hook Form + Zod e validação inline.
- `app/api/auth/{signup,login,logout,forgot-password}` — proxy same-origin para a API.

Segurança: senhas com **Argon2**, **JWT** com `JwtAuthGuard` global (opt-out via `@Public()`), **rotação de refresh token** com detecção de reuso, **rate limiting** (`ThrottlerGuard`) nos endpoints de auth, e sessão no navegador via **iron-session** (cookies HTTP-only).

### Upload e Processamento de Vídeos (Fase 03)

Upload direto ao object storage via **URLs presignadas** (multipart), sem que o arquivo passe pela API — a API só coordena o processo e delega os bytes ao MinIO. Após o upload, o vídeo é processado de forma assíncrona: extração de metadados e geração de thumbnail via FFmpeg, rodando em um worker separado.

**Ciclo de vida do vídeo:** `draft` → `processing` → `ready` | `failed`

Endpoints da API:

| Método & Rota | Descrição |
|---------------|-----------|
| `POST /videos` | Inicia o upload — cria o vídeo como `draft` e retorna URLs presignadas para envio direto ao storage |
| `POST /videos/{publicId}/complete` | Confirma o upload (com os ETags de cada parte), transiciona para `processing` e enfileira o job |
| `GET /videos/{publicId}` | Metadados e status do vídeo (dono vê vídeos não-`ready` e `error_code`; demais só veem vídeos `ready`) |
| `GET /videos/{publicId}/stream` | Redireciona (302) para URL presignada de reprodução inline (suporta HTTP Range / 206 Partial Content, servido nativamente pelo MinIO) |
| `GET /videos/{publicId}/download` | Redireciona (302) para URL presignada de download (`Content-Disposition: attachment`, nome derivado do título do vídeo) |

Uma rotina agendada (`VideoSweepService`) expira rascunhos (`draft`) abandonados há muito tempo e marca como `failed` vídeos presos em `processing` além do limite esperado.

#### Fluxo de upload via `curl`

```bash
TOKEN="<access_token>"   # obtido em POST /auth/login

# 1. Iniciar upload
INIT=$(curl -s -X POST http://localhost:3000/videos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Meu vídeo",
    "description": "Descrição opcional",
    "file_name": "clip.mp4",
    "file_size": 3050,
    "content_type": "video/mp4"
  }')

PUBLIC_ID=$(echo $INIT | python3 -c "import sys,json; print(json.load(sys.stdin)['public_id'])")
UPLOAD_URL=$(echo $INIT | python3 -c "import sys,json; print(json.load(sys.stdin)['upload']['urls'][0]['url'])")

# 2. Enviar o arquivo diretamente ao MinIO
#    A assinatura da URL inclui o header Host — ao testar fora da rede Docker,
#    resolva o hostname sem alterar a URL, em vez de trocar "minio" por "localhost":
PUT_RESPONSE=$(curl -s -i --resolve minio:9000:127.0.0.1 -X PUT "$UPLOAD_URL" \
  --data-binary "@clip.mp4")
ETAG=$(echo "$PUT_RESPONSE" | grep -i '^etag:' | sed 's/[^"]*"\([^"]*\)".*/\1/')

# 3. Completar o upload
curl -s -X POST "http://localhost:3000/videos/$PUBLIC_ID/complete" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"parts\":[{\"part_number\":1,\"etag\":\"$ETAG\"}]}"

# 4. Acompanhar o processamento
curl -s http://localhost:3000/videos/$PUBLIC_ID
```

> **`file_size` precisa bater exatamente com o tamanho real do arquivo enviado.** Uma divergência resulta em `VIDEO_UPLOAD_SIZE_MISMATCH` na etapa de complete, e o vídeo é automaticamente marcado como `failed` — proteção contra uploads incompletos ou adulterados.

> O `channelId` **não é informado em nenhuma etapa**: o canal do usuário autenticado é resolvido automaticamente no backend a partir do `userId` presente no JWT (relação 1:1 usuário↔canal).

#### Inspecionando a fila (BullMQ/Redis)

Para depuração local, os jobs podem ser inspecionados diretamente via `redis-cli`:

```bash
docker compose exec redis redis-cli
127.0.0.1:6379> KEYS bull:*
```

Ou visualmente, subindo o [Redis Commander](https://github.com/joeferner/redis-commander) apontado para a rede do projeto:

```bash
docker run -d --name redis-commander \
  --network nestjs-project_default \
  -p 8081:8081 \
  -e REDIS_HOSTS=local:redis:6379 \
  rediscommander/redis-commander
# UI em http://localhost:8081
```

Verificação de ponta a ponta (registro → confirmação → login → upload → processamento → thumbnail → download) executada e validada contra a infraestrutura real (Postgres, Redis, MinIO, FFmpeg), sem mocks.

## 🛠️ Estrutura do Projeto

```
green-field-ia-project/
├── docs/
│   ├── project-plan.md                  # Planejamento geral do projeto
│   ├── decisions/                       # Decisões técnicas por fase (research)
│   ├── phases/                          # Planos e implementação por fase
│   │   ├── phase-01-configuracao-base/
│   │   ├── phase-02-auth/               # Auth (backend)
│   │   ├── phase-02-auth-frontend/      # Auth (frontend)
│   │   └── phase-03-videos/             # Upload e processamento de vídeos
│   └── diagrams/
│       └── software-arch.mermaid        # Diagrama de arquitetura (C4)
├── nestjs-project/                      # Backend API (NestJS 11)
│   ├── src/
│   │   ├── auth/                        # Cadastro, login, JWT, refresh, reset de senha
│   │   ├── users/                       # Entidade e serviço de usuários
│   │   ├── channels/                    # Canal 1:1 por usuário (nickname do e-mail)
│   │   ├── videos/                      # Upload, metadados, streaming, download, sweep de rascunhos
│   │   ├── queue/                       # Configuração da fila (BullMQ) e producer
│   │   ├── storage/                     # Serviço de object storage (S3/MinIO)
│   │   ├── worker/                      # Worker de processamento de vídeo (FFmpeg)
│   │   ├── mail/                        # Envio de e-mails (templates Handlebars)
│   │   ├── common/                      # Filtros, pipes e exceptions de domínio
│   │   ├── config/                      # Configs namespaced (Joi)
│   │   └── database/                    # data-source, migrations e seeds
│   ├── test/                            # Testes e2e + fixtures
│   ├── compose.yaml                     # Docker Compose (API + PostgreSQL + Mailpit + MinIO + Redis + Worker)
│   └── Dockerfile.dev
├── next-frontend/                       # Frontend (Next.js 16, App Router)
│   ├── app/                             # Rotas, layouts, páginas e Route Handlers BFF
│   ├── components/                      # Componentes de auth, UI (shadcn) e ícones
│   ├── lib/                             # env, api (openapi-fetch), auth/session
│   ├── mocks/                           # MSW (handlers + server)
│   ├── tests/                           # E2E (Playwright)
│   ├── compose.yaml                     # Docker Compose (dev server)
│   └── Dockerfile.dev
├── CLAUDE.md                            # Instruções para IA
├── FC Tube.fig                          # Design system do projeto (Figma)
├── whiteboard.png                       # Quadro branco do projeto
└── README.md
```

## 📚 Fases do Projeto

| Fase | Descrição | Status |
|------|-----------|--------|
| **01** | Configuração Base do Projeto | ✅ Concluída |
| **02** | Cadastro, Login e Gerenciamento de Conta | ✅ Concluída |
| **03** | Upload e Processamento de Vídeos | ✅ Concluída |
| **04** | Gerenciamento de Vídeos e Canal | ⏳ Planejada |
| **05** | Página de Visualização do Vídeo | ⏳ Planejada |
| **06** | Interações Sociais (Likes, Comentários, Inscrições) | ⏳ Planejada |
| **07** | Página Inicial, Busca e Finalização | ⏳ Planejada |

Detalhes completos em `docs/project-plan.md` e, para a Fase 03 especificamente, em `docs/phases/phase-03-videos/`.

## 📖 Stack Tecnológica

| Camada | Tecnologia |
|--------|------------|
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS 4, shadcn/ui, React Hook Form + Zod, iron-session, openapi-fetch |
| Backend | NestJS 11, TypeScript, TypeORM, JWT, Argon2, Mailer (Handlebars), BullMQ |
| Banco de Dados | PostgreSQL 17 |
| Object Storage | MinIO (S3-compatível) |
| Fila | Redis 7.4 (backend do BullMQ) |
| Processamento de vídeo | FFmpeg / ffprobe |
| E-mail (dev) | Mailpit |
| Containerização | Docker, Docker Compose |
| Testes | Jest, Supertest (backend); Vitest, MSW, Playwright (frontend) |
| Qualidade | ESLint, Prettier |