# Task: Configuração do Cliente ChromaDB Cloud via .env

**Status**: Validated - Committed
**Created**: 2026-08-26

## Description (PT)

Configurar o cliente do ChromaDB Cloud (`chromadb.CloudClient`) em `resume-embeddings`, lendo as
credenciais (`api_key`, `tenant`, `database`) de variáveis de ambiente carregadas de um arquivo `.env`
local, que **nunca deve ser commitado**. O usuário forneceu a configuração gerada pelo painel do
ChromaDB Cloud (API key, tenant ID e nome do database) diretamente no chat — esses valores reais **não
são reproduzidos neste arquivo** (que é versionado em git); eles serão colocados apenas no `.env` local
pelo assistente durante `/speckit-implement`.

**Nota de segurança**: como a API key foi compartilhada em texto puro no chat, ela deve ser considerada
potencialmente exposta. Recomenda-se rotacioná-la no painel do ChromaDB Cloud após esta configuração
estar em uso.

## Description (EN)

Configure the ChromaDB Cloud client (`chromadb.CloudClient`) in `resume-embeddings`, reading credentials
(`api_key`, `tenant`, `database`) from environment variables loaded from a local `.env` file that must
never be committed. The real credential values (provided by the user in chat) are intentionally not
reproduced in this tracked file; they will only be written to the local, gitignored `.env` during
`/speckit-implement`.

## Business Rules / Constraints

- Segredos (API key, tenant, database) MUST viver exclusivamente em `.env` local — nunca em código-fonte
  ou em qualquer arquivo versionado (incluindo este `task.md` e commits futuros).
- `.env` MUST ser adicionado ao `.gitignore` do serviço **antes** de qualquer commit relacionado a esta
  task.
- `.env.example` (mesmas chaves, sem valores reais) MUST ser commitado como documentação do que precisa
  ser configurado.
- A configuração do client MUST viver em `config/` (Constitution Principle II: "Configurações gerais...
  variáveis de ambiente").
- Se alguma variável obrigatória estiver ausente, a inicialização do client MUST falhar com uma mensagem
  de erro clara (fail-fast), em vez de silenciosamente criar um client inválido.
- Testes unitários MUST mockar `chromadb.CloudClient` — não fazer chamada real à nuvem durante os testes
  (Constitution Principle III).

## Affected Service(s)

- `/services/resume-embeddings` (único serviço responsável pelo banco de dados vetorial, conforme
  Constitution Principle I)

## Implementation Checklist

- [x] `requirements.txt`: adiciona `chromadb` e `python-dotenv` (arquivo não existia neste serviço
      ainda; criado com fastapi/uvicorn/pydantic/pytest também, alinhado aos outros serviços)
- [x] `.gitignore`: criado incluindo `.env`, `.venv/`, `__pycache__/`, `*.pyc`, `.pytest_cache/`,
      `.coveragerc`, `.coverage` — criado **antes** de qualquer outro arquivo, e confirmado via
      `git check-ignore -v .env` antes de prosseguir
- [x] `.env.example`: template versionado com as chaves `CHROMA_API_KEY`, `CHROMA_TENANT`,
      `CHROMA_DATABASE` vazias (sem segredos reais)
- [x] `.env` (local, **fora do controle de versão**): populado com os valores reais fornecidos pelo
      usuário no chat; confirmado que não aparece em `git status --porcelain`
- [x] `config/__init__.py`
- [x] `config/chroma_client.py`: carrega `.env` via `python-dotenv` (`load_dotenv()` no import), lê
      `CHROMA_API_KEY`, `CHROMA_TENANT`, `CHROMA_DATABASE`, e instancia `chromadb.CloudClient(...)`
      através de `get_chroma_client()`; levanta `RuntimeError` listando a(s) variável(is) ausente(s)
      se alguma estiver faltando
- [x] Testes unitários (pytest) — `tests/config/test_chroma_client.py`, 4 testes (1 caminho feliz +
      3 parametrizados para cada variável obrigatória ausente), mockando `chromadb.CloudClient` via
      `unittest.mock.patch`, sem chamada real à nuvem. 4/4 passando localmente (verificação de
      desenvolvimento com `pytest` direto — não é o gate formal de `/speckit-complete`)
- [x] Teste para o endpoint `GET /` de `main.py` (via `fastapi.testclient.TestClient`) —
      `tests/test_main.py`, 1 teste. `httpx` (necessário pelo `TestClient`, antes apenas transitivo via
      `chromadb`) adicionado explicitamente em `requirements-dev.txt`. Cobertura do serviço agora em
      100% (antes 66.67%), gate de 80% passando

## Adjustment Requests

**2026-08-26 (/speckit-validate, round 2)**: Approved and committed at the user's request without the
file-by-file walkthrough ("no review needed"). Commit `4f747f3` in `services/resume-embeddings`: adds
`tests/test_main.py`, the explicit `httpx` dev dependency, and the pipeline scaffolding
(`Makefile`, `scripts/gen_coveragerc.py`) from `/speckit-implement`.

**2026-08-26 (/speckit-complete)**: `make validate-pipeline` ran for real for the first time (the
`Makefile`, `requirements-dev.txt`, `scripts/gen_coveragerc.py` were created as part of this run — not
yet committed). Lint passed and all 4 existing tests passed, but the coverage gate failed:
**66.67% vs the required 80%**, entirely due to `main.py` (pre-existing stub, untouched by this task) at
0% coverage. Reopened for `/speckit-implement` to add a trivial test for `main.py`'s `GET /` endpoint.

Original approval (2026-08-26, still valid for the code covered so far): Approved as-is without the
file-by-file walkthrough ("no review needed"). Before staging, confirmed `.env` was gitignored
(`git check-ignore -v .env`), absent from `git status`, and that neither the real API key nor tenant ID
appeared anywhere in the staged diff (`git diff --cached | grep`). Commit `951a2aa` in
`services/resume-embeddings`.
