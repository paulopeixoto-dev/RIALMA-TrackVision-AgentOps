# Task 8 Documentation Review Fixes

## Correcoes

- O teste de campo agora confirma a criacao do `CaptureEvent` e registra a presenca ou ausencia inicial de `support_image` antes da recuperacao.
- O comando recorrente e validado antes de qualquer acao manual; `Recuperar apoio` ficou explicitamente separado como acao opcional do operador.
- O runbook declara que o processamento LPR continua quando o NVR esta indisponivel, nao configurado ou fora da retencao.
- O runbook permite somente endpoints historicos/API suportados e proibe scraping da interface web do NVR.
- O checklist exige confirmar que o canal do NVR mapeado esta ativamente gravando.
- A secao de seguranca documenta Laravel encrypted casts e proibe credenciais em Resources, payloads de bootstrap e bundles do frontend.

## Verificacao

- Revisao textual dos dois documentos operacionais e do diff do commit.
- Busca por todos os termos dos cinco findings e por marcadores de trabalho pendente.
- Nenhum backend ou frontend foi alterado.
