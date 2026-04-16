---
name: privacy-consent-review
description: Use esta skill para revisar specs sob a ótica de privacidade (GDPR/LGPD/CCPA) e Consent Mode v2. Classifica campos em quatro classes (permitido default, sob consent analítico, sob consent adicional + jurídico, proibido), valida uso de user_id e user_provided_data, confere se o consent_profile faz sentido, e checa Consent Mode basic vs advanced. Invoque antes de qualquer release e sempre que surgir campo novo na spec.
---

# Revisão de Privacidade e Consent para Specs de Tracking

## Objetivo

Servir de gate legal/privacidade antes da geração de GTM e do release. Não substitui parecer jurídico, mas aplica mecanicamente o conjunto mínimo de checks documentado na spec do framework.

## Classificação obrigatória de campos

| Classe | Exemplos | Tratamento |
|---|---|---|
| **Permitido default** | IDs técnicos pseudônimos, moeda, valor, SKU, quantidade, `client_id`, `session_id` | Livre |
| **Sob consent analítico** | métricas de navegação, listas, promoções, `user_id` (autenticado) | Exige `consent_profile: analytics`, `analytics_storage = granted` |
| **Sob consent adicional + jurídico** | user-provided data (email hash, phone hash), sinais de Ads, enrichment identificável | Exige `ad_storage = granted`, base legal documentada, hashing SHA-256 + normalização, aprovação `user_provided_data_policy: approved-by-legal` |
| **Proibido no GA4/event stream** | email cru, telefone cru, nome, CPF, RG, endereço detalhado, cartão, CVV, data de nascimento | **Bloqueio imediato** |

## Regras aplicadas

1. **PII bruta** em qualquer campo ⇒ BLOCKER.
2. **`user_id`** deve ser:
   - first-party, sem PII;
   - ≤ 256 chars;
   - enviado apenas para usuário autenticado;
   - limpo como `null` no logout (não dummy value);
   - **não** registrado como Custom Dimension.
3. **Alerta BigQuery**: lembrar que `user_id` exportado no BigQuery **ignora consent status** — se propriedade está ligada ao BQ, registrar riscos.
4. **`client_id`** não armazenado quando `analytics_storage = denied`.
5. **Cookieless pings** em Consent Mode advanced podem produzir `user_pseudo_id` diferente por sessão no BigQuery ⇒ warn em specs que assumem persistência de identidade no warehouse.
6. **Consent Mode**:
   - decisão `basic` vs `advanced` documentada por região na `tracking/specs/shared/consent-profiles.yaml`;
   - Consent Mode configurado **apenas no container web** (sGTM recebe automaticamente via request); reprovar qualquer spec que tente configurar consent no server.
   - Não usar Custom HTML para setar consent; usar Consent APIs.
7. **User-provided data via Measurement Protocol**:
   - exigir SHA-256 + normalização;
   - entrada opt-in, aprovação jurídica explícita;
   - nunca rodar sem revisão caso a caso.
8. **GPC (Global Privacy Control)** reconhecido como opt-out válido em jurisdições CCPA. Spec deve documentar comportamento quando GPC=1.
9. **Consent profile** da spec deve existir em `consent-profiles.yaml`. `strictly_necessary` só em eventos de segurança/fraude.
10. **Sanitização server-side**: exigir que o mapping correspondente tenha `transformations: [redact_pii, allowlist_params]` — caso contrário BLOCKER.

## Saída

Relatório com:
- **BLOCKERS**: tudo que impede release (PII bruta, consent mode errado, falta de hashing em MP user_data, etc.).
- **WARNINGS**: pontos com risco, exigem justificativa (ex. `user_id` em propriedade BQ-exportada).
- **RECOMENDAÇÕES**: melhorias (ex. usar `region1` quando coleta UE; adicionar transformação de redaction adicional).
- **Classificação por campo**: tabela com `campo | classe | fonte | tratamento`.

## Passos

1. Carregar spec alvo (ou todas).
2. Para cada campo em `payload.properties` e em `mapping.params` + `items[*]`, resolver classe.
3. Verificar `consent_profile` vs `consent-profiles.yaml`.
4. Conferir transformações no server container.
5. Emitir relatório.
6. Se BLOCKERS, retornar `exit=1` e sugerir mudanças exatas.

## Delegação

Use o agente `privacy-consent-reviewer` quando:
- a revisão envolver mudanças em vários profiles;
- houver caso novo de user-provided data;
- for necessário redigir nota para o time jurídico.
