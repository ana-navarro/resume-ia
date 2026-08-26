# Task: Alinhar Serviço de Injeção às Regras de `features/injection/rules.md`

**Status**: Validated - Committed
**Created**: 2026-08-26

## Description (PT)

`features/injection/rules.md` define as regras finais (por enquanto) do sistema de injeção de
documentos (apresentação e currículos). O pedido do usuário foi: validar se todas essas regras já
estão implementadas e, para as que não estão, implementá-las — o documento é a fonte da verdade e
deve prevalecer sobre o comportamento atualmente implementado.

Uma auditoria do código de `resume-injections` e `resume-orchestrator` (as tasks
`upload-replace-presentation`, `public-curriculo-listing`, `wire-upload-embeddings-flow` e
`audit-log-document-listing`) encontrou **desvios reais em quase todas as regras do documento**:

1. **Nomes de atributos em português** (Regra Geral 3: "Todos os nomes dos atributos tem que ser em
   inglês"): campos como `idioma`, `atualizar`, `nome_arquivo`, `data_upload`, `usuario`,
   `tipo_documento`, `tipo_erro`, `detalhe`, `existe`, `tamanho_bytes`, `ultima_modificacao`, e os
   valores de status `"atualizado"`/`"antigo"` aparecem em DTOs, models, rotas e corpos de
   resposta/requisição em `resume-injections` e `resume-orchestrator`. Isso viola a regra de forma
   ampla — ~50 arquivos em `resume-injections` e ~45 em `resume-orchestrator` referenciam esses nomes
   (incluindo testes).
2. **Substituição de apresentação quebrada** (Regra "Endpoint separado para subida de upload
   apresentação"): a regra pede um atributo opcional `substituicao` — se `true`, permite substituir um
   documento já existente para aquele idioma; se ausente ou `false`, deve retornar erro. A task
   `audit-log-document-listing` **removeu completamente essa flag** e fez `POST /presentation` sempre
   retornar `409` quando já existe um documento para o idioma — ou seja, hoje **não existe nenhuma
   forma de substituir** uma apresentação já enviada, o que contraria a regra diretamente.
3. **Sem persistência em banco de dados dos metadados de apresentação**: a regra exige salvar no
   banco "o nome do arquivo, data de upload, se é o mais atual e linguagem". Hoje `GET /presentation`
   deriva tudo do filesystem (`Path.stat()`) — não existe nenhuma collection MongoDB para isso.
4. **Sem objeto de auditoria na substituição de apresentação**: a regra pede auditoria especificamente
   "quando acontece a substituição". A auditoria hoje existente (`resume-orchestrator`, CRUD de
   `audit_logs`) só é criada em **erros** (status ≥400 não-409) do relay — nunca em uma substituição
   bem-sucedida, que é o evento que a regra pede para auditar.
5. **Currículo: contrato de upload não corresponde à regra**: a regra pede os campos `tipo` (valores
   `contexto`|`atualizado`) e `confirmacao` (opcional, só relevante quando `tipo=atualizado` e já existe
   um `atualizado` para o idioma — sem confirmação, deve retornar erro). A implementação atual usa um
   único campo `atualizar: bool` que nunca gera erro (default `False` silenciosamente insere como
   histórico) — o mecanismo de confirmação obrigatória da regra não existe.
6. **Currículo: PDF nunca é armazenado em disco**: a regra pede que "somente o último currículo
   atualizado deve ser armazenado na pasta" `assets/curriculos`. Hoje `upload_curriculo_usecase.py`
   nunca chama nenhum `store_pdf` — o PDF é descartado após parsing, só o texto vai para MongoDB/
   ChromaDB.
7. **Sem subpastas `apresentacoes`/`curriculos` em `assets`**: a regra pede exatamente essas duas
   subpastas. Hoje `config/assets/` é usado de forma flat (`current_presentation_{idioma}.pdf`
   diretamente na raiz), sem nenhuma subpasta.
8. **Endpoints de listagem não filtram por língua**: a regra pede um endpoint "para pegar esse
   documento filtrando por língua", tanto para apresentação quanto para currículo. `GET /presentation`
   e `GET /curriculos` hoje sempre retornam tudo, sem parâmetro de filtro.

Esta task corrige os 8 pontos acima. Cresceu bem além do pedido original (~95 arquivos tocados pelo
rename, mais várias mudanças de comportamento em 2 serviços) — seguindo a preferência já demonstrada
pelo usuário nas tasks anteriores desta sessão, mantida como task leve em vez do pipeline completo.
Várias decisões de design abaixo são suposições documentadas do assistente (o usuário não foi
interrompido com perguntas de esclarecimento, seguindo o padrão já estabelecido nesta sessão) — revisar
com atenção no `/speckit-validate`.

## Description (EN)

`features/injection/rules.md` is the final (for now) rule set for the document-injection system
(presentation and résumés). The user asked to audit the current implementation against it and
implement whatever is missing — the document is the source of truth and overrides current behavior.

An audit of `resume-injections` and `resume-orchestrator` found real deviations from nearly every rule:
Portuguese attribute names throughout the API contract (violates the explicit "all attribute names
must be English" rule), a presentation-replace flow that was fully removed in favor of an
unconditional 409 (no way to replace a presentation anymore), no DB persistence of presentation
metadata, no audit record on successful replacement (only on error), a curriculo upload contract using
a single `atualizar` boolean instead of the required `tipo`/`confirmacao` pair with mandatory
confirmation error, curriculo PDFs never persisted to disk, no `assets/apresentacoes` /
`assets/curriculos` folder split, and no language filter on the listing endpoints. This task closes
all of these gaps.

## Business Rules / Constraints

- `features/injection/rules.md` é a fonte da verdade final; onde o comportamento implementado diverge,
  o documento prevalece (instrução explícita do usuário).
- Regra Geral 3 (nomes em inglês) aplica-se a **todos** os atributos do contrato público (form
  fields, query params, corpos de resposta) e aos nomes de campos internos (models/DTOs) nos dois
  serviços — mapeamento adotado: `idioma`→`language`, `nome_arquivo`→`file_name`,
  `data_upload`→`upload_date`, `usuario`→`user`, `tipo_documento`→`document_type`,
  `tipo_erro`→`error_type`, `detalhe`→`detail`, `existe`→`exists`, `ultima_modificacao`→`last_modified`,
  status `"atualizado"`→`"current"`, status `"antigo"`/`"contexto"`→`"context"`, `tipo` (currículo,
  campo de entrada)→`type` com valores `"context"`|`"current"`, `confirmacao`→`confirm`,
  `substituicao`→`replace`.
- `POST /presentation`: com `language` já existente, MUST exigir `replace=true` para prosseguir (erro
  caso contrário); sem `language` existente, cria normalmente independente de `replace`.
- Substituição de apresentação só ocorre após parse→chunk→embeddings→store bem-sucedidos (substituição
  atômica já implementada, preservada).
- Metadados de apresentação (`file_name`, `upload_date`, `is_current`, `language`) MUST ser persistidos
  em MongoDB, database `resume_injections` (mesmo padrão de `tasks/public-curriculo-listing`).
- Um registro de auditoria de apresentação MUST ser criado apenas quando uma substituição bem-sucedida
  ocorre (não na criação inicial, não em erros). **Suposição de arquitetura**: esse log vive localmente
  em `resume-injections` (nova collection Mongo), não em `resume-orchestrator` — uma chamada de volta de
  `resume-injections` para `resume-orchestrator` inverteria o fluxo estrito Frontend→bff→
  orchestrator→serviços da Constitution Principle II. É um conceito de auditoria distinto do CRUD de
  erros já existente em `resume-orchestrator` (que continua existindo, só com os campos renomeados).
- `POST /curriculo`: campos `type` (`"context"`|`"current"`) e `confirm` (opcional, default `false`).
  Se `type="current"` e já existe um `"current"` para o `language`, exige `confirm=true` (senão erro);
  ao confirmar, rebaixa os `"current"` existentes do idioma para `"context"` antes de inserir o novo.
  Primeiro currículo de um idioma sempre pode nascer `"current"` sem exigir `confirm` (nada para
  confirmar ainda — suposição mantida de `tasks/public-curriculo-listing`). `type="context"` nunca
  promove nem exige confirmação.
- Apenas o currículo `"current"` de cada idioma é persistido em `assets/curriculos/` (substitui o
  anterior); currículos `"context"` nunca tocam o filesystem — só existem via MongoDB + embeddings.
- `assets/` MUST ter exatamente duas subpastas: `apresentacoes/` e `curriculos/`.
- `GET /presentation` e `GET /curriculos` MUST aceitar um parâmetro opcional `language` para filtrar.
- Comunicação entre `domain` e `infra` exclusivamente via ports; um adapter por verbo/ação (Constitution
  Principle II).
- Testes unitários MUST mockar MongoDB e o filesystem — sem I/O real nos testes automatizados
  (Constitution Principle III).

## Affected Service(s)

- `/services/resume-injections` (a maior parte das mudanças: DTOs, models, usecases, adapters, rotas)
- `/services/resume-orchestrator` (rename dos campos de auditoria e do relay; propaga `replace` e
  `type`/`confirm` para `resume-injections`)

## Implementation Checklist

### Rename global de atributos para inglês (Regra Geral 3)

- [x] `resume-injections`: `idioma`→`language` em todos os DTOs, models, usecases, controllers, rotas
      (form fields e query params) e adapters/ports envolvidos: `upload_presentation_dto.py`,
      `presentation_document.py`, `replace_presentation_usecase.py`,
      `upload_presentation_controller.py`, `presentation_routes.py`, `upload_curriculo_dto.py`,
      `curriculo_document.py`, `upload_curriculo_usecase.py`, `upload_curriculo_controller.py`,
      `curriculo_routes.py`, `list_curriculos_controller.py`, `list_presentations_controller.py`,
      `list_presentations_usecase.py`, `check_presentation_exists_adapter.py`/`port.py`,
      `replace_embeddings_adapter.py`/`port.py`, `presentation_chunk_schema.py`, e os adapters/ports de
      `count`/`mark`/`insert`/`find` de currículo
- [x] `resume-injections`: `nome_arquivo`→`file_name` em `curriculo_document.py` e módulos relacionados
      (alinha com `presentation_document.py`, que já usa `file_name`)
- [x] `resume-injections`: `data_upload`→`upload_date`
- [x] `resume-injections`: status `"atualizado"`/`"antigo"`→`"current"`/`"context"` (ver também
      checklist de currículo abaixo)
- [x] `resume-injections`: `existe`→`exists`, `ultima_modificacao`→`last_modified` (ou remover, se
      substituídos pelos novos campos de metadados em Mongo — ver checklist de apresentação)
- [x] `resume-orchestrator`: `idioma`→`language`, `usuario`→`user`, `tipo_documento`→`document_type`,
      `tipo_erro`→`error_type`, `detalhe`→`detail`, `data`→`date` em: `audit_log_entry.py`,
      `create_audit_log_dto.py`, `update_audit_log_dto.py`, `audit_controller.py`, `audit_routes.py`,
      os 5 usecases e 5 adapters de auditoria, `notify_presentation_dto.py`/`usecase.py`/
      `controller.py`, `notify_curriculo_dto.py`/`usecase.py`/`controller.py`,
      `forward_presentation_adapter.py`/`port.py`, `forward_curriculo_adapter.py`/`port.py`,
      `presentation_routes.py`, `curriculo_routes.py`
- [x] Atualizar todas as suítes de teste afetadas pelo rename nos dois serviços (grep confirmou ~50
      arquivos em `resume-injections` e ~45 em `resume-orchestrator`, incluindo testes)

### Apresentação — substituição real via `replace` (Regra "Endpoint separado...")

- [x] `applications/dto/upload_presentation_dto.py`: adiciona `replace: bool = False`
- [x] `applications/routes/presentation_routes.py`: `POST /presentation` recebe
      `replace: bool = Form(False)` além de `language`
- [x] `domain/usecases/replace_presentation_usecase.py`: reverte o guard "sempre 409" introduzido em
      `tasks/audit-log-document-listing` — permite substituição quando já existe e `replace=True`;
      mantém erro (409) quando já existe e `replace` não é `True`; cria normalmente quando não existe

### Apresentação — metadados em MongoDB (Regra "No banco de dados, deve salvar...")

- [x] `domain/models/presentation_metadata.py` (novo): `language`, `file_name`, `upload_date`,
      `is_current`
- [x] `infra/ports/upsert_presentation_metadata_port.py` +
      `infra/adapters/upsert_presentation_metadata_adapter.py` (Output, MongoDB, collection
      `presentations`, database `resume_injections`)
- [x] `infra/ports/find_presentation_metadata_port.py` +
      `infra/adapters/find_presentation_metadata_adapter.py` (Output, busca por `language` opcional)
- [x] `replace_presentation_usecase.py`: chama `upsert_presentation_metadata` após `store_pdf.save(...)`
      bem-sucedido (tanto na criação quanto na substituição)

### Apresentação — auditoria na substituição (Regra "Quando acontece a substituição...")

- [x] `domain/models/presentation_audit_entry.py` (novo): `language`, `file_name`, `action="replaced"`,
      `date`
- [x] `infra/ports/insert_presentation_audit_port.py` +
      `infra/adapters/insert_presentation_audit_adapter.py` (Output, MongoDB, collection
      `presentation_audit_logs`, mesmo database)
- [x] `replace_presentation_usecase.py`: só no branch de substituição (já existia + `replace=True`),
      após persistir com sucesso, chama `insert_presentation_audit.execute(...)` — criação inicial não
      gera esse registro

### Apresentação — filtro por língua (Regra "Endpoint para pegar...") + subpasta `apresentacoes` (Regra 8)

- [x] `applications/routes/presentation_routes.py`: `GET /presentation` aceita `language` opcional via
      query string; com `language`, retorna um único objeto; sem, mantém a lista, agora construída a
      partir dos metadados do Mongo (não mais do filesystem)
- [x] `domain/usecases/list_presentations_usecase.py`: fonte de dados migra de
      `GetPresentationStatusAdapter` (filesystem) para `find_presentation_metadata` (Mongo) —
      necessário porque a regra pede nome do arquivo, data de upload e "se é o mais atual", dados que só
      existem na nova collection
- [x] `infra/adapters/store_pdf_adapter.py` → renomeia para `store_presentation_pdf_adapter.py`,
      salvando em `config/assets/apresentacoes/current_presentation_{language}.pdf`
- [x] `infra/adapters/check_presentation_exists_adapter.py`: ajustar path para `apresentacoes/`; avaliar
      se `get_presentation_status_adapter.py`/`port.py` ficam obsoletos (substituídos pelos metadados
      Mongo) e podem ser removidos
- [x] `config/assets/apresentacoes/.gitkeep` e `config/assets/curriculos/.gitkeep` (nova estrutura)

### Currículo — `type`+`confirm` em vez de `atualizar` (Regra "tipo (pode ser contexto ou atualizado)")

- [x] `applications/dto/upload_curriculo_dto.py`: substitui `atualizar: bool` por `type: str`
      (`"context"`|`"current"`, validado) e `confirm: bool = False`
- [x] `applications/routes/curriculo_routes.py`: `POST /curriculo` recebe `type: str = Form(...)` e
      `confirm: bool = Form(False)`
- [x] `domain/usecases/upload_curriculo_usecase.py`: reescreve a lógica — `type == "current"` com um
      `"current"` já existente para o `language` e `confirm` não `True` → levanta novo
      `CurriculoConfirmationRequiredError` (mapeado para 422 na rota); com `confirm=True`, rebaixa os
      `"current"` existentes do idioma para `"context"` e insere o novo como `"current"`; primeiro
      currículo do idioma sempre nasce `"current"` sem exigir `confirm`; `type == "context"` insere
      direto, sem promover nem exigir confirmação
- [x] `domain/models/curriculo_document.py`: `status` usa `"current"`/`"context"`

### Currículo — armazenar apenas o `"current"` em `assets/curriculos` (Regra 13)

- [x] `infra/ports/store_curriculo_pdf_port.py` + `infra/adapters/store_curriculo_pdf_adapter.py`
      (novo, Output): salva em `config/assets/curriculos/current_curriculo_{language}.pdf`, mesma
      escrita atômica (`.tmp` + `os.replace`) do adapter de apresentação
- [x] `domain/usecases/upload_curriculo_usecase.py`: chama `store_curriculo_pdf.save(...)` **somente**
      quando o currículo se torna `"current"`; currículos que nascem `"context"` não tocam o filesystem

### Currículo — filtro por língua (Regra "Endpoint para pegar...")

- [x] `applications/routes/curriculo_routes.py`: `GET /curriculos` aceita `language` opcional via query
      string
- [x] `domain/usecases/list_curriculos_usecase.py` + `infra/adapters/find_curriculos_adapter.py`:
      propaga o filtro opcional de `language` até a query MongoDB

### resume-orchestrator — acompanhar o novo contrato do relay

- [x] `notify_presentation_dto.py`, `notify_presentation_usecase.py`, `presentation_routes.py`,
      `forward_presentation_adapter.py`: propagar o novo campo `replace`
- [x] `notify_curriculo_dto.py`, `notify_curriculo_usecase.py`, `curriculo_routes.py`,
      `forward_curriculo_adapter.py`: trocar `atualizar` por `type`/`confirm`

### Testes (Constitution Principle III)

- [x] Atualizar todos os testes existentes afetados pelos renames acima nos dois serviços
- [x] Novos testes: guard de substituição de apresentação (`replace=False` com documento existente →
      erro; `replace=True` → substitui e audita; sem documento existente → cria sem auditar)
- [x] Novos testes: metadados de apresentação em Mongo (upsert/find), mockando `pymongo`
- [x] Novos testes: auditoria de substituição de apresentação (criada só no branch de replace
      bem-sucedido)
- [x] Novos testes: `GET /presentation?language=` e `GET /curriculos?language=` (com e sem filtro)
- [x] Novos testes: fluxo `type`/`confirm` de currículo — primeiro do idioma nasce `current` sem
      `confirm`; `type=current` com `current` já existente e `confirm=False` → erro; com `confirm=True`
      → promove e rebaixa os demais; `type=context` nunca promove nem exige confirmação
- [x] Novos testes: `store_curriculo_pdf_adapter.py` salva apenas quando `current`; nunca chamado para
      `context`
- [x] Rodar suíte local (`pytest`/`ruff`) como verificação de desenvolvimento em ambos os serviços antes
      de `/speckit-validate` (o gate formal de cobertura ≥80% continua responsabilidade do
      `/speckit-complete`, Constitution Principle V)

## Implementation Notes

- **`get_presentation_status_port.py`/`adapter.py` e `domain/models/presentation_status.py` foram
  removidos** (não apenas "avaliados" como o checklist original sugeria): totalmente substituídos por
  `find_presentation_metadata_port`/`adapter` + `PresentationMetadata`/`PresentationInfo`, já que a regra
  exige os dados virem do banco, não do filesystem. `check_presentation_exists_adapter.py` continua
  existindo (só ele ainda lê o filesystem, para o guard fail-fast de duplicata).
- **Extensão não pedida explicitamente**: o guard de "não auditar 409" em
  `notify_presentation_usecase.py` (`tasks/audit-log-document-listing`) foi replicado para
  `notify_curriculo_usecase.py`, agora que currículo também tem um 409 esperado
  (`CurriculoConfirmationRequiredError`) — mesma justificativa original (é um sinal de negócio esperado,
  não uma falha do sistema). Antes, currículo não tinha nenhum código de erro especial e todo erro
  gerava auditoria.
- **Valor default `usuario="desconhecido"` traduzido para `user="unknown"`**: como o próprio nome do
  atributo virou inglês, manter um valor padrão em português pareceu inconsistente com a Regra Geral 3;
  é um sentinel interno, não texto voltado ao usuário final.
- **Filtro `language` em `GET /presentation`/`GET /curriculos` não valida o valor** (aceita qualquer
  string; um idioma inexistente simplesmente não retorna resultados) — são endpoints públicos de leitura,
  então um valor inválido não tem efeito colateral, apenas resposta vazia/`exists:false`. Se isso não for
  aceitável, adicionar validação (`pt`|`en`) é uma mudança pequena e isolada na rota.
- **`README.md` de `resume-injections` não foi atualizado** — já estava desatualizado antes desta task
  (não mencionava os endpoints de currículo/auditoria adicionados por tasks anteriores) e atualizar
  documentação não fazia parte do checklist de alinhamento às regras; fica como pendência cosmética.
- Rodei `pytest` e `ruff check .` diretamente (não `make validate-pipeline`) em ambos os serviços como
  verificação de desenvolvimento, conforme o padrão já estabelecido nas tasks anteriores desta sessão —
  **resume-injections: 83/83 testes passando, ruff limpo; resume-orchestrator: 63/63 testes passando,
  ruff limpo**. O gate formal de cobertura ≥80% (Constitution Principle III) continua responsabilidade
  do `/speckit-complete`.

## Adjustment Requests

None. Approved as-is on 2026-08-26 without the file-by-file walkthrough (user: "no need approvals only
make the commit"). Commits: `2c7b8ff` (resume-injections, 75 files), `86fe4de` (resume-orchestrator, 45
files). No secrets touched in this task — verified `.env` absent from both nested repos' `git status`
before staging.
