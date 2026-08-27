# Task: Camada de Segurança (resume-guard-rails)

**Status**: Validated - Committed
**Created**: 2026-08-27

## Description (PT)

**Task 2.2** (conforme numerado pelo usuário — continuação do plano maior iniciado em
`tasks/persona-prompt-structure`, Task 2.1). Criar os validadores de input/output do
`resume-guard-rails`:

1. **Validador de input (escopo)**: bloquear perguntas fora de escopo (ex.: "faça um código
   malicioso", "me dê uma receita de bolo") **antes** de a pergunta chegar ao `resume-llm-engine`.
2. **Validador de output (alucinação)**: verificar que a resposta gerada não alucina — não afirma
   informação que não está fundamentada no conteúdo do currículo.

Igual à Task 2.1, esta task cobre **apenas** a lógica de validação dentro de `resume-guard-rails` —
per Constitution Principle II, o fluxo estrito é `Frontend → bff → orchestrator → (guard-rails,
embeddings, llm-engine)`; a fiação real (orchestrator chamando `resume-guard-rails` antes/depois de
chamar `resume-llm-engine`) fica para uma task futura, assim como a integração real com
`resume-embeddings`/`resume-llm-engine`.

### Suposições do assistente (não perguntadas; seguindo o padrão já estabelecido nesta sessão de
### documentar e prosseguir em vez de interromper com perguntas)

- **Sem integração com LLM real**: nenhum provedor de IA está configurado neste projeto ainda (a
  "chamada real" do `resume-llm-engine` também está fora de escopo, per `tasks/persona-prompt-structure`).
  Por isso, os dois validadores desta task são **baseados em regras/heurísticas determinísticas**
  (blocklist de padrões + lista de palavras-chave de escopo para o input; checagem de fundamentação
  textual heurística para o output), não classificação via IA/embeddings. Isso é uma limitação real e
  conhecida — falsos positivos/negativos são esperados — mas é a única abordagem viável sem uma
  integração de IA ainda inexistente no projeto. Documentado claramente no código e no README para não
  ser confundido com uma solução robusta de produção.
- **Validador de input**: heurística de duas camadas —
  1. **Blocklist**: padrões/palavras-chave que indicam pedido malicioso ou claramente prejudicial
     (ex.: pedidos de código malicioso/exploit/hacking, atividades ilegais, conteúdo de ódio/violência)
     → bloqueado imediatamente, motivo `"blocked_content"`.
  2. **Escopo**: a mensagem precisa conter pelo menos uma palavra-chave relacionada a
     carreira/currículo/experiência profissional (PT e EN) OU ser uma saudação/mensagem curta e
     genérica (ex.: "oi", "obrigado") → senão, bloqueada com motivo `"out_of_scope"`.
- **Validador de output**: recebe a resposta gerada **e** o contexto-fonte (texto do currículo que
  deveria fundamentar a resposta — passado como parâmetro, já que `resume-guard-rails` não busca
  currículo sozinho). Extrai heuristicamente "afirmações específicas" do output (sequências de
  palavras capitalizadas tipo nome próprio, números com unidade tipo "anos"/"years") e verifica se
  cada uma aparece (case-insensitive, substring) no contexto-fonte; se alguma não aparecer, marca como
  `is_grounded=False` com a lista de trechos suspeitos. **Suposição de nome de parâmetro**: como o
  contexto-fonte ainda não tem uma fonte real conectada (currículo via embeddings é integração
  futura), o parâmetro é só um `str` livre passado pelo chamador.
- **Endpoints REST**: `POST /validate-input` (`{"message": str}` → `{"allowed": bool, "reason":
  str|null}`) e `POST /validate-output` (`{"output": str, "source_context": str}` → `{"is_grounded":
  bool, "flagged_claims": list[str]}`) — pensados para serem chamados pelo `resume-orchestrator` numa
  task futura de fiação.
- **`resume-guard-rails` ainda é um stub completo** — esta task cria a estrutura hexagonal inteira do
  zero, seguindo o mesmo padrão de `resume-injections`/`resume-orchestrator`/`resume-embeddings`/
  `resume-llm-engine`, incluindo infra de testes (`Makefile`, `requirements-dev.txt`,
  `scripts/gen_coveragerc.py`, `conftest.py`, `.gitignore`).
- **Upgrade do CI**: `services/resume-guard-rails/.github/workflows/main.yml` hoje só roda `docker
  build` (era stub). Atualizado para `validate-pipeline` (install/lint/test) + `promote`, mesmo padrão
  dos demais serviços com código real — consistente com a política "todo repositório precisa de uma
  pipeline com o nome do repositório" já aplicada aos outros 8 repositórios.

## Description (EN)

Implements `resume-guard-rails`'s input/output validators: an **input validator** that blocks
out-of-scope or malicious questions (e.g. "write malicious code", "give me a cake recipe") before they
reach `resume-llm-engine`, and an **output validator** that flags potentially ungrounded/hallucinated
claims in a generated answer against a provided source context. Since no real LLM/embeddings
integration exists in this project yet, both validators are deterministic, rule-based heuristics
(blocklist + scope keywords for input; substring-grounding check for output) rather than AI-based
classification — a documented, known limitation, not a production-grade solution. Scope is limited to
the validators themselves; wiring `resume-orchestrator` to actually call these endpoints around its
`resume-llm-engine` call is a future task. This is also the first real feature in this service, so it
bootstraps the full hexagonal structure/test infra and upgrades its CI to the real
`validate-pipeline` + gate pattern.

## Business Rules / Constraints

- Validação de input MUST bloquear conteúdo malicioso/prejudicial (blocklist) e perguntas fora do
  escopo de carreira/currículo (scope keywords), retornando o motivo do bloqueio.
- Validação de output MUST identificar afirmações específicas no texto gerado que não estão presentes
  no contexto-fonte fornecido, retornando a lista de trechos suspeitos (sem bloquear automaticamente —
  cabe ao chamador decidir o que fazer com a sinalização).
- Ambos os validadores MUST ser funções determinísticas (mesma entrada → mesma saída), sem chamada de
  rede/IA externa (nenhum provedor configurado ainda).
- Fora de escopo: fiação real com `resume-orchestrator`, integração com `resume-embeddings`/
  `resume-llm-engine`, e qualquer validação baseada em modelo de IA/ML (tasks futuras).
- Comunicação entre `domain` e `infra` exclusivamente via ports; um adapter por verbo/ação
  (Constitution Principle II).
- Testes unitários MUST mockar os ports de infraestrutura, cobertura ≥80% (Constitution Principle III).
- Nomes de atributos/campos em inglês.

## Affected Service(s)

- `/services/resume-guard-rails` (estrutura hexagonal completa criada do zero: models/usecases/ports/
  adapters/controllers/rotas, infra de testes, `main.py`, CI)

## Implementation Checklist

### Domínio — validação de input (escopo/blocklist)

- [x] `domain/models/input_validation_result.py`: dataclass `InputValidationResult` (`allowed: bool`,
      `reason: str | None`)
- [x] `domain/ports/validate_input_port.py` (Input): interface chamada pelo controller
- [x] `domain/usecases/validate_input_usecase.py`: recebe `message: str`; checa contra blocklist
      (retorna `allowed=False, reason="blocked_content"` se match); senão checa escopo — mensagem
      curta/saudação OU contém ao menos uma scope keyword (retorna `allowed=False,
      reason="out_of_scope"` senão); caso contrário `allowed=True`

### Domínio — validação de output (fundamentação/alucinação)

- [x] `domain/models/output_validation_result.py`: dataclass `OutputValidationResult`
      (`is_grounded: bool`, `flagged_claims: list[str]`)
- [x] `domain/ports/validate_output_port.py` (Input): interface chamada pelo controller
- [x] `domain/usecases/validate_output_usecase.py`: recebe `output: str` e `source_context: str`;
      extrai candidatos a afirmação específica (regex: sequências de palavras capitalizadas tipo
      nome próprio, números com unidade); para cada um, verifica presença (case-insensitive) no
      `source_context`; monta `flagged_claims` com os que não aparecem; `is_grounded = not
      flagged_claims`

### Infra — conteúdo hardcoded (blocklist e scope keywords)

- [x] `infra/ports/get_blocked_patterns_port.py` + `infra/adapters/get_blocked_patterns_adapter.py`:
      retorna lista hardcoded de padrões/palavras-chave de conteúdo malicioso/proibido (código
      malicioso, hacking/exploit, atividade ilegal, ódio/violência)
- [x] `infra/ports/get_scope_keywords_port.py` + `infra/adapters/get_scope_keywords_adapter.py`:
      retorna lista hardcoded de palavras-chave de escopo (carreira, currículo, experiência,
      habilidades, projetos, formação, tecnologias — PT e EN)

### Aplicação — controllers e rotas

- [x] `applications/dto/validate_input_dto.py`: valida `message: str` (não vazio)
- [x] `applications/dto/validate_output_dto.py`: valida `output: str`, `source_context: str`
- [x] `applications/controllers/validate_input_controller.py` + rota `POST /validate-input`
- [x] `applications/controllers/validate_output_controller.py` + rota `POST /validate-output`
- [x] `applications/routes/guard_rails_routes.py`: exporta as duas rotas, incluída em `main.py`

### Bootstrap de infraestrutura (primeira feature real deste serviço)

- [x] Estrutura hexagonal completa (`applications/{dto,controllers,routes}`,
      `domain/{models,ports,usecases}`, `infra/{adapters,ports}`, `config/`), mesmo padrão dos outros
      serviços reais
- [x] `requirements-dev.txt`, `Makefile` (`install`/`lint`/`test`/`validate-pipeline`),
      `scripts/gen_coveragerc.py`, `conftest.py`, `.gitignore` — idênticos ao padrão já usado
- [x] `services/resume-guard-rails/.github/workflows/main.yml`: troca o job `build` (`docker build`)
      pelo padrão `validate-pipeline` + `promote`, nomeado `resume-guard-rails CI`

### Testes (Constitution Principle III)

- [x] Testes unitários para `validate_input_usecase.py`: bloqueia conteúdo malicioso; bloqueia fora de
      escopo; permite pergunta de carreira válida; permite saudação curta (mockando os 2 ports de
      Output)
- [x] Testes unitários para `validate_output_usecase.py`: marca alucinação quando afirmação não está
      no contexto-fonte; não marca quando está fundamentado; nenhuma afirmação específica extraída →
      `is_grounded=True`
- [x] Testes unitários para os 2 adapters hardcoded
- [x] Testes para os 2 controllers e as 2 rotas (`fastapi.testclient.TestClient`)
- [x] Rodar `pytest`/`ruff` localmente como verificação de desenvolvimento antes de `/speckit-validate`
      (o gate formal de cobertura ≥80% continua responsabilidade do `/speckit-complete`)

## Implementation Notes

- **`README.md` atualizado** (fora do checklist original, mesmo padrão de toda task anterior que
  bootstrapou a primeira feature real de um serviço): descreve os 2 endpoints, reforça a limitação de
  heurística determinística, documenta `make validate-pipeline`.
- **`requirements.txt`**: adicionado `pydantic` explicitamente (antes só vinha transitivamente via
  `fastapi`) já que os DTOs agora o usam diretamente.
- **Rotas JSON puras**: seguindo o padrão já usado em `resume-embeddings`/`resume-orchestrator` para
  endpoints sem multipart, as rotas declaram o DTO Pydantic diretamente como body
  (`payload: ValidateInputDTO`), deixando o FastAPI validar nativamente (422 automático) — os
  controllers recebem o DTO já validado, não strings soltas.
- **Bug real pego pelo `ruff` durante a verificação local**: `COMMON_LEADING_WORDS` (em
  `validate_output_usecase.py`) tinha `"as"` duplicado — a palavra inglesa "as" e o artigo plural
  português "as" colidem como a mesma string. `ruff` (regra B033) pegou isso antes do commit; corrigido
  removendo a duplicata (sem mudança de comportamento, já que era um `set`).
- **Verificação de desenvolvimento** (não é o gate formal de `/speckit-complete`): `.venv` local criado,
  `ruff check .` (0 issues após a correção acima) e `pytest --cov` — **23/23 testes passando, 91.73% de
  cobertura** (acima do gate de 80%).

## Adjustment Requests

None. Approved as-is on 2026-08-27, no interactive walkthrough (per the amended `/speckit-validate` —
Constitution v1.4.x). Commit: `702295e` (resume-guard-rails, 54 files). No secrets touched — confirmed
no `.env` among the staged files before committing.
