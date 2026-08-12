# RIALMA TrackVision AgentOps

Repositorio de orientacao agentica, documentacao de IA e governanca de contexto do
projeto RIALMA TrackVision.

Este repositorio nao contem codigo de produto. Ele centraliza instrucoes para agentes,
boas praticas de desenvolvimento agentico, templates de demanda, documentacao sobre
skills/MCPs e decisoes de contexto que devem orientar o trabalho nos repositorios de
backend e frontend.

## Repositorios do projeto

| Area | Repositorio local | Remoto | Responsabilidade |
| --- | --- | --- | --- |
| AgentOps | `.` | `https://github.com/paulopeixoto-dev/RIALMA-TrackVision-AgentOps.git` | IA, contexto, AGENTS.md, prompts, skills, MCPs e documentacao agentica |
| Backend | `./RIALMA-TrackVision-Backend` | `https://github.com/paulopeixoto-dev/RIALMA-TrackVision-Backend.git` | API, regras de negocio, banco, integracoes e servicos |
| Frontend | `./RIALMA-TrackVision-Frontend` | `https://github.com/paulopeixoto-dev/RIALMA-TrackVision-Frontend.git` | Interface web, experiencia do usuario e integracao com API |

As pastas de backend e frontend sao repositorios Git independentes e ficam ignoradas
no `.gitignore` deste repositorio.

## Como usar

1. Leia `AGENTS.md` antes de iniciar qualquer tarefa agentica.
2. Use `docs/prompt-templates/demanda-feature.json` para registrar demandas.
3. Consulte `docs/agentic-development.md` para seguir o fluxo de trabalho.
4. Consulte `docs/backend-laravel-guidelines.md` antes de alterar o backend Laravel.
5. Consulte `docs/frontend-vue-guidelines.md` antes de alterar o frontend Vue.
6. Consulte `docs/skills-and-mcp.md` antes de propor uma skill, plugin ou MCP.
7. Faca commits separados no repositorio correto: AgentOps, backend ou frontend.

## Principios

- Contexto duravel fica em documentacao versionada.
- Decisoes tecnicas importantes precisam ser registradas.
- Agentes devem planejar antes de alterar comportamento relevante.
- Backend e frontend nunca devem ser versionados dentro do AgentOps.
- Segredos, tokens, chaves e credenciais nunca entram no Git.
- Cada tarefa termina com verificacao objetiva e status limpo.
