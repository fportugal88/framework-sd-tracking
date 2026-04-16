---
name: generate-tracking-tests
description: Use esta skill para gerar a pirâmide de testes do framework (contract com AJV, integration com /debug/mp/collect, synthetic Playwright/Cypress, warehouse assertions BigQuery) a partir das specs. Escreve em tracking/tests/{unit,contract,integration,synthetic}. Invoque sempre após /create-event-spec ou /generate-ga4-mapping, e antes de /release-tracking-event.
---

# Geração da Pirâmide de Testes de Tracking

## Níveis gerados

1. **Lint/unit** — `tracking/tests/unit/<spec>.spec.ts`
   - Conferem estrutura da spec (naming, versao, owner, consent_profile, assertions não vazias).
2. **Contract** — `tracking/tests/contract/<evento>.contract.spec.ts`
   - AJV compila `tracking/mappings/datalayer/ecommerce.contract.json`.
   - Cada spec gera fixtures válidas e inválidas (boundary cases de `price`, `quantity`, `currency`, `transaction_id`, PII).
3. **Integration** — `tracking/tests/integration/<evento>.mp-debug.spec.ts`
   - `POST https://www.google-analytics.com/debug/mp/collect?measurement_id=<id>&api_secret=<secret>`
   - Body com `validation_behavior: ENFORCE_RECOMMENDATIONS`.
   - Espera `validationMessages: []`.
   - **Lembrete embutido no teste**: endpoint retorna `2xx` mesmo com payload inválido; só `validationMessages` prova correção.
4. **Synthetic** — `tracking/tests/synthetic/<fluxo>.e2e.ts` (Playwright ou Cypress)
   - Navega fluxo real (PDP → add_to_cart → checkout → purchase).
   - Intercepta `dataLayer` e request ao sGTM, afirmando eventos esperados em ordem.
5. **Warehouse assertions** — `tracking/tests/integration/bq-<evento>.sql`
   - SQL de cobertura: `%` de purchase sem `transaction_id`, duplicidade por `transaction_id`, `%` de eventos sem `items`, cobertura de `user_id` em sessões autenticadas, delta sGTM vs GA4.

## Regras

- Usar o JSON Schema real como fonte de verdade (nunca duplicar schema no teste).
- Tests NÃO devem fazer `fetch` a endpoints de produção; somente `/debug/mp/collect` em CI.
- Segredos (`GA4_API_SECRET`, `GA4_MEASUREMENT_ID`) apenas via env vars, nunca hardcoded.
- Fixtures inválidas cobrem:
  - `value` com `tax`/`shipping` incluídos (deve falhar);
  - `purchase` sem `transaction_id`;
  - `item` sem `item_id` e sem `item_name`;
  - `currency` fora de ISO 4217;
  - PII em qualquer campo (`email`, `phone`, `cpf`, …).
- Tests de `purchase` incluem caso de **duplicidade**: mesmo `transaction_id` disparado duas vezes deve ser detectado pelo monitor (não silenciosamente aceito).

## Template AJV (contract)

```typescript
import Ajv from 'ajv';
import addFormats from 'ajv-formats';
import schema from '../../mappings/datalayer/ecommerce.contract.json';

const ajv = new Ajv({ allErrors: true, strict: true });
addFormats(ajv);
const validate = ajv.compile(schema);

describe('<evento> contract', () => {
  it('payload válido passa', () => {
    const payload = {/* fixture válida */};
    expect(validate(payload)).toBe(true);
  });
  it('payload inválido falha com erro previsível', () => {
    const payload = {/* fixture inválida */};
    expect(validate(payload)).toBe(false);
    expect(validate.errors).toMatchSnapshot();
  });
});
```

## Template MP debug (integration)

```typescript
test('<evento> passa no Measurement Protocol debug', async () => {
  const url = `https://www.google-analytics.com/debug/mp/collect?measurement_id=${process.env.GA4_MEASUREMENT_ID}&api_secret=${process.env.GA4_API_SECRET}`;
  const body = {
    client_id: '12345.67890',
    validation_behavior: 'ENFORCE_RECOMMENDATIONS',
    events: [{ name: '<evento>', params: { /* ... */ session_id: '...', engagement_time_msec: 100 } }]
  };
  const res = await fetch(url, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) });
  const json = await res.json();
  expect(json.validationMessages).toEqual([]);
});
```

## Passos

1. Ler specs + mapping + contrato JSON Schema.
2. Para cada spec ativa, gerar arquivos nos 4+1 níveis.
3. Criar pasta `tracking/tests/fixtures/<evento>/` com casos válidos e inválidos.
4. Atualizar `tracking/tests/README.md` com como rodar localmente e em CI.
5. Registrar no changelog.

## Delegação

Use o agente `tracking-test-engineer` para:
- gerar testes de fluxos complexos multi-evento;
- criar fixtures em massa;
- manter snapshot de erros AJV em sincronia com mudanças de schema.
