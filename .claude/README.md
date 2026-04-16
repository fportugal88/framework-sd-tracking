# Claude Code — Skills e Agentes do Framework Spec-Driven de Tracking

Este diretório contém as **skills** (slash commands user-invocable) e os **agentes especializados** que operacionalizam o framework descrito em `Tracking-deep-research-report.md`.

A premissa central é: **tracking como produto, com contrato versionado** (specs) como fonte única de verdade. IA entra como fábrica de artefatos, nunca como dona da semântica.

## Skills (`.claude/skills/*/SKILL.md`)

Invocáveis por `/ <nome>` no Claude Code.

| Skill | Para quê |
|---|---|
| `/scaffold-tracking-repo` | Cria a árvore canônica `tracking/{specs,mappings,gtm,tests,tools,docs}` com schemas compartilhados, profiles de consent e governance inicial. |
| `/create-event-spec` | Rascunha nova spec de evento em YAML, respeitando GA4 canônico, consent profile, itemSchemaRef e versionamento SemVer. |
| `/validate-specs` | Lint estático das specs — naming, ecommerce, PII, consent, limites GA4, consistência cruzada. |
| `/generate-datalayer-schema` | Gera `tracking/mappings/datalayer/ecommerce.contract.json` consolidado a partir das specs. |
| `/generate-ga4-mapping` | Gera `tracking/mappings/ga4/ecommerce.map.yaml` com params, items, exclusões e consent. |
| `/generate-gtm-payload` | Gera artefatos prontos para Tag Manager API em `tracking/gtm/{web,server}/generated/` (idempotente). |
| `/generate-tracking-tests` | Gera a pirâmide (unit, contract AJV, integration MP debug, synthetic Playwright/Cypress, warehouse BQ). |
| `/generate-tracking-docs` | Mantém changelog, event-matrix, dictionary, privacy-matrix, runbooks e exemplos. |
| `/mp-debug-validate` | Dispara payload contra `/debug/mp/collect` com `ENFORCE_RECOMMENDATIONS` e interpreta `validationMessages`. |
| `/privacy-consent-review` | Revisa campos sob GDPR/LGPD/CCPA/GPC e Consent Mode v2; classifica em 4 classes. |
| `/release-tracking-event` | Orquestra pipeline completo: gates → Tag Manager API (`sync` → `create_version` → `publish`) → smoke. |

## Agentes (`.claude/agents/*.md`)

Disponíveis via `Agent(subagent_type=...)`.

| Agente | Domínio |
|---|---|
| `tracking-spec-author` | Autoria e evolução de specs YAML. |
| `tracking-linter` | Auditoria estática profunda (BLOCKER/WARNING/INFO). |
| `ga4-mapper` | Tradução spec → GA4 canônico; gera mapping e contrato JSON Schema. |
| `gtm-web-architect` | Container web: dataLayer, GA4 Event Tag, Consent Mode, transport_url first-party. |
| `gtm-server-architect` | sGTM: clients, transformations de privacidade, templates sandboxed, first-party context. |
| `tracking-test-engineer` | Pirâmide de testes, fixtures, snapshots, CI gates. |
| `privacy-consent-reviewer` | Gate legal/privacidade; classificação de campos; Consent Mode; user-provided data. |
| `tracking-docs-writer` | Changelog, matriz, dicionário, runbooks, governance. |
| `observability-engineer` | Monitores, alertas, dashboards, SLOs, BQ assertions. |
| `tracking-release-manager` | Orquestração de release e rollback via Tag Manager API. |
| `ai-automation-engineer` | Geradores/linters/validators; matriz de confiabilidade; humano no circuito. |

## Fluxo típico recomendado

```
/scaffold-tracking-repo
  → /create-event-spec              (cada evento)
    → /validate-specs
    → /privacy-consent-review
    → /generate-ga4-mapping
    → /generate-datalayer-schema
    → /generate-gtm-payload
    → /generate-tracking-tests
    → /mp-debug-validate
    → /generate-tracking-docs
    → /release-tracking-event       (publica via Tag Manager API)
```

## Princípios herdados da spec

1. **Contrato versionado antes de tag.** SemVer independente do container.
2. **GA4 standard ecommerce como semântica canônica.** `value = Σ(price*quantity)`; `transaction_id` obrigatório em `purchase`; `items` sempre.
3. **First-party sGTM** com transport_url em subdomínio próprio. Cloud Run ≥ 3 instâncias em prod.
4. **Consent Mode configurado apenas no web**; sGTM respeita. `basic` vs `advanced` por região.
5. **Measurement Protocol complementa, não substitui.** `/debug/mp/collect` é dev-only. `2xx` ≠ sucesso semântico.
6. **Identidade**: `user_id` first-party, sem PII, ≤ 256 chars, `null` no logout, não-CD; `client_id` vem do `_ga`.
7. **Pirâmide de qualidade**: lint → contract → MP debug → preview → runtime → warehouse.
8. **IA é fábrica de artefatos**, nunca dona da semântica de negócio, decisão jurídica ou modelagem de identidade.
