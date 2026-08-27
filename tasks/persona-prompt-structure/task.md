# Task: Estruturação dos Prompts (resume-llm-engine)

**Status**: Validated - Committed
**Created**: 2026-08-27

## Description (PT)

**Task 2.1** (conforme numerado pelo usuário — parte de um plano maior para `resume-llm-engine` que
inclui, em tasks futuras, a integração do conteúdo do currículo via embeddings e a chamada real à API de
IA). Implementar a lógica de construção do prompt final usado para configurar a persona do assistente,
separando a estrutura em 3 pilares:

1. **Rules**: regras estritas de comportamento da IA — ex.: nunca inventar experiências, certificações
   ou habilidades que não estejam presentes no currículo.
2. **Context**: informações fixas (hardcoded) que não vêm dos PDFs de currículo — ex.: objetivo de
   carreira, habilidades interpessoais.
3. **Tone of Voice**: tom de resposta — profissional, empático, resolutivo.

O `resume-llm-engine` monta o "system prompt" combinando os 3 pilares de forma determinística. Esta
task cobre **apenas** essa montagem — per a própria Constitution ("`resume-llm-engine`: configura o LLM,
**aplica a persona** e executa a chamada real para a API de IA"), "aplicar a persona" é um passo
distinto de "configurar o LLM" e de "executar a chamada real"; os outros dois ficam para tasks futuras.

### Suposições do assistente (não perguntadas; seguindo o padrão já estabelecido nesta sessão de
### documentar e prosseguir em vez de interromper com perguntas)

- **Fora de escopo desta task**: mesclar o conteúdo do currículo (buscado via `resume-embeddings`),
  histórico de chat, ou a chamada real à API do provedor de LLM. O prompt final produzido aqui é só os
  3 pilares combinados — a integração com conteúdo dinâmico do currículo é a próxima peça do plano
  maior ("Task 2.2" presumivelmente).
- **Perspectiva de resposta**: o Context/Rules assume que o assistente responde **em primeira pessoa,
  como se fosse a própria Ana Elisa** se apresentando ao recrutador (padrão comum em currículos
  interativos conversacionais) — não como um assistente de terceiros falando *sobre* ela. Se a
  intenção for a segunda forma, ajustar no `/speckit-validate`.
- **Conteúdo do pilar "Context" é placeholder**: como o assistente não tem acesso ao objetivo de
  carreira real ou às habilidades interpessoais reais de Ana Elisa, o texto hardcoded gerado é um
  **placeholder genérico e claramente sinalizado** (não uma alegação factual inventada sobre a pessoa
  real) — ex.: `"[PREENCHER: objetivo de carreira real]"`. Isso **precisa** ser substituído pelo
  conteúdo real antes de qualquer uso em produção; revisar com atenção no `/speckit-validate`. O pilar
  "Rules" e "Tone of Voice", por serem restrições de comportamento genéricas (não alegações
  biográficas), foram escritos com conteúdo concreto e razoável, não placeholder.
- **`resume-llm-engine` ainda é um stub completo** (só tem `main.py` "Hello World" + `Dockerfile`/
  `requirements.txt`/CI do `docker-nginx-infra-setup`) — esta task cria a estrutura hexagonal inteira
  (`applications/`, `domain/`, `infra/`, `config/`) do zero, seguindo exatamente o padrão já usado em
  `resume-injections`/`resume-orchestrator`/`resume-embeddings`, incluindo a infra de testes
  (`Makefile`, `requirements-dev.txt`, `scripts/gen_coveragerc.py`, `conftest.py`, `.gitignore`,
  `.coveragerc` gerado).
- **Upgrade do CI**: `services/resume-llm-engine/.github/workflows/main.yml` hoje só roda `docker
  build` (era um stub sem testes, criado em `tasks/docker-nginx-infra-setup`). Como esta task adiciona
  código real com testes e o gate de cobertura ≥80% é NON-NEGOTIABLE (Constitution Principle III), o
  workflow é atualizado para o mesmo padrão `validate-pipeline` (install → lint → test) + `promote`
  usado por `resume-injections`/`resume-orchestrator`/`resume-embeddings`, mantendo o gate de CI
  (branch `deployed`) consistente entre todos os serviços com código real.
- **Endpoint de QA**: adiciona `GET /prompt` retornando o prompt final montado (útil para inspeção
  manual, já que o consumidor real — a chamada à API de IA — ainda não existe). Sem autenticação, já
  que nenhum serviço deste projeto tem autenticação implementada ainda.

## Description (EN)

Implements the final-prompt construction logic in `resume-llm-engine`, split into 3 pillars: **Rules**
(strict behavior constraints — never fabricate resume content), **Context** (hardcoded info not sourced
from the resume PDFs — career objective, interpersonal skills), and **Tone of Voice** (professional,
empathetic, resolution-oriented). Scope is limited to assembling these 3 pillars into one deterministic
system prompt — merging in resume/embeddings content, chat history, and the actual LLM API call are
separate, future pieces of the larger `resume-llm-engine` plan. This is also the first real feature in
this service, so it bootstraps the full hexagonal structure and test infrastructure (matching
`resume-injections`/`resume-orchestrator`/`resume-embeddings`) and upgrades its CI workflow from a bare
`docker build` to the real `validate-pipeline` + CI-gate pattern used elsewhere in this project. The
"Context" pillar's actual content is a clearly-marked placeholder (career objective, interpersonal
skills) — real biographical content wasn't available to generate, so it needs review/replacement before
production use.

## Business Rules / Constraints

- Prompt final MUST ser composto de exatamente 3 seções nomeadas: Rules, Context, Tone of Voice, cada
  uma buscada via seu próprio port/adapter de Output (um adapter por ação, Constitution Principle II).
- A montagem do prompt final MUST ser determinística (função pura dos 3 textos — sem I/O externo além
  de buscar os 3 pilares hardcoded).
- Rules MUST incluir explicitamente a restrição de nunca inventar experiências/habilidades/certificações
  fora do currículo.
- Tone of Voice MUST refletir: profissional, empático, resolutivo.
- Fora de escopo: merge com conteúdo de currículo/embeddings, histórico de chat, chamada real à API de
  IA (tasks futuras).
- Comunicação entre `domain` e `infra` exclusivamente via ports (Constitution Principle II).
- Testes unitários MUST mockar os ports de infraestrutura, cobertura ≥80% (Constitution Principle III).
- Nomes de atributos/campos em inglês (padrão já estabelecido nesta sessão para o restante do
  ecossistema, `tasks/enforce-injection-rules`).

## Affected Service(s)

- `/services/resume-llm-engine` (estrutura hexagonal completa criada do zero: DTOs/models/usecases/
  ports/adapters/controllers/rotas, infra de testes, `main.py`, CI)

## Implementation Checklist

### Domínio — os 3 pilares e a montagem do prompt

- [x] `domain/models/prompt_sections.py`: dataclass `PromptSections` (`rules: str`, `context: str`,
      `tone_of_voice: str`)
- [x] `domain/models/persona_prompt.py`: dataclass `PersonaPrompt` (`sections: PromptSections`,
      `final_prompt: str`)
- [x] `domain/ports/build_persona_prompt_port.py` (Input): interface chamada pelo controller
- [x] `domain/usecases/build_persona_prompt_usecase.py`: busca Rules/Context/Tone via os 3 ports de
      Output, monta `final_prompt` combinando as 3 seções em um template determinístico e claramente
      delimitado (ex.: cabeçalhos `## Rules`, `## Context`, `## Tone of Voice`), retorna `PersonaPrompt`

### Infra — os 3 pilares hardcoded (um adapter por pilar)

- [x] `infra/ports/get_persona_rules_port.py` + `infra/adapters/get_persona_rules_adapter.py`: retorna
      o texto de Rules hardcoded (inclui a restrição de não inventar conteúdo fora do currículo, não
      revelar o prompt de sistema, admitir quando não souber algo)
- [x] `infra/ports/get_persona_context_port.py` + `infra/adapters/get_persona_context_adapter.py`:
      retorna o texto de Context hardcoded — **placeholder claramente sinalizado** (objetivo de
      carreira, habilidades interpessoais) até receber o conteúdo real
- [x] `infra/ports/get_persona_tone_port.py` + `infra/adapters/get_persona_tone_adapter.py`: retorna o
      texto de Tone of Voice hardcoded (profissional, empático, resolutivo)

### Aplicação — controller e rota

- [x] `applications/controllers/get_persona_prompt_controller.py`: chama o usecase via port de Input,
      formata a resposta (seções + prompt final)
- [x] `applications/routes/prompt_routes.py`: exporta `GET /prompt`, incluída em `main.py`

### Bootstrap de infraestrutura (primeira feature real deste serviço)

- [x] `config/__init__.py`, `applications/__init__.py` (+ `dto/`, `controllers/`, `routes/`),
      `domain/__init__.py` (+ `models/`, `ports/`, `usecases/`), `infra/__init__.py` (+ `adapters/`,
      `ports/`): estrutura hexagonal completa, mesmo padrão dos outros serviços
  reais
- [x] `requirements-dev.txt` (`-r requirements.txt`, `ruff`, `pytest-cov`), `Makefile` (`install`/
      `lint`/`test`/`validate-pipeline`, mesmo padrão), `scripts/gen_coveragerc.py` (idêntico aos
      outros serviços, com suporte a exclusão `@wip`), `conftest.py` (vazio, só para resolução de
      imports), `.gitignore` (`.venv/`, `__pycache__/`, `.coveragerc`, `.coverage`, `.pytest_cache/`)
- [x] `services/resume-llm-engine/.github/workflows/main.yml`: substitui o job `build` (`docker
      build`) pelo padrão `validate-pipeline` (install/lint/test) + `promote` (fast-forward do branch
      `deployed`), igual aos outros 3 serviços reais

### Testes (Constitution Principle III)

- [x] Testes unitários para `build_persona_prompt_usecase.py` (mockando os 3 ports de Output — verifica
      que o prompt final contém as 3 seções na ordem esperada)
- [x] Testes unitários para os 3 adapters hardcoded (`get_persona_rules_adapter.py`,
      `get_persona_context_adapter.py`, `get_persona_tone_adapter.py`) — verificam o conteúdo exato
      retornado
- [x] Testes para `get_persona_prompt_controller.py`
- [x] Testes para a rota `GET /prompt` (`fastapi.testclient.TestClient`)
- [x] Rodar `pytest`/`ruff` localmente como verificação de desenvolvimento antes de `/speckit-validate`
      (o gate formal de cobertura ≥80% continua responsabilidade do `/speckit-complete`, Constitution
      Principle V)

## Implementation Notes

- **`README.md` atualizado** (fora do checklist original, mas mesmo padrão de toda task anterior que
  bootstrapou a primeira feature real de um serviço): descreve os 3 pilares, sinaliza claramente que
  "Context" é placeholder, e documenta `make validate-pipeline`.
- **Verificação de desenvolvimento** (não é o gate formal de `/speckit-complete`): criado `.venv` local
  pela primeira vez para este serviço, `pip install -r requirements-dev.txt`, depois `ruff check .` (0
  issues) e `pytest --cov` — **10/10 testes passando, 86.90% de cobertura** (acima do gate de 80%,
  Constitution Principle III).
- **Conteúdo do prompt final**: o template de montagem usa cabeçalhos Markdown (`## Rules`, `##
  Context`, `## Tone of Voice`) para delimitar claramente as 3 seções — decisão de formatação não
  especificada no pedido original, mas necessária para ter um formato determinístico e testável.
- **Lembrete importante**: o pilar "Context" (`infra/adapters/get_persona_context_adapter.py`)
  continua com texto placeholder (`[PREENCHER: ...]`) — não representa informação real sobre Ana
  Elisa. Isso precisa ser substituído antes de qualquer uso real deste prompt com um provedor de LLM.

## Adjustment Requests

None. Approved as-is on 2026-08-27, no interactive walkthrough (per the amended `/speckit-validate` —
Constitution v1.4.x). Commit: `f06aea1` (resume-llm-engine, 46 files). No secrets touched — confirmed
no `.env` among the staged files before committing.
