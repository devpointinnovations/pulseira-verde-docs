# Plano de migração do app de JavaScript para TypeScript — v1.0

> Proposta para análise do PO (23/11/2026). **Só documentação: nenhum código do app foi alterado.**
> Operacionaliza o ADR-010 (`08-analise tecnica/adrs.md`, "Proposto") e o ARQ-11 do backlog, sobre o estado real do `pulseira-verde-mobile` na `develop` (`d3bd6db`).

## 1. Escopo e leitura do pedido

O alvo é o app `pulseira-verde-mobile` (Expo SDK 55, React Native 0.83, React 19). Conferido nos quatro repositórios:

| Repositório | Linguagem hoje | Entra na migração? |
|---|---|---|
| `pulseira-verde-mobile` | JavaScript: 98 arquivos de produção (14.038 linhas) + 74 de teste, **zero** `.ts/.tsx` | **Sim, é o alvo** |
| `pulseira-verde-admin` | Angular, já em TypeScript (`src/main.ts` + `src/app/**`) | Não |
| `pulseira-verde-backend` | Java (Spring Boot); só `perf/` tem 4 scripts JS de carga (k6 e Mongo) | Não (scripts de carga ficam em JS) |
| `pulseira-verde-docs` | Markdown | Não |

## 2. Estado de partida (medido)

- `tsconfig.json` já existe: `extends expo/tsconfig.base`, `strict: true`, `allowJs: true`, **`checkJs: false`**. Por isso o `npm run typecheck` (passo do CI) roda sem verificar nada; o próprio `ci.yml` registra isso em comentário.
- `typescript ~5.9.2` e `@types/react` já estão instalados. Faltam `@types/jest` e, se quisermos, `@types/node` (só testes/config).
- Jest: `testMatch: **/__tests__/**/*.test.js` e `collectCoverageFrom: src/**/*.js`; ambos precisam aceitar `.ts/.tsx`. O preset `jest-expo` já transpila TS via Babel, sem `ts-jest`.
- Sonar: `sonar.inclusions=src/**,App.js`, já agnóstico à extensão; `sonar.javascript.lcov.reportPaths` lê o `lcov.info` do Jest, que continua valendo para TS.
- Trava de contagem de testes (`.github/scripts/contagem_testes.py`): usa `src/*` e `App.js` como "código de produção" e `src/*/css/*`, `src/theme/*` como exceção. **`App.js` vira `App.tsx`**: o recorte precisa ser atualizado no mesmo PR.
- Piso de cobertura atual: 79/67/77/80 (statements/branches/functions/lines); lint com teto de 16 avisos. 942 testes.
- Maiores arquivos (risco): `GeofencingTaskService.js` (1.124 linhas), `GeoLocalizacaoScreen.js` (739), `FeedbackFormScreen.js` (651), `HospitaisScreen.js` (452), `HistoricoScreen.js` (437), `FeedbackNotificationService.js` (431).
- Superfície sem dependência circular conhecida: `src/theme`, `src/utils`, `src/core/api`, `src/config` são a base que o resto importa; ótimos primeiros alvos.

## 3. Decisões propostas (para o PO aprovar ou ajustar)

| # | Decisão | Motivo |
|---|---|---|
| T1 | **Incremental, arquivo a arquivo**, `allowJs: true` até o fim; sem "big bang" | App em homologação (rc.19); 942 testes protegem cada passo |
| T2 | **`strict: true` desde o primeiro `.ts`** (já está ligado); o que ainda é `.js` fica sob `checkJs: false` | Evita dívida de "strict depois" |
| T3 | **Sem `any` implícito e `any` explícito só com `// TODO(ts): motivo`** | Lint (`@typescript-eslint/no-explicit-any` em `warn`) conta no teto |
| T4 | Tipos de domínio em `src/core/types/` (`Hospital`, `Visita`, `Feedback`, `Usuario`, `PaginaDe<T>`), derivados da `Especificacao-API-v2.4.md` | Já previsto no ADR-010 |
| T5 | Contrato da API tipado à mão no início; **gerar tipos do OpenAPI do backend** só se o backend passar a publicá-lo (decisão futura, não bloqueia) | Evita nova dependência agora |
| T6 | Validar respostas da API em runtime com **Zod** só na borda (`apiClient`) e nas filas offline lidas do AsyncStorage | Tipo estático não protege contra dado salvo por versão antiga do app (rc anteriores) |
| T7 | Testes migram **depois** do código que cobrem, e só os que ganham valor (tipos de mocks); podem ficar em `.js` até o fim | Menos churn e menos risco |
| T8 | Conversão de arquivo = **renomear + tipar, sem mudar comportamento**. Refatoração fica fora do PR (Anti-Efeito Cobra: nada de PR pro-forma) | Diff revisável e `git blame` preservado (`git mv` em commit próprio) |

Pergunta ao PO (única de negócio): **a migração entra antes ou depois do Design System (documento irmão)?** Recomendo **tipar a base e o `theme/` primeiro (fases 0 a 2), depois construir o Design System já em TS**, e só então converter as telas. Assim os componentes novos já nascem tipados e não são migrados duas vezes.

## 4. Fases

Cada fase = 1 a 4 PRs pequenos, cada um saindo da `develop` (sem empilhar), com CI verde e commits assinados pelo desktop.

### Fase 0: fundação do tooling (1 PR, ~0,5 dia)
- `tsconfig.json`: `"checkJs": false` mantido; incluir `"types": ["jest"]`, `"paths"` não (evita mudar import em massa).
- `npm i -D @types/jest`; `@typescript-eslint` já vem do `eslint-config-expo`.
- `jest.config.js`: `testMatch: ["**/__tests__/**/*.test.{js,ts,tsx}"]`, `collectCoverageFrom: ["src/**/*.{js,ts,tsx}", "App.{js,tsx}", ...]`.
- `contagem_testes.py`: aceitar `App.js` ou `App.tsx` no recorte.
- `ci.yml`: o passo "Typecheck" deixa de ter o comentário "não verifica nada"; ganha `tsc --noEmit` como gate real.
- **Aceite:** CI verde, 942 testes, cobertura e Sonar sem variação; um `.ts` de teste (ex.: `src/utils/distancia.ts`) compila e é coberto.

### Fase 1: base sem UI (2 a 3 PRs, ~2 dias)
Ordem (do que não depende de nada para o que todos usam):
1. `src/theme/tokens.js` → `.ts` com `as const` (cores e espaçamentos viram tipos literais; base do Design System).
2. `src/utils/*` (`distancia`, `format`, `geojson`, `indicadores`, `mapStyle`, `normalize`, `alertas`, `mapkit/*`; atenção ao par `index.js`/`index.web.js`: vira `index.ts`/`index.web.ts` com uma `index.d.ts`/interface única compartilhada).
3. `src/core/types/*` (novo) e `src/core/api/*` (`apiClient`, `apiError`, `sessaoApi`, `envioComFila`): `request<T>()` genérico, `ApiError` tipado, binário (PDF) com assinatura própria.
- **Aceite:** `tsc --noEmit` zero erros nos migrados; nenhum `any` sem TODO; testes dessas áreas inalterados e verdes.

### Fase 2: estado, config e serviços (2 a 3 PRs, ~3 dias)
`src/core/query/*` (chaves de query tipadas, `queryClient`), `src/core/stores/sessaoStore.js` (Zustand tipado), `src/config/*` (`filaOffline`, `sessao`, `http`, `monitoramento`), `src/services/*` (`CatalogoLocal`, `SincronizacaoOffline`, `TokenStorage`…).
- **Ponto de atenção:** a fila offline e o catálogo persistem JSON no AsyncStorage escrito por rcs antigas. Aqui entra o **Zod na leitura** (T6) com teste de "payload da rc.17" para evitar quebrar quem atualiza o app por cima.
- **Aceite:** testes de offline (`filaOffline`, `SincronizacaoOffline`, `CatalogoLocal`) verdes sem alteração de asserção.

### Fase 3: Design System em TS (parte do documento irmão)
Os componentes `CS*` (12 hoje) e os novos entram já em `.tsx`, com `Props` exportadas. Ver `Proposta-Design-System-App-v1.0.md`.

### Fase 4: telas por feature (1 PR por feature, ~6 a 8 dias)
Ordem do menor para o maior risco, **uma feature por PR**, `service/` e `hooks/` antes de `view/`:
1. `onboarding` e `atualizacao` (pequenas).
2. `auth` (+ `LoginService`: toca token; passa por `security-review`).
3. `user` e `perfil`.
4. `hospitais` (lista, detalhe, ranking, mapa).
5. `feedback` (formulário de 651 linhas e notificações).
6. `visitas` (inclui o `GeofencingTaskService`, 1.124 linhas: **por último, sozinho num PR**, com revisão extra e roteiro de campo, porque roda em segundo plano e o erro só aparece no aparelho).
7. `geolocalizacao` (739 linhas).
8. `App.js` → `App.tsx` e `index.js` → `index.ts`; tipos de navegação (`RootStackParamList`, `TabParamList`) em `src/navigation/types.ts`, e `useNavigation<...>` tipado.

### Fase 5: endurecimento (1 PR, ~1 dia)
- `checkJs` fica irrelevante (sem `.js` de produção); **`allowJs: false`**.
- Config em JS (`jest.config.js`, `metro.config.js`, `eslint.config.js`) pode ficar em CJS; migrar só se sobrar tempo (sem ganho de produto).
- Regras extras em `error`: `no-explicit-any`, `consistent-type-imports`, `no-floating-promises` (promessas soltas são onde o Sonar já reclamou, S9383).
- Subir o piso de cobertura conforme a meta do PO (frontend ≥ 90%), sem baixar nenhum.
- Registrar em `Historico-Melhorias.md`; ADR-010 passa a "Implementado".

## 5. Estimativa de esforço

| Fase | Esforço | Risco |
|---|---|---|
| 0 Fundação | 0,5 dia | Baixo |
| 1 Base sem UI | 2 dias | Baixo |
| 2 Estado, config, serviços | 3 dias | **Médio** (dados persistidos) |
| 3 Design System | ver doc irmão (~8 dias) | Médio |
| 4 Telas | 6 a 8 dias | **Médio/alto** (geofencing, formulário) |
| 5 Endurecimento | 1 dia | Baixo |
| **Total só da migração** | **≈ 13 a 15 dias úteis de uma pessoa**, em 14 a 18 PRs | |

Estimativa por contagem de arquivos e linhas, sem medir produtividade real. Revisar ao fim da fase 1 (que serve de amostra: se levar o dobro, o total dobra).

## 6. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Tipo "bonito" que mente sobre a API | Zod na borda (T6) + tipos derivados da `Especificacao-API-v2.4.md`, não do palpite |
| Quebrar quem atualiza o app com fila/cache antigos | Teste com payload da rc.17 na fase 2; leitura tolerante com valor padrão |
| Geofencing em segundo plano regredir | Último PR, sozinho; mesmos testes + roteiro de campo na homologação seguinte |
| Cobertura/Sonar cair por troca de extensão | Fase 0 já ajusta `collectCoverageFrom`, contagem de testes e Sonar; medir antes e depois |
| Teto de 16 avisos do lint estourar | Regras novas entram em `warn` com teto medido e descem por fase |
| Mocks do Jest sem tipo | `jest.mocked()` e fábricas tipadas em `src/__tests__/helpers/` |
| Metro/Expo web: par `.js`/`.web.js` | Mesma assinatura via `.d.ts` comum; verificar o `expo export` do CI |
| Frentes em paralelo conflitando | Uma feature por vez; a migração congela refatorações grandes na feature em curso |
| Esforço sem valor de produto | Parar na fase 2 já entrega rede de proteção nas camadas críticas; as demais fases são opcionais e independentes |

## 7. Critérios de aceite (por PR e final)

Por PR: `tsc --noEmit` verde; `lint` dentro do teto; testes e cobertura sem queda; Sonar sem novo problema; diff só renomeia e tipa; documentado em `Historico-Melhorias.md` quando a fase fecha.
Final: zero `.js` em `src/` (exceto configs), `allowJs: false`, `tsc` como gate, `any` residual ≤ 5 com TODO, piso de cobertura mantido ou maior, ADR-010 "Implementado".

## 8. Rollback

- Cada PR é pequeno e independente: **reverter o PR** (Squash/revert) volta o arquivo ao `.js`, porque `allowJs: true` convive com ambos até a fase 5.
- A fase 5 (`allowJs: false`) é a única de "mão única"; só entra com todas as anteriores estáveis em homologação, e também é revertível por um PR.
- Nenhuma fase muda comportamento nem versão de dependência de runtime (só `devDependencies` e, se aprovado, Zod), então não há rollback de dados.

## 9. Fora de escopo

Trocar bibliotecas, reorganizar para feature-first (ADR-008), migrar scripts de `perf/` do backend, testes migrados em massa, gerar cliente da API.

## 10. Próximo passo

Aprovando este plano (e a ordem com o Design System), abre-se o PR da **fase 0** a partir da `develop`. Itens de backlog sugeridos: ARQ-11 passa de "incremental" para "planejado" no `De-Para-Backlog-Features.md` somente quando a fase 0 for entregue.
