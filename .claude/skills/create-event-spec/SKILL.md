---
name: create-event-spec
description: Use esta skill sempre que o usuário pedir para criar, rascunhar ou "spec-ar" um novo evento de tracking (ex. "cria spec do begin_checkout", "adiciona evento remove_from_cart"). A skill produz o arquivo YAML em tracking/specs/events/<evento>.v<major>.yaml respeitando semântica canônica GA4, consent profile, dedupe_key, assertions e rastreabilidade. Invocar também quando houver um PRD/ticket descrevendo intenção de evento e for preciso traduzi-lo em contrato formal.
---

# Criação de Spec de Evento (GA4 + sGTM)

## Quando usar

- Pedido para criar novo evento de tracking.
- Tradução de PRD/ticket/wireframe em contrato formal.
- Versionamento de evento existente (bump de MAJOR).

## Entradas esperadas

- Nome semântico pretendido (o agente deve propor o canônico GA4 se houver equivalente: `view_item`, `add_to_cart`, `begin_checkout`, `purchase`, `refund`, etc.).
- Gatilho (browser, backend, backoffice, MP).
- Owner (default: `analytics`).
- Campos de negócio relevantes.
- Restrição de consentimento (default: `analytics`).

## Regras obrigatórias

1. **Naming GA4**: `snake_case`, só `[a-z0-9_]`, começa com letra, sem prefixes reservados (`ga_`, `google_`, `firebase_`). Case-sensitive.
2. **Canônico > custom**: Se houver evento oficial do GA4 para o caso, o `ga4.event_name` deve ser ele. Só crie custom event quando não houver correspondência.
3. **Ecommerce payload**:
   - `view_item` / `add_to_cart`: `items` obrigatório; `currency` obrigatório se `value` presente; `value = Σ(price*quantity)` (NÃO incluir `shipping`/`tax`).
   - `purchase`: `transaction_id` obrigatório e único; `shipping`/`tax` fora do `value`.
   - Item precisa ter pelo menos `item_id` OU `item_name` + `price` + `quantity`.
4. **Identidade**: nunca colocar PII bruta. `user_id` só para autenticado, first-party, sem e-mail/telefone/CPF/nome. `client_id` vem do `_ga`.
5. **Campos técnicos** ficam em `_meta`, nunca misturados com o payload semântico.
6. **Dedupe**: se o evento representa transação/commit, `quality.dedupe_key` deve ser definido (ex. `transaction_id`).
7. **Versionamento SemVer** independente do container. Semântica mudou ⇒ novo arquivo `.v<MAJOR+1>.yaml`.

## Template YAML

```yaml
id: <dominio>.<evento>
versao: 1.0.0
owner: analytics
descricao: <o que significa, quando dispara, quem consome>
consent_profile: analytics            # analytics | ads | strictly_necessary | <custom>
ga4:
  event_name: <evento_canonico_ou_custom>
trigger:
  source: web                         # web | backend | backoffice | mp
  datalayer_event: <evento>
payload:
  required:
    - <campos>
  properties:
    currency:
      type: string
      format: iso4217
      example: BRL
    value:
      type: number
      regra: "soma(price * quantity) dos items"
    items:
      type: array
      minItems: 1
      itemSchemaRef: ../shared/item.schema.json
quality:
  dedupe_key: null                    # ex. transaction_id
  pii_allowed: false
  assertions:
    - "..."
observability:
  volume_min_daily: null
  rejection_rate_max: 0.01
  duplicate_rate_max: 0.001
```

## Passos que a skill executa

1. Conferir se `tracking/specs/events/` existe; se não, orientar `/scaffold-tracking-repo` antes.
2. Propor `event_name` canônico, confirmar com o usuário se diverge.
3. Gerar YAML seguindo template, preenchendo assertions específicas do evento.
4. Registrar entrada no `tracking/docs/changelog.md` na seção Unreleased.
5. Rodar um sanity check equivalente ao `/validate-specs` e retornar violações.
6. Sugerir próximos comandos: `/generate-datalayer-schema`, `/generate-tests`, `/privacy-consent-review`.

## Exemplos de boa saída

Ver exemplos canônicos de `view_item`, `add_to_cart` e `purchase` no `Tracking-deep-research-report.md` (seção "Mapeamento técnico").

## Quando delegar

Use o agente `tracking-spec-author` para casos com múltiplos eventos, refinamento iterativo ou quando for preciso manter consistência com specs já versionadas.
