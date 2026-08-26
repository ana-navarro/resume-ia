# Task: Rota Pública de Currículos-Base

**Status**: Validated - Committed
**Created**: 2026-08-26

## Description (PT)

Introduz o conceito de **currículos-base** (histórico versionado, múltiplos documentos, PT-BR e EN)
como algo **distinto** de **apresentação** (exatamente 2 documentos fixos — um PT, um EN — para a aba de
apresentação). Entrega uma rota pública `GET` que lista os metadados de todos os currículos-base
cadastrados. Isso exige revisar o comportamento já implementado em `POST /presentation`
(`tasks/upload-replace-presentation`) e adicionar um novo fluxo de upload de currículo com
versionamento (`atualizado`/`antigo`).

**Este task cresceu bastante em relação ao pedido original de 4 palavras** ("Rota Pública de
Currículos-Base"). O assistente recomendou migrar para o pipeline completo
(`/speckit-specify` → `/speckit-clarify` → `/speckit-plan` → `/speckit-tasks` → `/speckit-implement`)
dado o tamanho real do escopo (dois tipos de documento, máquina de estados de versionamento, nova
persistência em banco de dados, revisão de uma task já completada). O usuário optou explicitamente por
manter como task leve mesmo assim. Por isso, várias decisões de design abaixo são **suposições
documentadas** do assistente (marcadas como tal) — revise com atenção no `/speckit-validate`.

### Modelo de dados (conforme especificado pelo usuário no chat)

- **Apresentação** (`POST /presentation`, já implementado): passa a aceitar um parâmetro `idioma`
  (`pt` | `en`) e armazenar **no máximo 2 documentos simultâneos**, um por idioma — substituindo o
  modelo atual de único documento global. Isso é uma **correção ao comportamento de
  `tasks/upload-replace-presentation`**, não uma nova feature isolada.
- **Currículo** (`POST /curriculo`, novo): cada upload cria uma **nova entrada no histórico** (não
  substitui/apaga as anteriores). Cada currículo tem: `idioma` (`pt` | `en`), `nome_arquivo`,
  `data_upload`, `status` (`atualizado` | `antigo`), e o conteúdo/referência ao PDF. O texto extraído
  MUST ser limpo/deduplicado antes de armazenar (evitar informação repetida).
- **Versionamento** (`atualizado`/`antigo`): a requisição de upload de currículo recebe um parâmetro
  `atualizar` (boolean). **Suposição**: o invariante "só um `atualizado` por vez" é **por idioma**
  (um currículo PT atualizado + um currículo EN atualizado podem coexistir).
  - Se for o primeiro currículo daquele idioma na base, ele MUST nascer `atualizado` (independente do
    parâmetro `atualizar`).
  - Se `atualizar=true`: o novo currículo nasce `atualizado`; todos os currículos anteriores do mesmo
    idioma viram `antigo`.
  - Se `atualizar=false` (e já existe um `atualizado` para aquele idioma): o novo currículo nasce
    `antigo`, sem afetar o `atualizado` existente.
  - O currículo `atualizado` deve ter prioridade/peso maior nas buscas da IA — **fora de escopo desta
    task**: `resume-embeddings`/`resume-llm-engine` ainda são stubs e não consomem esse atributo ainda;
    fica registrado aqui como requisito para uma task futura de busca/ranqueamento.
- **Rota pública** (`GET`, novo, o pedido original): lista **todos os dados** de todos os
  currículos-base cadastrados (id, idioma, nome_arquivo, data_upload, status).

## Description (EN)

Introduces **base résumés** (versioned history, multiple documents, PT-BR and EN) as distinct from
**presentation** (exactly 2 fixed documents — one PT, one EN — for the presentation tab), and adds a
public `GET` route listing all base-résumé metadata. Requires revising the already-implemented
`POST /presentation` behavior and adding a new versioned résumé-upload flow (`atualizado`/`antigo`
status, one "current" version per language, higher AI-retrieval weight for the current one — the
weighting itself is out of scope since the AI services are still stubs).

This grew well beyond the original 4-word request; the assistant recommended the full spec pipeline
given the real scope, but the user chose to keep it as a lightweight task. Several design decisions
below are the assistant's documented assumptions — review carefully during `/speckit-validate`.

## Business Rules / Constraints

- `POST /presentation` MUST be revised to require an `idioma` (`pt`|`en`) param and enforce **no máximo
  2 documentos simultâneos** (um por idioma) — corrige o comportamento de
  `tasks/upload-replace-presentation` (que hoje trata um único documento global).
- `POST /curriculo` (novo) MUST criar uma nova entrada de histórico a cada upload — **nunca apaga**
  entradas anteriores (ao contrário do modelo de apresentação).
- Texto extraído de cada currículo MUST ser limpo/deduplicado antes de persistir (reaproveitar/estender
  a lógica de chunking já existente em `infra/adapters/chunk_text_adapter.py` como referência, mas sem
  alterar o comportamento do endpoint de apresentação).
- Persistência de metadados/histórico de currículo e apresentação MUST usar MongoDB (`pymongo`), via
  `.env` local — nunca commitado — seguindo o mesmo padrão de `tasks/mongodb-env-config`.
  **Suposição de infraestrutura**: reaproveita a mesma connection string do MongoDB Atlas já configurada
  para `resume-orchestrator` (mesmo cluster, mesmo usuário), mas com um **nome de database próprio**
  (ex.: `resume_injections`) para não misturar collections com o histórico de mensagens de chat do
  orchestrator. O `.env` de `resume-injections` é local e isolado — não há segredo compartilhado entre
  serviços em código, cada serviço tem sua própria cópia.
- Invariante de versionamento (`atualizado`/`antigo`) é **por idioma** — ver suposição na Description.
- A rota pública `GET` de listagem MUST retornar todos os campos de metadados de cada currículo-base
  (`id`, `idioma`, `nome_arquivo`, `data_upload`, `status`) — sem autenticação, por ser explicitamente
  pública.
- Comunicação entre `domain` e `infra` exclusivamente via ports (Constitution Principle II).
- Testes unitários MUST mockar o client MongoDB — sem conexão real durante os testes (Constitution
  Principle III).

## Affected Service(s)

- `/services/resume-injections` (**suposição**: mesmo serviço que já hospeda `POST /presentation`,
  parsing e chunking — mantém tudo relacionado a currículo/apresentação em um único serviço, consistente
  com Constitution Principle I). Se estiver errado, corrigir no `/speckit-validate`.

## Implementation Checklist

### Infraestrutura (MongoDB para resume-injections)

- [x] `.gitignore`: `.env` **não estava coberto** (o `.gitignore` desta task anterior só tinha
      `.venv/`, `__pycache__/`, etc.) — adicionado como primeira linha, confirmado via
      `git check-ignore -v .env` antes de criar o arquivo
- [x] `.env.example`: chave `MONGODB_URI` vazia (sem segredo real)
- [x] `.env` (local, fora do controle de versão): populado com a mesma connection string já usada em
      `resume-orchestrator`; confirmado que não aparece em `git status --porcelain`
- [x] `config/mongo_client.py`: `get_mongo_client()` + `get_database()` (seleciona o database
      `resume_injections`) — mesmo padrão de `resume-orchestrator` (`tasks/mongodb-env-config`), com
      `RuntimeError` claro se `MONGODB_URI` ausente
- [x] `requirements.txt`: adicionado `pymongo` e `python-dotenv` (este último também estava faltando)

### Domínio — Apresentação (correção ao comportamento existente)

- [x] `applications/dto/upload_presentation_dto.py`: adicionado campo `idioma` (`pt`|`en`) obrigatório,
      validado
- [x] `domain/usecases/replace_presentation_usecase.py`: revisado para operar por idioma — substitui
      apenas o slot do idioma enviado, nunca afeta o outro. **Nota**: "no máximo 2 documentos" é
      garantido estruturalmente por `StorePdfAdapter` ter exatamente um slot de arquivo por idioma
      (`current_presentation_{pt,en}.pdf`) — não há como existir um terceiro
- [x] `infra/adapters/store_pdf_adapter.py`: revisado para salvar em
      `config/assets/current_presentation_{idioma}.pdf` em vez de um único `current_resume.pdf` global.
      **Correção em cascata**: `domain/ports/replace_presentation_port.py`,
      `infra/ports/store_pdf_port.py`, `infra/ports/replace_embeddings_port.py`,
      `infra/adapters/replace_embeddings_adapter.py`, `infra/schemas/presentation_chunk_schema.py`,
      `applications/controllers/upload_presentation_controller.py` e
      `applications/routes/presentation_routes.py` (agora recebe `idioma` via `Form`) também precisaram
      de `idioma` propagado pela cadeia inteira — não estavam listados individualmente no checklist
      original, mas são consequência direta deste item

### Domínio — Currículo-base (novo, com histórico)

- [x] `domain/models/curriculo_document.py`: entidade com `idioma`, `nome_arquivo`, `raw_text`,
      `data_upload`, `status` (`atualizado`|`antigo`), `id` (opcional, populado só na leitura)
- [x] `domain/ports/upload_curriculo_port.py` (Input): interface chamada pelo controller
- [x] `domain/usecases/upload_curriculo_usecase.py`: orquestra parse → chunk → dedup (remove chunks
      repetidos preservando ordem) → conta currículos existentes do idioma → aplica a lógica de
      `atualizado`/`antigo` (primeiro do idioma nasce `atualizado`; `atualizar=true` promove o novo e
      rebaixa os demais do mesmo idioma; `atualizar=false` com `atualizado` já existente só insere como
      `antigo`) → insere no histórico (MongoDB)
- [x] **Correção ao checklist original**: em vez de um único `store_curriculo_port`/`adapter`,
      Constitution Principle II exige "um adapter para cada verbo/ação" — implementado como 3 ports/
      adapters de Output separados: `infra/ports/count_curriculos_port.py` +
      `infra/adapters/count_curriculos_adapter.py` (conta por idioma), `mark_curriculos_antigo_port.py`
      + `_adapter.py` (rebaixa os `atualizado` existentes do idioma), `insert_curriculo_port.py` +
      `_adapter.py` (grava a nova entrada) — todos via `pymongo`, collection `curriculos` no database
      `resume_injections`
- [x] `applications/dto/upload_curriculo_dto.py`: valida arquivo (tipo PDF, tamanho), `idioma`, e
      `atualizar` (boolean)
- [x] `applications/controllers/upload_curriculo_controller.py`: recebe upload e chama o usecase via
      port de Input
- [x] `applications/routes/curriculo_routes.py`: exporta `POST /curriculo` (multipart: `file`, `idioma`
      via `Form`, `atualizar` via `Form`, default `False`), incluída em `main.py`

### Domínio — Rota pública de listagem (o pedido original)

- [x] `domain/ports/list_curriculos_port.py` (Input): interface chamada pelo controller
- [x] `domain/usecases/list_curriculos_usecase.py`: busca todos os currículos-base via port de Output,
      retorna todos os campos de metadados
- [x] `infra/ports/find_curriculos_port.py` (Output): interface para leitura no MongoDB
- [x] `infra/adapters/find_curriculos_adapter.py`: implementa a leitura via `pymongo` (lista completa,
      sem filtro/paginação nesta primeira versão), mapeia `_id` do Mongo para `id: str`
- [x] `applications/controllers/list_curriculos_controller.py`: chama o usecase via port de Input,
      formata a resposta pública (id, idioma, nome_arquivo, data_upload ISO, status) — **não** inclui
      `raw_text` (conteúdo completo do currículo), só metadados, conforme suposição documentada
- [x] `applications/routes/curriculo_routes.py`: exporta `GET /curriculos` (pública, sem autenticação),
      no mesmo arquivo de rotas de currículo. **Verificado fim a fim**: `GET /curriculos` chamado via
      `TestClient` contra o cluster MongoDB Atlas real retornou `200 []` (collection vazia) — confirma
      que toda a cadeia rota→controller→usecase→adapter→MongoClient está funcionando

### Testes (Constitution Principle III)

- [x] Testes unitários para `replace_presentation_usecase.py` revisado — 5 testes (substituição por
      idioma; substituir PT não afeta EN); mais testes de DTO (idioma inválido), controller (idioma
      inválido) e do `store_pdf_adapter` (cada idioma no seu próprio arquivo)
- [x] Testes unitários para `upload_curriculo_usecase.py` — 7 testes: primeiro currículo do idioma nasce
      `atualizado` mesmo sem a flag; `atualizar=true` promove o novo e rebaixa os demais do mesmo
      idioma; `atualizar=false` com `atualizado` existente não o afeta; idiomas diferentes não se
      afetam; dedup de chunks repetidos; PDF sem texto / sem chunks levanta `InvalidCurriculoError`
      (mockando todos os ports de infraestrutura)
- [x] Testes unitários para `list_curriculos_usecase.py`: retorna o que o port de leitura retornar
      (mockado) — 1 teste
- [x] Testes unitários para os 4 adapters MongoDB novos (`count`/`mark`/`insert`/`find_curriculos`),
      mockando `config.mongo_client.get_database` — sem conexão real (Constitution Principle III)
- [x] Testes unitários para `config/mongo_client.py` (`get_mongo_client`, `get_database`) — 3 testes,
      mockando `pymongo.MongoClient`
- [x] Testes unitários para `upload_curriculo_dto.py` e para os controllers
      (`upload_curriculo_controller.py`, `list_curriculos_controller.py`)
- [x] Testes para as rotas (`fastapi.testclient.TestClient`): `POST /curriculo`, `GET /curriculos`,
      `POST /presentation` revisado (incluindo o caso de `idioma` ausente no form). **54/54 testes
      passando localmente, `ruff check .` limpo** (verificação de desenvolvimento — não é o gate formal
      de `/speckit-complete`)

## Adjustment Requests

None. Approved as-is on 2026-08-26 without the file-by-file walkthrough — the assistant explicitly
recommended reviewing this one given its size, but the user confirmed skipping it anyway when asked a
second time. Before staging, confirmed `.env` was gitignored (`git check-ignore -v .env`), absent from
`git status`, and that none of the MongoDB or ChromaDB secret values appeared anywhere in the staged
diff (`git diff --cached | grep`). Commit `b9ec82e` in `services/resume-injections`.
