# Homologação do backend v1.0.0-rc.18

> **Data:** 23/11/2026 · **Ambiente:** homologação (`master` do `pulseira-verde-backend`, Cloud Run `saude-monitor-backend-hom`, banco `saude_monitor_hom`)
> **Rc anterior do backend:** [`v1.0.0-rc.17`](./v1.0.0-rc.17.md) (16/11/2026) · **Origem:** `develop` do `pulseira-verde-backend` em `dba6b7c` (merge do PR #10), com CI e CD de dev verdes
> **Responsável pela validação:** Gabriel Vogado (PO)

Primeira rc do backend desde a rc.17. Ela leva a visita ativa única, as fronteiras entre módulos
(R2 e R3 da auditoria de arquitetura) e as regras de monitoramento do E8-06. O app continua o da
rc.19, sem rc nova.

**Por que o arquivo tem o prefixo `backend-`.** Cada repositório numera a sua rc
(MANUAL §3.3). O backend parou na rc.17 e o app foi até a rc.19, então a próxima rc do backend é
a **rc.18**, número que já tem o registro do app ([`v1.0.0-rc.18.md`](./v1.0.0-rc.18.md)). O
prefixo evita sobrescrever aquele registro.

---

## 1. Como esta rc é gerada

1. **Só o backend.** PR `develop` → `master` no `pulseira-verde-backend`. No merge, o
   `cd-homologacao.yml` calcula a versão pelo `version` do `build.gradle` (`1.0.0`) e pela última
   tag do backend (`rc.17`): **`v1.0.0-rc.18`**. Publica o Cloud Run `saude-monitor-backend-hom`,
   roda os smoke tests (saúde, leitura do Mongo, filtro por raio), confere a URL fixa e cria a tag e
   a pré-release, sem anexo.
2. **App sem rc nova.** O `pulseira-verde-mobile` não mudou desde a rc.19.
3. **Painel admin** não tem homologação e fica fora desta rc.

**Par usado no teste:** APK `rc.19` + backend `rc.18`.

**Atenção ao primeiro boot (PR #6).** O `VisitaAtivaUnicaRunner` roda no startup do backend HML.
Se o banco `saude_monitor_hom` tiver duas visitas ativas da mesma pessoa, ele encerra a mais
antiga como `EXPIRADA` e só então cria os índices únicos. Uma visita expirada assim não entra no
indicador.

**Se o pipeline falhar:** nenhuma tag é criada, e o `alarme-cd.yml` abre a issue "CD vermelho"
para a `master` do backend. Corrigir na `develop` e promover de novo; o número `rc.18` é
reaproveitado.

**Rollback:** republicar a revisão anterior do Cloud Run `saude-monitor-backend-hom`. Os índices
únicos criados pelo #6 ficam no banco e não atrapalham a versão anterior. As visitas que o runner
expirou não voltam.

---

## 2. O que entrou desde a rc.17

### Backend (`pulseira-verde-backend`)

| Assunto | PR | Melhoria |
|---|---|---|
| Nomes Pulseira Verde nos comentários, no README e nos workflows; nenhuma mudança de comportamento | #5 | M-078 |
| Uma só visita ativa por pessoa, garantida por índices únicos parciais; o check-in que perde a corrida devolve a visita vencedora | #6 | M-081 |
| Teste de arquitetura (ArchUnit) congela ciclos e acessos diretos entre módulos; só teste | #7 | M-082 |
| Fachadas de consulta entre módulos no lugar de repository de outro módulo | #8 | M-083 |
| Regras de alerta e painel de latência por rota em `monitoramento/` e job novo no CI; nada muda na aplicação | #9 | M-084 |
| O módulo `user` deixa de usar os documents de visita e feedback: exclusão de conta e exportação (JSON e PDF) passam pelas fachadas, com o mesmo resultado | #10 | registro no Histórico pendente |

### App, Painel Admin e documentação

- App sem rc nova: o APK de teste é o da rc.19.
- Painel sem homologação.
- Documentação: M-081 a M-084 (pulseira-verde-docs #12, #14 e #13) e este registro.

---

## 3. Roteiro de homologação

Usar o APK `pulseira-verde-hom-1.0.0-rc.19.apk` (página Releases do `pulseira-verde-mobile`), que
já aponta para o backend HML.

### 3.1 Visita ativa única (#6, novo nesta rc)

- [ ] Abrir o detalhe de um hospital e tocar duas vezes seguidas em "Estou aqui agora". O
      Histórico mostra **uma** visita aberta, não duas.
- [ ] Com a visita aberta, tentar o check-in em **outro** hospital. A segunda visita não é criada
      (a API responde 409) e o Histórico continua com uma visita aberta só.
- [ ] Encerrar a visita e fazer um novo check-in. A nova visita abre normalmente.

### 3.2 Exclusão de conta e exportação de dados (#10, novo nesta rc)

Usar uma conta de teste criada para isso, com pelo menos uma visita e uma avaliação.

- [ ] No Histórico, tocar em "Exportar meus dados (PDF)". O PDF lista as visitas e as avaliações da
      conta, com os mesmos campos da rc.17. (O app só exporta em PDF; o JSON da exportação é
      coberto pelos testes do #10.)
- [ ] No Perfil, excluir a conta. O login com ela deixa de funcionar, e a nota e o tempo do hospital
      continuam aparecendo (as visitas e avaliações ficam anonimizadas, não somem).

### 3.3 Telas que passam pelas fachadas (#8, regressão)

- [ ] Lista de hospitais, mapa, detalhe com nota e tempo mediano, ranking e Histórico carregam como
      na rc.19.
- [ ] Enviar uma avaliação de uma visita encerrada. Ela aparece no Histórico.

### 3.4 Roteiros anteriores

- [ ] [Roteiro da rc.19](./v1.0.0-rc.19.md#3-roteiro-de-homologação) (e, dentro dele, rc.18, rc.17
      e rc.16), com o APK da rc.19 e o backend desta rc.

---

## 4. Resultado

| Item | Situação | Observação |
|---|---|---|
| Pipeline do backend (`pulseira-verde-backend`) | _a preencher_ | |
| Roteiro 3.1 (visita ativa única) | _a preencher_ | |
| Roteiro 3.2 (exclusão e exportação) | _a preencher_ | |
| Roteiro 3.3 (fachadas, regressão) | _a preencher_ | |
| Roteiro 3.4 (rc.19 e anteriores) | _a preencher_ | |
| Decisão | _a preencher_ | Aprovada vira `release/1.0.0` (MANUAL §3.4) |
