# Task: Upload e Substituição da Apresentação

**Status**: Completed
**Created**: 2026-08-26

## Description (PT)

Permitir que Ana Elisa (mantenedora) envie um novo arquivo PDF de currículo para substituir a
apresentação atual. O upload deve disparar o reprocessamento completo do conteúdo (parsing, chunking e
geração de novos embeddings), de forma que o assistente de IA passe a responder com base apenas no novo
currículo. Corresponde à User Story 4 e ao FR-007 da spec `001-interactive-resume-ecosystem`.

Escopo desta task: apenas o serviço `resume-injections` (endpoint de upload, parsing/chunking do PDF, e
acionamento da substituição dos embeddings via seu port de saída). Roteamento via `resume-bff` /
`resume-orchestrator` e qualquer UI de upload no frontend ficam fora do escopo — podem ser tratados em
tasks futuras.

## Description (EN)

Allow the project maintainer (Ana Elisa) to upload a new resume PDF that replaces the current one. The
upload triggers full reprocessing (parsing, chunking, re-embedding) so the AI assistant only ever
answers based on the latest resume. Scope is limited to the `resume-injections` service; BFF/orchestrator
routing and any frontend upload UI are out of scope for this task.

## Business Rules / Constraints

- Apenas um currículo "ativo" existe por vez (sem versionamento/histórico) — Assumption já registrada na
  spec.
- Ao substituir, o conteúdo antigo (arquivo PDF anterior e seus embeddings no ChromaDB) MUST ser
  removido (purge), não arquivado.
- **Substituição atômica**: o purge do conteúdo antigo só pode ocorrer DEPOIS que o novo PDF for
  parseado, chunkeado e reindexado com sucesso. Se o novo PDF for inválido/corrompido, o conteúdo
  anteriormente indexado MUST permanecer intacto (edge case já identificado na spec).
- O novo arquivo PDF MUST ser salvo em `config/assets/` (Constitution Principle II), substituindo o
  arquivo anterior somente após a reindexação bem-sucedida.
- Comunicação entre `domain` e `infra` exclusivamente via ports (Constitution Principle II).

## Affected Service(s)

- `/services/resume-injections` (parsing, chunking, orquestração local da substituição)
- Depende de um port de saída para `/services/resume-embeddings` (purge + criação de novos embeddings) —
  se esse port/adapter ainda não existir no lado de `resume-embeddings`, criar um stub mínimo o
  suficiente para o usecase funcionar fim a fim.

## Implementation Checklist

- [x] `applications/dto/upload_presentation_dto.py`: valida o arquivo recebido (tipo PDF, tamanho máximo)
- [x] `applications/controllers/upload_presentation_controller.py`: recebe o upload e chama o usecase
      via o port de Input
- [x] `applications/routes/presentation_routes.py`: exporta a rota `POST /presentation`, incluída em
      `main.py` via `app.include_router(...)`
- [x] `domain/ports/replace_presentation_port.py` (Input): interface chamada pelo controller,
      implementada pelo usecase
- [x] `domain/usecases/replace_presentation_usecase.py`: orquestra parse → chunk → gerar novos
      embeddings → só então purgar PDF/embeddings antigos → salvar novo PDF (substituição atômica)
- [x] `domain/models/presentation_document.py`: entidade de domínio representando o currículo ativo
- [x] `infra/ports/*.py` (Output): `parse_pdf_port`, `chunk_text_port`, `store_pdf_port`,
      `replace_embeddings_port` — chamadas pelo usecase. **Correção**: estas interfaces ficam em
      `infra/ports/`, não em `domain/ports/` como este checklist originalmente listava — Constitution
      Principle II define `domain/ports/` para o port de Input e `infra/ports/` para os ports de Output.
- [x] `infra/adapters/parse_pdf_adapter.py`: extrai texto do PDF via `pdfplumber`
- [x] `infra/adapters/chunk_text_adapter.py`: divide o texto extraído em chunks (1000 chars, 100 de
      overlap)
- [x] `infra/adapters/store_pdf_adapter.py`: persiste o novo PDF em `config/assets/current_resume.pdf`
      via escrita em arquivo temporário + `os.replace` (troca atômica no filesystem)
- [x] `infra/adapters/replace_embeddings_adapter.py`: chama `POST /embeddings/replace` no
      `resume-embeddings` via `httpx` (um adapter por ação, conforme Constitution Principle II)
- [x] `infra/schemas/presentation_chunk_schema.py`: schema dos chunks enviados ao serviço de embeddings
- [x] Testes unitários (pytest) cobrindo: upload válido substitui o currículo; PDF corrompido/sem texto
      não apaga o conteúdo anterior; chunking vazio não apaga o conteúdo anterior; falha na geração de
      embeddings não deixa o sistema em estado inconsistente. 4/4 testes passando localmente
      (Constitution Principle III — mocks completos das Portas de infraestrutura via `unittest.mock`)
- [x] Testes unitários para `UploadPresentationDTO` (validação de `content_type` e `size_bytes`, casos
      válido/inválido) — `tests/applications/dto/test_upload_presentation_dto.py`, 4 testes
- [x] Testes unitários para `UploadPresentationController.handle` (DTO inválido levanta `ValueError`;
      usecase é chamado com os parâmetros corretos; `InvalidPresentationError` do usecase vira
      `ValueError`) — `tests/applications/controllers/test_upload_presentation_controller.py`, 3 testes
      (usa `asyncio.run(...)` em vez de `pytest.mark.asyncio` para não adicionar dependência de
      `pytest-asyncio`)
- [x] Testes para a rota `POST /presentation` (`applications/routes/presentation_routes.py`) via
      `fastapi.testclient.TestClient` — upload válido retorna 200; DTO/usecase inválido retorna 422 —
      `tests/applications/routes/test_presentation_routes.py`, 2 testes (monkeypatcha `_controller` do
      módulo de rotas para isolar do usecase/adapters reais)
- [x] Testes unitários para os 4 adapters em `infra/adapters/` (`ParsePdfAdapter`, `ChunkTextAdapter`,
      `StorePdfAdapter`, `ReplaceEmbeddingsAdapter`), mockando as dependências externas
      (`pdfplumber`, filesystem, `httpx`) — `tests/infra/adapters/test_*.py`, 9 testes no total
- [ ] Rodar `make validate-pipeline` (ou equivalente) e confirmar cobertura ≥ 80% (Constitution
      Principle III) antes de retornar a `/speckit-validate`. **Não executado aqui**: Constitution
      Principle V, passo 2, proíbe explicitamente rodar a simulação de pipeline durante
      `/speckit-implement` — isso é responsabilidade do `/speckit-complete`. Como verificação de
      desenvolvimento (não a validação formal do gate), rodei `pytest` (22/22 passando) e
      `ruff check .` (0 issues, após um `--fix` automático de 2 ajustes de formatação/import)
      diretamente no `.venv` já existente do serviço.

## Implementation Notes

- **Bugfix found while writing route tests**: `requirements.txt` was missing `python-multipart`, which
  FastAPI requires at runtime to parse `UploadFile`/form data — without it, `POST /presentation` fails to
  even start up (`RuntimeError: Form data requires "python-multipart" to be installed`). Added it to
  `requirements.txt` (not `requirements-dev.txt`, since it's a real runtime dependency of the endpoint,
  not a test-only one).
- **Naming convention adapted for Python**: Constitution Principle II's example naming
  (`get-personal-info-adapter`, kebab-case) isn't valid as a Python module name/import path (hyphens
  aren't legal in Python identifiers). All new Python files here use the same `[verbo]_[função]_[tipo]`
  structure in `snake_case` instead (e.g. `parse_pdf_adapter.py`). Frontend/TypeScript code can keep
  literal kebab-case. Worth a small addendum to the constitution to make this explicit.
- **Out of scope, left as a follow-up dependency**: `replace_embeddings_adapter.py` calls
  `POST /embeddings/replace` on `resume-embeddings`, which does not exist yet on that service (it's
  still the FastAPI "Hello World" stub). This task stayed within its stated scope
  (`resume-injections` only) rather than also building that endpoint. A follow-up `/speckit-task` should
  implement `POST /embeddings/replace` on `resume-embeddings` before this feature works end-to-end.
- Added `requirements.txt` (fastapi, uvicorn, pydantic, pdfplumber, httpx, pytest) and an empty
  `conftest.py` at the service root so `pytest` resolves the `domain.*` / `infra.*` / `applications.*`
  absolute imports regardless of invocation directory.
- Verified the 4 usecase unit tests pass in a throwaway venv (removed afterwards, not committed) and
  byte-compiled every new/edited file to catch syntax errors; `fastapi`/`pdfplumber`/`httpx` themselves
  were not installed/exercised since no environment for this service exists yet in the repo.

## Adjustment Requests

**2026-08-26 (/speckit-validate, round 2)**: Approved and committed at the user's request without the
file-by-file walkthrough ("no review needed"). Commit `f96003d` in `services/resume-injections`: adds the
5 new test suites (DTO, controller, route, 4 adapters — 18 tests) plus the `python-multipart` fix and
pipeline scaffolding from `/speckit-implement`.

**2026-08-26 (/speckit-complete)**: `make validate-pipeline` ran for real for the first time (the
`Makefile`, `requirements-dev.txt`, `scripts/gen_coveragerc.py`, `.gitignore`, and `@wip`-skip support in
`conftest.py` were created as part of this run — not yet committed). Lint passed (ruff, 0 issues) and all
4 existing tests passed, but the coverage gate failed: **29.75% vs the required 80%** (Constitution
Principle III). Only `domain/usecases/replace_presentation_usecase.py` has test coverage; the DTO,
controller, routes, and all 4 adapters have 0% coverage. Reopened for `/speckit-implement` to add the five
new checklist items above before returning to `/speckit-validate` → `/speckit-complete`.

Original approval (2026-08-26, still valid for the code covered so far):
Two minor non-blocking observations were raised during review (not required by the checklist, left for a
future follow-up if desired):
- `ParsePdfAdapter.extract_text` and `ReplaceEmbeddingsAdapter.replace` let their underlying exceptions
  (`pdfplumber` parse errors, `httpx.HTTPStatusError`) propagate uncaught, so a truly malformed PDF or an
  embeddings-service error surfaces as an HTTP 500 rather than the controller's clean 422. Atomicity is
  still preserved in both cases (old content is never purged before these steps succeed).
- In `ReplacePresentationUseCase.execute`, `replace_embeddings.replace(...)` runs before
  `store_pdf.save(...)`; if the PDF write fails after embeddings succeeded, embeddings would point at
  content not yet reflected in the stored PDF file. Low likelihood, not required by the stated business
  rules.
