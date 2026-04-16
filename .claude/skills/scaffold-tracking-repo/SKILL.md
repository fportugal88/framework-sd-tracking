---
name: scaffold-tracking-repo
description: Use esta skill para inicializar a estrutura de repositório do framework spec-driven de tracking GA4 + GTM Server-Side (pastas tracking/specs, mappings, gtm/{web,server}, tests, tools, docs), incluindo schemas compartilhados (item, money, consent-profiles), governance.md, changelog.md e convenções de naming. Invoque sempre que o repositório ainda não tiver a árvore padrão descrita na spec.
---

# Scaffold do Framework Spec-Driven de Tracking

## Objetivo

Gerar a árvore de diretórios canônica descrita em `Tracking-deep-research-report.md`, com os schemas compartilhados, profiles de consentimento e documentos de governança mínimos. O scaffold deve ser idempotente (não sobrescreve arquivos existentes sem confirmação).

## Estrutura alvo

```
tracking/
  specs/
    events/                 # view_item.v1.yaml, add_to_cart.v1.yaml, purchase.v1.yaml
    shared/
      item.schema.json
      money.schema.json
      consent-profiles.yaml
  mappings/
    ga4/
      ecommerce.map.yaml
    datalayer/
      ecommerce.contract.json
  gtm/
    web/
      generated/
      manual-overrides/
    server/
      generated/
      templates/
      transformations/
  tests/
    unit/
    contract/
    integration/
    synthetic/
  tools/
    generators/
    linters/
    validators/
  docs/
    changelog.md
    governance.md
    runbooks/
```

## Passos

1. Verifique a existência de `tracking/` antes de criar. Se já existir, liste o que falta e proponha patch.
2. Crie `tracking/specs/shared/item.schema.json` seguindo o `anyOf: [{required: [item_id]}, {required: [item_name]}]` e `required: [price, quantity]`.
3. Crie `tracking/specs/shared/money.schema.json` com `currency` ISO 4217 (`^[A-Z]{3}$`) e `value: number >= 0`.
4. Crie `tracking/specs/shared/consent-profiles.yaml` com os três profiles (`analytics`, `ads`, `strictly_necessary`).
5. Crie `tracking/docs/governance.md` explicando SemVer independente do container, ownership (Produto/Analytics + revisão Eng + Privacidade) e gate de release.
6. Crie `tracking/docs/changelog.md` no padrão Keep a Changelog.
7. Gere `.gitignore` do framework excluindo `tracking/gtm/*/generated/` somente se `generated/` for reconstruído por CI — caso contrário mantenha versionado.
8. Ao terminar, imprima uma matriz do que foi criado vs pulado.

## Convenções obrigatórias

- Nomes de evento em `snake_case`, começando com letra, apenas `[a-z0-9_]`, sem prefix reservados do GA4.
- Toda spec vive em `tracking/specs/events/<nome>.v<major>.yaml`.
- Campos técnicos vão em `_meta` / `event_context` / `tracking`, nunca misturados ao payload semântico.
- Semântica é imutável: mudar significado ⇒ bump de MAJOR + arquivo novo.

## Entregável

Ao final, gere um resumo com:
- arquivos criados (paths);
- arquivos pulados (já existiam);
- próximos comandos recomendados (`/create-event-spec`, `/validate-specs`).
