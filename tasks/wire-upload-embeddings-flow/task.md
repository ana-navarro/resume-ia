# Task: Fluxo Orchestrator → Injections → Embeddings

**Status**: Completed
**Created**: 2026-08-26

## Description (PT)

Conecta os três serviços de backend que hoje existem isolados: `resume-orchestrator` ganha seus próprios
endpoints públicos de upload (`POST /presentation`, `POST /curriculo`), que repassam o arquivo recebido
para os endpoints já implementados em `resume-injections` — substituindo o fluxo atual em que esses
endpoints eram chamados diretamente (decisão de escopo já registrada como pendência desde
`tasks/upload-replace-presentation`). `resume-injections` continua fazendo parsing/chunking como já
implementado, e passa a enviar os chunks para o endpoint `POST /embeddings/replace` em
`resume-embeddings` — endpoint que ainda não existe e que esta task implementa, usando o
`config/chroma_client.py` já configurado (`tasks/chromadb-env-config`) para vetorizar (embedding padrão
do ChromaDB, sem provedor externo) e gravar no ChromaDB Cloud.

**Este task também cresceu além do usual** (3 serviços tocados). O assistente não perguntou de novo se
o usuário queria trocar para o pipeline completo, já que nas duas tasks anteriores dessa escala o usuário
optou por manter leve mesmo assim — presumindo a mesma preferência aqui. Revise com atenção no
`/speckit-validate`.

### Decisões tomadas (perguntadas ao usuário)

- O Orchestrator ganha seus **próprios** endpoints de upload; o fluxo atual de upload direto para
  `resume-injections` é substituído por esse novo fluxo (Orchestrator relaying o arquivo recebido para
  `resume-injections`).
- Esta task **inclui** a implementação real de `POST /embeddings/replace` em `resume-embeddings`.

### Suposições do assistente (não perguntadas, documentadas para revisão)

- **Vetorização sem provedor externo**: como nenhuma API key de embeddings foi fornecida, uso a
  embedding function padrão do próprio `chromadb` (`collection.add(documents=..., ...)` sem passar
  `embeddings=` explicitamente — o client já roda um modelo local ONNX incluso no pacote `chromadb`).
- **Gap na task anterior**: `upload_curriculo_usecase.py` (de `tasks/public-curriculo-listing`) nunca
  chamava o adapter de embeddings — só gravava no MongoDB. Esta task fecha esse gap, reaproveitando o
  `ReplaceEmbeddingsPort`/`ReplaceEmbeddingsAdapter` já existente (usado hoje só por
  `replace_presentation_usecase.py`), estendendo o payload com um campo `tipo` (`presentation`|
  `curriculo`) para diferenciar as duas origens no ChromaDB.
- **Currículo: só o `atualizado` é vetorizado**. Currículos marcados `antigo` ficam só no histórico do
  MongoDB (Constitution não define isso explicitamente; é a leitura mais consistente com "o currículo
  atualizado terá prioridade/peso maior para a IA" documentado em `tasks/public-curriculo-listing` — se
  o antigo também fosse buscável, teríamos conteúdo duplicado/conflitante no vetor).
- **Semântica de "replace" no ChromaDB**: cada chamada a `POST /embeddings/replace` MUST apagar
  (`collection.delete(where=...)`) os chunks existentes com o mesmo `idioma`+`tipo` antes de inserir os
  novos — mesmo princípio de substituição atômica já usado no restante do sistema. Uma única collection
  ChromaDB (`resume_content`) com `idioma` e `tipo` como metadata, em vez de collections separadas por
  idioma/tipo (mais simples de administrar).
- **Orchestrator relaying o arquivo bruto**: o novo `POST /presentation`/`POST /curriculo` do
  Orchestrator recebe o multipart/form-data e repassa via `httpx` para o endpoint correspondente em
  `resume-injections`, devolvendo a resposta (ou erro) tal como veio. Validação de conteúdo (tipo PDF,
  tamanho) continua sendo responsabilidade de `resume-injections` (fonte da verdade já implementada);
  Orchestrator só valida `idioma` (e `atualizar`, no caso de currículo) antes de repassar.

## Description (EN)

Wires the three backend services that currently exist in isolation: `resume-orchestrator` gets its own
public upload endpoints (`POST /presentation`, `POST /curriculo`) that relay the received file to the
already-implemented endpoints on `resume-injections` — replacing the current direct-upload flow (a scope
decision deferred since `tasks/upload-replace-presentation`). `resume-injections` keeps parsing/chunking
as already implemented, and now sends chunks to a new `POST /embeddings/replace` endpoint on
`resume-embeddings`, implemented here using the already-configured `config/chroma_client.py`
(`tasks/chromadb-env-config`) to vectorize (ChromaDB's default local embedding, no external provider) and
store in ChromaDB Cloud.

Also closes a gap from `tasks/public-curriculo-listing`: `upload_curriculo_usecase.py` never actually
called the embeddings adapter. This task wires it in — only the `atualizado` (current) résumé per
language gets vectorized/searchable; `antigo` (old) ones stay MongoDB-only history.

## Business Rules / Constraints

- Comunicação entre `domain` e `infra` exclusivamente via ports (Constitution Principle II).
- Um adapter por verbo/ação (Constitution Principle II) — ver ports/adapters granulares no checklist.
- Testes unitários MUST mockar `httpx` (chamadas entre serviços) e o client ChromaDB — sem chamadas de
  rede reais durante os testes (Constitution Principle III).
- `POST /embeddings/replace` MUST ser idempotente por `idioma`+`tipo`: chamadas repetidas com o mesmo
  par nunca acumulam chunks duplicados (delete-then-insert).
- Se `resume-injections` retornar erro ao repassar o upload, o Orchestrator MUST propagar o mesmo status
  code/mensagem, não mascarar como sucesso.
- Se o ChromaDB rejeitar a inserção (ex.: indisponibilidade), `resume-embeddings` MUST retornar erro
  claro; `resume-injections` já trata esse caso (falha na chamada de embeddings não persiste o novo
  PDF/MongoDB — substituição atômica já implementada).

## Affected Service(s)

- `/services/resume-orchestrator` (novos endpoints de upload, primeira feature real do serviço)
- `/services/resume-injections` (usecase de currículo passa a chamar embeddings; schema/port/adapter de
  embeddings ganha campo `tipo`)
- `/services/resume-embeddings` (implementação real de `POST /embeddings/replace`, primeira feature real
  do serviço)

## Implementation Checklist

### resume-injections — estender o adapter de embeddings com `tipo`

- [x] `infra/schemas/presentation_chunk_schema.py`: campo `tipo: str` adicionado
- [x] `infra/ports/replace_embeddings_port.py`: parâmetro `tipo` adicionado a `replace()`
- [x] `infra/adapters/replace_embeddings_adapter.py`: `tipo` propagado no payload JSON
- [x] `domain/usecases/replace_presentation_usecase.py`: chama `replace_embeddings.replace(chunks,
      idioma, tipo="presentation")`
- [x] `domain/usecases/upload_curriculo_usecase.py`: `ReplaceEmbeddingsPort` injetado como nova
      dependência; chama `replace_embeddings.replace(deduped_chunks, idioma, tipo="curriculo")`
      **somente** dentro do `if becomes_atualizado:` — currículos que nascem `antigo` não são vetorizados
- [x] `applications/routes/curriculo_routes.py`: `ReplaceEmbeddingsAdapter()` injetado na construção do
      `UploadCurriculoUseCase`

### resume-orchestrator — novos endpoints de upload (relay)

- [x] `requirements.txt`: `httpx` e `python-multipart` adicionados
- [x] `domain/ports/notify_presentation_port.py` (Input) e
      `domain/ports/notify_curriculo_port.py` (Input): interfaces chamadas pelos controllers
- [x] `domain/usecases/notify_presentation_usecase.py` e `domain/usecases/notify_curriculo_usecase.py`:
      chamam o port de Output; se `status_code >= 400`, levantam `PresentationForwardingError`/
      `CurriculoForwardingError` (com `.status_code`/`.detail`) propagando o erro de `resume-injections`
- [x] `infra/ports/forward_presentation_port.py` e `infra/ports/forward_curriculo_port.py` (Output):
      `forward(...) -> tuple[int, dict]` — devolve status code + corpo, deixando a decisão de
      erro/sucesso para o usecase (infra não conhece regra de negócio)
- [x] `infra/adapters/forward_presentation_adapter.py` e `infra/adapters/forward_curriculo_adapter.py`:
      chamam `POST /presentation` / `POST /curriculo` em `resume-injections` via `httpx`
      (`INJECTIONS_SERVICE_URL`, mesmo padrão de `EMBEDDINGS_SERVICE_URL` em resume-injections)
- [x] `applications/dto/notify_presentation_dto.py` e `applications/dto/notify_curriculo_dto.py`:
      validam `idioma` (`pt`|`en`) e, no caso de currículo, `atualizar` (boolean)
- [x] `applications/controllers/notify_presentation_controller.py` e
      `applications/controllers/notify_curriculo_controller.py`: recebem o upload e chamam o usecase
      via port de Input
- [x] `applications/routes/presentation_routes.py` e `applications/routes/curriculo_routes.py`:
      exportam `POST /presentation` e `POST /curriculo`, incluídas em `main.py`. Estrutura hexagonal
      completa (`applications/`, `domain/`, `infra/`) criada do zero neste serviço — antes só existia
      `config/mongo_client.py` e o stub

### resume-embeddings — implementar POST /embeddings/replace

- [x] `domain/ports/replace_embeddings_port.py` (Input): interface chamada pelo controller
- [x] `domain/usecases/replace_embeddings_usecase.py`: orquestra delete (chunks antigos do mesmo
      `idioma`+`tipo`) → insert (novos chunks) via ports de Output, nessa ordem
- [x] `infra/ports/delete_chunks_port.py` (Output) e `infra/adapters/delete_chunks_adapter.py`:
      `collection.delete(where={"$and": [{"idioma": ...}, {"tipo": ...}]})` — usei `$and` explícito em
      vez do dict multi-chave direto do checklist original, sintaxe mais robusta entre versões do
      `chromadb`
- [x] `infra/ports/add_chunks_port.py` (Output) e `infra/adapters/add_chunks_adapter.py`:
      `collection.add(documents=chunks, ids=[f"{tipo}_{idioma}_{i}", ...], metadatas=[{"idioma",
      "tipo"}, ...])` — sem `embeddings=` explícito (embedding function padrão local do `chromadb`,
      modelo ONNX baixado automaticamente no primeiro uso)
- [x] `config/chroma_client.py`: `get_collection()` adicionado, retorna/cria a collection
      `resume_content` via `get_or_create_collection`
- [x] `applications/dto/replace_embeddings_dto.py`: valida `chunks` (lista não vazia), `idioma`
      (`pt`|`en`), `tipo` (`presentation`|`curriculo`)
- [x] `applications/controllers/replace_embeddings_controller.py`: recebe os campos já validados e
      chama o usecase via port de Input
- [x] `applications/routes/embeddings_routes.py`: exporta `POST /embeddings/replace`, incluída em
      `main.py`. **Desvio de padrão**: como este endpoint recebe JSON puro (não multipart como os outros
      dois serviços), a rota declara `payload: ReplaceEmbeddingsDTO` diretamente como body — o FastAPI
      valida automaticamente (422 nativo), sem a construção manual de DTO dentro do controller usada nos
      endpoints de upload de arquivo. Estrutura hexagonal completa criada do zero neste serviço — antes
      só existia `config/chroma_client.py` e o stub. **Verificado fim a fim**: `POST /embeddings/replace`
      chamado via `TestClient` contra o cluster ChromaDB Cloud real (mesmo criado em
      `tasks/chromadb-env-config`) retornou `200 {"status": "replaced", "idioma": "pt", "tipo":
      "curriculo"}`, incluindo o download automático do modelo de embedding ONNX local

### Testes (Constitution Principle III)

- [x] Testes unitários para as mudanças em `replace_presentation_usecase.py` (tipo="presentation") e
      `upload_curriculo_usecase.py` (tipo="curriculo" só quando `becomes_atualizado`; `antigo` não chama
      embeddings) — resume-injections: **54/54 testes passando** (nenhum teste novo, só ajustes nos
      existentes), `ruff check .` limpo
- [x] Testes unitários para `notify_presentation_usecase.py` e `notify_curriculo_usecase.py` (mockando o
      port de Output; erro ≥400 de `resume-injections` propagado com status_code/detail corretos)
- [x] Testes unitários para os 2 adapters de forward em `resume-orchestrator` (mockando `httpx.post`)
- [x] Testes para as rotas de `resume-orchestrator` (`fastapi.testclient.TestClient` — sucesso, DTO
      inválido, erro repassado com o status code original) — resume-orchestrator: **24/24 testes
      passando**, `ruff check .` limpo
- [x] Testes unitários para `replace_embeddings_usecase.py` (delete chamado antes de insert, verificado
      via `Mock` com `attach_mock`, mockando os ports de Output)
- [x] Testes unitários para `delete_chunks_adapter.py` e `add_chunks_adapter.py` (mockando
      `config.chroma_client.get_collection`, sem chamada real ao ChromaDB Cloud nos testes automatizados)
- [x] Testes para a rota `POST /embeddings/replace` em `resume-embeddings` (sucesso e payload inválido)
      — resume-embeddings: **16/16 testes passando**, `ruff check .` limpo. Verificação de
      desenvolvimento (`pytest`/`ruff` diretos) — não é o gate formal de `/speckit-complete`

## Adjustment Requests

None. Approved as-is on 2026-08-26 without the file-by-file walkthrough ("no review needed"). Commits:
`9694f2e` (resume-injections), `6ebe5a4` (resume-orchestrator), `53b95f2` (resume-embeddings). No
secrets touched in this task; confirmed no ChromaDB/MongoDB values leaked into any staged diff.
