# Auditoria de Arquitetura e Padrões — Pulseira Verde v1.0

**Data:** 23/11/2026 · **Pedido do PO:** "verifique que modelo ela segue: MVC, SOA, Eventos, DDD, entre outros"
**Tipo:** auditoria de leitura. Nenhum código foi alterado, nenhum teste ou build foi executado.
**Base medida:** `pulseira-verde-backend@39c9d8d`, `pulseira-verde-mobile@d3bd6db`, `pulseira-verde-admin@a6b87f6`, `pulseira-verde-docs@181ecab` (develop de cada repositório em 23/11/2026).
**Método:** contagem de arquivos e linhas, grafo de `import` entre pacotes/pastas, busca por anotações e APIs (`@TransactionalEventListener`, `@Transactional`, `TaskManager.defineTask` etc.) e leitura dos trechos citados. Toda afirmação abaixo aponta o arquivo de onde saiu.

---

## 1. Resumo executivo

**O sistema é um monólito em camadas, organizado por funcionalidade, com três clientes separados e integração síncrona por REST/JSON.** Não é SOA, não é microsserviços, não é DDD tático, não é hexagonal nem CQRS. Usa eventos em um único ponto, dentro do próprio processo.

| Pergunta do PO | Resposta curta |
|---|---|
| **MVC?** | **Sim, no backend**, na forma usual do Spring: `@RestController` → `Service` → `Repository` (Spring Data MongoDB) → `Document`. Sem *views* no servidor: o "V" está nos clientes. |
| **SOA / microsserviços?** | **Não.** Um único serviço Spring Boot (`settings.gradle`: só `saude-monitor`), um banco, um deploy no Cloud Run com `--max-instances=1`. Os quatro repositórios dividem **entregáveis**, não serviços. |
| **Eventos?** | **Muito pouco, e só dentro do processo.** Há um evento (`FeedbackSalvoEvent`) e um ouvinte. O restante do trabalho assíncrono é **polling por agendamento** (5 jobs, `@Scheduled`/`POST /internal/jobs`). Sem fila, sem broker, sem *outbox*. |
| **DDD?** | **Só no nível estratégico e parcial.** Os pacotes seguem contextos com nomes do negócio (`visita`, `feedback`, `indicador`, `hospital`), em português (linguagem ubíqua). No nível tático não: as 12 classes `*Document` são modelos anêmicos (só `@Getter/@Setter/@Builder`, nenhum método de regra), as regras moram nos `*ServiceImpl`, e os módulos leem os repositórios uns dos outros. |
| **Hexagonal / Clean?** | **Não.** Os serviços dependem direto do Spring Data e do `MongoTemplate` (13 arquivos); não há portas nem adaptadores. |
| **CQRS?** | **Não formal.** Existe um *read model* materializado (`IndicadorHospitalDocument`, recalculado por evento e job) e um catálogo cacheável com ETag. É separação de leitura pontual, não CQRS. |
| **App** | Arquitetura **offline-first** madura (fila persistida, envio em segundo plano, catálogo local, cache das consultas no aparelho), com **TanStack Query** para estado de servidor e **Zustand** para a sessão. Pastas ainda por tipo dentro de cada tela (ADR-008 pela metade). |
| **Painel** | **Angular 21 standalone, zoneless, com signals e rotas *lazy*** separadas em `core/` (serviços) e `features/` (telas). Aderente ao ADR-012. |

**O que mais preocupa (detalhe na §7):**

1. **Check-in pode duplicar visita ativa** sob concorrência: lê "existe visita ativa?" e depois grava, sem índice único que impeça a segunda (`VisitaServiceImpl.java:121-160`). O app tem duas fontes que podem disparar ao mesmo tempo (geofencing e fila offline).
2. **Ciclos de dependência entre módulos** (`visita ⇄ feedback`, `feedback ⇄ indicador`, `hospital ⇄ indicador`, `auth ⇄ user`, `config ⇄ user/visita`). `indicador` lê direto os repositórios de `visita`, `feedback` e `hospital`. Um "monólito modular" de fato não tem esses ciclos.
3. **Estado em memória da instância** (rate limit em `ConcurrentHashMap`, cache do evento `@Async`) combinado com `--max-instances=1`: escalar horizontalmente muda comportamento.
4. **Nenhuma escrita é transacional** (nenhum `@Transactional` nem `MongoTransactionManager`). Hoje isso é contornado pela ordem das gravações (a exclusão de conta anonimiza visitas e feedbacks antes de apagar a conta, então uma falha no meio pode ser repetida), mas a garantia depende de cada autor lembrar disso.
5. **Os documentos de arquitetura estão defasados** em relação ao código: o resumo de status dos ADRs marca como "Proposto" coisas que já estão no código, não existe ADR-016, e nenhum ADR declara o estilo do backend.

**Recomendação central:** assumir oficialmente o estilo que o código já segue (**monólito modular em camadas, por funcionalidade**) num ADR novo, e então torná-lo verdadeiro: eliminar os ciclos, dar a cada módulo uma fachada pública, fechar o check-in duplicado com índice único e registrar a regra de ordem das gravações enquanto não houver transação. Nada disso pede microsserviços, filas ou reescrita (§8).

---

## 2. Visão geral do sistema (medida)

```
 ┌───────────────────────┐      ┌───────────────────────┐
 │  App (Expo / RN 0.83) │      │ Painel (Angular 21)   │
 │  JS, 98 arquivos      │      │ TS, 24 arquivos       │
 │  TanStack Query+Zustand│     │ signals, zoneless     │
 │  fila offline + tasks │      │ Leaflet               │
 └──────────┬────────────┘      └──────────┬────────────┘
   Bearer JWT (SecureStore)       access em memória + refresh
            │ HTTPS/JSON           em cookie __Host- (ADR-013)
            ▼                                ▼
 ┌──────────────────────────────────────────────────────┐
 │  Backend Spring Boot 4 (WebMVC, Java 25) — 1 processo│
 │  202 arquivos Java, 12 controllers, ~46 endpoints    │
 │  auth · user · hospital · visita · feedback ·        │
 │  indicador · regiao · configuracao · jobs · config   │
 │  Cloud Run: min=0, max=1 instância                   │
 └──────────────────────────┬───────────────────────────┘
                            ▼
                 ┌─────────────────────┐
                 │ MongoDB (Atlas)     │  2dsphere, travas_execucao
                 └─────────────────────┘
   Cloud Scheduler (opcional) ──OIDC──▶ POST /internal/jobs/{nome}
```

Fontes: `settings.gradle`, `build.gradle`, contagem com `find`, os mapeamentos dos 12 controllers (11 da API + `JobsController`), `.github/workflows/cd-backend-google.yml:199-200` do backend.

---

## 3. Tabela padrão × evidência × aderência

Legenda: ✅ segue · 🟡 parcial · ❌ não segue · ➖ não se aplica

| # | Padrão | Onde se aplicaria | Evidência no código | Aderência |
|---|---|---|---|:---:|
| P1 | MVC / arquitetura em camadas | Backend | Cada módulo tem `controller/`, `service/` (+`impl/`), `repository/`, `document/`, `dto/`. Controllers não importam repositórios; só 2 importam *enums* de `document` (`HospitalController.java:5-7`, `AdminHospitalController.java:3`). Respostas por DTO/`record`. | ✅ |
| P2 | Organização por funcionalidade (*package by feature*) | Backend, Painel | Backend: `api/{auth,user,hospital,visita,feedback,indicador,regiao,configuracao,jobs}`. Painel: `features/*` + `core/*`. | ✅ |
| P3 | Monólito modular | Backend | Um módulo Gradle, um deploy. Mas há ciclos entre módulos e acesso a repositório alheio (§4.2). | 🟡 |
| P4 | SOA / microsserviços | Sistema | Um serviço só; os repositórios separam clientes, não serviços. Decisão explícita de não usar (Análise de Escala §12.3). | ➖ (não adotado, de propósito) |
| P5 | DDD estratégico (contextos, linguagem ubíqua) | Backend | Pacotes e classes com nomes do negócio em português (`Visita`, `PresencaVisita`, `TipoPermanencia`, `StatusVisita.GPS_INTERROMPIDO`). Pacote `agregado` renomeado para `indicador` para não confundir com o *aggregate* do DDD (Especificação API v2.4 §2). Fronteiras furadas (§4.2). | 🟡 |
| P6 | DDD tático (agregados, entidades ricas, objetos de valor) | Backend | 12 `*Document` só com `@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor`, zero método de regra. Regras nos `*ServiceImpl` (VisitaServiceImpl 665 linhas, HospitalServiceImpl 651). Ilhas de regra fora dos serviços: `PresencaVisita`, `VisitaDono`, `GeofenceValidator`. | ❌ (modelo anêmico) |
| P7 | Hexagonal / Clean Architecture | Backend | Serviços dependem de `MongoRepository` e `MongoTemplate` (13 arquivos usam `MongoTemplate`). Os `Document` do Mongo são o próprio modelo de domínio. Sem portas/adaptadores. | ❌ |
| P8 | Interface + implementação de serviço | Backend | 8 pares `XService`/`XServiceImpl` com uma implementação cada. | ✅ (convencional; sem ganho de troca) |
| P9 | Orientado a eventos | Backend | 1 evento (`indicador/event/FeedbackSalvoEvent.java`), publicado em `FeedbackServiceImpl.java:131,175`, ouvido por `FeedbackSalvoEventListener` (`@Async` + `@TransactionalEventListener(AFTER_COMMIT, fallbackExecution = true)`). Sem broker. | 🟡 (pontual, em memória) |
| P10 | Processamento agendado (*polling*) | Backend | 5 jobs em `jobs/AgendamentoInternoJobs.java` (indicadores, expiração, GPS interrompido, feedback sem resposta a cada 15 min; retenção às 03:30) com `TravaDistribuida` e disparo externo por `POST /internal/jobs/{nome}` com OIDC. | ✅ |
| P11 | CQRS | Backend | *Read model* materializado `IndicadorHospitalDocument`; catálogo `GET /api/v1/catalogo` com ETag/`s-maxage`. Mesmo modelo para escrever e ler no resto. | 🟡 (separação pontual) |
| P12 | Consistência transacional | Backend | Nenhum `@Transactional` e nenhum gerenciador de transação no código. Checkout usa atualização condicional (`atualizarSeAtiva`), atômica num documento; a exclusão de conta depende da ordem das gravações (§4.4). | 🟡 |
| P13 | Idempotência / *at-least-once* | Backend + App | App reenvia com `ocorridoEm` no corpo (`config/filaOffline.js`); check-in devolve a visita ativa existente (`VisitaServiceImpl.java:127-135`). Falta garantia no banco (§7, R2). | 🟡 |
| P14 | Tratamento de erro centralizado | Backend | `config/exception/GlobalExceptionHandler` + hierarquia `ApiException` (8 subclasses). `ConflitoGeofenceException` fica fora da hierarquia (`visita/service/`). | ✅ |
| P15 | Segurança *stateless* (JWT) | Backend | `SecurityConfig` com resource server, `JwtAuthenticationFilter`, chaves com `kid` (`ChavesJwt`), cadeia separada para `/internal/**` (`JobsSecurityConfig`). Rotas públicas de visita/feedback anônimo (`SecurityConfig.java:82-90`). | ✅ |
| P16 | Offline-first | App | Fila persistida (`config/filaOffline.js`, teto 50 itens, validade 24 h), reenvio (`services/SincronizacaoOffline.js`), envio com app fechado (`services/EnvioFilaSegundoPlano.js`, `expo-background-task`), catálogo local (`services/CatalogoLocal.js`), cache do TanStack Query persistido (`core/query/cachePersistido.js`). | ✅ |
| P17 | Estado de servidor × estado de cliente | App | TanStack Query (`core/query/*`, `queryKeys.js`) para dados do servidor; Zustand só para a sessão (`core/stores/sessaoStore.js`). | ✅ |
| P18 | Cliente HTTP único | App, Painel | App: `core/api/apiClient.js` + `config/http.js` (único `fetch(` do app, linha 178). Painel: `HttpClient` + `authInterceptor` (`app.config.ts:13`). | ✅ |
| P19 | Feature-first no app | App | Existe `src/core/` (api, query, stores), mas as telas ainda são `screens/<área>/{service,view,hooks}` e há 8 das 10 áreas importando outras áreas (§5.2). | 🟡 |
| P20 | Componentes standalone, signals, *lazy loading* | Painel | Sem `NgModule`; 9 componentes, 9 arquivos com `signal(`; todas as rotas com `loadComponent` (`app.routes.ts`); sem `zone.js` nas dependências (zoneless). | ✅ |
| P21 | Contrato de API compartilhado | Sistema | Backend publica OpenAPI (`springdoc-openapi`), mas app (JS) e painel (`hospital.models.ts`) escrevem os tipos à mão. Nenhum teste de contrato. | ❌ |
| P22 | Escala horizontal sem estado na instância | Backend | Rate limit em `ConcurrentHashMap` (`RateLimitService.java:57`); evento `@Async` em memória; `--max-instances=1` (`cd-backend-google.yml:200`). | ❌ (hoje mascarado pelo limite de 1 instância) |

---

## 4. Backend em detalhe

### 4.1 Camadas e tamanho por módulo

| Módulo | Arquivos | Linhas | Observação |
|---|---:|---:|---|
| `hospital` | 52 | 4.262 | Maior módulo: CRUD, sugestões, catálogo, seed DBF/SHP, migração CNES |
| `config` | 36 | 2.141 | Infraestrutura transversal: segurança, rate limit, exceções, trava, contenção |
| `visita` | 28 | 1.507 | Núcleo do produto (check-in/out, heartbeat, expiração) |
| `feedback` | 17 | 1.469 | Avaliação pós-saída + gerador de massa |
| `auth` | 23 | 1.354 | Login, refresh, painel, verificação de e-mail |
| `user` | 19 | 1.232 | Conta, LGPD (exclusão, exportação JSON/PDF) |
| `indicador` | 9 | 646 | *Read model* público |
| `jobs` | 6 | 322 | Disparo de jobs |
| `configuracao` | 7 | 318 | Configuração remota do app |
| `regiao` | 4 | 304 | Camadas geográficas |

Dentro de cada módulo a forma é a mesma, e é o MVC clássico do Spring:

```
Controller (@RestController, DTO record, @Valid)
    │
Service (interface) ── ServiceImpl (regra de negócio)
    │
Repository (Spring Data) / MongoTemplate
    │
Document (@Document, Lombok, sem comportamento)
```

### 4.2 Fronteiras entre módulos (grafo de imports medido)

Contagem de `import br.com.saude_monitor.api.<outro>`:

```
auth ──10──▶ user          user ──1──▶ auth           (ciclo)
visita ──1──▶ feedback     feedback ──10──▶ visita    (ciclo)
feedback ──2──▶ indicador  indicador ──2──▶ feedback  (ciclo)
hospital ──3──▶ indicador  indicador ──3──▶ hospital  (ciclo)
config ──7──▶ user         user ──7──▶ config         (ciclo)
config ──2──▶ visita       visita ──7──▶ config       (ciclo)
```

Acesso direto a **repositório ou documento de outro módulo** (fora seed): `AuthServiceImpl` → `UserRepository`; `FeedbackServiceImpl` → `VisitaRepository`, `HospitalRepository`; `IndicadorServiceImpl` → `VisitaRepository`, `FeedbackRepository`, `HospitalRepository`; `VisitaServiceImpl` → `HospitalRepository`; `UserServiceImpl` → `VisitaDocument`, `FeedbackDocument` (por `MongoTemplate.updateMulti`); `config/security/*` → `UserRepository`.

**Leitura:** os nomes dos pacotes desenham contextos, mas o código os atravessa pelo banco. Quem muda um campo de `VisitaDocument` precisa conferir `feedback`, `indicador`, `user` e os seeds. É o principal motivo para não chamar isto de "monólito modular" sem ressalva, embora a Análise de Escala (§12.3) diga "monólito modular com 6 contextos claros". Hoje são 8 módulos de negócio e os contextos não são fechados.

**Estado da R3 em 24/11/2026: atendido** (backend #7, #8, #10 e #13 a #22, M-082 a M-087, todos mergeados).

- **`repository` de outro módulo:** 42 usos → 0, por quatro fachadas (`HospitalConsulta`, `VisitaConsulta`, `FeedbackConsulta`, `UsuarioConsulta`).
- **`document` de outro módulo:** 221 usos → 0, com projeções (records em `dto` do módulo dono, com `@Builder`) no lugar do `Document`, inclusive na carga de massa de dev.
- **Ciclos entre módulos:** 28 arestas → 0, por inversão de dependência (interfaces no módulo de baixo, implementadas por quem está acima: `FonteDeContas`, `FonteDoModoSomenteLeitura`, `IndicadoresDosHospitais`, `EnvioDeConfirmacaoDeEmail`) e por mover cada tipo para o módulo que o usa (`FeedbackSalvoEvent` no `feedback`, `RetencaoDadosPessoaisJob` em `jobs`, carga de massa no módulo `seed`).
- **Teste de arquitetura:** o `ArquiteturaModulosTest` (ArchUnit) tem as três regras e **nenhuma exceção congelada** no `archunit_store`; violação nova quebra o CI.
- **Padrão do PO adotado no caminho:** nenhum tipo aninhado em `src/main` e `@Builder` nos records novos.
- **Fora da R3 (dívida registrada em M-086):** `@Builder` nos 25 records de resposta da API anteriores à R3 (38 pontos de criação), mobile e painel (a R3 é só do backend) e os demais riscos da §8.

### 4.3 Eventos e processamento assíncrono

```
FeedbackServiceImpl.salvar ──publishEvent(FeedbackSalvoEvent)──▶ FeedbackSalvoEventListener
                                                               (@Async, executor padrão)
                                                                       │
                                                     IndicadorService.recalcular(hospitalId)
         falha ou instância morre ──▶ IndicadorHospitalJob a cada 15 min (recalcularPendentes)
```

- O `@TransactionalEventListener(AFTER_COMMIT)` **nunca encontra transação** (não há `@Transactional`), e só roda porque `fallbackExecution = true`. Na prática é um `@EventListener` assíncrono; o nome promete uma garantia que não existe. O próprio Javadoc reconhece ("o save de feedback não é transacional no MVP").
- O evento vive em memória: se a instância for recolhida pelo Cloud Run logo depois da resposta, o recálculo se perde e o job de 15 min corrige. É um desenho **aceitável** (RN-18 admite até 15 min) e **coerente** com o ADR-004 da Árvore Tecnológica ("eventos in-process; Kafka só com volume ou segundo consumidor").
- Os demais fluxos assíncronos (expiração de visita, GPS interrompido, feedback sem resposta, retenção LGPD) são **varredura periódica**, não eventos. Isso é o estilo dominante e combina com Cloud Run + Cloud Scheduler.

### 4.4 Consistência e concorrência

- **Atualização condicional atômica:** checkout e heartbeat usam `Update` condicionado ao status ativo (`atualizarSeAtiva`), bom padrão para um documento só.
- **Trava distribuída** por upsert com `_id` único (`config/trava/TravaDistribuida.java`), usada no seed e nos jobs: correta para mais de uma instância.
- **Sem transação em escrita de vários documentos:** `UserServiceImpl.excluirConta` (linhas 189-204) anonimiza visitas e feedbacks (`updateMulti`) e **só depois** apaga a conta. A ordem é a segura: se cair no meio, a conta continua existindo, o pedido devolve erro e repetir a exclusão termina o trabalho (o `updateMulti` é idempotente). A proteção vem da ordem, não de uma transação, e não há teste que a fixe.
- **Check-in com verificar-e-gravar:** `VisitaServiceImpl.checkin` busca a visita ativa (linhas 121-125) e só depois grava (linha 159). Os índices de `VisitaDocument` (linhas 33-41) não têm índice único parcial por usuário/dispositivo com status ativo. **Corrigido em 23/11/2026** (R2, backend #6): índices únicos parciais criados no startup por `VisitaAtivaUnicaRunner`, e o check-in que perde a corrida devolve a visita ativa.

### 4.5 Segurança como parte da arquitetura

- Estilo *stateless* com JWT assinado e rotação por `kid` (ADR do SEG-04), refresh do painel em cookie `__Host-` com CSRF por `Origin`/`X-Requested-With` (ADR-013), cadeia própria para jobs internos com OIDC do Google.
- Rotas anônimas de visita e feedback (`SecurityConfig.java:82-90`) são decisão de produto (RN-20) e a posse é conferida por `dispositivoId` (`VisitaDono`), conforme SEG-02.
- `/v3/api-docs/**` e `/swagger-ui/**` estão `permitAll` (`SecurityConfig.java:126`) em todos os ambientes: o contrato completo da API fica público. Não é vulnerabilidade por si, mas expõe a superfície; vale desligar em produção.

---

## 5. App (pulseira-verde-mobile) em detalhe

### 5.1 Estilo

O app segue um estilo **"serviços + hooks + telas"**, com uma camada `core/` transversal:

```
App.js (native-stack + bottom tabs, 23 rotas)
  └─ screens/<área>/view      telas
     screens/<área>/hooks     hooks de dados (TanStack Query)
     screens/<área>/service   chamadas à API, tarefas de fundo
  core/api      apiClient, envioComFila, sessaoApi, apiError
  core/query    queryClient, queryKeys, cache persistido
  core/stores   sessaoStore (Zustand)
  services/     catálogo local, ids do aparelho, configuração remota, envio em 2º plano
  config/       http, fila offline, sessão, monitoramento
```

### 5.2 Acoplamento entre áreas

Imports de uma área de `screens/` para outra (medidos): `visitas` → `feedback`, `hospitais`, `onboarding`; `perfil` → `auth`, `feedback`, `visitas`; `hospitais` → `visitas` (4); `auth` → `feedback`, `visitas`; `onboarding` → `perfil`, `visitas`; `geolocalizacao` → `hospitais`. É exatamente o "anti-padrão 1" do ADR-008, que continua presente.

O maior arquivo é `screens/visitas/service/GeofencingTaskService.js` com **1.124 linhas**: define duas tarefas de fundo (`GEOFENCING_TASK` linha 682 e `ACOMPANHAMENTO_TASK` linha 849), recálculo de região, check-in e saída. Concentra a regra mais crítica do produto num módulo só, e é o candidato número um a quebrar em partes testáveis.

### 5.3 Offline e integração

- Padrão **fila de saída persistida** (*outbox* no cliente) com idempotência pelo servidor (`ocorridoEm` + "visita ativa"). Bem documentado no próprio código (`config/filaOffline.js`).
- Tokens no `expo-secure-store` no aparelho e `AsyncStorage` só no Web (`services/TokenStorage.js:31`).
- A URL da API é `*.run.app` em `app.json:46-47`: o app está preso ao endereço do Cloud Run até existir o domínio próprio (ADR-015).
- Sem TypeScript (0 arquivos `.ts/.tsx`): os contratos da API existem só como convenção. O ADR-010 não começou no código; o plano de migração foi publicado em 23/11/2026 ([`Plano-Migracao-TypeScript-App-v1.0.md`](../02-arquitetura-tecnica/Plano-Migracao-TypeScript-App-v1.0.md)) e é o caminho natural para a R5.

---

## 6. Painel (pulseira-verde-admin) em detalhe

- **Angular 21 moderno**: componentes standalone, `inject()`, signals, rotas *lazy* com `loadComponent` e `authGuard` (`app.routes.ts`), `HttpClient` com interceptor funcional, sem `zone.js` (detecção de mudança zoneless).
- **Separação `core/` × `features/`**: `core/{auth,hospitais,sugestoes,camadas,mapa,config}` tem serviços e modelos; `features/*` tem as 8 telas. É o equivalente "feature-first" que o app ainda não tem.
- **Pequeno** (24 arquivos TS fora de testes), sem gerenciador de estado global (NgRx não faz falta nesse tamanho).
- Sessão conforme ADR-013 (access em memória, refresh em cookie).
- Modelos da API escritos à mão (`core/hospitais/hospital.models.ts`), sem geração a partir do OpenAPI.

---

## 7. Divergências entre o que está documentado e o código

| # | Documento diz | Código mostra | Gravidade |
|---|---|---|:---:|
| D1 | Nenhum ADR declara o estilo arquitetural do backend; a Árvore (ADR-004) e a Análise de Escala (§12.3) falam em "monólito modular" | Monólito em camadas por funcionalidade, com ciclos entre módulos (§4.2) | Média |
| D2 | Análise de Escala §12.3: "6 contextos claros" | 8 módulos de negócio + `config` + `jobs`, com fronteiras atravessadas por repositório | Baixa |
| D3 | Árvore, ADR-004: eventos com `@TransactionalEventListener` | Existe, mas sem transação: roda por `fallbackExecution` | Baixa |
| D4 | `adrs.md`, Resumo de Status: "Os ADR-001 a ADR-010 continuam `Proposto`" | ADR-001 implementado (o próprio ADR diz, M-032); ADR-002 implementado (`createNativeStackNavigator`, `App.js:39`); ADR-004 substituído (SDD); ADR-005 implementado (`TokenStorage.js`); ADR-006 parcial (9 arquivos ainda com `TouchableOpacity`, 14 com `Pressable`); ADR-008 parcial (`core/` existe, `screens/views` removido); ADR-009 parcial (heartbeat parado por `VisitaAtivaSync`, ainda `setInterval`) | Média (o índice das decisões engana quem chega) |
| D5 | Análise de Escala propõe um ADR-016 (borda + catálogo local + Cloud Scheduler) | `adrs.md` pula de ADR-015 para ADR-017; não existe ADR-016 | Baixa |
| D6 | Comentário em `cd-backend-google.yml:215-221`: `--max-instances=1` porque o seed não é idempotente | O seed já usa `TravaDistribuida` (`SeedRunner.java:71`) desde o SOL-02; o motivo do comentário não vale mais. Continua `max=1` | Média (trava a escala por um motivo vencido; conferir antes se mais nada depende de instância única, p. ex. o rate limit em memória) |
| D7 | Árvore v2.2 §1.1 ("1 teste de controller", "sem JWT real", senha em texto puro) | 86 arquivos de teste, JWT com `kid`, BCrypt: a §1 é retrato de agosto e não está marcada como tal | Baixa |
| D8 | Meta do PO: cobertura backend ≥ 95%, frontend ≥ 90% | Backend: `jacocoTestCoverageVerification` com mínimo 0,65/0,50 (`build.gradle:133,139`); app: 79/67/77/80 (`jest.config.js:66-72`); nos três repositórios o portão real é o `diff-cover --fail-under=90` só nas linhas do PR | Média (já apontado na auditoria dos testes de 18/11) |

---

## 8. Riscos, dívidas e recomendações priorizadas

| Prior. | ID | Risco / dívida | Evidência | Recomendação | Esforço |
|:---:|---|---|---|---|:---:|
| **P1** | R2 | Duas visitas ativas para a mesma pessoa sob concorrência (geofencing + fila offline + toque manual) | `VisitaServiceImpl.java:121-160`; índices em `VisitaDocument.java:33-41` | Índice único parcial em `usuarioId` e outro em `dispositivoId` com `status ∈ {EM_ATENDIMENTO, SUSPEITA}`, tratando `DuplicateKeyException` como "devolve a ativa" | Baixo <br>✅ **Corrigido em 23/11/2026** (backend #6, M-081) |
| **P2** | R3 | Ciclos e acesso a repositório alheio: mudança num módulo quebra outro sem aviso; impede extrair um módulo no futuro | §4.2 | Cada módulo expõe uma fachada (`VisitaConsulta`, `HospitalConsulta`…) e só ela é importada de fora; teste de arquitetura (ArchUnit ou Spring Modulith `ApplicationModules.verify()`) no CI para impedir regressão | Médio <br>✅ **Atendido em 24/11/2026** (backend #7, #8, #10 e #13 a #22, M-082 a M-087, todos mergeados): `repository` 42 → 0, `document` 221 → 0 e ciclos 28 → 0, com o ArchUnit sem nenhuma exceção congelada, ver §4.2 |
| **P2** | R4 | Estado na instância impede escalar: rate limit em memória e `max=1` com motivo vencido | `RateLimitService.java:57`; D6 | Decidir no ADR-015 o caminho do rate limit (borda/Cloudflare ou Mongo) e só então subir `max-instances`; atualizar o comentário do CD | Médio |
| **P2** | R5 | Contrato da API duplicado à mão em JS e TS, sem teste de contrato | P21 | Gerar os tipos do painel a partir do `/v3/api-docs` (`openapi-typescript`) no CI e, com o ADR-010, no app; publicar o `openapi.json` como artefato do backend | Médio |
| **P3** | R6 | `GeofencingTaskService.js` com 1.124 linhas concentra a regra mais crítica do app | §5.2 | Separar em funções puras (decisão de entrada/saída, recálculo de região) e adaptadores do Expo; testar as puras sem *mock* do TaskManager | Médio |
| **P3** | R1 | Escritas em vários documentos sem transação; a exclusão de conta (LGPD) só é segura pela ordem das gravações | `UserServiceImpl.java:189-204` | Um teste que simule falha na anonimização e confirme que a conta continua e que a segunda tentativa conclui; regra escrita no ADR de estilo (R8): "apagar o dado identificável por último". Transação Mongo (`MongoTransactionManager`, o Atlas é *replica set*) só se surgir escrita que a ordem não resolva | Baixo |
| **P3** | R7 | Modelo anêmico: regra de visita espalhada em `VisitaServiceImpl` (665 linhas) | P6 | Sem reescrita: mover regras de transição de status para métodos da própria visita ou para objetos como `PresencaVisita`, conforme as tocar | Contínuo |
| **P3** | R8 | Índice de ADRs desatualizado e estilo não declarado | D1-D5 | Atualizar o Resumo de Status; criar o ADR-016 (ou marcar o número como não usado); criar um ADR "Estilo arquitetural: monólito modular em camadas por funcionalidade", com as regras de fronteira da R3 | Baixo |
| **P3** | R9 | OpenAPI e Swagger públicos em produção | `SecurityConfig.java:126` | Ligar só fora de produção (`springdoc.api-docs.enabled` por perfil) | Baixo |
| **P4** | R10 | `@TransactionalEventListener` sem transação confunde quem lê | §4.3 | Trocar por `@EventListener` + `@Async`, ou manter se um dia houver transações | Mínimo |

### O que **não** recomendo agora (e por quê)

- **Microsserviços / SOA:** equipe pequena, um banco, um domínio coeso; os gatilhos da Análise de Escala §12.3 continuam não atingidos. A R3 deixa a porta aberta sem pagar o custo.
- **Kafka/RabbitMQ ou event sourcing:** um único evento e um único consumidor; o job de 15 min já é a rede de segurança. Revisar se surgir um segundo consumidor (ex.: notificação push do servidor).
- **Hexagonal completo / DDD tático de livro:** reescreveria 8 serviços e 86 arquivos de teste sem mover nenhum indicador medido. Melhor aplicar a R7 de forma incremental.
- **CQRS formal:** o *read model* do indicador e o catálogo com ETag já cobrem a necessidade de leitura.

---

## 9. Conclusão para o PO

Em uma frase: **o Pulseira Verde é um monólito Spring Boot em camadas (MVC), organizado por funcionalidade, com um app offline-first e um painel Angular moderno, falando REST/JSON com JWT.** Os padrões de "evento" e "DDD" aparecem só de forma leve (um evento em memória, nomes do negócio nos pacotes), e isso é adequado ao tamanho do produto. O que precisa de atenção não é trocar de estilo, e sim **fechar as fronteiras do monólito** (R3), **garantir a escrita crítica do check-in** (R2) e **atualizar a documentação de decisões** (R8) para que ela volte a descrever o sistema real.

Próximo passo sugerido: o PO escolhe quais itens P1/P2 viram estória; cada um vira um PR no repositório da área.
