<!--
Sync Impact Report
- Version change: 1.2.0 → 1.3.0
- Rationale: Minor amendment to redesign the Speckit workflow into a four-step lifecycle (`/speckit-task`, `/speckit-implement`, `/speckit-validate`, `/speckit-complete`), introduce a `tasks/` directory for tracking, enforce file-by-file interactive validation, and explicitly mandate bilingual support (PT/EN) across the project.
- Modified principles:
  - II. Arquitetura Hexagonal e Fluxo Estrito: Added `assets/` subfolder under `config/` for temporary
    storage of resume PDFs and other files (documentation gap fixed by the validation pass following
    this amendment — not present in the original 1.3.0 sync note).
  - V. Workflow de Implementação Automatizada (Speckit): Fully rewritten to reflect the new 4-step command lifecycle.
  - VI. Suporte Bilíngue (PT/EN): Added explicit requirement for dual-language support.
  - Comandos de Desenvolvimento (Prompt Commands): Updated the command list and actions.
-->

# Constituição do Projeto Currículo Interativo

**Propósito**: Este projeto é um Currículo Interativo focado em processos seletivos para posições de Engenharia de Software, que expõe as competências técnicas de Ana Elisa. Todo código gerado, revisado ou validado neste repositório MUST seguir estritamente os princípios abaixo.

## Core Principles

### I. Fronteiras do Monorepo de Micro-Serviços

O ecossistema adota uma arquitetura de micro-serviços em um monorepo. Código MUST ser alocado respeitando estritamente estas fronteiras de diretórios:

- `/apps/resume-app`: Frontend React (TypeScript) — "frontend anêmico", focado em UI, diagramação
  e chat.
- `/services/resume-bff`: Backend for Frontend (Python) — proxy e formatador/limpador de dados
  para o frontend; NÃO possui regras de orquestração de domínio, apenas repassa requisições ao
  orquestrador e adapta as respostas para a UI.
- `/services/resume-orchestrator`: o Maestro (Python) — recebe chamadas do BFF e coordena o fluxo
  de dados entre os serviços de IA e validação (Saga pattern).
- `/services/resume-injections`: Ingestão de Dados (Python) — lê arquivos de assets (currículos),
  realiza chunking e envia ao banco vetorial.
- `/services/resume-embeddings`: Serviço Vetorial (Python) — gera embeddings locais e gerencia o
  banco de dados vetorial para buscas semânticas.
- `/services/resume-llm-engine`: Motor de Inferência (Python) — configura o LLM, aplica a persona
  e executa a chamada real para a API de IA.
- `/services/resume-guard-rails`: Serviço de Segurança (Python) — valida inputs e outputs,
  evitando prompt injections e alucinações.
- `/resume-server`: Infraestrutura e Gateway de Rede — configurações do NGINX, orquestração de
  containers (`docker-compose.yml`) e scripts.

**Rationale**: fronteiras de diretório explícitas impedem vazamento de responsabilidades entre as
camadas de IA, segurança e apresentação, mantendo cada serviço independentemente substituível.

### II. Arquitetura Hexagonal e Fluxo Estrito

O fluxo de chamadas MUST seguir exatamente a direção: Frontend → `bff` → `orchestrator` → (`guard-rails`, `embeddings`, `llm-engine`). Saltos diretos entre camadas não adjacentes são proibidos.

Todos os serviços Python MUST respeitar os princípios SOLID e seguir estritamente a estrutura de pastas e regras da Arquitetura Hexagonal (Ports & Adapters) abaixo:

**Estrutura de Pastas Padrão (Backend)**:
- `applications/`: Camada de entrada/saída.
  - `controllers/`: Recebem as requisições.
  - `routes/`: Definição dos endpoints.
  - `middlewares/`: Guards, annotations e interceptadores.
  - `dto/`: Interfaces Pydantic para tipagem e validação de dados de entrada/saída.
- `domain/`: Coração da aplicação.
  - `usecases/`: Contém TODAS as regras de negócio. Nunca mexem com schemas, apenas models.
  - `models/`: Entidades de domínio chamadas pelos usecases.
  - `ports/`: Interfaces de Input/Output (os controllers chamam os ports de Input para acessar os usecases).
- `infra/`: Detalhes técnicos e dependências externas.
  - `adapters/`: Chamadas para banco de dados e manipulação de dados. **NÃO podem conter regras de negócio**. Deve existir **um adapter para cada verbo/ação** (ex: get, delete).
  - `schemas/`: Estruturas de dados para infraestrutura (ex: geração de dados para ChromaDB, ORM).
  - `ports/`: Interfaces de Output (os usecases chamam estes ports para acessar os adapters).
- `config/`: Configurações gerais (conexões de banco de dados, variáveis de ambiente, etc.).
  - `assets/`: Armazenamento temporário de dados como pdfs para currículos ou outros arquivos.
- `main.py`: Ponto de entrada (root) do serviço.

**Regras de Comunicação e Responsabilidade**:
1. **Estrutura de Ports**: Todos os ports MUST ter estrutura clara de Input e Output.
2. **Isolamento via Ports**: 
   - A camada de `domain` comunica com a de `infra` APENAS via ports (Usecases nunca chamam Adapters diretamente, apenas seus ports).
   - A camada de `applications` comunica com a de `domain` APENAS via ports (Controllers nunca chamam Usecases diretamente, apenas seus ports).
3. **Responsabilidade**: Usecases = Exclusivos para regras de negócio; Adapters = Exclusivos para manipulação/persistência de dados e integrações.

**Convenção de Nomenclatura**:
- **Backend (Arquivos e Módulos)**: `[verbo]-[função em inglês]-[tipo]`. Exemplo: `get-personal-info-adapter`, `delete-personal-info-usecase`.
- **Frontend**: Componentes MUST usar `PascalCase`. A arquitetura do front MUST sempre, SEMPRE, priorizar a componentização e a reutilização.

**Rationale**: isolar domínio de infraestrutura através de injeção de dependência via ports estritos permite trocar provedores e focar em testabilidade unitária. Ter um adapter por verbo respeita o Princípio da Responsabilidade Única (SRP).

### III. Test-First e Qualidade (NON-NEGOTIABLE)

- Backend (Python): `pytest` MUST ser usado para testes unitários; `pytest-bdd` (Gherkin/Cucumber) MUST ser usado para testes de componente/comportamento.
- Frontend (React): `Jest` MUST ser usado para testes unitários; `Cypress` MUST ser usado para testes de componente/E2E.
- A abrangência dos testes unitários MUST envolver todos os arquivos, garantindo testes completos para a rota (como um todo), controllers, adapters e use cases.
- Testes unitários de backend MUST mockar completamente as Portas (interfaces) de infraestrutura, focando exclusivamente na camada de Domínio.
- A cobertura de testes unitários MUST ser de no mínimo 80%.
- **Work In Progress (WIP)**: Qualquer arquivo marcado com a anotação/comentário `@wip` MUST ter seus testes unitários ignorados (skipped) na execução e MUST ser totalmente desconsiderado do cálculo da métrica de cobertura de código.

**Rationale**: fixar a estratégia de testes e garantir que todas as camadas do hexágono sejam validadas evita regressões. O uso de `@wip` permite iteração contínua sem quebrar os portões de qualidade da CI durante o desenvolvimento local.

### IV. Infraestrutura Reproduzível com Hot-Reloading

A infraestrutura em `/resume-server` MUST utilizar volumes Docker para permitir modificações em tempo real ("real-time modifications") durante o desenvolvimento, sem exigir rebuild de imagens a cada alteração de código.

**Rationale**: hot-reloading acelera o ciclo de feedback ao validar mudanças em um ecossistema com múltiplos micro-serviços independentes.

### V. Workflow de Implementação Automatizada (Ciclo Speckit)

A implementação de novas funcionalidades MUST seguir o ciclo de 4 etapas estruturado abaixo para garantir rastreabilidade, validação granular e estabilidade:

1. **Planejamento (`/speckit-task`)**: O assistente processa a requisição do usuário, cria o contexto da tarefa em uma subpasta dentro do diretório `/tasks` e levanta requisitos.
2. **Execução (`/speckit-implement`)**: O assistente gera estritamente o código com base no checklist do planejamento sem realizar commits ou rodar pipelines.
3. **Revisão (`/speckit-validate`)**: O assistente apresenta o código de forma iterativa e interativa (arquivo por arquivo) até a aprovação e commit por parte do usuário.
4. **Finalização (`/speckit-complete`)**: O assistente aciona a automação de CI local e realiza o envio do código.

**Rationale**: separar planejamento, codificação, validação de usuário e push automático evita commits prematuros, quebra de build inesperada e garante que a arquitetura hexagonal seja rigorosamente seguida em cada arquivo modificado.

### VI. Suporte Bilíngue (PT/EN)

O projeto MUST conter suporte nativo para os idiomas Português e Inglês.
- O Frontend e mensagens de erro do Backend devem prever internacionalização (i18n).
- A documentação das tarefas, os checklists dentro da pasta `/tasks`, e descrições do sistema geradas pelo assistente devem prever contexto claro que acomode equipes e processos de avaliação bilíngues.

---

## Comandos de Desenvolvimento (Prompt Commands)

Quando o usuário acionar os comandos abaixo, o assistente MUST atuar da seguinte forma:

### Comandos de Workflow Speckit
- `/speckit-task [descrição]`:
  - Se a descrição for insuficiente, o assistente MUST solicitar mais informações (ex: razão da task, regras de negócio).
  - Cria uma pasta dentro do diretório `tasks/` com um nome descritivo (ex: `tasks/get-embeddings/`).
  - Cria um arquivo de planejamento dentro dessa pasta contendo um **checklist de implementação** detalhado (ex: `[ ] - adapter e port de get embeddings`, `[ ] - usecase e port`, `[ ] - controller`, `[ ] - exportar rotas`).
- `/speckit-implement`:
  - Lê a task ativa e seu checklist gerado.
  - Implementa as mudanças de código solicitadas nos diretórios apropriados.
  - **Atenção**: Esta etapa NÃO realiza commit, NÃO faz push e NÃO roda a pipeline. Apenas altera/cria os arquivos localmente.
- `/speckit-validate`:
  - Mostra ao usuário o checklist da task, marcando o que foi concluído.
  - O assistente exibe o código gerado/modificado **um arquivo por vez**. O usuário precisa responder ("next" ou "próximo") para o assistente exibir o próximo arquivo.
  - Ao final do checklist, caso algo não tenha sido implementado, o assistente exibe a justificativa da omissão.
  - Pergunta ao usuário se ele deseja aprovar e continuar.
    - Se **SIM**: O assistente gera a mensagem de commit (em inglês, padronizada) e comita as alterações.
    - Se **NÃO**: O assistente pergunta o que o usuário deseja alterar (texto livre) e reinicia o processo de ajuste.
- `/speckit-complete`:
  - Roda a simulação da pipeline localmente (`make validate-pipeline`), checando lint, testes unitários e se a meta de cobertura (80%) foi atingida.
  - Se aprovado, realiza o *push* nos repositórios relevantes.

### Comandos de Utilitários
- `/test-unit`: gerar testes unitários cobrindo cenários de sucesso e falha (respeitando a regra de pular itens com `@wip`).
- `/test-component`: gerar arquivo `.feature` e a implementação dos "steps".
- `/update-repos`: sugerir comandos para atualizar dependências e sincronizar branches de todos os projetos.
- `/run-server`: ajustar os scripts em `/resume-server` para subir os containers com hot-reloading ativo.
- `/run-pipeline`: executar a validação da pipeline de forma isolada localmente (lint, testes, **validação da meta de cobertura de testes unitários** e build) ANTES do usuário confirmar alterações.

---

## Governance

- Esta Constituição prevalece sobre qualquer outra prática, convenção ou preferência estilística adotada no projeto.
- Emendas requerem: (1) registro da mudança proposta e sua justificativa; (2) atualização do número de versão conforme a política de versionamento semântico abaixo; (3) atualização da data de "Last Amended".
- Versionamento semântico dos princípios:
  - MAJOR: remoção ou redefinição incompatível de um princípio existente.
  - MINOR: adição de um novo princípio ou seção, ou expansão material de uma diretriz existente.
  - PATCH: esclarecimentos, correções de redação ou ajustes não semânticos.
- Toda revisão de código ou geração automatizada MUST verificar conformidade com os princípios acima antes de considerar uma tarefa concluída.
- Complexidade que viole a Arquitetura Hexagonal (Princípio II) ou as fronteiras de serviço (Princípio I) MUST ser explicitamente justificada no plano de implementação ou rejeitada.

**Version**: 1.3.0 | **Ratified**: TODO(RATIFICATION_DATE): data original de adoção não registrada (projeto sem histórico git) | **Last Amended**: 2026-08-26