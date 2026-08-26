# Task: CRUD de Auditoria, Listagem de Documentos e Guard de Duplicata

**Status**: Validated - Committed
**Created**: 2026-08-26

## Description (PT)

Três entregas relacionadas ao fluxo de upload de documentos já implementado:

1. **CRUD de auditoria** (`resume-orchestrator`): armazena registros de erro (tipo de documento,
   tipo de erro, data, usuário) toda vez que um upload repassado para `resume-injections` falha —
   **exceto** o caso de "documento duplicado" (ver item 3), que não gera auditoria.
2. **Endpoint de listagem de documentos de apresentação** (`resume-injections`): hoje só existe
   `GET /curriculos`; apresentação (PT/EN) não tem nenhuma forma de consulta.
3. **Guard de duplicata, só em apresentação**: se `POST /presentation` for chamado para um idioma que
   já tem documento salvo, a chamada MUST falhar com erro em vez de substituir silenciosamente — **muda
   o comportamento de substituição atômica implementado em `tasks/upload-replace-presentation`**.
   Currículo **não** entra nesse guard — o usuário confirmou que não é necessário, já que o upload de
   currículo sempre é aceito como novo histórico (o mecanismo `atualizado`/`antigo` já resolve isso).

**Este task também é maior que o usual** (CRUD completo de 5 operações + 2 features novas, 2 serviços).
Seguindo a preferência já demonstrada nas tasks anteriores desta sessão, mantive como task leve em vez
de sugerir o pipeline completo.

### Suposições do assistente (usuário recusou responder perguntas de esclarecimento; decisões abaixo
### documentadas para revisão no `/speckit-validate`)

- **CRUD é literal**: o usuário usou a palavra "CRUD", então implemento as 4 operações completas
  (criar, listar, buscar por id, atualizar, apagar) — mesmo sendo incomum permitir editar/apagar
  registros de auditoria (que normalmente são imutáveis). Se isso não for realmente necessário, ajustar
  no `/speckit-validate`.
- **Serviço da auditoria**: `resume-orchestrator`, reaproveitando o MongoDB já configurado
  (`tasks/mongodb-env-config`) — aquela task já registrou "auditoria de erros do sistema em geral" como
  motivação para o MongoDB deste serviço. Nova collection `audit_logs`.
- **Quando a auditoria é criada**: automaticamente, dentro de `notify_presentation_usecase.py` e
  `notify_curriculo_usecase.py`, sempre que a chamada a `resume-injections` retornar erro (≥400) — **com
  exceção do status 409** (ver duplicata abaixo). Além disso, a rota `POST /audit` fica disponível para
  criação manual/direta, caso necessário.
- **Como distinguir "duplicata" de outros erros para não auditar**: `resume-injections` passa a
  responder **409 Conflict** especificamente para o caso de apresentação duplicada (em vez de 422, usado
  para os demais erros de validação). `notify_presentation_usecase.py` verifica o status code: 409 nunca
  gera auditoria; qualquer outro erro ≥400 gera.
- **Campo "usuário"**: não existe autenticação no sistema ainda. Os endpoints de upload do Orchestrator
  passam a aceitar um campo opcional `usuario` (form field, default `"desconhecido"`) usado só para
  preencher a auditoria em caso de erro — não é enviado para `resume-injections`. Um sistema de
  identidade de verdade fica para uma task futura.
- **Campo "tipo de erro"**: string livre derivada do status code retornado por `resume-injections`
  (ex.: `"validacao"` para 4xx, `"erro_interno"` para 5xx) — sem um enum fechado, para não travar em
  categorias que ainda não existem.
- **Listagem de apresentação**: como `StorePdfAdapter` não persiste metadados (só sobrescreve
  `current_presentation_{idioma}.pdf`), `GET /presentation` lista, para cada idioma, se existe arquivo e
  seu tamanho/data de modificação lidos diretamente do filesystem (`Path.stat()`) — sem nova
  infraestrutura de persistência só para isso.
- **Nome da usecase existente mantido**: `ReplacePresentationUseCase`/`replace_presentation_usecase.py`
  continuam com esse nome mesmo não "substituindo" mais nada (agora rejeita se já existir) — renomear em
  cascata (DTO, port, controller, rota, testes) não parece valer o churn só por consistência de nome;
  registrado aqui como pendência cosmética menor.

## Description (EN)

Three related deliverables on top of the already-implemented upload flow: (1) a full audit-log CRUD in
`resume-orchestrator` (MongoDB), auto-populated whenever a relayed upload to `resume-injections` fails —
except for the "duplicate document" case, which is never audited; (2) a `GET /presentation` listing
endpoint in `resume-injections` (currently only `GET /curriculos` exists); (3) a duplicate guard that
applies **only** to presentation uploads — `POST /presentation` now rejects (409) if a document already
exists for that idioma, instead of silently replacing it, reversing the atomic-replace behavior from
`tasks/upload-replace-presentation`. Curriculo is explicitly excluded from the duplicate guard per the
user's clarification, since its `atualizado`/`antigo` versioning already handles repeated uploads.

## Business Rules / Constraints

- `POST /presentation` MUST reject with `409 Conflict` if a document already exists for the given
  `idioma` — checked **before** parsing (fail fast, no wasted work).
- Currículo uploads MUST NOT be affected by any duplicate guard.
- Every non-409 error (≥400) returned by `resume-injections` to `resume-orchestrator`'s relay endpoints
  MUST create an audit log entry (`tipo_documento`, `tipo_erro`, `data`, `usuario`, `detalhe`).
  409 (duplicate) MUST NOT create an audit log entry.
- Audit CRUD MUST use MongoDB (`pymongo`) in `resume-orchestrator`, collection `audit_logs`.
- `GET /presentation` (novo) MUST report, per idioma (`pt`, `en`), whether a document exists and its
  file size / last-modified timestamp.
- Comunicação entre `domain` e `infra` exclusivamente via ports (Constitution Principle II); um adapter
  por verbo/ação.
- Testes unitários MUST mockar MongoDB e o filesystem — sem I/O real nos testes automatizados
  (Constitution Principle III).

## Affected Service(s)

- `/services/resume-orchestrator` (CRUD de auditoria completo; auto-log nos usecases de notify; campo
  `usuario` nos endpoints de upload)
- `/services/resume-injections` (guard de duplicata em `POST /presentation`; novo `GET /presentation`)

## Implementation Checklist

### resume-orchestrator — modelo e persistência da auditoria

- [x] `domain/models/audit_log_entry.py`: entidade com `tipo_documento`, `tipo_erro`, `detalhe`, `data`,
      `usuario`, `id` (opcional, só na leitura)
- [x] `requirements.txt`: confirmado — já tinha `pymongo`, nenhuma dependência nova. `bson` (usado para
      `ObjectId`) já vem embutido no `pymongo`
- [x] `config/mongo_client.py`: **adicionado além do checklist original** — `get_database()` (banco
      `resume_orchestrator`), mesmo padrão de `resume-injections`/`resume-embeddings`; não existia ainda
- [x] `infra/ports/insert_audit_log_port.py` (Output) + `infra/adapters/insert_audit_log_adapter.py`:
      `collection.insert_one(...)` na collection `audit_logs`, retorna o id inserido
- [x] `infra/ports/find_audit_logs_port.py` (Output) + `infra/adapters/find_audit_logs_adapter.py`:
      lista todos os registros
- [x] `infra/ports/find_audit_log_by_id_port.py` (Output) +
      `infra/adapters/find_audit_log_by_id_adapter.py`: busca um registro por id (retorna `None` se o
      id não for um `ObjectId` válido ou não existir)
- [x] `infra/ports/update_audit_log_by_id_port.py` (Output) +
      `infra/adapters/update_audit_log_by_id_adapter.py`: `collection.update_one(...)`
- [x] `infra/ports/delete_audit_log_by_id_port.py` (Output) +
      `infra/adapters/delete_audit_log_by_id_adapter.py`: `collection.delete_one(...)`

### resume-orchestrator — usecases e rota CRUD (`/audit`)

- [x] `domain/ports/create_audit_log_port.py`, `list_audit_logs_port.py`, `get_audit_log_port.py`,
      `update_audit_log_port.py`, `delete_audit_log_port.py` (Input): uma interface por operação
- [x] `domain/usecases/create_audit_log_usecase.py`, `list_audit_logs_usecase.py`,
      `get_audit_log_usecase.py`, `update_audit_log_usecase.py`, `delete_audit_log_usecase.py`: cada
      um orquestra o port de Output correspondente; `get`/`update`/`delete` levantam
      `AuditLogNotFoundError` (definida em `get_audit_log_usecase.py`, reaproveitada pelas outras duas)
      se o id não existir
- [x] `applications/dto/create_audit_log_dto.py` e `applications/dto/update_audit_log_dto.py`: validam
      `tipo_documento` (`presentation`|`curriculo`); update tem todos os campos opcionais (`None` por
      padrão) para permitir atualização parcial
- [x] `applications/controllers/audit_controller.py`: um controller com um método por operação (`create`,
      `list`, `get`, `update`, `delete`), cada um chamando o usecase correspondente via port de Input
- [x] `applications/routes/audit_routes.py`: `POST /audit`, `GET /audit`, `GET /audit/{id}`,
      `PUT /audit/{id}`, `DELETE /audit/{id}` (204 No Content) — incluída em `main.py`.
      **Verificado fim a fim**: `GET /audit` via `TestClient` contra o cluster MongoDB Atlas real
      retornou `200 []`

### resume-orchestrator — auto-log de erros nos uploads existentes

- [x] `applications/dto/notify_presentation_dto.py` e `applications/dto/notify_curriculo_dto.py`:
      campo opcional `usuario` (default `"desconhecido"`) adicionado
- [x] `applications/routes/presentation_routes.py` e `applications/routes/curriculo_routes.py`:
      `usuario: str = Form("desconhecido")` adicionado, propagado ao controller/usecase
- [x] `domain/usecases/notify_presentation_usecase.py`: ao capturar erro ≥400 de `resume-injections`
      **com status ≠ 409**, chama `create_audit_log` (injetado via construtor, instanciado na rota)
      antes de levantar `PresentationForwardingError`; status 409 nunca audita. `tipo_erro` derivado do
      status (`"validacao"` para 4xx, `"erro_interno"` para 5xx)
- [x] `domain/usecases/notify_curriculo_usecase.py`: mesmo comportamento, mas **sem** a exceção de
      status — todo erro ≥400 de currículo audita (não há guard de duplicata para currículo)

### resume-injections — guard de duplicata (apresentação apenas)

- [x] `infra/ports/check_presentation_exists_port.py` (Output) +
      `infra/adapters/check_presentation_exists_adapter.py`: verifica se
      `config/assets/current_presentation_{idioma}.pdf` já existe
- [x] `domain/usecases/replace_presentation_usecase.py`: verifica existência **antes** de parsear;
      levanta nova exceção `PresentationAlreadyExistsError` se já existir (nome da classe/arquivo
      mantido por não valer o churn de renomear em cascata — ver suposição na Description)
- [x] `applications/routes/presentation_routes.py`: captura `PresentationAlreadyExistsError` e responde
      `409 Conflict` (distinto do `422` usado para os demais erros de validação — é assim que
      `resume-orchestrator` decide se audita ou não). O controller (`upload_presentation_controller.py`)
      deixa esse erro propagar sem converter para `ValueError`, para a rota conseguir distinguir os dois

### resume-injections — listagem de apresentação

- [x] `domain/models/presentation_status.py` (**adicionado além do checklist original**): entidade
      `PresentationStatus` (`idioma`, `existe`, `tamanho_bytes`, `ultima_modificacao`)
- [x] `infra/ports/get_presentation_status_port.py` (Output) +
      `infra/adapters/get_presentation_status_adapter.py`: para um idioma, retorna
      `PresentationStatus` (com `existe=False` e os demais campos `None` se o arquivo não existir, em
      vez de retornar `None` como o checklist original sugeria — mais fácil de tipar/testar)
- [x] `domain/ports/list_presentations_port.py` (Input) + `domain/usecases/list_presentations_usecase.py`:
      chama o port de Output para `pt` e `en`, monta a lista de status
- [x] `applications/controllers/list_presentations_controller.py`: formata a resposta pública
- [x] `applications/routes/presentation_routes.py`: exporta `GET /presentation` (pública, sem
      autenticação, mesmo padrão de `GET /curriculos`). **Verificado fim a fim**: `GET /presentation`
      via `TestClient` retornou `200` com `pt`/`en` ambos `existe: false` (nenhum upload feito ainda)

### Testes (Constitution Principle III)

- [x] Testes unitários para os 5 usecases de auditoria e os 5 adapters MongoDB (mockando
      `config.mongo_client.get_database`, sem conexão real) — resume-orchestrator: **61/61 testes
      passando**, `ruff check .` limpo
- [x] Testes para as 5 rotas de `/audit` (`fastapi.testclient.TestClient`)
- [x] Testes unitários para `notify_presentation_usecase.py` (audita em erro ≠409; não audita em 409,
      inclusive teste explícito de status 409) e `notify_curriculo_usecase.py` (sempre audita em erro),
      mockando os ports envolvidos
- [x] Testes unitários para `replace_presentation_usecase.py` revisado: rejeita com
      `PresentationAlreadyExistsError` se já existir (e não chega a chamar `parse_pdf`); segue o fluxo
      normal se não existir — resume-injections: **64/64 testes passando**, `ruff check .` limpo
- [x] Testes para a rota `POST /presentation` revisada (409 em duplicata, além dos casos já existentes)
      e para a nova rota `GET /presentation`
- [x] Testes unitários para `list_presentations_usecase.py` e para os 2 adapters de filesystem
      (`check_presentation_exists_adapter.py`, `get_presentation_status_adapter.py`), usando `tmp_path`
      real do pytest em vez de mockar `Path.stat()` diretamente — mais simples e igualmente isolado

## Adjustment Requests

None. Approved as-is on 2026-08-26 without the file-by-file walkthrough ("no review needed"). Commits:
`5395adb` (resume-orchestrator), `4582f0b` (resume-injections). No secrets touched in this task.
