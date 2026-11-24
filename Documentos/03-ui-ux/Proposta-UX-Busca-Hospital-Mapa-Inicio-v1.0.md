# Proposta de UX: busca e seleção de hospital no mapa do Início (v1.0)

Data: 24/11/2026 · Pedido do PO · Status: **Opção B escolhida pelo PO em 24/11/2026 e implementada ([M-091](../09-melhoria-continua/Historico-Melhorias.md))**
Repositório da tela: `pulseira-verde-mobile` (develop em `d3bd6db`). Referências visuais: `Padrao-UI-UX-v2.1.md` e protótipo Open Design.

## 1. O que o PO relatou

1. Ao digitar ou selecionar um hospital, o mapa do Início não mostra o hospital.
2. Ao tocar no marcador de hospital no mapa do Início, o app abre o "Mapa de Geolocalização" e não o detalhe do hospital.
3. A barrinha acima de "Perto de você" parece arrastável (para ver a lista inteira) e não faz nada.

## 2. Causa de cada item (confirmada no código)

Arquivos em `src/screens/hospitais/view/`.

### 2.1 O mapa não mostra o hospital buscado ou selecionado

- Não existe estado de "hospital selecionado" no Início. `HospitaisScreen.js` não guarda seleção alguma; o toque num card chama `abrirDetalhe` (`HospitaisScreen.js:184-189`, ligado ao card em `207-215`) e navega direto para o detalhe, sem passar pelo mapa.
- A câmera é fixa na pessoa: `MapaPertoDeVoce.js:46` define o centro como `posicao ?? CENTRO_SEM_GPS` e `:52-55` aplica zoom 12,5 (cerca de 3 a 4 km de largura). Nada move a câmera para um hospital. Um hospital longe fica fora da tela, mesmo que o pin exista.
- A busca só filtra a lista (`HospitaisScreen.js:128-131`); os pins acompanham o filtro (`:182`), mas só os 12 mais perto entram (`PINS_NO_MAPA = 12`, `MapaPertoDeVoce.js:12` e `:28-33`). Um hospital que casa com a busca mas está além do 12º não ganha pin.
- O hospital da visita ativa é colocado no topo da lista (`HospitaisScreen.js:168-179`) mas não entra nos pins (`:182` usa `comDistancia`, sem ele).
- Hospital sem coordenada nunca ganha pin (`MapaPertoDeVoce.js:31`).

### 2.2 O toque no marcador leva ao mapa completo

- É comportamento escrito de propósito: o mapa fica dentro de `pointerEvents="none"` (`MapaPertoDeVoce.js:50`) e um `Pressable` cobre a área inteira (`:78-83`) chamando `onAbrirMapa`, que navega para `MapaTela` (`HospitaisScreen.js:190`). O comentário `:22-24` explica a razão: BUG-04, toque atrasado num mapa já destruído trava o app. Por isso nenhum pin recebe toque no Início.
- O padrão seguro já existe no mapa completo: pin tocável, card de resumo e o mapa desmontado antes de navegar (`GeoLocalizacaoScreen.js:173-190`, `:258-300`, `:405-423`, `:528-540`).

### 2.3 A alça não faz nada

- É uma `View` estática de 40×4 (`HospitaisScreen.js:293`, estilo `:394-401`). A folha tem altura fixa (`flex: 1`, `marginTop: -28`, `:386-393`) e o mapa ocupa sempre 40% da tela (`FRACAO_DO_MAPA`, `:42` e `:112`). Não há gesto nenhum.
- `react-native-gesture-handler` e `react-native-reanimated` já estão no `package.json`, mas o único uso de gesto no `src` é a importação em `index.js`; a folha não os usa.
- O padrão do app prevê bottom sheet com handle 4×40, topo `radius-2xl` e arraste (`Padrao-UI-UX-v2.1.md:592`, animação de 300 ms emphasized em `:665`). A alça do Início promete isso e não cumpre.

## 3. Princípios da proposta

1. Tocar em um hospital (lista, busca ou pin) deve sempre mostrar **onde ele está** antes de abrir o detalhe.
2. Toque no pin leva ao **cartão resumo** e só o botão "Ver detalhes" abre o detalhe. O mapa completo continua existindo, mas por botão próprio.
3. A alça só existe se arrastar de verdade.
4. Reaproveitar o que o mapa completo já resolve: card `CSHospitalCard`, câmera por ref, desmontar o mapa antes de navegar (BUG-04), caixa com meia largura mínima de 0,005° (M-064).

## 4. Opções

### Opção A: Selecionar pela lista e pela busca, sem mexer na folha (menor)

- Toque num card da lista **seleciona**: a câmera enquadra a pessoa e o hospital, o pin ganha destaque e o card mostra o botão "Ver detalhes". Busca com um único resultado seleciona sozinha.
- Pins continuam sem toque (o mapa segue "só vista"); tocar no mapa ainda abre o mapa completo.
- A alça é **removida** (some a promessa de arrastar).

```
┌────────────────────────┐
│ [🔍 Buscar hospital  ◯]│
│                        │
│   ●você    (+4,6)←zoom │  mapa 40%
│                  ◎     │
│  ╭────────────────────╮│
│  │ Perto de você  Rank││  sem alça
│  │ [Todos][Público]…  ││
│  │▌Hosp. Base  1,2 km ││  ← card selecionado
│  │ ★4,6  [Ver detalhes]│
│  │ Hosp. Regional 3 km││
└────────────────────────┘
```

- Resolve: item 1 (parcial: pelo toque na lista e pela busca) e item 3 (por remoção). **Não resolve o item 2**: o pin continua sem toque.
- Esforço: 1 a 1,5 dia. Risco baixo (mapa continua sem toque).
- Testes: ajustar os testes que esperam toque no card abrindo o detalhe (`HospitaisScreen.test.js`); novos testes de seleção e de câmera. Sem teste de gesto.
- Contra: dois toques para chegar ao detalhe a partir da lista e o pin segue mudo.

### Opção B (recomendada): Folha de 3 alturas + cartão resumo + pins tocáveis

**Folha com arraste de verdade**, 3 paradas com encaixe (snap):

| Altura | Folha | Mapa | Quando |
|---|---|---|---|
| Recolhida | só alça, título e chips (~18% da tela) | quase tela cheia | abrir para olhar o mapa; ao selecionar um hospital |
| Meio (padrão) | ~58% (como hoje) | ~42% | abertura do app |
| Expandida | ~92%, lista quase inteira | faixa de busca | ver a lista toda |

A alça e o título "Perto de você" são a área de arraste; a lista só rola com a folha expandida (evita brigar com o gesto). Animação de 300 ms emphasized, como no padrão.

**Seleção única, três entradas** (lista, busca, pin do mapa):

1. A folha desce para "recolhida".
2. A câmera enquadra pessoa e hospital (`fitBounds` com padding: topo = busca, base = cartão). Sem GPS, centraliza no hospital em zoom 14. Muito perto: caixa mínima de 0,005° para não ir ao zoom máximo.
3. O pin selecionado fica maior e verde cheio; os demais ficam como hoje.
4. Cartão resumo (`CSHospitalCard` com distância e nota) sobre o mapa, com **"Ver detalhes"** e **X** para limpar. Só "Ver detalhes" navega.

**Busca:** ao digitar, uma lista de sugestões (até 5) abre sob o campo; tocar numa sugestão seleciona; "buscar" no teclado seleciona a primeira. A lista da folha continua filtrada.

**Mapa:** pin tocável seleciona (usa o mesmo padrão do mapa completo: o pin é uma `View` do React, e o mapa é desmontado antes de navegar). Toque em área vazia limpa a seleção. O botão de localização passa a **recentralizar na pessoa** e um botão novo "Mapa completo" (ícone de expandir) abre o `MapaTela`. O hospital da visita ativa e o selecionado sempre entram nos pins, mesmo fora dos 12 mais perto.

```
 Recolhida + selecionado          Meio (padrão)            Expandida
┌──────────────────────┐   ┌──────────────────────┐  ┌──────────────────────┐
│[🔍 Hosp. Ba…      ◯] │   │[🔍 Buscar…        ◯] │  │[🔍 Buscar…        ◯] │
│ ┌ sugestões ───────┐ │   │                      │  ├──────────────────────┤
│ └──────────────────┘ │   │  ●você   (4,6)       │  │        ━━            │
│   ●você  ╱╲          │   │            (4,1)     │  │ Perto de você  Rank. │
│        ╱  ◉ 4,6 ←pin │   │ ╭──────────────────╮ │  │ [Todos][Púb][Priv]   │
│  [⌖][⛶]              │   │ │       ━━         │ │  │ Hosp. Base    1,2 km │
│ ┌──────────────────┐ │   │ │ Perto de você    │ │  │ Hosp. Regional 3 km  │
│ │Hosp. Base  1,2 km│ │   │ │ [Todos][Púb][Pr] │ │  │ UPA Norte     4 km   │
│ │★4,6 [Ver detalhes]│ │   │ │ Hosp. Base 1,2km │ │  │ …(lista inteira)     │
│ └──────────────────┘ │   │ │ Hosp. Reg. 3 km  │ │  │                      │
│ ━━ Perto de você ▲   │   │ ╰──────────────────╯ │  │                      │
└──────────────────────┘   └──────────────────────┘  └──────────────────────┘
   [⌖] minha posição   [⛶] mapa completo
```

- Resolve os três itens.
- Esforço: 3 a 4 dias (folha com gesto e encaixe, estado de seleção, câmera por ref no mapa do Início, sugestões da busca, pins tocáveis com o desmonte seguro).
- Testes: o gesto não roda no Jest; a regra de encaixe (`paradaMaisProxima(posição, velocidade)`) vira função pura testada, e o gesto fica na casca fina. Reanimated/gesture-handler precisam de mock no setup. Mudam os testes que esperam o botão de localização e o toque no mapa abrindo o mapa completo (`HospitaisScreen.test.js:191` e `:262`, `MapaPertoDeVoce.test.js:79`) e os de toque no card. Novos: seleção por lista, busca e pin, enquadramento, limpar seleção, pin do selecionado fora dos 12. Mantém a meta de cobertura do app (≥ 90%).
- Riscos: BUG-04 (mitigado com a mesma ordem do mapa completo); desempenho do mapa com a folha animando (animar só a folha, sem re-renderizar o mapa a cada quadro).

### Opção C: Início vira o mapa completo (mapa único, folha por cima)

- Funde `MapaTela` e o Início: um mapa interativo em tela cheia com a folha de 3 alturas, filtros e busca; a aba Mapa deixa de existir ou aponta para o mesmo componente.
- Resolve tudo e elimina o mapa duplicado, mas muda a navegação do app, o design aprovado (Início com mapa de 40%) e a aba Mapa.
- Esforço: 6 a 8 dias; reescreve boa parte dos testes de `HospitaisScreen` e `GeoLocalizacaoScreen`. Risco maior (geofencing e check-in vivem no mapa completo).
- Faz sentido só como etapa da migração para TypeScript e Design System, não agora.

## 5. Comparação

| | A: selecionar | **B: folha 3 alturas + cartão** | C: mapa único |
|---|---|---|---|
| Item 1 (hospital no mapa) | sim, por lista e busca | sim, por lista, busca e pin | sim |
| Item 2 (toque no pin) | não | sim (cartão, depois detalhe) | sim |
| Item 3 (alça) | removida | arrasta de verdade | arrasta de verdade |
| Esforço | 1 a 1,5 dia | 3 a 4 dias | 6 a 8 dias |
| Risco | baixo | médio | alto |
| Fiel ao design aprovado | sim | sim (acrescenta paradas e cartão) | não |

## 6. Recomendação

**Opção B.** Entrega o comportamento esperado de app de mapa (arrastar a folha, tocar no pin, ver o hospital antes do detalhe), usa padrões que o app já tem (cartão e câmera do mapa completo, bottom sheet do padrão UI/UX) e mantém o design aprovado. A Opção A serve se o PO preferir uma correção rápida e adiar o gesto.

## 7. Decisões do PO (24/11/2026)

1. **Opção B** (folha de 3 alturas, cartão resumo e pins tocáveis).
2. O primeiro toque na lista **seleciona e mostra no mapa**; "Ver detalhes" no cartão abre o detalhe.
3. O botão de localizar recentraliza na pessoa; o mapa completo ganhou botão próprio.
4. A busca ganhou sugestões sob o campo (até 5).
5. Hospital sem coordenada: o cartão aparece com o aviso "sem localização no mapa".

Diferenças entre o protótipo e o app: tocar numa área vazia do mapa não limpa a seleção (o adaptador do mapa não expõe o toque no mapa); limpa-se pelo X do cartão ou pelo botão de localizar. O pin do hospital da visita ativa só aparece se ele estiver entre os mais perto ou for o selecionado.
