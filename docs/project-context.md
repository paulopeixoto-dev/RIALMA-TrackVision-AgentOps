# Contexto do Projeto

Atualizado em: 2026-08-12

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
- Registrar padroes de backend Laravel que agentes e devs devem seguir.
- Registrar padroes de frontend Vue que agentes e devs devem seguir.
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

## Padrao inicial do backend

O backend deve seguir Laravel com Controllers magros, validacao por Form Requests,
Eloquent usado de forma eficiente e logica de negocio isolada em Services ou Actions.
Esse padrao esta detalhado em `docs/backend-laravel-guidelines.md` e deve orientar
qualquer demanda futura de API, regra de negocio, banco ou integracao.

## Padrao inicial do frontend

O frontend deve seguir Vue 3 com Composition API, Single-File Components,
componentes pequenos, composables para logica reutilizavel, Pinia quando houver estado
compartilhado e services/clients para consumo de API. Esse padrao esta detalhado em
`docs/frontend-vue-guidelines.md` e deve orientar qualquer demanda futura de tela,
componente, rota, estado ou integracao com API.

## CI/CD

Backend e frontend possuem workflows GitHub Actions para validacao em `push` e
`pull_request` para `main`.

- Backend: Composer validate, Pint e testes Laravel.
- Frontend: ESLint, Vitest e build Vite.

Deploy automatico ainda nao faz parte do escopo; sera definido junto com a
infraestrutura de producao e homologacao de campo.
