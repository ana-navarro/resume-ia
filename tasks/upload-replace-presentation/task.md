# Task: Upload e Substituição da Apresentação

**Status**: Validated - Committed
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

## Implementation Notes

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

None. Approved as-is on 2026-08-26. Two minor non-blocking observations were raised during review
(not required by the checklist, left for a future follow-up if desired):
- `ParsePdfAdapter.extract_text` and `ReplaceEmbeddingsAdapter.replace` let their underlying exceptions
  (`pdfplumber` parse errors, `httpx.HTTPStatusError`) propagate uncaught, so a truly malformed PDF or an
  embeddings-service error surfaces as an HTTP 500 rather than the controller's clean 422. Atomicity is
  still preserved in both cases (old content is never purged before these steps succeed).
- In `ReplacePresentationUseCase.execute`, `replace_embeddings.replace(...)` runs before
  `store_pdf.save(...)`; if the PDF write fails after embeddings succeeded, embeddings would point at
  content not yet reflected in the stored PDF file. Low likelihood, not required by the stated business
  rules.
