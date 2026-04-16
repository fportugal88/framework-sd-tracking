---
name: mp-debug-validate
description: Use esta skill para validar um payload de evento contra o endpoint oficial /debug/mp/collect do Measurement Protocol do GA4 com validation_behavior=ENFORCE_RECOMMENDATIONS. Útil durante desenvolvimento ou investigação de incidente. A skill monta o payload a partir de uma spec (ou de um JSON fornecido), dispara a requisição e interpreta validationMessages - nunca usar /mp/collect direto (produção) para validar.
---

# Validação via Measurement Protocol Debug

## Objetivo

Rodar um check semântico de um evento contra o endpoint `/debug/mp/collect`, usado apenas em dev/CI. **Não** publica dados em relatórios; **não** valida `api_secret`. `2xx` não é sucesso semântico — o que importa é `validationMessages: []`.

## Entradas

- Spec alvo (por id) **ou** JSON do evento bruto.
- `GA4_MEASUREMENT_ID` e `GA4_API_SECRET` via env vars.
- Região (default: `www.google-analytics.com`; usar `region1.google-analytics.com` para coleta UE).

## Regras obrigatórias

1. **Apenas HTTPS POST**.
2. Body sempre inclui:
   - `client_id` (fake `12345.67890` se dev).
   - `validation_behavior: "ENFORCE_RECOMMENDATIONS"`.
   - `events[].name` (= `ga4.event_name` da spec).
   - `events[].params.session_id`.
   - `events[].params.engagement_time_msec` (mínimo `1`, default `100` em dev).
3. Payload respeita o mapping GA4 gerado; não adiciona campos não mapeados.
4. Se a spec tem `items`, inclui `items` com pelo menos um item e `item_id` OU `item_name`.
5. Para `purchase`, incluir `transaction_id` único por execução (ex. `DEBUG-${timestamp}`).
6. Se coleta UE: usar `region1.google-analytics.com`.
7. **Nunca** disparar contra produção (`/mp/collect`) dentro da skill; há um guard explícito.
8. Segredos NUNCA logados. Skill mascara `api_secret` no output.

## Interpretação de resposta

- `validationMessages: []` ⇒ válido.
- `validationMessages: [...]` ⇒ cada entrada tem `fieldPath`, `description`, `validationCode`. A skill lista em tabela, explica causa provável e sugere correção na spec ou no mapping.
- Códigos comuns:
  - `VALUE_INVALID`: tipo ou formato fora do esperado.
  - `NAME_INVALID`: event_name ou parameter name ferindo convenção GA4.
  - `REQUIRED`: campo obrigatório ausente (ex. `items`, `currency` quando `value`).
  - `VALUE_OUT_OF_BOUNDS`: limite de tamanho (`> 100` chars string, `> 40` chars name).

## Passos

1. Resolver spec → payload via mapping (ou usar JSON cru fornecido).
2. Montar body conforme regras.
3. `POST` para `https://<host>/debug/mp/collect?measurement_id=...&api_secret=...`.
4. Parsear `json.validationMessages`.
5. Retornar:
   - resultado (OK / INVALID);
   - tabela de mensagens;
   - diff sugerido na spec/mapping para fixar;
   - lembrete: sucesso aqui **não** garante que vai aparecer em Realtime; para isso validar também com Tag Assistant / DebugView.

## Exemplo de chamada

```bash
curl -sS -X POST \
  "https://www.google-analytics.com/debug/mp/collect?measurement_id=${GA4_MEASUREMENT_ID}&api_secret=${GA4_API_SECRET}" \
  -H "Content-Type: application/json" \
  -d '{
    "client_id": "12345.67890",
    "validation_behavior": "ENFORCE_RECOMMENDATIONS",
    "events": [{
      "name": "purchase",
      "params": {
        "session_id": "1712345678",
        "engagement_time_msec": 100,
        "transaction_id": "DEBUG-000123",
        "currency": "BRL",
        "value": 199.8,
        "items": [{"item_id":"SKU-001","item_name":"Camiseta básica","price":99.9,"quantity":2}]
      }
    }]
  }'
```

## Delegação

Para validação em lote (todas as specs), use o agente `tracking-test-engineer`, que orquestra a pirâmide completa.
