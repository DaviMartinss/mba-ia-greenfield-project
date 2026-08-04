# Dando sequência ao StreamTube com IA — Fase 03: Upload e Processamento de Vídeos

## Descrição

O StreamTube é a plataforma de compartilhamento de vídeos que vem sendo construída ao longo do curso. O professor já entregou as Fases 01 (configuração base) e 02 (autenticação, usuários e canais), tanto no backend quanto no frontend, seguindo o workflow de desenvolvimento orientado por IA ensinado no curso.

O desafio agora é avançar para a próxima etapa da sequência: a Fase 03 — Upload e Processamento de Vídeos, entregue por completo. Ao contrário de uma feature simples de CRUD, essa fase é um desafio de engenharia em si mesma: exige lidar com arquivos grandes, processamento assíncrono via fila, um worker dedicado de vídeo, streaming e infraestrutura nova rodando em Docker. Todo esse trabalho deve ser conduzido com IA como ferramenta central, seguindo o workflow do projeto do início ao fim.

Trata-se de um desafio voltado a backend: o que será entregue é a API, o worker, a infraestrutura e os artefatos gerados ao longo do processo. (Existe um frontend no repositório, mas a interface de vídeo está fora do escopo desta fase.)

## Ferramenta de IA

O projeto base foi montado pensando no Claude Code, que é a ferramenta recomendada: toda a base de IA (skills, sub-agents, CLAUDE.md, rules e .mcp.json) já está pronta para funcionar com ele.

É possível usar outra ferramenta agêntica, como Gemini CLI, OpenAI Codex ou similares — mas atenção: essa base de IA do repositório foi construída especificamente para o Claude Code. Ao optar por outra ferramenta, cabe a você portar essa base para a convenção correspondente antes de começar:

- O CLAUDE.md deve virar o arquivo de instruções equivalente da outra ferramenta (por exemplo, GEMINI.md no Gemini CLI, AGENTS.md no Codex).
- As skills e sub-agents do workflow precisam ser recriados no mecanismo equivalente da ferramenta escolhida (comandos, extensões, custom modes) — ou, na ausência de um equivalente direto, o mesmo workflow deve ser conduzido manualmente.
- O .mcp.json e os servidores MCP (Postgres, context7) devem virar a configuração de MCP própria da ferramenta escolhida.

Consulte sempre a documentação oficial da ferramenta escolhida para confirmar nomes corretos de arquivos, pastas e comandos. Independentemente de qual ferramenta for usada, o workflow permanece o mesmo e os artefatos entregues também (detalhados adiante) — só a "máquina" por trás muda. Trocar de ferramenta não altera os Critérios de Aceite.

## Sobre o uso de IA

A IA é a ferramenta principal de produção, e seu uso é obrigatório. Seu papel aqui é o de maestro do processo: conduzir o workflow na sequência correta, revisar criticamente cada saída gerada, refinar os prompts sempre que o resultado ficar raso, consultar a documentação das bibliotecas antes de implementar qualquer coisa e manter todos os artefatos coerentes entre si.

O uso de IA precisa ficar visível no repositório: decisões técnicas, artefatos de planejamento, o plano da fase e o registro de progresso — tudo isso deve ser fruto do fluxo conduzido com IA.

## Objetivo

Entregar, em um fork público do repositório base, dando continuidade ao projeto mba-ia-greenfield-project:

- As decisões técnicas da fase (fila, estratégia de upload, streaming, processamento etc.), registradas em `docs/decisions/`
- Os artefatos de planejamento da fase, dentro de `docs/phases/phase-03-videos/` (`context.md`, `validation.md`, o plano `phase-03-videos.md`, o `progress.md` e o `library-refs.md` quando houver bibliotecas novas a fixar)
- O módulo de vídeos implementado no backend, com a infraestrutura nova (storage, fila e worker) rodando via Docker
- A Fase 03 funcionando de ponta a ponta: upload de até 10GB, processamento automático, geração de thumbnail, URL única, streaming e download
- O CLAUDE.md (ou o arquivo equivalente da ferramenta escolhida) atualizado com a seção referente a vídeos

Toda informação registrada nos artefatos precisa ser rastreável até o plano ou o código. Não é permitido inventar requisitos, decisões ou comportamentos sem origem identificável.

## Contexto

### O que já existe no projeto

- Backend em NestJS 11 + TypeORM + PostgreSQL 17, dentro de `nestjs-project/`, com as Fases 01 e 02 já concluídas: módulos `auth/`, `users/`, `channels/`, `mail/`, `common/`, `config/`, `database/` e `swagger/` (OpenAPI).
- Cada usuário possui um canal (relação 1:1), criado automaticamente no cadastro. Os vídeos da Fase 03 pertencem a um canal.
- Guard JWT global, filtro de exceções de domínio, ValidationPipe global, rate limiting, migrations versionadas e seeds já implementados.
- A infraestrutura atual em `nestjs-project/compose.yaml` conta apenas com API, PostgreSQL e Mailpit.
- Um frontend em Next.js dentro de `next-frontend/` (cobrindo as Fases 01–02), que fica fora do escopo desta fase.

O que ainda não existe, e que você vai construir: o módulo de vídeo, a tabela de vídeos, o serviço de object storage, a fila de processamento e o worker de vídeo (com FFmpeg). A arquitetura-alvo (documentada em `docs/diagrams/software-arch.mermaid` e no CLAUDE.md) já prevê esses três componentes como parte da Fase 03.

### O workflow do projeto

O projeto segue um workflow de planejamento em pipeline definido pelo Luiz, e essa sequência deve ser respeitada. Cada estágio corresponde a uma skill do projeto e produz um artefato:

1. Pesquisar as opções e decidir o caminho técnico — skill `research` — artefato `docs/decisions/technical-decisions-phase-03-videos.md`
2. Consolidar o contexto da fase — skill `plan-context` — artefato `docs/phases/phase-03-videos/context.md`
3. Validar o contexto (checando inconsistências, decisões faltando, gaps) — skill `plan-validate` — artefato `validation.md` (com veredito clean/dirty)
4. Resolver pendências e fixar bibliotecas — skill `plan-resolve` — atualiza decisões e contexto, além de gerar `library-refs.md`
5. Gerar o plano executável — skill `plan-build` — artefato `phase-03-videos.md` (Step Implementations, Technical Specs, Dependency Map e Deliverables)
6. Gerar specs de teste (etapa opcional) — skill `plan-test-specs` — artefato com as specs de teste
7. Implementar passo a passo — skill `implement` — artefato: código + `progress.md`

Pontos de formato que precisam ser respeitados:

- A fase corresponde a uma pasta (`docs/phases/phase-03-videos/`) contendo `context.md`, `validation.md`, o plano `phase-03-videos.md` e o `progress.md` — além do `library-refs.md` quando a fase fixa novas bibliotecas (que é justamente o caso aqui, por conta de storage/fila/FFmpeg). Use a pasta `docs/phases/phase-02-auth/` como referência de formato.
- O plano é organizado em Step Implementations (`SI-03.1`, `SI-03.2`, ...), acompanhadas das Technical Specifications (Data Model, API Contracts, Authorization Matrix, Error Catalog e, por causa da fila, também Events/Messages), do Dependency Map e dos Deliverables.
- O `validation.md` precisa fechar com status clean antes de partir para a implementação.
- Os sub-agents de leitura (`.claude/agents/`) são acionados pelas skills internamente — você não os invoca diretamente.

Para quem optar por outra ferramenta: produza os mesmos artefatos (decisões, contexto, validação, plano com SIs e Technical Specs, progresso), seguindo o mesmo encadeamento. O formato da pasta da fase é o contrato a ser cumprido; a skill que o gera é apenas um detalhe de implementação da ferramenta.

## Regras e Definition of Done

O CLAUDE.md define as regras do projeto, que se aplicam também a esta fase:

- **Definition of Done:** a fase só está concluída quando a suíte de testes relevante passa (unit + integração + e2e), a suíte completa passa, `npx tsc --noEmit` retorna código 0 e `npm run lint` passa sem erros.
- **Docker:** tudo deve rodar em containers; use sempre o nome do serviço definido no Compose como host (por exemplo, `db`) — nunca `localhost`.
- **Documentação de bibliotecas:** antes de implementar com qualquer biblioteca, consulte a documentação oficial via context7 (MCP) e siga a versão instalada no projeto.
- **Git Flow:** branches `feature/*` partem da `dev` e retornam para ela; nunca commite diretamente na `main`. Commits devem ser curtos e descritivos.
- **Testes:** use os sufixos `*.spec.ts` (unit), `*.integration-spec.ts` (integração com banco/serviços reais) e `*.e2e-spec.ts` (e2e via supertest).

## Escopo da Fase 03 — Upload e Processamento de Vídeos

As capacidades a serem entregues (com definição completa em `docs/project-plan.md`, Fase 03) são:

- Object storage para armazenar os arquivos de vídeo e as thumbnails.
- Fila de processamento em segundo plano, com um worker que a consome.
- Upload de vídeos de até 10GB sem travar o sistema (isto é, sem segurar a API durante todo o envio).
- Pré-cadastro automático do vídeo como rascunho assim que o upload é iniciado.
- Processamento automático logo após o upload: extração de duração e metadados.
- Geração automática de thumbnail a partir de um frame do vídeo.
- URL única por vídeo, sem risco de conflito com outros.
- Reprodução via streaming, sem exigir o download completo do arquivo.
- Possibilidade de download do vídeo pelo usuário.

Entregáveis previstos no plano original: upload de até 10GB funcionando, processamento automático do vídeo, streaming operante e URLs únicas sendo geradas.

**Persistência:** é necessária uma entidade/tabela de vídeos vinculada ao canal, contendo no mínimo: identificação, dono (canal), título, status (por exemplo: rascunho → processando → pronto/erro), chaves de storage do arquivo original e da thumbnail, duração, metadados e o identificador da URL única. O modelo exato de dados fica a cargo do plano (seção Data Model).

**Decisões a tomar e justificar durante a etapa de research:**

- A tecnologia de fila — o plano do projeto deixa esse ponto explicitamente em aberto ("TBD"). É a principal decisão de stack desta fase.
- A estratégia de upload de arquivos de até 10GB sem travar o sistema (por exemplo, upload direto ao storage via URL pré-assinada / multipart, evitando que o arquivo passe pela API).
- Como o worker deve rodar (processo ou container separado) e de que forma ele extrai os metadados e gera o thumbnail (via FFmpeg/ffprobe).
- A estratégia de geração de URL única e a de streaming (por exemplo, requisições com range / resposta 206 Partial Content).
- O ciclo de status do vídeo e o que acontece quando o processamento falha.

Sobre o object storage: ele não é uma decisão em aberto. O projeto já direciona para S3 (ou compatível) — na prática, você roda o MinIO localmente em Docker (que replica a API do S3) e o trocaria por S3 real em produção. O que fica sob sua decisão é como usar esse storage (organização de buckets e chaves, estratégia de upload pré-assinado) — não qual storage escolher. A única decisão de stack genuinamente aberta nesta fase é a da fila.

Essas decisões formam o núcleo da etapa de research. Pesquise as alternativas, registre os trade-offs e a escolha final no documento de decisões, e só depois avance para o planejamento.

## Requisitos

### 1. Decisões técnicas (research)

Conduza a etapa de research sobre as decisões em aberto listadas acima. Registre o resultado em `docs/decisions/technical-decisions-phase-03-videos.md`, seguindo o formato já usado nos documentos de decisão existentes (opções avaliadas, trade-offs e recomendação para cada decisão). É esse documento que alimenta todo o planejamento seguinte.

### 2. Planejamento (pipeline)

Conduza a pipeline de planejamento até chegar ao plano final, gerando os artefatos dentro da pasta `docs/phases/phase-03-videos/`:

- `plan-context` → `context.md`
- `plan-validate` → `validation.md` (precisa fechar em clean)
- `plan-resolve` → resolve as pendências apontadas e gera o `library-refs.md` (com bibliotecas confirmadas via context7)
- `plan-build` → `phase-03-videos.md`, contendo Step Implementations (`SI-03.x`), Technical Specifications (incluindo Events/Messages para a fila), Dependency Map e Deliverables

Revise criticamente cada saída gerada. O `validation.md` aponta decisões pendentes e gaps de dependência: alterne entre validate e resolve até o resultado ficar clean. Um plano mal amarrado leva a uma implementação igualmente mal amarrada.

### 3. Implementação (implement)

Implemente a fase conduzido pela skill `implement`, avançando SI por SI, rodando os testes a cada passo e só seguindo adiante quando a suíte daquele SI estiver verde. Isso inclui:

- O módulo de vídeos no backend, respeitando as convenções e regras do projeto (separação de camadas, repository pattern, uso de fila/eventos, transações). Use o módulo `auth/` como referência de forma — a estrutura concreta de arquivos é decisão do seu plano, não algo definido pelo enunciado.
- A infraestrutura nova no `compose.yaml`: os serviços de object storage, fila e worker, subindo junto com o restante da stack do backend.
- A migration que cria a tabela de vídeos.
- Os testes nos níveis apropriados (unit, integração com banco/serviços reais e e2e), seguindo as skills de teste do projeto. Evite mockar o que pode ser testado de verdade usando a infraestrutura do Compose.
- O `progress.md` da fase atualizado (status e testes por SI), no mesmo formato usado na Fase 02.

Ao final, a Definition of Done definida no CLAUDE.md precisa passar por completo.

### 4. Atualização da documentação de IA

Atualize o CLAUDE.md (ou o arquivo equivalente da sua ferramenta) para refletir o estado real do código após a conclusão da fase: o módulo de vídeos, os endpoints, a fila/worker e o storage. Documentação que faça referência a arquivos ou comportamentos que não existem de fato é motivo de reprovação.

## Critérios de Aceite

Todos os itens abaixo são obrigatórios. Esta é a lista única usada na avaliação.

**Decisões e planejamento**
- `technical-decisions-phase-03-videos.md` com todas as decisões em aberto resolvidas e devidamente justificadas (fila, estratégia de upload, streaming, processamento/thumbnail, ciclo de status)
- Pasta `docs/phases/phase-03-videos/` contendo `context.md`, `validation.md` (com status clean), o plano `phase-03-videos.md`, o `progress.md` e o `library-refs.md` (quando houver bibliotecas novas a fixar — o que é esperado nesta fase)
- O plano segue o formato do projeto: SIs no padrão `SI-03.x`, Technical Specifications (Data Model, API Contracts, Authorization Matrix, Error Catalog, Events/Messages), Dependency Map e Deliverables

**Implementação — feature**
- Upload de vídeo de até 10GB sem travar a API, com pré-cadastro automático do vídeo como rascunho ao iniciar o processo
- Processamento automático após o upload: extração de duração/metadados e geração de thumbnail
- URL única por vídeo, sem risco de conflito
- Streaming funcionando (sem exigir download completo) e download do vídeo disponível
- Ciclo de status do vídeo (rascunho → processando → pronto/erro) refletido corretamente no banco

**Implementação — infraestrutura e qualidade**
- Object storage, fila e worker subindo via `docker compose` junto com o backend
- Migration cria a tabela de vídeos, com a entidade devidamente ligada ao canal
- Testes nos níveis adequados, todos passando (`npm test` e `npm run test:e2e`)
- Definition of Done completa: suíte verde + `npx tsc --noEmit` (código 0) + `npm run lint` sem erros
- Git Flow respeitado (trabalho realizado em `feature/*` a partir de `dev`, sem commits diretos na `main`)

**Documentação e ferramenta**
- CLAUDE.md (ou equivalente) atualizado com a seção de vídeos, coerente com o código entregue
- Se outra ferramenta que não o Claude Code foi utilizada: a base de IA foi devidamente portada para a convenção dela e os artefatos da pasta da fase foram entregues no mesmo formato exigido

**Reprova automática**
- Pular etapas do workflow: implementar sem passar pelas etapas de research, planejamento e implementação (e sem seus respectivos artefatos)
- Plano sem SIs ou sem as Technical Specifications, ou `validation.md` que não fecha em clean
- Passar o arquivo de 10GB pela API de um jeito que trave o sistema (ou seja, sem uma estratégia de upload assíncrono/direto)
- Não ter fila, worker e storage reais subindo no Compose
- `tsc` retornando erro, lint quebrado ou suíte de testes vermelha
- Commit feito diretamente na `main`
- CLAUDE.md/equivalente com informações inconsistentes em relação ao código
- Usar outra ferramenta sem portar a base de IA para a convenção dela

## Estrutura do entregável

Tudo deve estar dentro do fork do `mba-ia-greenfield-project`. Abaixo está representado apenas o que é novo ou foi alterado (os nomes de arquivo do módulo são ilustrativos — a estrutura final é decisão do seu plano):

```
mba-ia-greenfield-project/
├── docs/
│   ├── decisions/
│   │   └── technical-decisions-phase-03-videos.md     ← research
│   └── phases/
│       └── phase-03-videos/                           ← pasta da fase
│           ├── context.md                             ← plan-context
│           ├── validation.md                          ← plan-validate (clean)
│           ├── library-refs.md                        ← plan-resolve (se houver libs novas)
│           ├── phase-03-videos.md                     ← plan-build (o plano)
│           └── progress.md                            ← implement
├── nestjs-project/
│   ├── CLAUDE.md (ou equivalente)                     ← atualizado
│   ├── compose.yaml                                   ← + storage, fila, worker
│   ├── src/
│   │   ├── videos/                                    ← novo módulo (forma de referência: auth/)
│   │   │   └── ...
│   │   └── database/migrations/
│   │       └── <timestamp>-CreateVideos.ts
│   └── (worker de vídeo — local conforme o seu plano)
└── CLAUDE.md (ou equivalente)                         ← atualizado
```

## Repositório base

https://github.com/devfullcycle/mba-ia-greenfield-project

O fork funciona como sua estrutura de trabalho: não é necessário criar um repositório novo, apenas adicionar e editar arquivos dentro dele. O repositório já traz as Fases 01 e 02 (backend e frontend), todo o workflow em `.claude/` (skills, sub-agents e rules), o CLAUDE.md, o `docs/project-plan.md` e o `compose.yaml` do backend já configurado com Postgres e Mailpit.

## Ordem de execução sugerida

1. **Setup.** Faça o fork, suba o backend (`cd nestjs-project && docker compose up -d`), instale as dependências, rode as migrations e confirme que a suíte atual está verde. Caso vá usar outra ferramenta, porte a base de IA antes de começar.
2. **Research.** Pesquise e feche as decisões em aberto (fila, estratégia de upload, streaming, processamento). O object storage já está definido (S3/MinIO).
3. **Planejamento.** Rode a pipeline completa (context → validate → resolve → build) até o `validation.md` fechar em clean e o plano estar completo. Revise tudo criticamente.
4. **Implementação.** Conduza a implementação SI a SI: módulo, infraestrutura no Compose, migration, testes e `progress.md`.
5. **Fechamento.** Garanta que a Definition of Done esteja cumprida (testes + tsc + lint), atualize o CLAUDE.md e revise os Critérios de Aceite item a item antes de dar push.

## Dicas finais

- A Fase 03 é grande; é o plano que sustenta tudo. Quanto melhores forem as decisões e o plano (SIs bem fatiados, contratos e eventos bem definidos), mais limpa sai a implementação. Vale a pena investir tempo no planejamento.
- O upload de 10GB é uma decisão de arquitetura, não de força bruta. Pesquise a estratégia correta antes de codar — deixar o arquivo inteiro passar pela API é o caminho errado.
- Infraestrutura de verdade, testada de verdade. Fila, worker e storage precisam subir no Compose e ser exercitados pelos testes: evite simular o que pode ser rodado de fato.
- Continuidade, não retrabalho. Reaproveite os padrões já estabelecidos no projeto (guard, filtro de exceções, repository, migrations, rules). Você está somando uma nova fase, não reescrevendo o que já existe.
- A ferramenta é escolha sua, o workflow não é. Use o Claude Code ou porte a base para a sua ferramenta de preferência — mas o encadeamento research → planejamento → implementação, assim como os artefatos da fase, permanece o mesmo.
