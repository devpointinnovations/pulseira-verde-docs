# Migração para repositórios por área — devpointinnovations (29/10/2026)

**Status:** histórico migrado e sincronizado em 16/11/2026; ajustes de CI/CD em PR em cada repositório; passos de conta do PO pendentes (§5). O monorepo continua sendo o repositório ativo até o PO decidir a virada (§6).
**Base:** `GabrielVogado/saude-monitor`. Cópia inicial em 29/10 (`develop` em `3bf45be`, tags `v1.0.0-rc.1` a `rc.10`); sincronização em 16/11 (`develop` em `666f68f`, `master` em `ea7e8b0`, tags até `v1.0.0-rc.16`).
**Nomes (16/11/2026):** a pedido do PO, os repositórios foram renomeados de `radar-saude-mobile`, `radar-saude-backend`, `radar-saude-docs` e `painel-admin-frontend` para `pulseira-verde-mobile`, `pulseira-verde-backend`, `pulseira-verde-docs` e `pulseira-verde-admin`. O GitHub redireciona os endereços antigos. As chaves dos projetos no SonarQube Cloud continuam as antigas (só os nomes exibidos são Pulseira Verde), e o APK passa a se chamar `pulseira-verde-hom-X.Y.Z-rc.N.apk` a partir da rc.18.
**Pedido do PO (29/10/2026):** "migrar o projeto para o github devpointinnovations, onde cada setor terá seu repositório próprio, front end mobile, painel admin e back end". Decisões do PO: repositórios públicos, histórico inteiro preservado, commits antigos do Claude reescritos no nome do PO.

---

## 1. Para onde foi cada pasta

| Repositório | Veio de | Também levou |
|---|---|---|
| [`pulseira-verde-mobile`](https://github.com/devpointinnovations/pulseira-verde-mobile) | `frontend/` (vira a raiz) e os arquivos do app que ficaram na raiz entre 26/02 e 21/03/2026 | `deploy/android/`, `doc/assets/` (GIFs do README), o `README.md` de apresentação, `.github/` |
| [`pulseira-verde-admin`](https://github.com/devpointinnovations/pulseira-verde-admin) | `admin/` (vira a raiz) | `.github/` |
| [`pulseira-verde-backend`](https://github.com/devpointinnovations/pulseira-verde-backend) | `backend/` (vira a raiz) | `deploy/google/`, `deploy/oracle/` (histórico), `perf/`, os dados geoespaciais de origem (histórico), o `render.yaml` da raiz em `deploy/render/` (histórico), `.github/` |
| [`pulseira-verde-docs`](https://github.com/devpointinnovations/pulseira-verde-docs) (este) | `Documentos/` | `doc/tutorial-usuario/`, `site/` (Worker do saudemonitor.dev) |

Cada repositório leva o histórico de `.github/` inteiro; o PR de ajuste de cada um tira os workflows que não são dele.

Ficaram só no repositório antigo (`GabrielVogado/saude-monitor`, intacto, não arquivado):

- as ~200 branches de trabalho já mergeadas (foram levadas só `develop`, `master` e as tags rc);
- issues, PRs, histórico do Actions e as **Releases com os APKs das rc 1 a 10**;
- as pastas de configuração de agentes que entraram no git por engano em agosto e setembro, por regra do PO de 03/09/2026.

PRs citados em mensagens de commit e nos documentos (`#123`) são do repositório antigo.

## 2. Como o histórico foi separado

- `git filter-repo` por repositório, mantendo autor, data e mensagem de cada commit.
- **Mobile:** o app ficou na raiz do repositório até 19/03/2026 e conviveu com a cópia em `frontend/` até 21/03/2026. A regra junta as duas fases sem deixar a limpeza da raiz apagar os arquivos do app.
- **Backend:** o `render.yaml` da raiz (aposentado em 22/10, M-060) colidia com o `backend/render.yaml`; foi para `deploy/render/render.yaml` no histórico.
- **Autoria (decisão do PO):** os 154 commits com autor `Claude <noreply@anthropic.com>` passaram a `GabrielVogado <gabriel.vogado@gmail.com>`, e as linhas `Claude-Session`, `Co-Authored-By: Claude` e os links do claude.ai saíram de 174 mensagens. Datas, conteúdo e ordem não mudaram.
- **APKs em Git LFS (mobile):** três APKs antigos (`saude-monitor-preview.apk` e dois em `android/app/build/outputs/`) estavam no LFS. O ambiente da migração não consegue enviar objetos LFS e o GitHub recusa ponteiro sem objeto, então os três viraram arquivos comuns do git, com o mesmo conteúdo (`git lfs migrate export`). O maior tem 74 MB, abaixo do limite de 100 MB.
- Conferência: para cada commit, a árvore da pasta no monorepo é igual à árvore do repositório novo; `develop`, `master` e as 10 tags conferidas com `git ls-remote` depois do push.

## 3. O que muda no fluxo

- **CI:** cada repositório roda só o job da sua área (`Frontend (Expo Web)`, `Admin (Angular)`, `Backend (Spring Boot)`) e a trava de contagem de testes, agora com uma área só. Os rulesets exigem o check da área.
- **Homologação por repositório.** Antes, um merge na `master` gerava uma rc com o APK e o backend juntos. Agora:
  - **mobile:** `master` → APK HML → tag `vX.Y.Z-rc.N` (X.Y.Z do `app.json`) + Release com o APK;
  - **backend:** `master` → deploy `saude-monitor-backend-hom` → tag `vX.Y.Z-rc.N` (X.Y.Z do `version` do `build.gradle`, que passou a `1.0.0`) + Release sem anexo;
  - as tags `rc.11` a `rc.16` vieram do monorepo na sincronização de 16/11, então as duas numerações começam em `rc.17` e andam separadas;
  - quando o app depende de uma mudança da API, promover o **backend antes** do app. O APK continua apontando para a URL fixa de homologação.
  - o registro em [`homologacoes/`](./homologacoes/) passa a citar o par usado no teste (APK `rc.N` + backend `rc.M`).
- **Produção:** o `release.yml` de cada repositório cria a `release/<versão>` a partir da rc dele; no backend dispara o deploy de produção, no mobile o AAB.
- **SonarQube Cloud:** organização `devpointinnovations`, projetos `devpointinnovations_radar-saude-mobile`, `devpointinnovations_painel-admin-frontend` e `devpointinnovations_radar-saude-backend`. O histórico de análises da organização `gabrielvogado` não migra.
- **Google Cloud (WIF):** a federação aceitava só `GabrielVogado/saude-monitor`. Os dois comandos que liberam o repositório novo estão em `deploy/google/README.md` do pulseira-verde-backend.

## 4. PRs de ajuste

| Repositório | PR |
|---|---|
| pulseira-verde-admin | [#1](https://github.com/devpointinnovations/pulseira-verde-admin/pull/1) |
| pulseira-verde-backend | [#1](https://github.com/devpointinnovations/pulseira-verde-backend/pull/1) |
| pulseira-verde-mobile | [#1](https://github.com/devpointinnovations/pulseira-verde-mobile/pull/1) |
| pulseira-verde-docs | o PR deste documento |

## 5. Passos de conta do PO

1. ~~Em cada repositório: branch padrão `develop` e apagar a `main` criada junto com ele.~~ Feito: conferido em 16/11, os quatro têm `develop` como padrão e nenhuma `main`.
2. Segredos do Actions: `SONAR_TOKEN` nos três repositórios de código; no mobile, `ANDROID_RELEASE_KEYSTORE_BASE64`, `ANDROID_RELEASE_STORE_PASSWORD`, `ANDROID_RELEASE_KEY_ALIAS`, `ANDROID_RELEASE_KEY_PASSWORD`, `ANDROID_KEYSTORE_BASE64`, `EXPO_TOKEN`, as variáveis `EXPO_PUBLIC_API_BASE_URL` e `EXPO_PUBLIC_API_BASE_URL_PROD` e, opcionais (sem eles o APK sai sem Sentry), `EXPO_PUBLIC_SENTRY_DSN`, `SENTRY_AUTH_TOKEN`, `SENTRY_ORG` e `SENTRY_PROJECT`. O `EXPO_PUBLIC_MAPBOX_TOKEN` deixou de ser usado com a volta ao MapLibre (M-071). Os valores não podem ser lidos do repositório antigo; vêm de onde foram guardados.
3. SonarQube Cloud: importar a conta devpointinnovations e criar os três projetos com as chaves do §3.
4. Google Cloud: rodar os dois comandos de `deploy/google/README.md` (pulseira-verde-backend) antes do primeiro deploy por lá.
5. Importar os rulesets de `.github/rulesets/` em cada repositório e ligar o Code Scanning.
6. Cópias locais: clonar os repositórios novos (o `D:\saude-monitor` continua apontando para o antigo).
7. Primeira homologação pelo fluxo novo (backend `rc.17`, depois APK `rc.17`) valida tudo de ponta a ponta.

## 6. Sincronização com o monorepo

Enquanto o trabalho continuar no `GabrielVogado/saude-monitor`, cada merge lá só chega aos repositórios da org por uma nova sincronização. O filtro é determinístico: rodado de novo sobre o monorepo, ele gera os mesmos commits já publicados e acrescenta só os novos, então o envio é fast-forward e não reescreve nada. Ferramentas na pasta do projeto (`ferramentas/migracao-org/`):

- `admin.txt`, `backend.txt`, `docs.txt`: `git filter-repo --paths-from-file <arquivo> --mailmap mailmap --message-callback "$(cat msg_cb.py)"`;
- mobile: `git filter-repo --mailmap mailmap --message-callback "$(cat msg_cb.py)" --commit-callback "$(cat mobile_cb.py)"`, depois `git lfs fetch --all` do monorepo e `git lfs migrate export --everything --include="*.apk" --yes`;
- conferência antes do envio: a tag `v1.0.0-rc.10` gerada tem de ter o mesmo hash da que está no repositório da org, e a `develop` da org tem de ser ancestral da nova;
- envio de `develop`, `master` e das tags novas sem `--force`, com o `CD - Homologação` desligado durante o push na `master` (sem os segredos ele só falharia, mas não deve rodar sem querer).

A sincronização para quando o PO fizer a virada: os PRs passam a ser abertos nos repositórios da org e o monorepo recebe só o aviso no README.
