---
name: generate-ga4-mapping
description: Use esta skill para gerar tracking/mappings/ga4/ecommerce.map.yaml, que traduz cada spec em mapeamento explícito dataLayer -> parâmetros GA4 (event_name, event_params, items[*], user_properties). Garante alinhamento com os eventos recomendados do GA4 standard ecommerce, respeita limites (25 params/event, 27 custom item params), e documenta campos excluídos por política de privacidade.
---

# Geração de Mapping dataLayer → GA4

## Objetivo

Produzir um arquivo declarativo que associa cada evento da spec ao seu payload final GA4, incluindo:
- `event_name` canônico ou custom;
- `event_params` (currency, value, transaction_id, coupon, customer_type, …);
- `items[*]` com parâmetros obrigatórios e custom;
- `user_properties` quando aplicável;
- campos explicitamente excluídos (com motivo).

## Formato

```yaml
# tracking/mappings/ga4/ecommerce.map.yaml
version: 1
events:
  - spec_id: ecommerce.purchase
    spec_version: 1.0.0
    ga4_event_name: purchase
    source: dataLayer.ecommerce
    params:
      transaction_id: ecommerce.transaction_id
      currency: ecommerce.currency
      value: ecommerce.value
      tax: ecommerce.tax
      shipping: ecommerce.shipping
      coupon: ecommerce.coupon
      customer_type: ecommerce.customer_type
    items:
      source: ecommerce.items[*]
      required: [price, quantity]
      any_of: [item_id, item_name]
      map:
        item_id: item_id
        item_name: item_name
        item_brand: item_brand
        item_category: item_category
        item_variant: item_variant
        price: price
        quantity: quantity
    user_properties:
      user_type: user.type
    excluded:
      - field: user.email
        reason: "PII - bloqueado em GA4 event stream"
      - field: user.cpf
        reason: "PII - bloqueado em GA4 event stream"
    consent:
      profile: analytics
      cookieless_behavior: allowed
```

## Regras

1. **Ecommerce canônico** tem prioridade. `purchase`, `view_item`, `add_to_cart`, `begin_checkout`, `add_payment_info`, `add_shipping_info`, `view_cart`, `remove_from_cart`, `select_item`, `view_item_list`, `select_promotion`, `view_promotion`, `refund`.
2. **Limites GA4**: até 25 event params customizados; até 27 custom item parameters por item; nome ≤ 40 chars; valor string ≤ 100 chars.
3. **`value`** deve vir documentado como `Σ(price*quantity)` sem shipping/tax. Se a spec declara tax/shipping no ecommerce, mapeie-os como params independentes (`tax`, `shipping`), NÃO como parte de `value`.
4. **PII** listada em `excluded` com motivo. Nunca mapear email/telefone/CPF/RG/nome diretamente.
5. **User properties** são pseudônimas, nunca PII. `user_id` vai como parâmetro de evento (`user_id` no nível do evento/GA4 SDK), NÃO como user property nem Custom Dimension.
6. **Deduplicação**: se `dedupe_key` está na spec, replicar aqui em `quality.dedupe_key`.
7. **Consent**: profile herdado da spec; `cookieless_behavior` vai `allowed | blocked` conforme política.

## Passos

1. Ler specs e schemas compartilhados.
2. Para cada spec, emitir entrada em `events:`.
3. Validar limites GA4. Se exceder, falhar com mensagem clara indicando quais params cortar ou consolidar em um campo composto.
4. Gerar `tracking/mappings/ga4/ecommerce.map.yaml`.
5. Diff contra versão anterior se existir, logar mudanças no `tracking/docs/changelog.md`.

## Saída esperada

- Arquivo YAML gerado + relatório de cobertura (specs → params).
- Matriz de campos excluídos por privacidade.
- Próximo passo: `/generate-gtm-payload`.

## Delegação

Para conflitos entre spec e schema standard, ou decisão sobre event_name custom vs canônico, use o agente `ga4-mapper`.
