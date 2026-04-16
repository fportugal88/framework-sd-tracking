---
name: generate-datalayer-schema
description: Use esta skill para gerar ou atualizar o contrato JSON Schema do dataLayer (tracking/mappings/datalayer/ecommerce.contract.json) a partir das specs em tracking/specs/events. Produz um schema consolidado que o frontend pode consumir para validar dataLayer.push em runtime e que os contract tests podem carregar. Respeita regras do GA4 standard ecommerce e inclui _meta obrigatório com spec_id e spec_version.
---

# Geração de JSON Schema do dataLayer

## Objetivo

Derivar, a partir do conjunto de specs YAML, um único JSON Schema (Draft 2020-12 ou 2019-09) que descreve tudo que pode legitimamente ser empurrado em `window.dataLayer` pelo site/app.

## Princípios

- **Única fonte de verdade = specs**. Nunca editar o schema manualmente.
- **Enum `event`**: lista fechada dos `trigger.datalayer_event` das specs ativas (não-deprecated).
- **`ecommerce` como objeto discriminado**: cada evento mapeia para um subset obrigatório (`items`, `currency`, `value`, `transaction_id`…).
- **`_meta` obrigatório**: `spec_id` + `spec_version` + `consent_snapshot`.
- **Shared schemas** (`item.schema.json`, `money.schema.json`) são referenciados via `$ref`, não inlinados.

## Contrato base

```json
{
  "$id": "schemas/datalayer/ga4-ecommerce.events.schema.json",
  "type": "object",
  "required": ["event", "_meta"],
  "properties": {
    "event": { "type": "string", "enum": ["..."] },
    "ecommerce": { "type": "object" },
    "_meta": {
      "type": "object",
      "required": ["spec_id", "spec_version"],
      "properties": {
        "spec_id": { "type": "string" },
        "spec_version": { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
        "consent_snapshot": { "type": "object" }
      }
    }
  },
  "allOf": [
    { "if": { "properties": { "event": { "const": "purchase" } } },
      "then": { "required": ["ecommerce"],
                "properties": { "ecommerce": { "required": ["transaction_id", "currency", "value", "items"] } } } }
  ]
}
```

## Regras de geração

1. Varre `tracking/specs/events/*.yaml` ignorando `deprecated: true`.
2. Para cada evento, cria um bloco `allOf[{ if, then }]` que condiciona por `event`.
3. `items` sempre referencia `../../specs/shared/item.schema.json`.
4. `currency` vem de `shared/money.schema.json` (pattern `^[A-Z]{3}$`).
5. `value` tem `minimum: 0`, comentário `$comment` alertando "não incluir tax/shipping".
6. Emite `additionalProperties: false` em `ecommerce` para evitar drift silencioso.
7. Inclui `$comment` com o `id` + `versao` da spec em cada ramo.
8. Quando há custom parameters (fora do canônico), cria definição em `$defs`.

## Passos

1. Carregar todas specs.
2. Carregar `tracking/specs/shared/item.schema.json` e `money.schema.json` (falhar se faltarem).
3. Gerar schema consolidado.
4. Escrever em `tracking/mappings/datalayer/ecommerce.contract.json` (pretty-printed, 2-spaces).
5. Rodar `ajv compile` (ou equivalente) para garantir que o schema é sintaticamente válido.
6. Atualizar `tracking/docs/changelog.md` registrando mudança.
7. Imprimir diff resumido (eventos adicionados/removidos/alterados).

## Saída esperada

- Arquivo JSON Schema completo e válido.
- Lista de eventos cobertos.
- Avisos para eventos `deprecated` ainda presentes no datalayer legado.
- Sugestão de rodar `/generate-tracking-tests` a seguir.

## Delegação

Para fluxos complexos multi-domínio ou quando o contrato precisa ser fatiado por área (checkout, catálogo, pós-venda), use o agente `ga4-mapper`.
