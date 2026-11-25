# Homologação do backend v1.0.0-rc.20

> **Data:** 25/11/2026 · **Ambiente:** homologação (`master` do `pulseira-verde-backend`, Cloud Run `saude-monitor-backend-hom`, banco `saude_monitor_hom`)
> **Rc anterior do backend:** [`backend-v1.0.0-rc.19`](./backend-v1.0.0-rc.19.md) (24/11/2026) · **Origem:** `develop` do `pulseira-verde-backend` em `91b0183` (merge do PR #27), com CI e CD de dev verdes
> **Responsável pela validação:** Gabriel Vogado (PO)

Esta rc leva ao backend de homologação o suporte à escolha e à troca de unidade no check-in
quando os geofences de dois hospitais se sobrepõem (E2-04/RN-05, M-093, pulseira-verde-backend #26)
e o workflow que publica a pré-release `dev` na aba Releases (#27, M-094). O comportamento visível
é aditivo: campos opcionais novos no check-in e um endpoint novo; o app da rc.21 segue compatível.

**Por que o arquivo tem o prefixo `backend-`.** Cada repositório numera a sua rc (MANUAL §3.3), e o
número `rc.20` já tem o registro do app ([`v1.0.0-rc.20.md`](./v1.0.0-rc.20.md)). O prefixo evita
sobrescrever aquele registro.

---

## 1. Como esta rc é gerada

1. **Backend primeiro.** PR `develop` → `master` no `pulseira-verde-backend`. No merge, o
   `cd-homologacao.yml` calcula a versão pelo `version` do `build.gradle` (`1.0.0`) e pela última
   tag do backend (`rc.19`): **`v1.0.0-rc.20`**. Publica o Cloud Run `saude-monitor-backend-hom`,
   roda os smoke tests, confere a URL fixa e cria a tag e a pré-release, sem anexo.
2. **App depois.** O APK da rc.22 usa o `PATCH /api/v1/visitas/{id}/hospital` e os campos novos do
   check-in; por isso este PR é mergeado antes da promoção do app
   ([`v1.0.0-rc.22.md`](./v1.0.0-rc.22.md)).
3. **Painel admin** não tem homologação e fica fora desta rc.

**Par usado no teste:** APK `rc.22` + backend `rc.20`.

**Pré-release `dev` convive com a rc.** O `release-dev.yml` (#27) só reage ao CD da `develop`
(`workflow_run` na branch `develop`, evento `push`); a homologação roda na `master`, e a tag `dev`
não casa com `v*-rc.*`. Conferir na página Releases que a `v1.0.0-rc.20` sai como pré-release sem
anexo e que a Release `dev` continua como estava.

**Se o pipeline falhar:** nenhuma tag é criada, e o `alarme-cd.yml` abre a issue "CD vermelho"
para a `master` do backend. Corrigir na `develop` e promover de novo; o número `rc.20` é
reaproveitado.

**Rollback:** republicar a revisão anterior do Cloud Run `saude-monitor-backend-hom` (a da rc.19).
Os campos novos são opcionais e a versão anterior os ignora; o APK da rc.22 perde só a troca de
unidade (o `PATCH` passa a responder 404).

---

## 2. O que entrou desde a rc.19

### Backend (`pulseira-verde-backend`)

| Assunto | PR | Melhoria |
|---|---|---|
| Check-in com geofences sobrepostos: `CheckinRequest.escolhidoNoDesempate`, `CheckinResponse.alternativas`, `VisitaDocument.hospitaisAlternativos` e o novo `PATCH /api/v1/visitas/{id}/hospital` para trocar a unidade da visita ativa (só entre as alternativas, mantendo a entrada) | #26 | M-093 |
| Workflow `release-dev.yml`: pré-release `dev` na aba Releases, recriada a cada deploy verde de dev (sem anexo no backend) | #27 | M-094 |

### App, Painel Admin e documentação

- App: rc.22 própria ([`v1.0.0-rc.22.md`](./v1.0.0-rc.22.md)); o APK de teste é o da rc.22.
- Painel sem homologação.
- Documentação: M-093 e M-094 (pulseira-verde-docs #26 e #27) e este registro.

---

## 3. Roteiro de homologação

Usar o APK `pulseira-verde-hom-1.0.0-rc.22.apk` (página Releases do `pulseira-verde-mobile`), que
já aponta para o backend HML.

### 3.1 Escolha e troca de unidade (#26, novo nesta rc)

- [ ] `POST /api/v1/visitas/checkin` por geofence num ponto coberto por **uma** unidade responde
      como antes, sem `alternativas`.
- [ ] Num ponto coberto por duas unidades (Hospital Regional de Taguatinga e Cepav - Azaleia), a
      resposta traz `alternativas` com a outra unidade.
- [ ] `PATCH /api/v1/visitas/{id}/hospital` com um id que está em `alternativas` troca a unidade e
      mantém a entrada; a unidade anterior passa a ser alternativa. Com um id fora das alternativas,
      responde erro (não troca).
- [ ] No empate (diferença de até 10 m), o check-in automático responde 409 com candidatos; repetido
      com `escolhidoNoDesempate` de um hospital que contém a posição, cria a visita.

### 3.2 Pipeline e Releases (#27)

- [ ] A página Releases do backend mostra `v1.0.0-rc.20` como pré-release **sem anexo**.
- [ ] A Release `dev` não foi alterada pelo CD de homologação.

### 3.3 Roteiros anteriores

- [ ] [Roteiro do backend rc.19](./backend-v1.0.0-rc.19.md#3-roteiro-de-homologação) (tendência da
      nota e regressão dos módulos), ainda não preenchido, e dentro dele o da rc.18.

---

## 4. Resultado

| Item | Situação | Observação |
|---|---|---|
| Pipeline do backend (`pulseira-verde-backend`) | _a preencher_ | |
| Roteiro 3.1 (escolha e troca de unidade) | _a preencher_ | |
| Roteiro 3.2 (pipeline e Releases) | _a preencher_ | |
| Roteiro 3.3 (rc.19 do backend) | _a preencher_ | |
| Decisão | _a preencher_ | Aprovada vira `release/1.0.0` (MANUAL §3.4) |
