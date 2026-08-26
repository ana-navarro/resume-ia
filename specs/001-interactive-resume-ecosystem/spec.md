# Feature Specification: Interactive Resume Ecosystem

**Feature Branch**: `001-interactive-resume-ecosystem`

**Created**: 2026-08-26

**Status**: Draft

**Input**: User description: "Project Specification: Interactive Resume Ecosystem — portfólio vivo com apresentação tradicional, visão arquitetural (diagramas) e agente de IA (RAG sobre o currículo em PDF) para interação com recrutadores internacionais de Engenharia de Software, construído sobre arquitetura de micro-serviços em monorepo."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Recrutador visualiza a apresentação do currículo (Priority: P1)

Como recrutador, quero visualizar a apresentação interativa do currículo da Ana Elisa para entender
rapidamente sua experiência profissional, habilidades técnicas e formação em Engenharia de Software.

**Why this priority**: é o valor mínimo do produto — sem esta apresentação não existe portfólio. Deve
funcionar mesmo que as demais camadas (IA, arquitetura) estejam indisponíveis.

**Independent Test**: pode ser testado acessando apenas a aba de Apresentação, sem depender do chat de
IA nem da aba de Arquitetura, e confirmando que todas as informações profissionais estão visíveis e
legíveis.

**Acceptance Scenarios**:

1. **Given** um recrutador acessa o link do currículo pela primeira vez, **When** ele abre a aba de
   Apresentação, **Then** ele vê experiência profissional, habilidades técnicas e formação organizadas
   de forma clara e legível.
2. **Given** o assistente de IA está indisponível, **When** o recrutador acessa a aba de Apresentação,
   **Then** o conteúdo do currículo continua totalmente visível e funcional.

---

### User Story 2 - Recrutador interage com o assistente de IA (Priority: P2)

Como recrutador, quero fazer perguntas em linguagem natural sobre a experiência técnica da Ana Elisa e
receber respostas precisas e profissionais, para avaliar sua adequação a uma vaga sem precisar ler o
currículo inteiro.

**Why this priority**: é o principal diferencial do projeto, demonstrando competência prática em IA
aplicada. Depende do conteúdo da apresentação (P1) já existir como base de conhecimento.

**Independent Test**: pode ser testado enviando perguntas pelo chat e verificando que as respostas
refletem informações reais do currículo, mesmo sem a aba de Arquitetura estar disponível.

**Acceptance Scenarios**:

1. **Given** o recrutador está na aba de IA, **When** ele pergunta sobre uma tecnologia específica
   presente no currículo, **Then** o assistente responde de forma técnica e correta, referenciando
   informações reais do currículo.
2. **Given** o recrutador faz uma pergunta fora do escopo profissional, **When** o assistente processa a
   pergunta, **Then** ele responde educadamente redirecionando para o escopo do currículo, sem inventar
   informações.
3. **Given** o recrutador tenta manipular o assistente com uma instrução maliciosa (prompt injection),
   **When** a mensagem é processada, **Then** o sistema neutraliza a tentativa antes de gerar qualquer
   resposta, preservando a integridade do agente.

---

### User Story 3 - Recrutador explora a visão arquitetural (Priority: P3)

Como recrutador ou entrevistador técnico, quero visualizar diagramas da arquitetura de micro-serviços do
projeto para avaliar as competências de design de sistemas da Ana Elisa.

**Why this priority**: reforça a credibilidade técnica, mas não é essencial para o valor central do MVP
(mostrar o currículo e permitir perguntas).

**Independent Test**: pode ser testado abrindo a aba de Arquitetura isoladamente e confirmando que os
diagramas representam corretamente a topologia dos serviços do projeto.

**Acceptance Scenarios**:

1. **Given** o recrutador abre a aba de Arquitetura, **When** a página carrega, **Then** ele visualiza um
   diagrama representando os micro-serviços do sistema e o fluxo de comunicação entre eles.

---

### User Story 4 - Mantenedora atualiza o conteúdo do currículo (Priority: P4)

Como Ana Elisa (mantenedora do projeto), quero adicionar ou atualizar arquivos PDF do meu currículo e ter
o conteúdo automaticamente reprocessado, para que o assistente de IA sempre responda com base nas
informações mais recentes.

**Why this priority**: é uma capacidade operacional necessária para manter o produto atualizado ao longo
do tempo, mas não bloqueia o lançamento inicial (pode ser executada uma única vez antes do lançamento).

**Independent Test**: pode ser testado adicionando um novo PDF à pasta de origem, executando o processo
de atualização de conteúdo, e confirmando que uma pergunta subsequente ao assistente reflete o novo
conteúdo.

**Acceptance Scenarios**:

1. **Given** um novo arquivo PDF de currículo é adicionado à pasta de origem, **When** o processo de
   atualização de conteúdo é executado, **Then** as novas informações passam a estar disponíveis nas
   respostas do assistente de IA.
2. **Given** o processo de reprocessamento falha para um arquivo corrompido, **When** o erro ocorre,
   **Then** o conteúdo anteriormente indexado permanece intacto e disponível.

---

### Edge Cases

- O que acontece quando o assistente de IA não encontra informação relevante no currículo para responder
  a uma pergunta?
- Como o sistema lida com múltiplos recrutadores conversando simultaneamente com o assistente?
- O que acontece se o assistente de IA ficar indisponível ou muito lento para responder?
- Como o sistema reage a tentativas de prompt injection ou perguntas ofensivas/fora de escopo?
- O que acontece quando um PDF de currículo mal formatado ou corrompido é enviado para atualização de
  conteúdo?
- O que acontece quando o conteúdo do currículo é atualizado enquanto um recrutador está em uma conversa
  ativa?
- O que acontece se a persistência da conversa em JSON falhar (ex.: disco cheio, erro de escrita)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema MUST exibir uma apresentação interativa do currículo de Ana Elisa, incluindo
  experiência profissional, habilidades técnicas e formação, de forma legível para recrutadores.
- **FR-002**: O sistema MUST exibir uma visão arquitetural com diagramas que representem a estrutura de
  micro-serviços do próprio projeto.
- **FR-003**: O sistema MUST fornecer uma interface de chat onde recrutadores podem enviar perguntas em
  linguagem natural sobre a experiência técnica de Ana Elisa.
- **FR-004**: As respostas do assistente de IA MUST ser fundamentadas no conteúdo real do currículo
  (abordagem de recuperação aumentada), evitando informações inventadas (alucinações).
- **FR-005**: A persona do assistente de IA MUST manter tom técnico e profissional, com foco geral nas
  competências de Ana Elisa, sem direcionamento a uma empresa específica.
- **FR-006**: O sistema MUST validar e sanitizar toda entrada do usuário e saída gerada pela IA antes de
  exibi-la, prevenindo prompt injection e conteúdo inseguro ou alucinado.
- **FR-007**: O sistema MUST permitir que novos arquivos de currículo (PDF) sejam adicionados e
  reprocessados, atualizando o conhecimento disponível ao assistente de IA sem exigir reconstrução
  completa do sistema.
- **FR-008**: Uma falha ou indisponibilidade do assistente de IA MUST NOT impedir o acesso às abas de
  Apresentação e de Arquitetura.
- **FR-009**: O sistema MUST indicar visualmente ao recrutador quando uma resposta do assistente está
  sendo processada (estado de carregamento).
- **FR-010**: Quando o assistente não encontrar informação relevante para responder, o sistema MUST
  retornar uma resposta educada informando a limitação, em vez de inventar uma resposta.
- **FR-011**: O sistema MUST suportar múltiplas conversas simultâneas de diferentes recrutadores sem
  misturar o contexto entre sessões.
- **FR-012**: O sistema MUST permitir acesso público ao currículo e ao assistente de IA, sem exigir login
  ou qualquer credencial; a proteção contra abuso e tráfego não intencional é responsabilidade da camada
  de guard rails (FR-006), não de controle de acesso do usuário.
- **FR-013**: O sistema MUST persistir cada conversa entre um recrutador e o assistente de IA em formato
  JSON, associada a um identificador único de conversa, contendo as perguntas e respostas trocadas, para
  permitir que Ana Elisa revise posteriormente as interações.
- **FR-014**: O sistema MUST disponibilizar a apresentação do currículo e as respostas do assistente de
  IA em inglês e em português, permitindo que o visitante escolha o idioma de sua preferência.
- **FR-015**: Uma falha ao persistir uma conversa em JSON MUST NOT impedir a entrega da resposta ao
  recrutador; a falha de persistência deve ser tratada como um problema secundário e não visível ao
  usuário.

### Key Entities *(include if feature involves data)*

- **Documento de Currículo (PDF)**: arquivo de origem contendo o histórico profissional de Ana Elisa,
  usado como base de conhecimento do assistente.
- **Fragmento de Conhecimento**: trecho do currículo processado e indexado para busca semântica, usado
  para fundamentar as respostas do assistente.
- **Sessão de Conversa**: interação entre um recrutador e o assistente de IA, identificada por um ID
  único e persistida em formato JSON, contendo as perguntas e respostas trocadas durante a visita, para
  revisão posterior por Ana Elisa.
- **Persona do Assistente**: definição do tom, escopo e limites de resposta do agente de IA, incluindo o
  direcionamento geral (não específico a uma empresa).
- **Diagrama de Arquitetura**: representação visual da estrutura de micro-serviços do projeto, exibida na
  aba de Arquitetura.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Um recrutador consegue visualizar a apresentação completa do currículo em até 5 segundos
  após acessar o link, sem necessidade de login ou instalação de software.
- **SC-002**: Em pelo menos 90% das perguntas típicas sobre a experiência técnica de Ana Elisa, o
  assistente de IA responde em até 10 segundos.
- **SC-003**: 100% das respostas do assistente de IA revisadas em testes de validação são consistentes
  com o conteúdo real do currículo, sem informações inventadas.
- **SC-004**: Um recrutador (técnico ou não-técnico) consegue compreender a estrutura geral dos
  micro-serviços do projeto após visualizar a aba de Arquitetura por até 2 minutos.
- **SC-005**: 100% das tentativas de prompt injection ou manipulação identificadas em testes de segurança
  são neutralizadas antes de gerar uma resposta ao usuário.
- **SC-006**: Após uma atualização do currículo-fonte, o assistente de IA passa a refletir o novo
  conteúdo em suas respostas sem exigir reinício manual das abas de Apresentação ou Arquitetura.

## Assumptions

- O público-alvo (recrutadores) possui conexão de internet estável e utiliza navegadores modernos; não há
  requisito de suporte a navegadores legados.
- Suporte a dispositivos móveis é desejável, mas um layout responsivo básico é suficiente para a primeira
  versão (não é necessário um aplicativo nativo).
- Os diagramas de arquitetura são baseados na estrutura real dos serviços descrita na Constituição do
  projeto e podem ser atualizados manualmente quando a arquitetura mudar; não é exigida geração
  automática a partir do código-fonte.
- O motor de IA e o banco vetorial são executados localmente (self-hosted), portanto não há custo por
  chamada de API externa a ser controlado, mas existe um limite prático de capacidade de processamento
  simultâneo do hardware local.
- Não há garantia de suporte a um número mínimo de recrutadores simultâneos além do que a infraestrutura
  local suporta; picos extremos de tráfego estão fora do escopo desta versão.
- O acesso ao currículo e ao assistente de IA é público e não requer autenticação; a moderação de abuso
  fica a cargo da camada de guard rails, não de controle de identidade do visitante.
- As conversas armazenadas contêm apenas o texto digitado voluntariamente pelo recrutador e as respostas
  do assistente; não é coletado nenhum outro dado pessoal (nome, e-mail, IP) além do necessário para gerar
  o identificador único da conversa. Conformidade formal com regulações como GDPR/LGPD está fora do
  escopo desta versão, dado o caráter de portfólio pessoal.
- O idioma padrão exibido no primeiro acesso (antes de o visitante escolher explicitamente) pode ser
  detectado pelo navegador ou definido como um padrão fixo; a decisão de qual dos dois é deixada para a
  fase de planejamento.
