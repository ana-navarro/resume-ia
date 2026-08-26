<!--
Sync Impact Report
- Version change: 1.1.0 → 1.2.0
- Rationale: Minor amendment to expand Principle II with strict folder structures, explicit Hexagonal boundaries (Input/Output ports), responsibility segregation, and naming conventions for both Backend and Frontend.
- Modified principles:
  - II. Arquitetura Hexagonal e Fluxo Estrito: Added detailed folder structure (applications, domain, infra, config), strict port-based communication rules, single-verb adapter rule, and naming conventions (kebab-case for backend, PascalCase for frontend).
-->

# Constituição do Projeto Currículo Interativo

**Propósito**: Este projeto é um Currículo Interativo focado em processos seletivos para posições
de Engenharia de Software no polo tecnológico de Helsinque (especialmente no distrito de Kamppi),
expondo as competências técnicas de Ana Elisa. Todo código gerado, revisado ou validado neste
repositório MUST seguir estritamente os princípios abaixo.

## Core Principles

### I. Fronteiras do Monorepo de Micro-Serviços

O ecossistema adota uma arquitetura de micro-serviços em um monorepo. Código MUST ser alocado
respeitando estritamente estas fronteiras de diretórios:

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

- Backend (Python): `pytest` MUST ser usado para testes unitários; `pytest-bdd`
  (Gherkin/Cucumber) MUST ser usado para testes de componente/comportamento.
- Frontend (React): `Jest` MUST ser usado para testes unitários; `Cypress` MUST ser usado para
  testes de componente/E2E.
- A abrangência dos testes unitários MUST envolver todos os arquivos, garantindo testes completos para a rota (como um todo), controllers, adapters e use cases.
- Testes unitários de backend MUST mockar completamente as Portas (interfaces) de infraestrutura,
  focando exclusivamente na camada de Domínio.
- A cobertura de testes unitários MUST ser de no mínimo 80%.
- **Work In Progress (WIP)**: Qualquer arquivo marcado com a anotação/comentário `@wip` MUST ter seus testes unitários ignorados (skipped) na execução e MUST ser totalmente desconsiderado do cálculo da métrica de cobertura de código.

**Rationale**: fixar a estratégia de testes e garantir que todas as camadas do hexágono sejam validadas evita regressões. O uso de `@wip` permite iteração contínua sem quebrar os portões de qualidade da CI durante o desenvolvimento local.

### IV. Infraestrutura Reproduzível com Hot-Reloading

A infraestrutura em `/resume-server` MUST utilizar volumes Docker para permitir modificações em
tempo real ("real-time modifications") durante o desenvolvimento, sem exigir rebuild de imagens a
cada alteração de código.

**Rationale**: hot-reloading acelera o ciclo de feedback ao validar mudanças em um ecossistema
com múltiplos micro-serviços independentes.

### V. Workflow de Implementação Automatizada

Quando a ferramenta `speckit-implement` for acionada, o assistente MUST assumir a execução da tarefa e seguir estritamente estas etapas, nesta ordem:

1. **Description**: descrever detalhadamente o plano de ação e a arquitetura da solução proposta.
2. **Realization**: gerar e aplicar o código nos diretórios corretos, respeitando a topologia
   (Princípio I) e a Arquitetura Hexagonal (Princípio II).
3. **Validation**: pausar e solicitar que o usuário teste localmente. Se o usuário relatar falha,
   o assistente MUST perguntar detalhes e corrigir o código antes de prosseguir.
4. **Pipeline**: executar automaticamente o comando de simulação da pipeline
   (`make validate-pipeline`). Esta validação MUST checar a execução dos testes e validar se a cobertura de testes unitários atingiu o mínimo de 80% (ignorando os itens `@wip`). Se a pipeline falhar, o assistente MUST tentar corrigir o código autonomamente e apenas INFORMAR o usuário do que foi ajustado.
5. **Commit**: se a validação manual e a pipeline forem bem-sucedidas, o assistente MUST gerar a
   mensagem de commit em inglês padronizada e realizar o commit das alterações.

**Rationale**: um ciclo determinístico de implementação evita commits prematuros ou não
validados, mesmo quando o trabalho é delegado à IA.

## Comandos de Desenvolvimento (Prompt Commands)

Quando o usuário acionar os comandos abaixo, o assistente MUST atuar da seguinte forma:

- `/test-unit`: gerar testes unitários cobrindo cenários de sucesso e falha (respeitando a regra de pular itens com `@wip`).
- `/test-component`: gerar arquivo `.feature` e a implementação dos "steps".
- `/update-repos`: sugerir comandos para atualizar dependências e sincronizar branches de todos
  os projetos.
- `/run-server`: ajustar os scripts em `/resume-server` para subir os containers com
  hot-reloading ativo.
- `/run-pipeline`: executar a validação da pipeline de forma isolada localmente (lint, testes, **validação da meta de cobertura de testes unitários** e build) ANTES do usuário confirmar alterações.

## Governance

- Esta Constituição prevalece sobre qualquer outra prática, convenção ou preferência estilística
  adotada no projeto.
- Emendas requerem: (1) registro da mudança proposta e sua justificativa; (2) atualização do
  número de versão conforme a política de versionamento semântico abaixo; (3) atualização da
  data de "Last Amended".
- Versionamento semântico dos princípios:
  - MAJOR: remoção ou redefinição incompatível de um princípio existente.
  - MINOR: adição de um novo princípio ou seção, ou expansão material de uma diretriz existente.
  - PATCH: esclarecimentos, correções de redação ou ajustes não semânticos.
- Toda revisão de código ou geração automatizada (incluindo o uso do `speckit-implement`) MUST verificar
  conformidade com os princípios acima antes de considerar uma tarefa concluída.
- Complexidade que viole a Arquitetura Hexagonal (Princípio II) ou as fronteiras de serviço
  (Princípio I) MUST ser explicitamente justificada no plano de implementação ou rejeitada.

**Version**: 1.2.0 | **Ratified**: TODO(RATIFICATION_DATE): data original de adoção não
registrada (projeto sem histórico git) | **Last Amended**: 2026-08-26