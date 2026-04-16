---
name: generate-tracking-docs
description: Use esta skill para gerar e manter a documentação operacional do framework (changelog, matriz de cobertura de eventos, dicionário de parâmetros, runbooks, exemplos de dataLayer). Consome specs, mappings e tests, e escreve em tracking/docs. Invoque após qualquer geração para manter artefatos sincronizados.
---

# Geração de Docs do Framework de Tracking

## Artefatos gerados

1. **`tracking/docs/changelog.md`** — Keep a Changelog, separado em Unreleased / versões publicadas por container GTM.
2. **`tracking/docs/event-matrix.md`** — tabela `spec_id | versao | owner | ga4_event | consent | destinos | status`.
3. **`tracking/docs/dictionary.md`** — cada parâmetro usado em qualquer spec, com tipo, descrição, PII? e origem.
4. **`tracking/docs/examples/<evento>.md`** — exemplo de `dataLayer.push` + request esperado ao sGTM + resposta MP debug esperada.
5. **`tracking/docs/runbooks/`**
   - `on-drop-in-purchase-volume.md`
   - `on-duplicate-transactions.md`
   - `on-consent-mode-rejection-spike.md`
   - `on-sgtm-error-spike.md`
   - `rollback-container-version.md` (republicar versão anterior via Tag Manager API).
6. **`tracking/docs/governance.md`** — fluxo draft → analytics → eng → privacidade → geração → preview → publish → monitoramento → deprecação.
7. **`tracking/docs/privacy-matrix.md`** — classificação de campos em: permitidos por default, sob consent analítico, sob consent adicional + jurídico, proibidos.

## Regras

- **Fonte única de verdade = specs**. Documentação NUNCA contradiz spec.
- Todo exemplo de código usa fixture válida dos testes (não inventar payload).
- Referências a comportamento GA4 devem citar a regra exata (ex. "value = Σ(price*quantity) sem tax/shipping"; "`transaction_id` obrigatório em `purchase`"; "até 27 custom params por item").
- Changelog segue SemVer independente por spec + notas da versão publicada do container.
- Runbooks têm estrutura fixa: sintoma → diagnóstico em 5 passos → mitigação → rollback → postmortem mínimo.

## Passos

1. Ler specs, mapping, contrato, fixtures de teste.
2. Regerar `event-matrix.md` e `dictionary.md` a cada run (arquivos são derivados, não manuais).
3. Preservar seções manuais marcadas com `<!-- MANUAL:begin -->...<!-- MANUAL:end -->` em `governance.md` e runbooks.
4. Abrir `changelog.md` seção `Unreleased` com bullets por spec alterada.
5. Emitir relatório final: arquivos gerados + arquivos preservados + warnings (ex. spec sem owner, parâmetro sem descrição).

## Delegação

Para docs longas, padronização editorial e tradução PT-BR ↔ EN, use o agente `tracking-docs-writer`.
