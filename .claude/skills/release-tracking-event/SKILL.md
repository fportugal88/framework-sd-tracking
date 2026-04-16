---
name: release-tracking-event
description: Use esta skill para orquestrar o pipeline completo de release de um evento de tracking (governança → infra → core → qualidade → automação). Em ordem, executa validate-specs, privacy-consent-review, generate-datalayer-schema, generate-ga4-mapping, generate-gtm-payload, generate-tracking-tests, generate-tracking-docs, roda mp-debug-validate e depois chama a Tag Manager API (workspaces.sync → workspaces.create_version → versions.publish). Nunca publica direto sem passar pelos gates.
---

# Release End-to-End de Evento de Tracking

## Quando usar

- Evento novo pronto para produção.
- Bump de MAJOR/MINOR de evento existente.
- Revisão massiva do container (ex. deprecação, rename, migração).

## Gates obrigatórios (em ordem)

1. **Governança**
   - Spec existe em `tracking/specs/events/`.
   - `owner` preenchido; CODEOWNERS marcado; PR template com checklist.
2. **Lint** via `/validate-specs` — zero BLOCKERS.
3. **Privacy** via `/privacy-consent-review` — zero BLOCKERS; WARNINGS justificados no PR.
4. **Contratos**
   - `/generate-datalayer-schema` atualizado;
   - `/generate-ga4-mapping` atualizado;
   - diff revisado no PR.
5. **GTM artefatos**
   - `/generate-gtm-payload` escreve em `tracking/gtm/{web,server}/generated/`;
   - manual overrides respeitados.
6. **Testes**
   - `/generate-tracking-tests` executado;
   - unit + contract + integration verdes em CI;
   - synthetic em ambiente de staging OK.
7. **MP debug**
   - `/mp-debug-validate` para cada evento tocado;
   - `validationMessages: []`.
8. **Docs**
   - `/generate-tracking-docs` executado;
   - changelog em `Unreleased` com bullet por evento;
   - matriz e dicionário atualizados.
9. **Preview GTM**
   - sGTM Preview ativo;
   - Tag Assistant / DebugView mostrando evento como esperado em smoke test manual.
10. **Publicação via Tag Manager API**
    - `workspaces.sync` para resolver drift;
    - `workspaces.getStatus` ⇒ esperado sem conflito;
    - `workspaces.create_version` com nome `release/<evento>@<semver>-<date>`;
    - `versions.publish`;
    - registrar `versionId` no changelog.
11. **Pós-release**
    - ativar/checar monitores (volume, duplicidade, PII leaks, consent rejection, delta sGTM↔GA4);
    - smoke com Realtime + DebugView;
    - plano de rollback: `versions.publish` da versão anterior (republish) descrito em `runbooks/rollback-container-version.md`.

## Rollback

- Operação de rollback = republicar `versionId` anterior via Tag Manager API (operação **não-destrutiva**: cria nova versão que replica a anterior).
- Sempre registrar motivo do rollback + postmortem curto.
- Não resolver incident reescrevendo histórico.

## Segredos e ambiente

- `GTM_API_OAUTH_TOKEN`, `GA4_MEASUREMENT_ID`, `GA4_API_SECRET` só por env / secret manager.
- Pipeline rodando em service account com escopo mínimo (`tagmanager.edit.containers` + `publish` apenas no container alvo).

## Saída

Relatório consolidado com:
- eventos publicados + versão semântica;
- container GTM `versionId` web e server;
- link do changelog;
- status dos monitores nos primeiros 30 minutos;
- próximo checkpoint recomendado (24h, 7 dias).

## Delegação

Use o agente `tracking-release-manager` para coordenar a execução, consolidar aprovações e operar a Tag Manager API.
