# Task: READMEs dos Microsserviços, Server, App e do Repositório

**Status**: Completed
**Created**: 2026-08-26

## Description (PT)

Criar um `README.md` para cada microsserviço em `/services`, para `/resume-server` e para
`/apps/resume-app`, além de um `README.md` na raiz do repositório `resume-ia` explicando o ecossistema
como um todo (arquitetura de monorepo de micro-serviços, os princípios da Constituição, e como usar o
projeto — subir o ambiente, rodar cada serviço, e seguir o workflow Speckit). Tarefa puramente de
documentação: nenhum código de aplicação é alterado.

## Description (EN)

Add a `README.md` to each microservice under `/services`, to `/resume-server`, and to
`/apps/resume-app`, plus a root-level `README.md` for the `resume-ia` repository explaining the overall
ecosystem (micro-services monorepo architecture, the Constitution's principles, and how to use the
project — bringing up the environment, running each service, and following the Speckit workflow).
Documentation-only task: no application code changes.

## Business Rules / Constraints

- Cada README MUST refletir o estado real do serviço (Constitution Principle I: fronteiras do
  monorepo) — não inventar funcionalidades inexistentes.
- `resume-bff`, `resume-orchestrator`, `resume-embeddings`, `resume-llm-engine` e `resume-guard-rails`
  hoje contêm apenas o stub FastAPI "Hello World" (`main.py`) — os READMEs desses serviços MUST deixar
  isso explícito (papel pretendido conforme a Constituição + status "não implementado ainda"), em vez de
  descrever endpoints/comportamento que não existem.
- O README de `resume-injections` MUST refletir a funcionalidade já implementada (upload/substituição
  atômica de apresentação, arquitetura hexagonal Ports & Adapters) e como rodar a pipeline local
  (`make validate-pipeline` / `.venv` + `pytest` + `ruff`), incluindo a dependência pendente de
  `POST /embeddings/replace` em `resume-embeddings`.
- O README de `apps/resume-app` MUST substituir o conteúdo padrão gerado pelo template Vite (atual:
  "React + TypeScript + Vite") por uma descrição real do projeto (Currículo Interativo).
- O README de `resume-server` MUST documentar o papel do NGINX (`nginx/nginx.conf`) e do
  `docker-compose.yml` (hoje vazio) na orquestração dos containers, incluindo a exigência de
  hot-reloading via volumes Docker (Constitution Principle IV).
- O README raiz MUST descrever a arquitetura de monorepo (Principle I), o fluxo estrito de chamadas
  Frontend → bff → orchestrator → (guard-rails, embeddings, llm-engine) (Principle II), e o ciclo de 4
  etapas do Speckit (`/speckit-task` → `/speckit-implement` → `/speckit-validate` → `/speckit-complete`,
  Principle V), com suporte bilíngue PT/EN (Principle VI).
- Esta etapa é apenas de planejamento/documentação: sem commit, push ou execução de pipeline aqui.

## Affected Service(s)

- `/services/resume-bff`
- `/services/resume-orchestrator`
- `/services/resume-injections`
- `/services/resume-embeddings`
- `/services/resume-llm-engine`
- `/services/resume-guard-rails`
- `/resume-server`
- `/apps/resume-app`
- Raiz do repositório `resume-ia` (`/README.md`)

## Implementation Checklist

- [x] `/services/resume-bff/README.md`: papel de BFF (proxy/formatador para o frontend, sem regras de
      orquestração de domínio — Constitution Principle I), stack (Python/FastAPI), como rodar
      localmente, status atual (stub)
- [x] `/services/resume-orchestrator/README.md`: papel de orquestrador/Maestro (coordena chamadas entre
      guard-rails, embeddings e llm-engine via Saga pattern), stack, como rodar, status atual (stub)
- [x] `/services/resume-injections/README.md`: papel de ingestão de dados (parsing/chunking/embeddings),
      estrutura hexagonal já implementada (`applications/`, `domain/`, `infra/`), endpoint
      `POST /presentation`, como rodar (`make validate-pipeline`, `.venv`), dependência pendente de
      `POST /embeddings/replace` em `resume-embeddings`
- [x] `/services/resume-embeddings/README.md`: papel de serviço vetorial (embeddings locais + banco
      vetorial para busca semântica), stack, como rodar, status atual (stub)
- [x] `/services/resume-llm-engine/README.md`: papel de motor de inferência (configura o LLM, aplica a
      persona, chama a API de IA), stack, como rodar, status atual (stub)
- [x] `/services/resume-guard-rails/README.md`: papel de segurança (valida inputs/outputs, evita prompt
      injection e alucinações), stack, como rodar, status atual (stub)
- [x] `/resume-server/README.md`: papel do NGINX (`nginx/nginx.conf`) e do `docker-compose.yml` na
      orquestração dos containers, hot-reloading via volumes Docker, como subir o ambiente
- [x] `/apps/resume-app/README.md`: substitui o README padrão do template Vite pela descrição real do
      frontend do Currículo Interativo, stack (React + TypeScript), como rodar em dev
- [x] `/README.md` (raiz do repo): visão geral do ecossistema, arquitetura de monorepo, fluxo estrito de
      chamadas entre serviços, ciclo de 4 etapas do Speckit, como subir o ambiente localmente, links para
      os READMEs de cada serviço/app — em PT e EN (Constitution Principle VI)

## Adjustment Requests

None. Approved as-is on 2026-08-26 without the file-by-file walkthrough ("no review needed"). Commits:
`eae5a2f` (resume-bff), `826f07b` (resume-orchestrator), `d0f550b` (resume-injections), `b7851db`
(resume-embeddings), `4cff611` (resume-llm-engine), `83d49e7` (resume-guard-rails), `7c75b51`
(resume-server), `4124f4d` (resume-app), plus this task's own commit in `resume-ia` for the root
`README.md` and task status.
