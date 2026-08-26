# resume-ia

## Visão geral (PT)

Currículo Interativo focado em processos seletivos para posições de Engenharia de Software, que expõe as
competências técnicas de Ana Elisa através de um assistente de IA capaz de responder perguntas sobre seu
currículo. Ver `.specify/memory/constitution.md` para a Constituição completa do projeto — todo código
gerado, revisado ou validado neste repositório MUST segui-la estritamente.

## Overview (EN)

An Interactive Resume built for Software Engineering hiring processes, showcasing Ana Elisa's technical
skills through an AI assistant that can answer questions about her résumé. See
`.specify/memory/constitution.md` for the full project Constitution — all code generated, reviewed, or
validated in this repository MUST strictly follow it.

## Arquitetura: monorepo de micro-serviços

O ecossistema adota uma arquitetura de micro-serviços em um monorepo (Constitution Principle I), com
fronteiras de diretório estritas:

| Diretório | Papel | Status |
|---|---|---|
| [`apps/resume-app`](apps/resume-app/README.md) | Frontend React (TypeScript) — "frontend anêmico": UI, diagramação, chat | Scaffold Vite inicial |
| [`services/resume-bff`](services/resume-bff/README.md) | Backend for Frontend — proxy/formatador para o frontend | Stub |
| [`services/resume-orchestrator`](services/resume-orchestrator/README.md) | Maestro — coordena os serviços de IA/validação (Saga) | Stub |
| [`services/resume-injections`](services/resume-injections/README.md) | Ingestão de dados — parsing/chunking do currículo, envio ao banco vetorial | **Implementado** (upload/substituição de currículo) |
| [`services/resume-embeddings`](services/resume-embeddings/README.md) | Serviço vetorial — embeddings locais + busca semântica | Stub |
| [`services/resume-llm-engine`](services/resume-llm-engine/README.md) | Motor de inferência — configura o LLM e a persona | Stub |
| [`services/resume-guard-rails`](services/resume-guard-rails/README.md) | Segurança — valida inputs/outputs, evita prompt injection | Stub |
| [`resume-server`](resume-server/README.md) | Infraestrutura — NGINX + `docker-compose.yml` | Configuração vazia (a implementar) |

**Fluxo de chamadas estrito** (Constitution Principle II — saltos entre camadas não adjacentes são
proibidos):

```
Frontend → resume-bff → resume-orchestrator → (resume-guard-rails, resume-embeddings, resume-llm-engine)
```

Cada serviço Python segue Arquitetura Hexagonal (Ports & Adapters):
`applications/` (controllers, routes, dto) → `domain/` (usecases, models, ports de Input) → `infra/`
(adapters, ports de Output, schemas), com `config/assets/` para armazenamento de arquivos como PDFs de
currículo. Comunicação entre `domain` e `infra` acontece exclusivamente via ports.

## Como usar este repositório

1. Leia `.specify/memory/constitution.md` — ele rege toda a forma de desenvolver aqui (arquitetura,
   testes, workflow, bilinguismo).
2. Cada serviço/app tem seu próprio README com instruções específicas de setup (tabela acima). Muitos
   ainda são apenas stubs FastAPI — comece por `services/resume-injections` para ver um exemplo
   completo de serviço implementado (hexagonal + testes + pipeline local).
3. `resume-server/` concentra a subida do ambiente completo via Docker Compose — hoje ainda vazio,
   pendente de implementação (ver seu README).

### Workflow de implementação (Speckit)

Novas funcionalidades seguem um ciclo de 4 etapas (Constitution Principle V):

```
/speckit-task → /speckit-implement → /speckit-validate → /speckit-complete
```

1. **`/speckit-task`**: planeja uma tarefa pequena e autocontida em `tasks/<slug>/task.md`, com um
   checklist de implementação por camada hexagonal.
2. **`/speckit-implement`**: implementa o checklist (sem commit, push ou pipeline).
3. **`/speckit-validate`**: revisão interativa arquivo por arquivo com o mantenedor; aprova e comita.
4. **`/speckit-complete`**: roda a pipeline local (lint + testes + cobertura ≥ 80%) e faz o push.

Para features grandes e multi-story, use o pipeline completo (`/speckit-specify` →
`/speckit-clarify` → `/speckit-plan` → `/speckit-tasks` → `/speckit-implement`) em vez do fluxo
leve acima.

## Repository usage (EN)

1. Read `.specify/memory/constitution.md` first — it governs architecture, testing, workflow, and
   bilingual requirements for everything in this repo.
2. Each service/app has its own README with setup instructions (table above). Most backend services are
   still FastAPI stubs — start with `services/resume-injections` to see a fully implemented example
   (hexagonal architecture + tests + local pipeline).
3. `resume-server/` is meant to bring up the full environment via Docker Compose — currently empty,
   pending implementation.
4. New features follow the 4-step Speckit lifecycle: `/speckit-task` → `/speckit-implement` →
   `/speckit-validate` → `/speckit-complete` (or the full `specs/` pipeline for large, multi-story
   features).
