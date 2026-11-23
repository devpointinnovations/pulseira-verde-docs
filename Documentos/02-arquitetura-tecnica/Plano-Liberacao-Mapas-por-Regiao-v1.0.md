# Plano de aplicação — Liberação dos mapas por região (região, estado, município e área)

> **Versão:** 1.0 · **Data:** 22/11/2026 · **Status:** 🟡 Proposto; o PO decidiu L1 e L3 em 22/11/2026 (Pedregal primeiro, DF depois, liga/desliga por estabelecimento; §9), L2, L4 e L5 seguem com a recomendação
> **Base de código:** `develop` dos quatro repositórios em 22/11/2026 (backend `39c9d8d`, admin `a6b87f6`, mobile `209b97c`)
> **Amplia:** [`Desenho-Hospitais-por-Estado-v1.0.md`](./Desenho-Hospitais-por-Estado-v1.0.md) (ADR-017, aceito em 13/11/2026).
> Tudo o que lá foi decidido (D1 a D7) continua valendo; este plano troca a unidade de liberação
> "UF" por uma regra com quatro níveis. ADR correspondente: ADR-018 (proposto) em `08-analise tecnica/adrs.md`.
> **Sem código neste documento.**

---

## 1. O pedido

O PO pediu, em 21/11/2026:

1. **Buscar os dados abertos dos estabelecimentos de saúde do Brasil**: o que existe, em que
   formato, com que licença e atualização, e como entra na base.
2. **Gerenciar pelo painel administrativo onde o app funciona**, em qualquer nível: o DF, Goiás
   inteiro, só um pedaço de um município (exemplo do PO: o Pedregal, em Novo Gama/GO) ou uma região
   inteira (exemplo do PO: toda a região Norte).
3. **Um plano de aplicação**: fases, PRs por repositório, riscos e ordem de entrega.

Em 22/11/2026, depois da primeira versão deste plano, o PO fixou o começo: **liberar o Pedregal e o
DF, nessa ordem**, com uma ação no painel para **desligar o DF** quando quiser testar em escala menor,
e o painel com poder de **ligar e desligar cada estabelecimento de cada região**. Só documentação,
sem código.

O ADR-017 já resolveu o difícil: catálogo por UF, catálogo em arquivo no app, UF descoberta no
aparelho, importação do CNES por UF. O que falta é a **regra de liberação** ser mais fina e mais
larga que a UF, e o painel ter a tela que a controla.

### O que seria fracasso mesmo funcionando

- Liberar "região Norte" e o app mostrar, sem revisão, os pontos com coordenada ruim de sete estados
  de uma vez (o critério de liberação do ADR-017 §10 deixa de valer).
- Recolher uma área e quem está numa visita naquele hospital perder a saída.
- Uma mudança no painel derrubar o catálogo do DF para os APKs já instalados.

---

## 2. Fontes de dados abertos de saúde

Conferido nesta sessão (22/11/2026). O ambiente na nuvem só alcança o espelho S3 do OpenDataSUS; os
sites do DATASUS, do IBGE e a API de dados abertos do Ministério recusam conexão pelo proxy. O que
não foi lido aparece como "não conferido".

### 2.1 Estabelecimentos (o que vira hospital no mapa)

| Fonte | O que é | Formato | Atualização | Acesso | Uso no Pulseira Verde |
|---|---|---|---|---|---|
| **CNES – Estabelecimentos (OpenDataSUS)** | Cadastro nacional, uma linha por estabelecimento, com posição | CSV `;`, Latin-1, dentro de zip | Retrato único; o arquivo do espelho é de **11/06/2025** (`Last-Modified`) | `s3.sa-east-1.amazonaws.com/ckan.saude.gov.br/CNES/cnes_estabelecimentos.zip` (50 MB zip, 184,7 MB CSV, 574.006 linhas, 36 colunas) — **baixado e medido** | **Fonte principal da carga.** Tem `CO_UF`, `CO_IBGE` (município, 6 dígitos), `NO_BAIRRO`, `NU_LATITUDE`, `NU_LONGITUDE`, `TP_UNIDADE`, `CO_NATUREZA_JUR`, `CO_MOTIVO_DESAB` |
| **CNES – arquivos de disseminação (FTP DATASUS)** | 13 grupos por UF e mês: ST estabelecimentos, LT leitos, EQ equipamentos, SR serviços especializados, HB habilitações, EP equipes, PF profissionais, DC dados complementares, RC regra contratual, IN incentivos, EE ensino, EF filantrópico, GM gestão e metas | DBC (DBF comprimido) | Mensal, um arquivo por grupo, UF e competência | `ftp.datasus.gov.br/dissemin/publicos/CNES/` — **não conferido** (bloqueado aqui). Espelho não oficial com sincronização diária: `datasus-ftp-mirror.nyc3.digitaloceanspaces.com` | O ST **não traz coordenadas** (documentação do pacote healthbR); serve para **atualização mensal** de situação (ativo/desativado). LT e HB servem para enriquecer o hospital (leitos, UTI) numa fase futura |
| **API de dados abertos do Ministério da Saúde** | `apidadosabertos.saude.gov.br/cnes/estabelecimentos` | JSON paginado | Não conferido | Bloqueada pelo proxy (já em 13/11) | Alternativa ao CSV para consultar uma unidade por código; não para carga nacional |
| **CNES – consulta pública** | `cnes.datasus.gov.br` (fonte citada pelo Ministério no FAQ "Como acessar o Cadastro Nacional") | Site | Contínua | Navegador | Conferência manual de um estabelecimento na revisão de pontos suspeitos |
| **InfoSaúde/GDF** | Camadas do DF (já usadas) | DBF/SHP | Pontual | Já no repositório | **Continua sendo a base do DF** (341 vs 257 do CNES, ADR-017 §3.1) |

**Licença:** os conjuntos do OpenDataSUS e do DATASUS são dados públicos abertos do governo federal
(Política de Dados Abertos, Decreto 8.777/2016). A licença exata de cada conjunto (normalmente
atribuição) **não foi conferida** nesta sessão; registrar a fonte e a data no relatório de cada
importação, como já se faz em `07-dados/`, cobre a atribuição. São dados de estabelecimentos, não de
pessoas; telefone e e-mail do CNES não entram no catálogo público.

**Risco da fonte:** o CSV do espelho tem 17 meses. Antes da primeira carga fora do DF, a Fase 2
precisa de um retrato atual: baixar no desktop do PO pelo portal do OpenDataSUS (que o desktop
alcança) ou cruzar o CSV com o ST mensal mais recente para desligar o que foi desativado depois.

### 2.2 Divisão do território (o que vira região, estado, município)

| Fonte | O que é | Uso |
|---|---|---|
| **IBGE – API de localidades** (`servicodados.ibge.gov.br/api/v1/localidades`) | Regiões (5), UFs (27), municípios com código de 7 dígitos, distritos, subdistritos, regiões imediatas e intermediárias | Tabela de localidades embutida no backend (§4.2). Não conferida nesta sessão |
| **IBGE – API de malhas** (`servicodados.ibge.gov.br/api/v3/malhas`) | Contornos em GeoJSON/TopoJSON por país, região, UF e município, com resolução escolhida | Malha das UFs no app (D4, já decidido) e contorno do município no mapa do painel. Não conferida |
| **IBGE – malhas territoriais anuais** (`geoftp.ibge.gov.br`, shapefile) | Mesma malha, para baixar inteira | Alternativa offline para gerar os arquivos simplificados com `mapshaper`, como já foi feito com as camadas do DF |

Bairro **não é divisão do IBGE**: o IBGE desce até distrito e subdistrito. O Pedregal não é
município nem distrito; é um bairro de Novo Gama (§3.3). Por isso o quarto nível é uma **área
desenhada no painel**, e não uma lista oficial.

### 2.3 O que não buscar agora

"Todos os dados" do CNES são 13 grupos mensais por UF. Para o mapa só um importa: estabelecimentos
com posição. Os demais (profissionais, equipamentos, regras contratuais, incentivos) não mudam nada
que o app mostra hoje. Leitos e habilitações ficam anotados como enriquecimento futuro, a pedido.

---

## 3. Números medidos no CNES para os exemplos do PO

Mesmo corte do ADR-017 §3.1: ativos, natureza jurídica pública, tipos que o app trata.

### 3.1 Por região do IBGE

| Região | UFs | Estabelecimentos no escopo |
|---|---|---|
| Norte | AC, AM, AP, PA, RO, RR, TO | **5.414** |
| Nordeste | 9 | 23.388 |
| Sudeste | 4 | 18.334 |
| Sul | 3 | 8.669 |
| Centro-Oeste | DF, GO, MS, MT | **4.281** |

5.571 códigos de município aparecem no escopo. Com a tabela do ADR-017, o Norte inteiro são 7 UFs e
7 revisões de pontos suspeitos (PA sozinho tem 2.754 unidades e 93 pontos repetidos).

### 3.2 Municípios dos exemplos

| Município | Código CNES (6) | Código IBGE (7) | No escopo |
|---|---|---|---|
| Brasília (o DF inteiro é um município) | 530010 | 5300108 | 257 (base atual: 341) |
| Goiânia/GO | 520870 | 5208707 | 133 |
| Novo Gama/GO | 521523 | 5215231 | 27 |
| Luziânia/GO | 521250 | — | 48 |
| Águas Lindas/GO | 520025 | — | 31 |
| Valparaíso/GO | 521975 | — | 18 |

O 7º dígito é o verificador do IBGE; os códigos de 7 dígitos acima devem ser conferidos na tabela de
localidades quando ela entrar no backend (Fase 1).

### 3.3 Pedregal

No CNES, Novo Gama tem 27 unidades no escopo. Seis do bairro trazem "PEDREGAL" ou "VALE DO PEDREGAL"
no campo `NO_BAIRRO`, e uma delas se chama "USF Vila Zequinha", mas está cadastrada no bairro
Pedregal. O nome do bairro é texto livre e não serve de regra.

O centro das unidades do bairro fica em **−16,0587, −48,0344**. Um círculo de **1,5 km** pega cinco das
seis unidades no escopo; a USF Vale do Pedregal fica a 2,9 km e exige 3 km. É o exemplo de por que a
área é uma forma no mapa revisada pelo ADMIN, com a prévia do que entra (§6.2).

---

## 4. Modelo de dados

### 4.1 A regra de liberação

Nova coleção `liberacoes` no MongoDB. Uma regra por lugar:

```text
liberacoes {
  _id,
  nivel:        REGIAO | UF | MUNICIPIO | AREA | ESTABELECIMENTO,
  codigo:       "1" (Norte) | "52" (GO) | "5215231" (Novo Gama) | "area-pedregal-novo-gama"
                | id do hospital (ESTABELECIMENTO),
  acao:         LIBERAR | BLOQUEAR,
  nome:         "Região Norte" | "Goiás" | "Novo Gama" | "Pedregal (Novo Gama)",
  uf:           "GO"           (MUNICIPIO e AREA; nulo para REGIAO; a própria UF para UF)
  area:         { centro: Point(lon, lat), raioMetros: 1500 }   (só AREA)
  motivo:       texto curto obrigatório
  criadoPor, criadoEm, atualizadoPor, atualizadoEm
}
índice único { nivel: 1, codigo: 1 }
```

- `BLOQUEAR` existe para abrir o todo e fechar uma parte: "Norte liberado, menos o Amapá".
- **ESTABELECIMENTO** é o liga/desliga de um hospital, UPA ou UBS dentro de uma região, pedido pelo
  PO em 22/11/2026. Não é o mesmo que o `ativo` que o painel já tem (`PATCH` de status, E7-07):
  `ativo = false` diz que a unidade não existe mais ou não funciona, e vale em todo lugar;
  `ESTABELECIMENTO BLOQUEAR` diz só "não aparece no app por enquanto" (ponto em revisão, teste em
  escala menor) e some quando a regra é removida. Os dois continuam separados.
- O histórico fica em `liberacoes_historico` (só inserção: quem, quando, antes, depois, motivo).
  Volta atrás vira uma regra nova, nunca apagar o histórico.
- **Área é um círculo** (centro + raio) nesta versão: o Leaflet do painel desenha círculo sem
  plugin, o MongoDB testa com `$centerSphere`, e o raio se ajusta olhando a prévia.
  `shortcut: círculo não segue a forma do bairro; trocar por polígono (GeoJSON) se uma área real precisar e o círculo pegar unidade de bairro vizinho.`

### 4.2 O que muda em `hospitais` e no backend

- `endereco.codigoMunicipio` (7 dígitos do IBGE), novo. Hoje só existe `endereco.cidade` em texto.
  O importador preenche a partir de `CO_IBGE`; o DF recebe `5300108` numa reconciliação de uma vez
  (mesmo padrão do `ReconciliacaoRegiaoAdministrativaRunner`).
- Índice `{ ativo: 1, 'endereco.uf': 1, 'endereco.codigoMunicipio': 1 }` (substitui o índice
  `{ ativo, endereco.uf }` previsto no ADR-017 §4.1).
- **Tabela de localidades no classpath** (regiões, UFs, municípios: código, nome, UF, região), gerada
  da API de localidades do IBGE e versionada, como as camadas do DF em `RegiaoService`. Muda uma vez
  por ano. Sem coleção nova e sem chamada ao IBGE em produção.

### 4.3 Herança entre níveis

Para cada estabelecimento, vale a regra **mais específica** que o alcança:

```text
0. ESTABELECIMENTO regra do próprio hospital
1. ÁREA      círculos que contêm o ponto  → se algum BLOQUEAR: fechado; se algum LIBERAR: aberto
2. MUNICÍPIO regra do endereco.codigoMunicipio
3. UF        regra do endereco.uf
4. REGIÃO    regra da região da UF (1º dígito do código da UF)
5. padrão    fechado
```

O que estiver com `ativo = false` fica fora antes de tudo isso, como hoje.

Entre duas áreas sobrepostas, `BLOQUEAR` vence (o lado seguro). O cálculo roda em memória, na
montagem do catálogo da UF: no máximo 7.220 estabelecimentos (MG) contra algumas dezenas de regras.
Não existe campo "liberado" gravado no hospital, então ligar ou desligar não exige reprocessar a base.

### 4.4 Exemplos

| O PO quer | Regras no painel | O que o app mostra |
|---|---|---|
| Só o DF (hoje) | `UF 53 LIBERAR` | DF inteiro. Igual ao atual |
| DF + Pedregal | `UF 53 LIBERAR` · `AREA Pedregal (Novo Gama) 1,5 km LIBERAR` | DF inteiro; em GO, só as unidades dentro do círculo |
| Goiás inteiro | `UF 52 LIBERAR` | 1.947 unidades de GO |
| Goiás, menos um município | `UF 52 LIBERAR` · `MUNICIPIO X BLOQUEAR` | GO sem o município X |
| Só Goiânia | `MUNICIPIO 5208707 LIBERAR` | 133 unidades; o resto de GO fechado |
| Toda a região Norte | `REGIAO 1 LIBERAR` | 5.414 unidades em 7 UFs |
| Norte, menos o Amapá até a revisão | `REGIAO 1 LIBERAR` · `UF 16 BLOQUEAR` | 6 UFs; AP mostra "ainda não chegou" |
| Amapá fechado, mas Macapá aberta | o anterior + `MUNICIPIO Macapá LIBERAR` | só Macapá no AP |
| DF aberto, menos um hospital em revisão | `UF 53 LIBERAR` · `ESTABELECIMENTO <id> BLOQUEAR` | DF sem aquele hospital |
| Pedregal fechado, mas uma UBS dele aberta | `ESTABELECIMENTO <id> LIBERAR` | só aquela UBS em GO |

### 4.5 A ordem decidida pelo PO: Pedregal primeiro, DF depois

| Etapa | Regras | O que o app mostra | No painel |
|---|---|---|---|
| **1. Só o Pedregal** (teste em escala menor) | `AREA Pedregal (Novo Gama) LIBERAR` · `UF 53 BLOQUEAR` | Em GO, as unidades do círculo; no DF, a tela "ainda não chegou" | Liga a área; desliga o DF |
| **2. Pedregal + DF** | `AREA Pedregal LIBERAR` · `UF 53 LIBERAR` | Pedregal e o DF inteiro | Liga o DF |
| **Voltar ao teste menor** (quando quiser) | `UF 53 BLOQUEAR` de novo | Volta à etapa 1 | Desliga o DF |
| **Tirar um estabelecimento** (qualquer etapa) | `ESTABELECIMENTO <id> BLOQUEAR` | Some só aquele | Desliga a linha dele |

Desligar o DF é uma regra, não apaga nada: os 341 estabelecimentos, as visitas e as avaliações ficam
no banco, e ligar de novo devolve tudo como estava. Quem estiver no meio de uma visita num hospital do
DF quando ele for desligado termina a visita normalmente (§5.2).

---

## 5. API

### 5.1 Painel (ADMIN, `/api/v1/admin/...`)

| Rota | Faz |
|---|---|
| `GET /admin/liberacoes` | Lista as regras vigentes |
| `PUT /admin/liberacoes/{nivel}/{codigo}` | Cria ou troca a regra (`acao`, `motivo`, `area` quando AREA). Grava o histórico |
| `DELETE /admin/liberacoes/{nivel}/{codigo}` | Remove a regra: o lugar volta a herdar do nível de cima. Grava o histórico |
| `POST /admin/liberacoes/previa` | Recebe a regra que o ADMIN vai salvar e devolve, sem gravar: quantos estabelecimentos entram e saem, por UF e município, quantos pontos suspeitos e quais UFs ainda não têm importação |
| `GET /admin/liberacoes/historico?nivel=&codigo=` | Histórico paginado |
| `GET /admin/liberacoes/estabelecimentos?nivel=&codigo=` | Os estabelecimentos de um lugar (região, UF, município ou área), paginados, cada um com a situação efetiva e de qual regra ela vem. É a lista do liga/desliga por estabelecimento (§6.1) |

O liga/desliga de um estabelecimento usa o mesmo `PUT`/`DELETE` com `nivel = ESTABELECIMENTO`.
Ligar ou desligar o DF inteiro é `PUT /admin/liberacoes/UF/53` com `LIBERAR` ou `BLOQUEAR`.
| `GET /admin/localidades/regioes`, `/ufs?regiao=`, `/municipios?uf=` | Da tabela do classpath, com a contagem de estabelecimentos importados e a situação de cada um (aberto, fechado, herdado de onde) |

Validações: `codigo` existe na tabela de localidades; `motivo` obrigatório; raio entre 200 m e 20 km;
área precisa cair dentro da UF informada; **liberar um lugar sem nenhum estabelecimento importado é
recusado** (409), para não abrir uma UF vazia.

### 5.2 Público (o app)

- **Configuração remota** ganha `coberturas` no lugar do `ufsLiberadas` do ADR-017:

  ```json
  "coberturas": [
    { "uf": "DF", "total": true,  "versaoCatalogo": "…" },
    { "uf": "GO", "total": false, "versaoCatalogo": "…", "locais": ["Pedregal (Novo Gama)"] }
  ]
  ```

  `versaoCatalogo` (o campo único de hoje) continua com a versão do DF, para os APKs instalados.
- `GET /api/v1/catalogo?uf=GO`, indicadores e ranking: como no ADR-017 §4.1, mas o filtro é a regra
  da §4.3, não só a UF. Sem `uf`, só o DF, sempre, **e também pela regra**: com o DF desligado, o
  pedido sem `uf` devolve o catálogo vazio. Os APKs instalados hoje (rc.17 e anteriores) não sabem
  mostrar "ainda não chegou" e ficariam com a lista vazia; por isso a etapa 1 da §4.5 só começa com a
  versão mínima do app (`versaoMinimaApp`, já existente) apontando para a versão da Fase 6, e esses
  APKs passam a mostrar o aviso de atualização. A versão de cada catálogo continua sendo o SHA-256
  do conteúdo, então qualquer mudança de regra muda a versão sozinha.
- Busca por raio e detalhe: escondem o que estiver fechado.
- **Check-in novo** em estabelecimento fechado: 409 `FORA_DA_COBERTURA`. Hoje a D5 ("visita em UF não
  liberada não registra") depende só de o app não ter o geofence; com regras finas, o servidor
  precisa garantir. **Saída e sinal de uma visita já aberta são sempre aceitos**, mesmo que o lugar
  tenha sido fechado no meio dela.

### 5.3 Quanto tempo uma mudança leva para chegar

O catálogo por UF fica 5 min em cache por instância e a configuração remota é lida ao abrir o app e
ao voltar ao primeiro plano. Na instância que recebe o `PUT`, o cache é limpo na hora; nas outras, em
até 5 min. A tela do painel avisa: "vale para todos em até 5 minutos". Sem fila ou evento entre
instâncias nesta versão.

---

## 6. Painel administrativo

### 6.1 Tela "Cobertura" (nova rota `/cobertura`)

```text
┌ Cobertura do app ─────────────────────────────────────────────┐
│ Região ▸ UF ▸ Município          │  [mapa Leaflet]            │
│ ▾ Centro-Oeste   herdado: fechado│   UF e municípios abertos  │
│   ● DF   ABERTO (regra UF)  341  │   em verde, áreas como     │
│   ◐ GO   PARCIAL           1.947 │   círculos                 │
│     ▸ Novo Gama  fechado     27  │                            │
│       ◉ Pedregal ABERTO (área) 5 │                            │
│ ▸ Norte          fechado   5.414 │                            │
│ [Liberar] [Bloquear] [Voltar a herdar] [+ Área no mapa]       │
└───────────────────────────────────────────────────────────────┘
```

- No topo, um interruptor fixo **"DF no app: ligado/desligado"**, o atalho para a etapa 1 ↔ 2 da §4.5,
  com a prévia ("vai tirar 341 estabelecimentos do app") e a confirmação de sempre.
- Cada linha mostra a situação efetiva e **de onde ela vem** ("herdado da região", "regra própria").
- Abrir uma linha (região, UF, município ou área) mostra os **estabelecimentos daquele lugar**, com
  busca e filtro por tipo, e um interruptor em cada um. Desligar um cria a regra `ESTABELECIMENTO
  BLOQUEAR`; ligar de volta num lugar aberto remove a regra (volta a herdar). A linha mostra quando
  o estado do estabelecimento é diferente do lugar ("desligado à mão em 22/11 por …").
- "+ Área no mapa": clicar no centro, arrastar o raio, ver os pontos que entram.
- **Toda gravação passa pela prévia** (§5.1): "Vai abrir 5.414 estabelecimentos em 7 UFs; 3 UFs sem
  revisão de pontos suspeitos". Liberar uma região ou UF com revisão pendente pede confirmação
  digitando o nome do lugar.
- Aba de histórico com quem mudou, quando e por quê.

### 6.2 Telas existentes

- Hospitais, Mapa e Dashboard: filtro por UF e município (ADR-017 §6) e a marca "fora da cobertura"
  em cada estabelecimento fechado, para o ADMIN revisar antes de abrir.
- Detalhe do hospital: ganha o mesmo interruptor "aparece no app", ao lado do "ativo" que já existe,
  com o texto que diferencia os dois (§4.1).

---

## 7. Impacto no app e na carga de hospitais

### 7.1 App

Quase nada além do ADR-017 Fase 3, porque o servidor já entrega o catálogo da UF filtrado:

- UF com `total: false`: o topo da lista e do mapa diz "Em Goiás, por enquanto: Pedregal (Novo Gama)".
- UF fora de `coberturas`: a tela "O Pulseira Verde ainda não chegou em <estado>" do ADR-017 §5.4.
- Geofencing, união dos catálogos e fila offline: sem mudança. O que está fechado não vem no catálogo,
  então não vira geofence.
- O app **não precisa** da malha dos municípios: só a das UFs (D4) para saber qual catálogo baixar.
- Check-in guardado na fila offline que volta com 409 `FORA_DA_COBERTURA`: sai da fila com o aviso
  "esse local ainda não está no Pulseira Verde", como os outros 409 de hoje.

### 7.2 Carga de hospitais

- **Importar e liberar são passos separados.** A importação do CNES (ADR-017 Fase 2) traz a UF
  inteira para o banco, fechada. A liberação abre o pedaço que o ADMIN escolher. Assim, liberar o
  Pedregal exige importar GO inteiro (1.947 unidades, ~4 MB no banco), o que é barato e já deixa GO
  pronto para revisão.
- O importador passa a gravar `codigoMunicipio` e a calcular os pontos suspeitos (repetidos, fora do
  polígono da UF) por município, para a prévia mostrar onde está o problema.
- Norte inteiro no banco: 5.414 documentos × ~2 KB ≈ 11 MB. O teto do Atlas M0 (ADR-017 §3.3)
  continua longe.

---

## 8. Caminhos considerados

| Caminho | Como | Veredito |
|---|---|---|
| **A. Regras por nível com herança** (§4) | Poucas regras; o filtro roda na montagem do catálogo | **Recomendado**: cobre todos os exemplos do PO com 1 a 3 regras cada |
| B. Marcar cada hospital como liberado | Campo `liberado` em cada documento, atualizado em lote | Descartado: liberar o Norte grava 5.414 documentos; hospital importado depois nasce sem saber se deveria estar aberto |
| C. Só UF e região (sem município e área) | O `ufsLiberadas` do ADR-017 mais uma lista de regiões | Mais barato (~3 dias a menos), mas não atende o Pedregal nem "só Goiânia" |
| D. Área como lista de bairros do CNES | Liberar pelo texto de `NO_BAIRRO` | Descartado: o texto é livre ("PEDREGAL", "VALE DO PEDREGAL", USF de outro nome no bairro) |

---

## 9. Decisões do PO

| # | Pergunta | Recomendação |
|---|---|---|
| **L1** | Os quatro níveis (região, UF, município, área) entram já, ou só UF e região agora? | ✅ **Decidido pelo PO em 22/11/2026:** os quatro, **mais o liga/desliga por estabelecimento** de cada região (nível ESTABELECIMENTO, §4.1) |
| **L2** | Área: círculo agora e polígono depois, ou polígono já? | Círculo agora (sem plugin, prévia mostra o que entra) |
| **L3** | Primeira liberação fora do DF: GO inteiro (D6) ou só o Pedregal como piloto? | ✅ **Decidido pelo PO em 22/11/2026:** Pedregal primeiro, DF depois, com o DF podendo ser desligado no painel para testes em escala menor (§4.5). GO inteiro fica para depois da revisão |
| **L4** | Liberar com revisão de pontos suspeitos pendente: bloquear ou permitir com confirmação? | Permitir com confirmação digitada, registrada no histórico |
| **L5** | Retrato atual do CNES (§2.1): baixar no desktop pelo OpenDataSUS ou cruzar com o ST mensal? | Baixar no desktop: é um passo manual por mês e não exige ler DBC |

---

## 10. Fases e PRs

Uma branch por PR, saindo da `develop`, sem empilhar. **Backend antes do painel, painel antes do app.**
O app só começa depois que os PRs da migração de layout (#6 a #9 e os fluxos 5 e 6) estiverem
mergeados, porque mexe nas mesmas telas de lista e mapa, e o PR #7 (offline) mexe no cache do
catálogo.

| Fase | Repositório | Branch | Entrega | Dias |
|---|---|---|---|---|
| 0 | — | — | Teste no aparelho da leitura de um catálogo de 3 MB (ADR-017 Fase 0) | — |
| 1 | backend | `feature/catalogo-por-uf` | ADR-017 Fase 1 (catálogo, indicadores e ranking por `uf`, DF como padrão) **mais** tabela de localidades no classpath, `endereco.codigoMunicipio`, reconciliação do DF para `5300108` e índice novo | 4 |
| 2 | backend | `feature/liberacao-por-regiao` | Coleção `liberacoes` e histórico, resolvedor da §4.3 com o nível ESTABELECIMENTO, rotas do §5.1, `coberturas` na configuração remota, catálogo filtrado pela regra (inclusive o pedido sem `uf`), 409 `FORA_DA_COBERTURA` no check-in novo, regra inicial `UF 53 LIBERAR` criada na subida se a coleção estiver vazia (o app continua como hoje até o ADMIN mudar) | 4,5 |
| 3 | backend + docs | `feature/importador-cnes-por-uf` / `doc/importacao-cnes-go` | ADR-017 Fase 2 com `codigoMunicipio` e pontos suspeitos por município; relatório de GO em `07-dados/` | 4 |
| 4 | admin | `feature/tela-cobertura` | Tela Cobertura (§6.1) com o interruptor do DF, árvore, mapa, área em círculo, lista de estabelecimentos com liga/desliga, prévia, confirmação e histórico | 5 |
| 5 | admin | `feature/filtro-uf-municipio` | ADR-017 Fase 4 (filtros e dashboard por UF/município) e a marca "fora da cobertura" | 2 |
| 6 | mobile | `feature/catalogo-por-uf` | ADR-017 Fase 3 mais o aviso de cobertura parcial e o tratamento do 409 na fila | 9 a 11 |
| 7 | — | homologação | rc do backend (Fases 1–3), depois rc do app (Fase 6) com o par anotado. Na homologação, nesta ordem: (a) `versaoMinimaApp` na versão da Fase 6; (b) etapa 1 da §4.5, só o Pedregal, com o DF desligado; teste em campo no Pedregal e na divisa; (c) etapa 2, ligar o DF e conferir que o DF volta inteiro (341) sem perder visitas e avaliações; (d) desligar e religar um estabelecimento | 2 + campo |

Em cada PR de código, o PR correspondente no `pulseira-verde-docs` atualiza a
`Especificacao-API-v2.4.md`, o `De-Para-Backlog-Features.md` e o `Historico-Melhorias.md` só com o que
foi entregue.

**Total:** cerca de 30 a 33 dias até o piloto do Pedregal em campo, contra 21 a 23 do ADR-017 só por
UF. A diferença (≈ 9 a 10 dias) é o custo dos níveis município, área e estabelecimento e da tela
Cobertura.

---

## 11. Riscos

| Risco | Efeito | Mitigação |
|---|---|---|
| CSV do CNES velho (jun/2025) | Unidades fechadas depois aparecem no mapa | L5; relatório da importação traz a data do retrato |
| Liberar região inteira sem revisão | Pontos de centroide viram conflito de check-in em 7 UFs | Prévia com pontos suspeitos e confirmação digitada (L4) |
| Cache de 5 min por instância | ADMIN fecha uma área e alguns aparelhos ainda veem por minutos | Aviso na tela; check-in fechado já é recusado pelo servidor |
| APKs antigos | Catálogo muda sob eles | Sem `uf` → só DF, nunca outra UF (ADR-017 §4.1); `versaoCatalogo` mantido |
| DF desligado com APKs antigos instalados | Lista vazia, sem explicação, para os testadores da rc | `versaoMinimaApp` antes da etapa 1 (§5.2) |
| Confundir "aparece no app" com "ativo" | ADMIN desativa um hospital que só queria esconder, e ele some do histórico do painel | Dois interruptores com textos diferentes (§6.2) |
| Conflito com a migração de layout no app | Retrabalho nas telas de lista e mapa | Fase 6 só depois dos PRs de layout mergeados |
| Círculo pega bairro vizinho | Unidade errada aberta | Prévia lista as unidades; polígono como evolução (L2) |
| Código de município com 6 dígitos no CNES | Junção errada com o IBGE | Tabela de localidades faz a ponte 6 → 7 dígitos, com teste |

---

## 12. O que fica para quem implementar

- Nomes exatos de classes e campos, desde que mantenham as rotas e a compatibilidade da §5.2.
- Onde a prévia calcula (mesmo resolvedor da §4.3, sem duplicar regra).
- O formato da tabela de localidades no classpath (CSV ou JSON), desde que venha da API do IBGE com
  a data registrada.
