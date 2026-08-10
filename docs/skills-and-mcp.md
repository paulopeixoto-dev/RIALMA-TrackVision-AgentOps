# Skills e MCP

Este documento define quando usar AGENTS.md, skills, plugins e MCP no TrackVision.

## AGENTS.md

Use `AGENTS.md` para regras duraveis que todo agente deve seguir no repositorio:

- mapa de repositorios;
- comandos padrao;
- convencoes de documentacao;
- limites de seguranca;
- expectativas de review;
- orientacoes que devem valer sempre.

Mantenha o arquivo curto. Quando um fluxo crescer demais, transforme em skill.

## Skills

Use skills para workflows repetiveis que precisam de processo proprio. Exemplos:

- preparar especificacao de demanda;
- revisar contrato de API;
- criar plano de testes;
- gerar release notes;
- validar documentacao de integracoes;
- revisar PR com checklist especifico do TrackVision.

Estrutura sugerida:

```text
.agents/
`-- skills/
    `-- nome-da-skill/
        |-- SKILL.md
        |-- references/
        |-- scripts/
        `-- assets/
```

Uma skill deve ser criada somente quando houver repeticao real ou risco de perda de
padrao entre agentes. Para tarefas raras, documente o processo em `docs/`.

## MCP

MCP conecta o agente a ferramentas externas e fontes de contexto. Use quando a tarefa
exigir algo fora do repositorio local.

Possiveis candidatos para o TrackVision:

| MCP ou conector | Uso esperado | Status |
| --- | --- | --- |
| GitHub | issues, PRs, reviews e automacoes de repositorio | a avaliar |
| Browser/Playwright | validacao visual e smoke tests de frontend | a avaliar |
| Figma | leitura de designs e tokens visuais | a avaliar |
| Linear/Jira | triagem e rastreamento de demandas | a avaliar |
| Docs internos | consulta de regras, processos e decisoes | a avaliar |

## Regras de seguranca para MCP

- Conectar somente ferramentas necessarias.
- Usar permissao minima.
- Evitar MCP com escrita quando leitura resolve a tarefa.
- Nunca versionar tokens ou credenciais.
- Registrar qual dado externo e acessado.
- Revisar riscos antes de automatizar acoes destrutivas.

## Skills + MCP

Combine skills e MCP quando um workflow repetivel depende de ferramentas externas.

Exemplo:

- Skill: "triagem-de-demanda"
- MCP: issue tracker para ler ticket e GitHub para localizar PRs relacionados
- Saida: resumo, riscos, repositorios afetados e plano de execucao

O workflow fica na skill. O acesso externo fica no MCP.

## Fontes oficiais usadas como referencia

- OpenAI Codex: `AGENTS.md` como orientacao duravel do repositorio.
- OpenAI Codex: skills como workflows reutilizaveis com `SKILL.md`, referencias,
  scripts e assets.
- OpenAI Codex: MCP como ponte para ferramentas e contexto externo.
- OpenAI Codex: boas praticas recomendam contexto claro, plano quando necessario,
  validacao e melhoria continua das instrucoes.
