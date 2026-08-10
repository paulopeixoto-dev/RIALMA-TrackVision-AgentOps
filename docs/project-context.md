# Contexto do Projeto

Atualizado em: 2026-08-10

## Visao geral

RIALMA TrackVision e organizado em tres repositorios:

- AgentOps: documentacao agentica, contexto de IA, AGENTS.md, prompts, skills e MCPs.
- Backend: API, regras de negocio, persistencia e integracoes.
- Frontend: aplicacao web, experiencia do usuario e consumo da API.

O AgentOps e a fonte de verdade para como agentes devem colaborar com o projeto.
Ele nao substitui documentacao tecnica especifica de backend e frontend, mas define
o modo de trabalho para que todos os agentes e devs sigam o mesmo processo.

## Layout local

```text
C:\projetos\rialma\RIALMA-TrackVision-AgentOps
|-- AGENTS.md
|-- README.md
|-- docs/
|-- RIALMA-TrackVision-Backend/    # repo separado, ignorado pelo AgentOps
`-- RIALMA-TrackVision-Frontend/   # repo separado, ignorado pelo AgentOps
```

## Repositorios remotos

- AgentOps: `https://github.com/paulopeixoto-dev/RIALMA-TrackVision-AgentOps.git`
- Backend: `https://github.com/paulopeixoto-dev/RIALMA-TrackVision-Backend.git`
- Frontend: `https://github.com/paulopeixoto-dev/RIALMA-TrackVision-Frontend.git`

## Responsabilidades do AgentOps

- Manter regras duraveis para agentes em `AGENTS.md`.
- Registrar prompts reutilizaveis e formatos de especificacao.
- Documentar quando criar ou usar skills.
- Documentar quando conectar MCPs e quais limites de seguranca aplicar.
- Registrar decisoes de contexto que afetam backend e frontend.
- Melhorar continuamente o processo conforme surgirem repeticoes, erros ou revisoes.

## O que nao pertence ao AgentOps

- Codigo da API.
- Codigo da interface.
- Artefatos de build.
- Variaveis de ambiente reais.
- Dumps de banco ou dados sensiveis.
- Documentacao operacional que seja exclusiva de um unico repositorio e nao tenha
  impacto agentico.

