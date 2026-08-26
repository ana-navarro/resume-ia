# Task: Docker Compose + NGINX com Hot-Reload e Auto-Update

**Status**: Validated - Committed
**Created**: 2026-08-26

## Description (PT)

`resume-server/docker-compose.yml` e `resume-server/nginx/nginx.conf` existem mas estão vazios (o
`README.md` do serviço já documenta isso como pendência). Esta task implementa os três pedidos do
usuário:

1. **Configuração Docker + NGINX dentro de `resume-server`**: um `docker-compose.yml` que sobe todos os
   7 serviços do ecossistema (`resume-app`, `resume-bff`, `resume-orchestrator`, `resume-injections`,
   `resume-embeddings`, `resume-llm-engine`, `resume-guard-rails`) e um `nginx.conf` que faz o roteamento
   reverso para eles.
2. **Hot-reloading obrigatório**: qualquer alteração de código local deve refletir no container já
   rodando, sem rebuild — via bind mount do código-fonte de cada serviço para dentro do container, mais
   `uvicorn --reload` (Python) e `vite --host` com HMR (frontend). Isso já é exigido pela Constitution
   Principle IV; esta task é a primeira a de fato implementá-lo.
3. **Atualização automática quando o repositório atualiza no GitHub**: cada um dos 7 serviços é um
   repositório GitHub **independente** dentro deste diretório de trabalho (confirmado via
   `git remote -v` em cada um — não são git submodules, são checkouts separados lado a lado), mais o
   repositório externo `resume-ia` que contém `apps/resume-app`, `resume-server`, `tasks/` etc. Não existe
   nenhum servidor remoto de deploy descrito neste projeto (a Constitution trata `resume-server`
   explicitamente como ambiente de desenvolvimento local, não produção) — então "atualizar
   automaticamente" só pode significar manter os checkouts locais sincronizados com o GitHub, para que o
   hot-reload já existente (item 2) pegue o novo código automaticamente assim que ele chegar no disco.

### Suposições do assistente (não perguntadas; seguindo o padrão já estabelecido nesta sessão de
### documentar e prosseguir em vez de interromper com perguntas)

- **Mecanismo de auto-update**: um serviço adicional no `docker-compose.yml` (`auto-updater`, imagem
  `alpine/git`) que roda um script (`resume-server/scripts/auto-update.sh`) em loop, fazendo
  `git fetch` + `git pull --ff-only` periodicamente em cada um dos 7 checkouts (bind-mount da raiz do
  monorepo inteira). Só faz fast-forward — nunca sobrescreve alterações locais não commitadas. Uma
  GitHub Action **não pode**, por si só, alcançar a máquina local do desenvolvedor para forçar um
  `git pull` nela; por isso ela não substitui esse script, apenas complementa (ver próximo item).
- **"Se necessário, crie uma action do GitHub"**: interpretado como CI (lint + testes + gate de
  cobertura) rodando automaticamente a cada push, usando a automação de pipeline (`make
  validate-pipeline`) já existente em `resume-injections`, `resume-orchestrator` e `resume-embeddings`
  (únicos com `Makefile` + suíte de testes reais até agora). Cada workflow vai para
  `.github/workflows/ci.yml` **dentro do respectivo repositório** (`services/<nome>/.github/...`), já
  que cada um é um repo GitHub separado. `resume-bff`, `resume-guard-rails` e `resume-llm-engine` ainda
  são stubs sem `Makefile`/testes — ficam **fora do escopo de CI** por enquanto (nada a validar); ainda
  assim ganham Dockerfile porque precisam rodar no `docker-compose` (pedido explícito do usuário: "não
  um service por vez"). O frontend (`apps/resume-app`) faz parte do repo externo `resume-ia` — seu CI
  (lint + build) vai em `.github/workflows/frontend-ci.yml` na raiz do `resume-ia`.
- **Portas**: todo serviço Python expõe `8000` dentro do container; mapeamento de host único por
  serviço (`8001`–`8006`) só para debug direto, sem depender do NGINX. `resume-app` usa `5173` (padrão
  Vite). NGINX escuta `80`.
- **Roteamento NGINX**: `/` → `resume-app` (com suporte a WebSocket para o HMR do Vite); `/bff/` →
  `resume-bff`; `/orchestrator/` → `resume-orchestrator` (hospeda os endpoints públicos de upload/
  auditoria de `features/injection/rules.md`). `resume-injections`, `resume-embeddings`,
  `resume-llm-engine` e `resume-guard-rails` **não** são expostos via NGINX — só acessíveis dentro da
  rede Docker por outros serviços, conforme o fluxo estrito de chamadas da Constitution Principle II
  (Frontend→bff→orchestrator→demais serviços).
- **`resume-bff`/`resume-guard-rails`/`resume-llm-engine` não têm `requirements.txt`** (só um
  `main.py` "Hello World" cada) — esta task adiciona um mínimo (`fastapi`, `uvicorn[standard]`) só para
  o Dockerfile funcionar; não implementa nenhuma lógica de negócio desses serviços (fora de escopo).
- **Variáveis de ambiente/segredos**: `resume-injections`, `resume-orchestrator` e `resume-embeddings`
  continuam lendo seu próprio `.env` local (gitignorado) via `env_file` no `docker-compose.yml` — nenhum
  segredo novo é introduzido ou centralizado.

## Description (EN)

Implements Docker + NGINX infra for the whole 7-service ecosystem (`docker-compose.yml` +
`nginx/nginx.conf` in `resume-server`, currently empty placeholders), with mandatory hot-reload (bind
mounts + `uvicorn --reload` / Vite HMR, per Constitution Principle IV) so a developer runs the full
system via `docker compose up` instead of one service at a time. Also adds a local auto-update
mechanism (a small `auto-updater` container that periodically fast-forward-pulls each of the 7
independent GitHub repos checked out side by side in this working tree) so the already-running,
hot-reloading containers pick up new commits pushed to GitHub without manual intervention — no remote
deploy target exists in this project, so a GitHub Action alone cannot reach the developer's machine;
CI (lint/test/coverage gate) workflows are added instead where pipeline automation already exists
(`resume-injections`, `resume-orchestrator`, `resume-embeddings`, plus a lint/build check for the
frontend).

## Business Rules / Constraints

- Hot-reload MUST work via bind mount + in-process file-watcher (`uvicorn --reload`, Vite HMR) — no
  rebuild required for a code change to take effect (Constitution Principle IV).
- `docker compose up` from `resume-server/` MUST start all 7 services plus NGINX plus the auto-updater
  in one command (explicit user requirement: "não um service por vez").
- The auto-updater MUST only fast-forward-pull (`git pull --ff-only`); it MUST NOT force-push, reset, or
  discard uncommitted local changes in any of the 7 checkouts.
- NGINX MUST only publicly route to `resume-app`, `resume-bff`, and `resume-orchestrator` — the other
  4 services stay internal-only, per the Constitution's strict Frontend→bff→orchestrator→others flow
  (Principle II).
- Secrets (`.env` files) are never baked into images or committed — passed via `env_file` from the
  already-existing local `.env` per service.
- New CI workflows only added where a `Makefile`/test suite already exists (`resume-injections`,
  `resume-orchestrator`, `resume-embeddings`); no fabricated tests for the stub services.
- **Merge-to-`main` MUST gate the local environment**: `auto-update.sh` MUST NOT pull a commit into a
  gated repo (one with a CI pipeline) until that exact commit has passed CI (unit tests + coverage ≥80%,
  Constitution Principle III) — implemented via a bot-managed `deployed` branch that CI only advances on
  a successful `push`-to-`main` run; the auto-updater tracks `deployed`, not `main`, for those repos.

## Affected Service(s)

- `/resume-server` (primary: `docker-compose.yml`, `nginx/nginx.conf`, `scripts/auto-update.sh`)
- `/services/resume-bff`, `/services/resume-guard-rails`, `/services/resume-llm-engine` (new
  `Dockerfile`, `.dockerignore`, minimal `requirements.txt`)
- `/services/resume-injections`, `/services/resume-orchestrator`, `/services/resume-embeddings` (new
  `Dockerfile`, `.dockerignore`, `.github/workflows/ci.yml` with `validate-pipeline` + `promote` jobs)
- `/apps/resume-app` (new `Dockerfile`, `.dockerignore`, `.github/workflows/ci.yml`; `vite.config.ts`
  updated for Docker-friendly HMR) — this repo, not `resume-ia`, is where its CI lives
- `resume-ia` root (task tracking only; the frontend CI workflow that was mistakenly placed here was
  removed)

## Implementation Checklist

### Dockerfiles por serviço

- [x] `services/resume-injections/Dockerfile`: `python:3.12-slim`, instala `requirements.txt`, `CMD
      uvicorn main:app --host 0.0.0.0 --port 8000 --reload`
- [x] `services/resume-orchestrator/Dockerfile`: idêntico padrão
- [x] `services/resume-embeddings/Dockerfile`: idêntico padrão
- [x] `services/resume-bff/requirements.txt` (novo, mínimo: `fastapi`, `uvicorn[standard]`) +
      `Dockerfile` (mesmo padrão)
- [x] `services/resume-guard-rails/requirements.txt` (novo) + `Dockerfile`
- [x] `services/resume-llm-engine/requirements.txt` (novo) + `Dockerfile`
- [x] `.dockerignore` em cada um dos 6 serviços Python (`.venv/`, `__pycache__/`, `.git/`,
      `.pytest_cache/`, `.env`) — evita copiar lixo local para a imagem no build (o bind mount de
      hot-reload ainda vai expor `.venv/` dentro do container em runtime; inofensivo, só não entra na
      imagem)
- [x] `apps/resume-app/Dockerfile`: `node:22-slim`, `npm install`, `CMD npm run dev -- --host 0.0.0.0`
- [x] `apps/resume-app/.dockerignore` (`node_modules/`, `dist/`, `.git/`)
- [x] `apps/resume-app/vite.config.ts`: adiciona `server: { host: true, watch: { usePolling: true } }`
      (polling é necessário para o file-watcher funcionar de forma confiável em bind mounts do Docker
      Desktop no Windows/Mac)

### docker-compose.yml

- [x] `resume-server/docker-compose.yml`: define os 7 serviços + `nginx` + `auto-updater`, cada
      serviço Python com bind mount do seu diretório (`../services/<nome>:/app`), `resume-app` com bind
      mount + volume anônimo em `/app/node_modules` (evita o bind mount sobrescrever os módulos
      instalados no container)
- [x] `env_file` apontando para o `.env` local de `resume-injections`, `resume-orchestrator` e
      `resume-embeddings` (os três que já leem segredos via `.env`)
- [x] Mapeamento de portas: `resume-app:5173`, `resume-bff:8001→8000`,
      `resume-orchestrator:8002→8000`, `resume-injections:8003→8000`, `resume-embeddings:8004→8000`,
      `resume-llm-engine:8005→8000`, `resume-guard-rails:8006→8000`, `nginx:80`
- [x] Serviço `nginx`: monta `./nginx/nginx.conf` read-only, `depends_on` todos os outros

### NGINX

- [x] `resume-server/nginx/nginx.conf`: `/` → `resume-app` (com upgrade de conexão para WebSocket/HMR
      do Vite), `/bff/` → `resume-bff`, `/orchestrator/` → `resume-orchestrator`; demais serviços não
      expostos publicamente (Constitution Principle II)

### Auto-update local

- [x] `resume-server/scripts/auto-update.sh`: loop (`sleep 15`) fazendo `git fetch` +
      `git pull --ff-only` em cada um dos 7 checkouts (6 serviços + a raiz `resume-ia`), logando
      sucesso/pulo sem nunca descartar alterações locais
- [x] Serviço `auto-updater` no `docker-compose.yml`: imagem `alpine/git`, bind mount da raiz inteira do
      monorepo, roda o script acima

### GitHub Actions (CI)

- [x] `services/resume-injections/.github/workflows/ci.yml`: `make validate-pipeline` em push/PR para
      `main`
- [x] `services/resume-orchestrator/.github/workflows/ci.yml`: idêntico
- [x] `services/resume-embeddings/.github/workflows/ci.yml`: idêntico
- [x] `.github/workflows/frontend-ci.yml` (raiz de `resume-ia`): `npm ci && npm run lint && npm run
      build` em `apps/resume-app`, em push/PR para `main`. **Corrigido abaixo**: esse arquivo estava no
      repositório errado (`resume-ia` nunca vê pushes do repo `resume-app`) — removido e recriado em
      `apps/resume-app/.github/workflows/ci.yml`

### CI gate (`deployed` branch) — reaberto em 2026-08-26

- [x] Corrigir localização: `.github/workflows/frontend-ci.yml` sai de `resume-ia` (raiz) e vira
      `apps/resume-app/.github/workflows/ci.yml` (repo próprio, mesmo padrão dos outros)
- [x] `services/resume-injections/.github/workflows/ci.yml`,
      `services/resume-orchestrator/.github/workflows/ci.yml`,
      `services/resume-embeddings/.github/workflows/ci.yml`,
      `apps/resume-app/.github/workflows/ci.yml`: adicionar um segundo job `promote`, que só roda em
      `push` para `main` (nunca em PR) e só depois (`needs:`) do job de pipeline ter passado — avança um
      branch `deployed` (`git push origin HEAD:refs/heads/deployed --force`) para o commit que acabou de
      passar em todos os testes + gate de cobertura. `permissions: contents: write` explícito
- [x] `resume-server/scripts/auto-update.sh`: reescrito para diferenciar repositórios "gated" (
      `resume-injections`, `resume-orchestrator`, `resume-embeddings`, `resume-app` — os 4 com pipeline
      de CI) dos demais (`resume-ia`, `resume-server`, `resume-bff`, `resume-guard-rails`,
      `resume-llm-engine` — sem pipeline ainda). Os "gated" fazem fetch do branch `deployed` (não de
      `main`) e só avançam (`merge --ff-only`) até onde o `deployed` apontar — nunca pulam para um commit
      de `main` que não passou pela pipeline. Os demais continuam com o comportamento antigo (pull direto
      do branch atual)
- [x] Tratar o caso do `deployed` ainda não existir (primeira execução da CI): logar e pular sem erro

### Documentação

- [x] `resume-server/README.md`: atualizar "Status atual" (não está mais vazio) e o passo "Como subir o
      ambiente" com o comando real (`docker compose up --build`) e a lista de portas/rotas
- [x] `resume-server/README.md`: documentar o gate de CI (`deployed` branch) e por que
      `resume-bff`/`resume-guard-rails`/`resume-llm-engine` ainda não têm esse gate (sem `Makefile`/testes
      ainda)

## Implementation Notes

- **Correção de contagem de repositórios**: a Description original contou 7 repositórios GitHub
  independentes (6 serviços + `resume-ia`), mas `resume-server` **também** é seu próprio repositório
  GitHub separado (`git -C resume-server remote -v` confirma `origin` próprio) — descoberto só durante a
  implementação. São **8 repositórios independentes** no total. `resume-server/scripts/auto-update.sh`
  foi ajustado para incluir `/workspace/resume-server` na lista `REPOS`, e o `README.md` atualizado já
  reflete os 8.
- **`.gitattributes` adicionado (fora do checklist original)**: `*.sh text eol=lf` em
  `resume-server/.gitattributes` (e também na raiz de `resume-ia`, por segurança). Necessário porque o
  Git deste ambiente Windows normaliza arquivos de texto para CRLF no checkout (confirmado pelos avisos
  "LF will be replaced by CRLF" ao commitar em tasks anteriores); sem isso, `auto-update.sh` seria
  checked out com `\r` em cada linha e quebraria ao rodar dentro do container Linux (`alpine/git`, shell
  `ash`), que não tolera `\r` embutido.
- **Validação possível sem Docker instalado nesta máquina**: `docker`/`docker compose` não estão
  disponíveis neste ambiente de desenvolvimento, então não foi possível rodar `docker compose config`
  ou `docker compose up` de fato. Validado o que deu para validar sem Docker: sintaxe YAML de
  `docker-compose.yml` e dos 4 workflows (`python -c "import yaml; yaml.safe_load(...)"`, todos OK) e
  sintaxe do shell script (`sh -n auto-update.sh`, OK). **Recomendo rodar `docker compose up --build`
  manualmente antes de considerar isso pronto para uso real** — não posso confirmar que os containers
  sobem/comunicam corretamente sem essa verificação real.
- **`resume-server/README.md`**: reescrito para descrever o estado real implementado, incluindo a lista
  de portas e o pré-requisito de `.env` local nos 3 serviços que precisam de segredos.
- **2026-08-26, rodada de correção do gate de CI**: `promote` como job separado (não um step a mais no
  job existente) para que `needs:`/`if:` deixem explícito que só roda **depois** do pipeline passar e
  **só** em push direto a `main` — em uma PR, `github.ref` é o merge ref sintético
  (`refs/pull/N/merge`), não `refs/heads/main`, então o `if:` já exclui PRs sem precisar de lógica extra.
  `git push origin HEAD:refs/heads/deployed --force` é o único ponto que usa `--force` neste projeto —
  aceitável aqui porque `deployed` é um ponteiro gerenciado só pelo bot da CI (nunca por commit humano
  direto), então "force" nele é reposicionar um marcador, não reescrever histórico de verdade.
  **Não testado fim a fim** (exigiria push real para o GitHub e observar a Action rodar, fora do alcance
  desta sessão) — sintaxe YAML validada, lógica revisada, mas recomendo observar a primeira execução
  real antes de confiar cegamente no gate.

## Adjustment Requests

None. Approved as-is on 2026-08-26 without the file-by-file walkthrough (user: "no need approvals only
make the commit"). Commits: `c3f85a2` + `6d07186` (resume-server), `a8f7ecc` (resume-bff), `879ca2a`
(resume-guard-rails), `232b4fa` (resume-llm-engine), `2202a96` (resume-injections), `7850395`
(resume-orchestrator), `9513d07` (resume-embeddings), `622674a` (resume-app).

**Correction made during validation**: `git ls-files apps/resume-app` from the outer `resume-ia` repo
returned nothing, which I initially (and incorrectly) read as "the frontend was never committed." In
fact `apps/resume-app` — like every directory under `/services/*` — is its own independent Git repo
(`github.com/ana-navarro/resume-app`, with prior history), which is exactly why the outer repo can't see
into it. This was the **9th** independent repo in the ecosystem, missed in the original count of 8
(`resume-ia` + `resume-server` + 6 services). Fixed before committing: `auto-update.sh`'s `REPOS` list
and `resume-server/README.md` now include `resume-app` (commit `6d07186`, on top of `c3f85a2`), and this
task's 3 frontend files (`Dockerfile`, `.dockerignore`, `vite.config.ts`) were committed directly inside
the `resume-app` repo, not the outer one.

**2026-08-26, before `/speckit-complete`** (user, in chat, not via `/speckit-validate`): "Antes de
completar dentro do resume-server valide que toda vez que o usuário mergea algo na main deve rodar uma
pipeline (testes unitários, testes de componentes, cobertura de testes unitários >=80%) antes de
publicar no server. Fazendo isso deve rolar o auto-update para o ambiente." Translation: on every merge
to `main`, CI (unit tests, component tests, coverage ≥80%) MUST run and pass **before** the change is
published to the local dev environment; only then should the auto-updater pick it up. This is a real gap
in what was just built: `auto-update.sh` currently fast-forward-pulls `origin/<branch>` directly, with no
CI gate at all — it would happily pull a commit that fails tests or was never even pushed through CI (a
direct push to `main`, or a merge on a repo with branch protection that doesn't require the check).
Also caught and fixed in the same pass: `frontend-ci.yml` was created in the **wrong repo**
(`resume-ia`'s `.github/workflows/`) — since `apps/resume-app` is its own separate GitHub repo (see the
9-repo correction above), pushes to it would never actually trigger that workflow. See the new
"CI gate (deployed branch)" section of the checklist below for the fix, reopened before `/speckit-complete`.

**2026-08-26, re-approved after the CI-gate fix**: approved as-is, again without the file-by-file
walkthrough ("no need approvals only make the commit"). Commits: `71eb92a` (resume-server, gate
auto-update.sh on `deployed`), `c0e37da` (resume-app, new CI workflow), `0ce7880`
(resume-injections), `7d88327` (resume-orchestrator), `514693d` (resume-embeddings) — all add the
`promote` job. Outer `resume-ia` repo: removed the misplaced `.github/workflows/frontend-ci.yml`, no
new code there beyond task tracking. Not committed anywhere: no secrets touched, confirmed via
`git status` in every repo before staging.
