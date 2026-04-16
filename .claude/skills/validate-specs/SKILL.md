---
name: validate-specs
description: Use esta skill para lintar e validar todas as specs de evento em tracking/specs/events. Verifica naming GA4 (snake_case, sem prefix reservado), presença de owner/consent_profile/versao, consistência ecommerce (items, currency quando há value, transaction_id em purchase), limites de custom definitions, proibição de PII, schemas compartilhados e coerência com mappings/datalayer/ecommerce.contract.json. Invoque antes de qualquer release, geração de GTM ou publicação.
---

# Lint e Validação de Specs de Tracking

## Objetivo

Detectar violações estáticas ANTES de gerar containers GTM, tests ou docs. Falhas aqui = bloqueio de release.

## Regras aplicadas

### 1. Naming (GA4)
- `^[a-z][a-z0-9_]{0,39}$`
- Não começar com `ga_`, `google_`, `firebase_`, `_`.
- Não colidir com eventos automáticos do GA4 (ex. `session_start`, `page_view`, `first_visit`, `user_engagement`).

### 2. Estrutura YAML
Campos obrigatórios em cada spec:
- `id` (formato `<dominio>.<evento>`)
- `versao` (SemVer: `MAJOR.MINOR.PATCH`)
- `owner`
- `descricao`
- `consent_profile`
- `ga4.event_name`
- `trigger.source` ∈ {web, backend, backoffice, mp}
- `payload.required` (lista, pode ser vazia só em eventos sem dados)
- `quality.pii_allowed` (boolean)
- `quality.assertions` (lista não vazia)

### 3. Ecommerce (se `ga4.event_name` for ecommerce canônico)
- `view_item`, `add_to_cart`, `view_cart`, `begin_checkout`, `add_shipping_info`, `add_payment_info`: `items` em required; `currency` obrigatório se `value` estiver em required.
- `purchase`: `transaction_id`, `currency`, `value`, `items` em required. `dedupe_key: transaction_id`.
- `refund`: `transaction_id` em required.
- `value` deve ter `regra` documentada explicando que é `Σ(price*quantity)` sem `shipping`/`tax`.

### 4. Item schema
- Referência `itemSchemaRef` deve existir.
- Schema de item precisa garantir `anyOf: [required: [item_id], required: [item_name]]` e `required: [price, quantity]`.
- Máximo de **27 custom parameters por item** (limite oficial GA4).

### 5. Limites GA4 (event parameters globais)
- Máximo 25 parâmetros custom por evento (fora dos automáticos).
- Nome de parâmetro: `snake_case`, ≤ 40 chars.
- Valor string ≤ 100 chars (alerta), truncamento conhecido em 500.

### 6. PII
Campos bloqueados no payload (case-insensitive match):
- `email`, `phone`, `telefone`, `cpf`, `cnpj`, `rg`, `passport`, `full_name`, `first_name`, `last_name`, `address`, `street`, `zipcode`, `cep`, `birthdate`, `ssn`, `credit_card`, `card_number`, `cvv`.

Se `quality.pii_allowed: true`, exigir `user_provided_data_policy: approved-by-legal` e hashing SHA-256 explícito.

### 7. Consent
`consent_profile` deve existir em `tracking/specs/shared/consent-profiles.yaml`. `strictly_necessary` só em eventos de segurança/fraude.

### 8. Identidade
- Se `user_id` presente: `pii_allowed: false` mantido; `user_id` marcado como não-PII, first-party, ≤ 256 chars.
- `user_id` nunca como Custom Dimension.
- `client_id` não deve aparecer em payload semântico (é metadata de transporte).

### 9. Consistência cruzada
- Todo evento em `tracking/specs/events/*.yaml` deve estar referenciado em `tracking/mappings/ga4/ecommerce.map.yaml` (quando aplicável) e em `tracking/mappings/datalayer/ecommerce.contract.json`.
- Não pode haver dois arquivos com o mesmo `id` + `versao`.
- Se existe `foo.v1.yaml` e `foo.v2.yaml`, v1 precisa estar marcado `deprecated: true` com `replaced_by: foo.v2`.

### 10. Measurement Protocol
Quando `trigger.source: mp`, exigir:
- `mp.client_id_source` documentado;
- `mp.session_id` e `mp.engagement_time_msec` previstos;
- nota explícita que `/debug/mp/collect` é usado só em dev, não em prod;
- `2xx` não é prova de sucesso semântico.

## Saída

Relatório markdown com três blocos:
1. **BLOCKERS** (impedem release)
2. **WARNINGS** (não bloqueiam mas exigem justificativa)
3. **INFO** (sugestões de melhoria)

Para cada violação: `file:line` + regra quebrada + sugestão de fix.

## Passos

1. Listar `tracking/specs/events/*.yaml`.
2. Carregar schemas em `tracking/specs/shared/`.
3. Aplicar regras 1–10.
4. Checar cruzamento com `tracking/mappings/`.
5. Retornar relatório. Se houver BLOCKER, a skill sinaliza `exit=1`.

## Delegação

Para batch grande ou auditoria profunda (100+ specs, histórico), delegue ao agente `tracking-linter`.
