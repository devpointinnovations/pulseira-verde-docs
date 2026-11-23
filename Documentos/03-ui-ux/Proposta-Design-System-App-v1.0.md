# Proposta de Design System escalável do app — v1.0

> Proposta para análise do PO (23/11/2026). **Só documentação: nenhum código do app foi alterado.**
> Base: `src/theme/tokens.js` e `src/components/CS*` do `pulseira-verde-mobile` (`develop` `d3bd6db`), o `Padrao-UI-UX-v2.1.md` (§5) e o protótipo aprovado do Open Design (`pulseira-verde-telas.html`, 11 telas, logo B). Complementa o `Plano-Migracao-TypeScript-App-v1.0.md`.

## 1. Ponto de partida: já existe um embrião, falta torná-lo sistema

| O que existe | Situação |
|---|---|
| `theme/tokens.js` com `colors`, `typography`, `spacing`, `radii`, `shadows`, `motion`, `touchTarget`, `gradients`, `fontes` | Bom, **já no verde #0B6B4F e na Inter**. É uma camada só (valores crus + papéis misturados) |
| 12 componentes `CS*` (`CSButton`, `CSChip`, `CSCard`, `CSHospitalCard`, `CSHeader`, `CSTextField`, `CSOptionSheet`, `CSRatingStars`, `CSEmptyState`, `CSLoading`, `CSIconButton`, `CSAvisoConexao`) + rodapé legal | Alguns com teste (`CSHospitalCard`, `CSTextField`, `CSAvisoConexao`, `CSRodapeLegal`); props sem tipo; variantes por `if` |
| Telas com estilo próprio | 30 arquivos usam `StyleSheet.create`; 1 cor hex solta em tela (`MapaPertoDeVoce.js`); estilos em `css/` por tela (`LoginStyle`, `PerfilStyle`, `UserStyle`) |
| Tab bar, Sheet do Início, badge de nota, pill de status, pin do mapa | Existem no protótipo; no app estão dentro de `App.js` ou de telas, **não como componentes** |
| Nome "Clinical Sanctuary" | Herdado da v2.0 azul. Proposta: **renomear para "Pulseira Verde DS"** (sem mexer em arquivo hoje, só na documentação futura) |

Conclusão: não é preciso construir do zero. A proposta **consolida, tipa, completa e documenta** o que existe, para que a próxima tela nasça de peças e não de estilo solto.

## 2. Princípios

1. **Tokens em três camadas** (primitivo → semântico → componente), nunca hex em componente ou tela.
2. **Componente decide, tela compõe:** tela só monta peças; estado visual (hover/pressed/disabled/erro/carregando) mora no componente.
3. **Acessibilidade é requisito do componente** (papel, rótulo, estado, alvo ≥ 48 dp, contraste AA), não da tela.
4. **Offline e carregando são estados de primeira classe** (skeleton, vazio, erro, "dado antigo").
5. **Um componente só entra no sistema com:** props tipadas, todas as variantes e estados, teste, e entrada no catálogo.
6. **Pouco e útil:** só componentes que já têm 2 usos ou estão no protótipo aprovado (YAGNI).

## 3. Tokens

### 3.1 Camadas

```
Primitivo  → green.700 = #0B6B4F, neutral.900 = #13211B, ...
Semântico  → color.action.primary = green.700, color.text.primary = neutral.900
Componente → button.primary.bg = color.action.primary
```

Hoje `tokens.js` mistura primitivo e semântico (`primary`, `ratingFilled`, `geoActive`). Proposta: manter os nomes atuais como **semânticos** (sem quebrar as telas) e acrescentar a camada primitiva por baixo, em `src/theme/primitives.ts`.

### 3.2 Valores (já aprovados no Open Design e presentes no `tokens.js`)

| Grupo | Valores |
|---|---|
| **Marca** | verde `#0B6B4F`; container `#D5EEE2`; sobre-container `#06402F`; gradiente `#0B6B4F → #06402F` |
| **Superfícies** | fundo `#F4F7F5`; cartão `#FFFFFF`; linha `#D9E2DC`; texto `#13211B`; texto secundário `#4C5B54` |
| **Semânticos** | erro `#B3261E` (badge/contador); âmbar `#8A5A00` sobre `#FCEFD2` (falta avaliar); estrela `#E8A317`; localização do usuário `#1A73E8` (só no mapa) |
| **Tipografia** | Inter 400/500/600/700/800. Escala: display 32/28/24, headline 22/20, title 18/16, body 16/14/12, label 14/12/11, numérico tabular. Títulos do protótipo: 800, tracking −0,4 a −0,6 |
| **Espaçamento** | 4 · 8 · 12 · 16 · 20 · 24 · 32 · 40 · 48 · 64 |
| **Raios** | 4 · 8 · 12 · 16 (botão, inputs) · 18 (card) · 24 · 28 (sheet) · pílula 9999 |
| **Sombras** | `cloud1`, `cloud2`, `primaryGlow`, `glass` (já existem); elevação Android equivalente |
| **Movimento** | 120/200/300 ms, curvas padrão/entrada/saída; **respeitar "reduzir movimento"** |
| **Alvo de toque** | mínimo 48 dp (`touchTarget.min`) |

### 3.3 Pontos a decidir/corrigir

- **Modo escuro:** hoje não existe. Proposta: **não entra agora**; os tokens semânticos já permitem um tema escuro depois sem mexer em componente (decisão do PO, ver §10).
- **Contraste:** `onSurfaceVariant #4C5B54` sobre `#F4F7F5` e o âmbar `#8A5A00` sobre `#FCEFD2` precisam de verificação AA registrada (o v2.1 §5.1.4 faz para a paleta azul antiga; **refazer para a verde**).
- **Escala tipográfica do app web** (react-native-web) usa as mesmas fontes por `carregarFontes.web.js`; manter.

## 4. Catálogo de componentes

Três níveis. Em **negrito** o que já existe (a evoluir); o resto é novo e vem do protótipo.

### 4.1 Base (átomos)

| Componente | Variantes | Estados | Observação |
|---|---|---|---|
| **`Button`** (`CSButton`) | `primary` 56 dp r16, `secondary`/`outline` 52, `tertiary`/`ghost` 48; `sm` | default, pressed, disabled, loading | ícone à direita; `fullWidth` |
| **`IconButton`** | filled, tonal, plain | pressed, disabled | rótulo de acessibilidade obrigatório |
| **`Chip`** (`CSChip`) | filtro (36 dp r18), selecionado `#D5EEE2` | default, selected, disabled | grupo com rolagem horizontal |
| `Badge`/`Pill` | contador (vermelho), status, "falta avaliar" (âmbar), distância | — | `StatusPill` com tom: neutro, sucesso, aviso, erro |
| **`RatingStars`** + `RatingBadge` | nota numérica com estrela, tamanhos sm/md/lg | somente leitura, editável | nota com `tabular-nums` |
| `Avatar` | iniciais, 36/52 dp | — | |
| **`TextField`** | texto, senha, busca | default, focus, erro, disabled, helper | rótulo visível, erro junto ao campo |
| `Icon` | wrapper de `lucide-react-native` com tamanhos e cor por token | — | evita import solto e emoji |

### 4.2 Compostos (moléculas)

| Componente | Notas |
|---|---|
| **`Card`** (`CSCard`) | `elevated`, `outlined`, `selected` (borda verde 1,5) |
| **`HospitalCard`** (`CSHospitalCard`) | ícone, nome (1 linha), tipo/distância, nota, tempo de espera "45 min"; **sem botão Check-in no Início** (decisão do PO 23/11) |
| `ListItem` | ícone, título, subtítulo, chevron (Perfil, Histórico, Notificações) |
| `SearchBar` | pílula 52 dp sobre o mapa, avatar à direita |
| **`AppBar`** (`CSHeader`) | título, voltar, ação; versão transparente sobre o mapa |
| `TabBar` | Início · Ranking · Visitas · Perfil; item ativo com pílula `#D5EEE2` 64×32; badge de contador (hoje dentro de `App.js`) |
| **`BottomSheet`** | alça, cantos 28, estados `peek`/`half`/`full` (Início com mapa) |
| **`OptionSheet`** (`CSOptionSheet`) | seleção em folha |
| **`Banner`** (`CSAvisoConexao`) | offline, pendências a enviar, erro; tons info/aviso/erro |
| **`EmptyState`**, **`Skeleton`/`Loading`** (`CSLoading`) | ilustração leve + ação |
| `MapPin` | pílula com nota; selecionado em verde com escala 1,12; "você está aqui" azul |
| `ProgressDots` | onboarding (ponto 8 dp, ativo 24 dp) |
| `FormQuestion` | pergunta de avaliação em 4 passos (Avaliar) |

### 4.3 Padrões de tela (organismos, só documentados)

Início com mapa + sheet, lista de hospitais, detalhe + "Estou aqui agora", avaliação em 4 perguntas, Visitas, Ranking, Perfil, onboarding. Cada padrão referencia **quais componentes usa**; não vira componente genérico até repetir.

### 4.4 Regras comuns de API

Prop `variant` (união de strings, não booleanos soltos), `size`, `disabled`, `loading`, `testID`, `accessibilityLabel` (obrigatório quando só ícone), `style` apenas para **layout externo** (margem), nunca para cor/tipografia. Exportação nomeada em `src/design-system/index.ts` (o barrel atual `components/index.js` tem teste; manter o teste).

## 5. Acessibilidade (critério de aceite de todo componente)

- Papel (`accessibilityRole`), rótulo, estado (`selected`, `disabled`, `busy`).
- Alvo ≥ 48 dp; espaçamento entre alvos ≥ 8 dp.
- Contraste AA (4,5:1 texto; 3:1 ícone/borda) verificado e registrado para a paleta verde.
- Não depender só de cor (nota tem número; status tem texto).
- Escala de fonte do sistema suportada (sem altura fixa em texto; `numberOfLines` com intenção).
- "Reduzir movimento" desliga animações não essenciais; foco/leitor de tela na ordem visual.
- Teste automatizado: `getByRole`/`getByLabelText` por componente (Testing Library já em uso).

## 6. Estrutura de pastas

```
src/design-system/
  tokens/        primitives.ts · semantic.ts · typography.ts · spacing.ts · index.ts
  components/
    Button/      Button.tsx · Button.test.tsx · Button.stories.tsx · index.ts
    Chip/ ...    (uma pasta por componente)
  patterns/      documentação dos padrões de tela (markdown)
  index.ts       API pública: tokens + componentes
```

- **Migração de nome:** `components/CS*` → `design-system/components/*` com **re-export temporário** em `components/index.js` para as telas não quebrarem; remove-se o re-export quando a última tela migrar.
- **Fronteira:** `design-system/` não importa nada de `screens/`, `services/` nem `core/` (só `react`, `react-native` e ícones). Regra de lint (`no-restricted-imports`) para impedir.
- Tema: `ThemeProvider` mínimo (contexto com os tokens), apenas se o modo escuro for aprovado; até lá, import direto dos tokens.

## 7. Documentação e catálogo

| Opção | Prós | Contras |
|---|---|---|
| **A. Storybook React Native em modo web** (react-native-web já está no app) | Catálogo navegável no navegador, publicável (Pages/Cloudflare), `play` de acessibilidade, padrão de mercado | Dependência e configuração (Vite + RN Web); stories são código a manter |
| B. Tela "Catálogo" dentro do app (só `__DEV__`) | Zero dependência; testa no aparelho real | Sem navegação por docs; fica só para quem roda o app |
| C. Só Markdown + imagens (`pulseira-verde-docs`) | Mais barato | Desatualiza; não testa o componente real |

**Recomendação: A para a fase de componentes, com C como contrato (cada componente tem 1 página curta: quando usar, variantes, estados, acessibilidade, "não faça").** B só se A travar no Expo SDK 55. Decisão do PO (§10). Antes de adotar, **fazer um spike de 0,5 dia** para provar que o Storybook web roda com Expo 55/RN 0.83/Reanimated 4 e não pesa no CI.

## 8. Como migrar as telas atuais

Fluxo por tela, na ordem do Open Design, **sem mudar comportamento nem layout aprovado** (o app já foi alinhado ao protótipo nos PRs #6 a #15; o DS tira o estilo duplicado):

1. **Ondas de componentes** (fase 3 do plano de TS), cada uma com PR próprio: (a) tokens em 3 camadas; (b) `Button`, `IconButton`, `Chip`, `Badge/Pill`, `RatingBadge`; (c) `Card`, `HospitalCard`, `ListItem`, `TextField`; (d) `AppBar`, `TabBar`, `BottomSheet`, `Banner`, `MapPin`.
2. **Tela a tela:** substituir `StyleSheet` local por componentes + tokens: onboarding → Início → Detalhe/Avaliação → Visitas/Ranking/Perfil → Auth/Perfil (os `css/*Style.js`).
3. **Medição de adoção:** script que conta hex literais e `StyleSheet.create` em `screens/` (hoje: 30 arquivos com estilo, 1 hex). Meta: hex em tela = 0.
4. **Regressão visual:** capturas das telas do protótipo (`telas/*.png`) comparadas com o app web (Playwright já disponível na nuvem) por componente e por tela; tolerância pequena e revisão humana. Não é gate do CI na primeira versão.
5. **Congelamento:** depois da onda (d), tela nova só com componentes do DS; exceção documentada.

## 9. Estimativa de esforço e riscos

| Etapa | Esforço |
|---|---|
| Tokens em 3 camadas + contraste AA verificado | 1 dia |
| Onda (b) átomos | 2 dias |
| Onda (c) compostos | 2 dias |
| Onda (d) estruturais (TabBar, Sheet, Banner, Pin) | 2 dias |
| Storybook web (spike + 12 componentes documentados) | 1,5 dia |
| Migração das telas (≈ 20 telas, 1 a 2 por PR) | 5 a 6 dias |
| **Total** | **≈ 14 a 15 dias úteis**, sobrepondo-se à migração TS (fase 3) |

| Risco | Mitigação |
|---|---|
| "Design system" virar fim em si | Regra YAGNI: só o que está no protótipo ou tem 2 usos; entrega por tela, não por catálogo |
| Mudar visual aprovado sem querer | Comparação com `telas/*.png` e revisão do PO por onda; DS **copia** o layout aprovado |
| Sheet/mapa exigem Reanimated/Gesture Handler (já instalados) com desempenho irregular em Android modesto | Protótipo do `BottomSheet` medido em aparelho real antes de generalizar |
| Storybook incompatível com Expo 55 | Spike de 0,5 dia; plano B (tela Catálogo `__DEV__`) |
| Cobertura/Sonar caírem ao mover arquivos | Mover com `git mv`, testes junto; piso de cobertura mantido |
| Divergência com o admin (Angular) | Fora de escopo agora; tokens em JSON neutro (`tokens.json`) como fonte única é decisão futura |

## 10. Perguntas ao PO

1. **Modo escuro** entra agora, depois, ou nunca no MVP? (Recomendo: depois; os tokens já ficam prontos.)
2. **Storybook web** (opção A) ou começar pelo mais simples (B/C)? (Recomendo A após spike.)
3. **Ordem com a migração TS:** DS já em `.tsx` (recomendado, plano de TS fases 0 a 2 antes).
4. **Nome:** "Pulseira Verde DS" no lugar de "Clinical Sanctuary" na documentação?
5. **Tokens compartilhados com o painel admin** (Angular) viram escopo depois do app?

## 11. Critérios de aceite

- Zero hex e zero espaçamento literal fora de `design-system/tokens`.
- Cada componente: tipos, variantes e estados do §4, teste de papel/rótulo/estado, página no catálogo.
- Telas migradas sem alteração de comportamento: testes existentes verdes e cobertura ≥ piso atual.
- Contraste AA da paleta verde registrado em tabela.
- Capturas das telas migradas conferidas contra o Open Design pelo PO.

## 12. Rollback

Cada onda e cada tela é um PR independente; reverter o PR devolve o estilo antigo, porque o re-export de `components/` fica até a última tela migrar. Tokens novos só **acrescentam** camadas; os nomes atuais continuam valendo.
