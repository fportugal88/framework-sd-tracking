---
name: generate-gtm-payload
description: Use esta skill para gerar payloads prontos para a Tag Manager API (workspaces/variables, triggers, tags), tanto do container Web quanto do Server container, a partir das specs e do mapping GA4. Escreve em tracking/gtm/web/generated e tracking/gtm/server/generated. Nunca publica diretamente; gera os artefatos para CI/CD rodar workspaces.create_version + versions.publish. Use antes de qualquer release.
---

# Geração de Payloads GTM (Web + Server) via Tag Manager API

## Objetivo

Transformar specs + mappings em configuração GTM versionada e aplicável via API:
- Variables (constantes, dataLayer variables, lookup tables);
- Triggers (custom events para cada `datalayer_event`);
- Tags (GA4 Event Tag no web container, GA4 client e tags server-side no server container);
- Transformations (server-side: redaction, enrichment, allowlist);
- Folders e notes com rastreabilidade para `spec_id` + `spec_version`.

**Nunca** publica direto. O output é consumido pelo pipeline (`workspaces.sync` → `workspaces.create_version` → `versions.publish`).

## Web container — por evento

- **Trigger**: Custom Event com `eventName = <datalayer_event>`.
- **Variables dataLayer**: uma por campo mapeado (`ecommerce.transaction_id`, `ecommerce.currency`, …). Nome do variable: `dlv - ecommerce.<campo>`.
- **Tag GA4 Event**: referência ao GA4 Configuration Tag; `event_name = <ga4_event_name>`; parâmetros mapeados 1:1; `send_to` apontando para o server container via `transport_url` (= domínio first-party do sGTM).
- **Folder**: `ecommerce/<evento>`.
- **Notes**: `spec_id=<id> spec_version=<ver> owner=<owner>`.

## Server container — por evento

- **GA4 Client**: reivindica `/collect`, `/g/collect`, `/j/collect` (nativo). Prioridade definida, sem sobreposição com outros clients.
- **Tag GA4**: envia ao stream correto; parâmetros vindo de `getAllEventData()`.
- **Transformations** (aplicadas antes das tags):
  - `redact_pii`: remove campos em denylist (`email`, `phone`, `cpf`, `rg`, `cnpj`, `full_name`, etc.).
  - `allowlist_params`: mantém só os campos previstos no mapping.
  - `enrich_geo/device` opcionalmente, respeitando limite do sGTM de derivação.
- **Tags adicionais** (Ads/Floodlight/outros): geradas SOMENTE se a spec declarou destino extra em `mappings/destinations/*`.
- **Policy**: qualquer template customizado deve declarar `send_http` permission com allowlist de domínios.

## Measurement Protocol (complemento)

Se a spec tem `trigger.source: mp`:
- gerar snippet de client MP separado (não tag web);
- `POST https://www.google-analytics.com/mp/collect?measurement_id=...&api_secret=...`;
- `region1.google-analytics.com` se coleta UE;
- validação em desenvolvimento: `/debug/mp/collect` com `validation_behavior: ENFORCE_RECOMMENDATIONS`;
- incluir `session_id` e `engagement_time_msec` para métricas de sessão.

## Estrutura de saída

```
tracking/gtm/web/generated/
  accounts/<aid>/containers/<cid>/workspaces/<wsid>/
    variables/*.json
    triggers/*.json
    tags/*.json
    folders/*.json
    manifest.json
tracking/gtm/server/generated/
  accounts/<aid>/containers/<cid>/workspaces/<wsid>/
    clients/*.json
    tags/*.json
    transformations/*.json
    templates/*.json
    manifest.json
```

`manifest.json` contém mapping `spec_id@spec_version -> [entity paths]`.

## Regras de geração

1. **Idempotência**: rerodar com mesma entrada gera mesmo output (hash determinístico).
2. **Rastreabilidade**: todo entity GTM referencia `spec_id` + `spec_version` em `notes`.
3. **Nenhum secret commitado**: `api_secret`, `measurement_id`, IDs de conta ficam em variáveis de ambiente / secret store.
4. **Manual overrides preservados**: nunca sobrescrever arquivos em `tracking/gtm/web/manual-overrides/` e `tracking/gtm/server/manual-overrides/`. O pipeline mescla via `workspaces.sync`.
5. **First-party**: `transport_url` do web aponta para subdomínio first-party (ex. `https://gtm.suaempresa.com.br`). Falhar se não estiver configurado.
6. **Permissões de template**: gerar `policy.yaml` por template customizado listando permissões mínimas (`send_http`, `get_cookie`, `set_cookie`, `read_event_data`).

## Passos

1. Carregar specs ativas + `mappings/ga4/ecommerce.map.yaml`.
2. Gerar entities web (variables → triggers → tags → folders).
3. Gerar entities server (clients → transformations → templates → tags).
4. Emitir `manifest.json` com rastreabilidade.
5. Rodar validação estrutural contra o schema da Tag Manager API.
6. Imprimir diff resumido; sugerir `/release-tracking-event` para publicação via pipeline.

## Delegação

- `gtm-web-architect` para configuração do container web.
- `gtm-server-architect` para sGTM, clients, transformations e templates customizados.
- `tracking-release-manager` para orquestrar a chamada real da Tag Manager API.
