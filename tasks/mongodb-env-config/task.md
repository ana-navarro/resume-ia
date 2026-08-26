# Task: Configuração do Cliente MongoDB via .env

**Status**: Validated - Committed
**Created**: 2026-08-26

## Description (PT)

Configurar o cliente MongoDB (`pymongo`) em `resume-orchestrator`, lendo a connection string de uma
variável de ambiente carregada de um arquivo `.env` local, que **nunca deve ser commitado**. O objetivo
final (fornecido pelo usuário) é armazenar em uma collection um JSON de cada mensagem que o usuário
pergunta ao assistente — algo como `nome_usuario`, `data`, `id` e `payload` — além de uma auditoria de
erros gerais do sistema. O usuário forneceu usuário, senha e connection string reais diretamente no chat
— esses valores **não são reproduzidos neste arquivo** (que é versionado em git); eles serão colocados
apenas no `.env` local pelo assistente durante `/speckit-implement`.

**Suposição sobre o serviço**: o usuário indicou "todos que forem necessários". Como a persona de
mensagem/auditoria descrita cobre o fluxo completo pergunta → resposta → possíveis erros de qualquer
serviço a jusante, e `resume-orchestrator` é o único serviço com essa visão de ponta a ponta (Constitution
Principle I: "recebe chamadas do bff e coordena o fluxo de dados entre os serviços de IA e validação"),
esta task escopa a configuração do cliente **apenas em `resume-orchestrator`**. Se outro serviço também
precisar gravar diretamente no MongoDB (ex.: `resume-guard-rails` logando suas próprias rejeições), isso
deve ser uma task separada — cada serviço tem seu próprio `.env`/config isolados, sem código
compartilhado entre serviços neste monorepo.

**Escopo desta task**: apenas a configuração do cliente MongoDB (conexão via `.env`), no mesmo padrão de
`tasks/chromadb-env-config`. A modelagem de domínio para efetivamente gravar mensagens/auditoria
(usecases, ports, adapters, schema da collection) **não está incluída aqui** — `resume-orchestrator`
ainda é só o stub "Hello World" e não tem nenhuma rota/usecase real ainda. Fica para uma task futura, a
ser detalhada quando o usuário especificar o restante do fluxo (mencionado como "se tiver mais coisa
peço depois").

**Nota de segurança**: como o usuário, senha e connection string foram compartilhados em texto puro no
chat, devem ser considerados potencialmente expostos. Recomenda-se rotacionar a senha no Atlas/MongoDB
Cloud após esta configuração estar em uso.

## Description (EN)

Configure the MongoDB client (`pymongo`) in `resume-orchestrator`, reading the connection string from an
environment variable loaded from a local `.env` file that must never be committed. The end goal (per the
user) is to persist a JSON document per user message asked to the assistant — roughly `nome_usuario`,
`data`, `id`, `payload` — plus a general system-error audit log. The real username/password/connection
string (provided by the user in chat) are intentionally not reproduced in this tracked file; they will
only be written to the local, gitignored `.env` during `/speckit-implement`.

**Scope assumption**: the user said "all services that need it." Since the message/audit flow described
spans the full question → answer → possible-downstream-error path, and `resume-orchestrator` is the only
service with that end-to-end visibility, this task scopes the client configuration to
`resume-orchestrator` only. Any other service needing its own direct MongoDB writes would be a separate
task. Only the connection setup is in scope here — the actual usecases/ports/adapters to write messages
and audit entries are left for a follow-up task once the user specifies the rest of the flow.

## Business Rules / Constraints

- Segredos (usuário, senha, connection string) MUST viver exclusivamente em `.env` local — nunca em
  código-fonte ou em qualquer arquivo versionado (incluindo este `task.md` e commits futuros).
- `.env` MUST ser adicionado ao `.gitignore` do serviço **antes** de qualquer commit relacionado a esta
  task (`resume-orchestrator` ainda não tem `.gitignore`).
- `.env.example` (mesma chave, sem valor real) MUST ser commitado como documentação do que precisa ser
  configurado.
- A configuração do client MUST viver em `config/` (Constitution Principle II: "Configurações gerais...
  variáveis de ambiente").
- Se a variável obrigatória estiver ausente, a inicialização do client MUST falhar com uma mensagem de
  erro clara (fail-fast), em vez de silenciosamente criar um client inválido.
- Testes unitários MUST mockar `pymongo.MongoClient` — não fazer conexão real ao MongoDB durante os
  testes (Constitution Principle III).
- Fora de escopo: usecases/ports/adapters para efetivamente gravar mensagens ou auditoria — apenas a
  configuração da conexão.

## Affected Service(s)

- `/services/resume-orchestrator` (ver suposição de escopo na Description acima)

## Implementation Checklist

- [x] `requirements.txt`: criado (não existia) com `fastapi`, `uvicorn[standard]`, `pydantic`,
      `pymongo`, `python-dotenv`, `pytest`
- [x] `.gitignore`: criado incluindo `.env`, `.venv/`, `__pycache__/`, `*.pyc`, `.pytest_cache/`,
      `.coveragerc`, `.coverage` — criado **antes** de qualquer outro arquivo, confirmado via
      `git check-ignore -v .env` antes de prosseguir
- [x] `.env.example`: template versionado com a chave `MONGODB_URI` vazia (sem segredo real)
- [x] `.env` (local, **fora do controle de versão**): populado com a connection string real fornecida
      pelo usuário no chat; confirmado que não aparece em `git status --porcelain`
- [x] `config/__init__.py`
- [x] `config/mongo_client.py`: carrega `.env` via `python-dotenv` (`load_dotenv()` no import), lê
      `MONGODB_URI`, e instancia `pymongo.MongoClient(uri)` através de `get_mongo_client()`; levanta
      `RuntimeError` claro se a variável estiver ausente
- [x] Testes unitários (pytest) — `tests/config/test_mongo_client.py`, 2 testes (caminho feliz +
      variável ausente), mockando `pymongo.MongoClient` via `unittest.mock.patch`, sem conexão real.
      2/2 passando localmente (verificação de desenvolvimento com `pytest` direto — não é o gate
      formal de `/speckit-complete`)

## Adjustment Requests

None. Approved as-is on 2026-08-26 without the file-by-file walkthrough ("no review needed"). Before
staging, confirmed `.env` was gitignored (`git check-ignore -v .env`), absent from `git status`, and that
neither the real username, password, nor connection string appeared anywhere in the staged diff
(`git diff --cached | grep`). Commit `b183793` in `services/resume-orchestrator`.
