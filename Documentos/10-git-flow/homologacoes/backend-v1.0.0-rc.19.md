# Homologação do backend v1.0.0-rc.19

> **Data:** 24/11/2026 · **Ambiente:** homologação (`master` do `pulseira-verde-backend`, Cloud Run `saude-monitor-backend-hom`, banco `saude_monitor_hom`)
> **Rc anterior do backend:** [`backend-v1.0.0-rc.18`](./backend-v1.0.0-rc.18.md) (23/11/2026) · **Origem:** `develop` do `pulseira-verde-backend` em `19ff437` (merge do PR #24), com CI e CD de dev verdes
> **Responsável pela validação:** Gabriel Vogado (PO)

Segunda rc do backend depois da troca de repositórios. Ela fecha a R3 da auditoria de arquitetura
(documents e ciclos entre módulos em zero, padrão `@Builder` nos records) e leva a tendência simples
da nota do hospital (E4-06). Tudo o que muda de comportamento visível é a tendência, que é um campo
opcional novo na resposta de indicadores; o resto é refatoração com os mesmos endpoints e JSON.

**Por que o arquivo tem o prefixo `backend-`.** Cada repositório numera a sua rc (MANUAL §3.3). A
próxima rc do backend é a **rc.19**, número que já tem o registro do app
([`v1.0.0-rc.19.md`](./v1.0.0-rc.19.md)). O prefixo evita sobrescrever aquele registro.

---

## 1. Como esta rc é gerada

1. **Backend primeiro.** PR `develop` → `master` no `pulseira-verde-backend`. No merge, o
   `cd-homologacao.yml` calcula a versão pelo `version` do `build.gradle` (`1.0.0`) e pela última
   tag do backend (`rc.18`): **`v1.0.0-rc.19`**. Publica o Cloud Run `saude-monitor-backend-hom`,
   roda os smoke tests (saúde, leitura do Mongo, filtro por raio), confere a URL fixa e cria a tag e
   a pré-release, sem anexo.
2. **App depois.** A rc.20 do app (E4-06 e busca no mapa) mostra a tendência que esta rc passa a
   enviar; por isso este PR é mergeado antes da promoção do app
   ([`v1.0.0-rc.20.md`](./v1.0.0-rc.20.md)).
3. **Painel admin** não tem homologação e fica fora desta rc.

**Par usado no teste:** APK `rc.20` + backend `rc.19`.

**Se o pipeline falhar:** nenhuma tag é criada, e o `alarme-cd.yml` abre a issue "CD vermelho"
para a `master` do backend. Corrigir na `develop` e promover de novo; o número `rc.19` é
reaproveitado.

**Rollback:** republicar a revisão anterior do Cloud Run `saude-monitor-backend-hom` (a da rc.18).
Os campos novos do `agregados_hospitais` (`tendenciaNota`, `variacaoNota`, `tendenciaPreliminar`)
são opcionais e a versão anterior os ignora; o app da rc.20 mostra a tela sem tendência quando o
campo falta.

---

## 2. O que entrou desde a rc.18

### Backend (`pulseira-verde-backend`)

| Assunto | PR | Melhoria |
|---|---|---|
| `document` de outro módulo: o `indicador` e a `visita` passam a usar projeções (`VisitaParaIndicador`, `FeedbackParaIndicador`, `HospitalResumo`, `HospitalNoPonto`); a consulta de geofence vai para o módulo `hospital` | #13 | M-085 |
| Tipos aninhados em arquivo próprio, com `@Builder` (padrão do PO); schemas `Periodo` e `Finalidade` do OpenAPI mantêm o nome | #14 | M-086 |
| `feedback` sem `VisitaDocument` e `HospitalDocument`: `VisitaParaAvaliacao` e `VisitaParaFeedback` | #15 | M-085 |
| `auth` e `config.security` sem `UserDocument`: `UsuarioAutenticavel`, sem o hash da senha no `toString` | #16 | M-085 |
| Carga de massa de avaliações (só perfil `dev`) sem document alheio; `document` de outro módulo chega a zero | #17 | M-085 |
| Ciclos, passo A: `config` não depende mais de módulos de negócio | #18 | M-087 |
| Ciclos, passo B: evento do feedback no `feedback` e job de retenção no módulo `jobs` | #19 | M-087 |
| Ciclos, passo C: carga de massa vira o módulo `seed` | #20 | M-087 |
| Ciclos, passo D: `hospital` lê indicadores pela porta `IndicadoresDosHospitais`; o endpoint `GET /api/v1/hospitais/{id}/indicadores` vai para o módulo `indicador` (mesmo caminho e mesmo grupo do OpenAPI) | #21 | M-087 |
| Ciclos, passo E: `user` envia o código de confirmação do e-mail pela interface `EnvioDeConfirmacaoDeEmail`; ciclos entre módulos em zero | #22 | M-087 |
| Records de resposta da API com `@Builder` em todos os módulos | #23 | M-088 |
| Tendência simples da nota do hospital: nota dos últimos 30 dias contra os 30 anteriores, campos opcionais `tendenciaNota`, `variacaoNota` e `tendenciaPreliminar` em `GET /api/v1/hospitais/{id}/indicadores` | #24 | M-090 |

O PR #12 (sincroniza a `develop` com a `master`) foi fechado sem merge e não entra.

### App, Painel Admin e documentação

- App: rc.20 própria ([`v1.0.0-rc.20.md`](./v1.0.0-rc.20.md)); o APK de teste é o da rc.20.
- Painel sem homologação.
- Documentação: M-085 a M-088 e M-090 (pulseira-verde-docs #16, #17 e #21) e este registro.

---

## 3. Roteiro de homologação

Usar o APK `pulseira-verde-hom-1.0.0-rc.20.apk` (página Releases do `pulseira-verde-mobile`), que
já aponta para o backend HML.

### 3.1 Tendência da nota (#24, novo nesta rc)

- [ ] `GET /api/v1/hospitais/{id}/indicadores` de um hospital com poucas avaliações responde 200
      **sem** `tendenciaNota` (com menos de 5 avaliações em algum dos dois períodos não há
      tendência).
- [ ] Em um hospital com pelo menos 5 avaliações nos últimos 30 dias e 5 nos 30 anteriores, a
      resposta traz `tendenciaNota` e `variacaoNota` (nota atual menos a anterior, com duas casas; variação abaixo de 0,3 em módulo é estável).
      Com menos de 10 avaliações em algum período, `tendenciaPreliminar` vem `true`. (O banco HML
      pode não ter um hospital assim; nesse caso fica coberto pelos testes automatizados do backend.)
- [ ] O `GET /api/v1/catalogo/indicadores` continua igual, sem os campos de tendência.

### 3.2 Regressão das fachadas e dos módulos (#13 a #23)

Sem mudança de endpoint nem de JSON; confere que nada quebrou.

- [ ] Lista de hospitais, mapa, detalhe com nota e tempo mediano, ranking e Histórico carregam.
- [ ] Check-in, encerramento da visita e envio de avaliação funcionam; a avaliação aparece no
      Histórico.
- [ ] Cadastro, login, confirmação de e-mail (o código chega) e recuperação de senha funcionam.
- [ ] O detalhe do hospital ainda abre o indicador (o endpoint mudou de módulo, não de caminho).
- [ ] Exportar os dados (PDF) e excluir uma conta de teste seguem como na rc.18.

### 3.3 Roteiros anteriores

- [ ] [Roteiro do backend rc.18](./backend-v1.0.0-rc.18.md#3-roteiro-de-homologação), em
      especial a visita ativa única (3.1) e a exclusão de conta (3.2), que ainda não foram
      preenchidos.

---

## 4. Resultado

| Item | Situação | Observação |
|---|---|---|
| Pipeline do backend (`pulseira-verde-backend`) | _a preencher_ | |
| Roteiro 3.1 (tendência da nota) | _a preencher_ | |
| Roteiro 3.2 (regressão dos módulos) | _a preencher_ | |
| Roteiro 3.3 (rc.18 do backend) | _a preencher_ | |
| Decisão | _a preencher_ | Aprovada vira `release/1.0.0` (MANUAL §3.4) |
