# Desenvolvimento Agentico

Este guia define como agentes de IA devem atuar no RIALMA TrackVision.

## Entrada de demanda

Toda demanda deve deixar claros quatro pontos:

- objetivo: o que precisa mudar ou existir;
- contexto: arquivos, repositorios, exemplos e decisoes relevantes;
- restricoes: tecnologia, prazo, seguranca, padroes e limites;
- criterio de aceite: como saber que a tarefa terminou.

Use `docs/prompt-templates/demanda-feature.json` quando a demanda envolver produto,
backend, frontend ou documentacao.

## Fluxo recomendado

1. Classificar a demanda: feature, bugfix, documentacao, infraestrutura ou pesquisa.
2. Identificar repositorios afetados: AgentOps, backend, frontend ou combinacao.
3. Ler contexto minimo necessario.
4. Separar duvidas reais de suposicoes seguras.
5. Propor design para mudancas relevantes.
6. Implementar em passos pequenos.
7. Verificar com comandos, testes, lint, build ou revisao documental.
8. Registrar decisoes e atualizar documentacao.
9. Fazer commit focado.
10. Informar resultado e pendencias.

## Definition of Done

Uma tarefa so deve ser considerada concluida quando:

- os arquivos alterados pertencem ao repositorio correto;
- o escopo pedido foi atendido;
- criterios de aceite foram verificados;
- comandos relevantes foram executados ou a impossibilidade foi explicada;
- documentacao necessaria foi atualizada;
- `git status --short --branch` foi conferido;
- o resumo final explica mudancas e verificacao.

## Contexto e higiene

- Leia arquivos especificos antes de editar.
- Prefira buscas com `rg` para localizar contexto rapidamente.
- Evite carregar documentacao inteira quando um arquivo de rota resolve a tarefa.
- Nao misture logs extensos com decisoes finais.
- Transforme feedback recorrente em regra duravel no `AGENTS.md` ou em uma skill.

## Planejamento

Planeje antes de implementar quando a tarefa envolver:

- nova funcionalidade;
- alteracao de arquitetura;
- contratos de API;
- banco de dados;
- autenticacao ou permissao;
- integracoes externas;
- UI complexa;
- risco de regressao;
- mudancas em mais de um repositorio.

O plano deve ser curto, verificavel e dividido por repositorio.

## Revisao

Ao revisar trabalho feito por agente:

- procure riscos antes de elogios;
- cite arquivos e linhas quando aplicavel;
- diferencie bug real de preferencia;
- indique teste ausente quando relevante;
- mantenha feedback acionavel.

## Commits

Commits devem ser pequenos e coerentes. Evite juntar documentacao agentica,
backend e frontend em um unico commit. Quando uma demanda tocar varios repositorios,
crie commits separados em cada um.

