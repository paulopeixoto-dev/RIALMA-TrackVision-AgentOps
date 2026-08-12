# AGENTS.md

## Proposito

Este arquivo orienta agentes de IA trabalhando no ecossistema RIALMA TrackVision.
Ele deve ser lido antes de qualquer alteracao em documentacao, backend ou frontend.

O repositorio atual e o AgentOps do projeto. Ele serve para contexto, governanca,
prompts, documentacao agentica, skills e MCPs. Codigo de produto deve ser criado nos
repositorios especificos de backend ou frontend.

## Mapa local

- AgentOps: `C:\projetos\rialma\RIALMA-TrackVision-AgentOps`
- Backend: `C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Backend`
- Frontend: `C:\projetos\rialma\RIALMA-TrackVision-AgentOps\RIALMA-TrackVision-Frontend`

Backend e frontend sao repositorios Git separados. Eles aparecem como pastas locais
para conveniencia, mas sao ignorados pelo Git do AgentOps.

## Regras de trabalho

- Nao criar codigo de backend ou frontend neste repositorio AgentOps.
- Antes de editar, rode `git status --short --branch` no repositorio correto.
- Nunca reverta mudancas que voce nao fez, a menos que o usuario peca explicitamente.
- Evite comandos destrutivos. Se forem inevitaveis, explique o risco antes.
- Preserve escopo: uma demanda deve alterar somente os repositorios necessarios.
- Nao assumir stack, framework, banco ou padroes sem verificar arquivos existentes ou
  especificacao aprovada.
- Escreva documentacao em pt-BR. Use termos tecnicos em ingles quando forem nomes de
  ferramentas, padroes ou comandos.
- Nao versionar `.env`, tokens, chaves, dumps com dados reais ou credenciais.

## Fluxo agentico padrao

1. Entender a demanda com objetivo, contexto, restricoes e criterio de aceite.
2. Verificar o estado dos repositorios relevantes.
3. Propor design ou plano quando a tarefa alterar arquitetura, comportamento,
   experiencia de usuario, dados, seguranca ou integracoes.
4. Implementar no repositorio correto.
5. Atualizar documentacao quando o comportamento, fluxo ou decisao mudar.
6. Rodar verificacoes adequadas ao tipo de mudanca.
7. Fazer commit focado com mensagem clara.
8. Informar o que mudou, onde mudou e como foi verificado.

## Uso de backend e frontend

Ao trabalhar no backend:

- Use o repositorio `RIALMA-TrackVision-Backend`.
- Siga o padrao Laravel documentado em `docs/backend-laravel-guidelines.md`.
- Mantenha Controllers magros: eles devem coordenar request, authorization,
  chamada de Service/Action e response, sem concentrar regra de negocio.
- Valide dados com Form Requests sempre que a entrada tiver regras, autorizacao ou
  mensagens proprias.
- Use Eloquent de forma eficiente, evitando N+1 e consultas espalhadas sem criterio.
- Isole logica de negocio em Services ou Actions testaveis.
- Em APIs, use versionamento, Resources para serializacao e middleware/policies para
  autorizacao recorrente.
- Em integracoes externas, configure timeouts e evite chamadas HTTP soltas no
  Controller.
- Leia configuracoes com `config()`, nao com `env()` fora de arquivos de config.
- Documente endpoints, contratos, regras de negocio e variaveis de ambiente.
- Inclua testes para regras de negocio, validacoes e integracoes criticas.

Ao trabalhar no frontend:

- Use o repositorio `RIALMA-TrackVision-Frontend`.
- Siga o padrao Vue documentado em `docs/frontend-vue-guidelines.md`.
- Use Vue 3, Composition API e Single-File Components como base.
- Mantenha componentes pequenos, com responsabilidade clara, props tipadas e emits
  explicitos.
- Isole chamadas de API em services/clients, nao em componentes espalhados.
- Use composables para logica reutilizavel e Pinia para estado compartilhado quando
  estado local nao bastar.
- Siga as regras essenciais, fortemente recomendadas e recomendadas do Vue Style Guide.
- Priorize fluxos completos, estados de carregamento, erro e vazio.
- Verifique responsividade e integracao com contratos da API.

## Documentacao

Documentacao agentica vive neste repositorio. Use preferencialmente:

- `docs/project-context.md` para mapa do projeto e responsabilidades.
- `docs/agentic-development.md` para fluxo de desenvolvimento agentico.
- `docs/backend-laravel-guidelines.md` para padroes obrigatorios do backend Laravel.
- `docs/frontend-vue-guidelines.md` para padroes obrigatorios do frontend Vue.
- `docs/skills-and-mcp.md` para criterios de uso de skills, plugins e MCPs.
- `docs/prompt-templates/` para prompts reutilizaveis.
- `docs/checklists/` para checklists operacionais.

## Skills

Use skills para fluxos repetiveis que precisam de instrucoes, exemplos, scripts ou
referencias. Skills especificas do projeto devem ser documentadas antes de serem
criadas. Quando forem criadas, use `.agents/skills/<nome-da-skill>/SKILL.md`.

Uma skill deve ter:

- nome claro;
- descricao com gatilho de uso e limites;
- passos objetivos;
- entradas esperadas;
- saidas esperadas;
- verificacoes obrigatorias.

## MCPs

Use MCP quando o agente precisar de ferramentas ou contexto externo, como GitHub,
issue tracker, design tool, browser controlado, documentacao interna ou observabilidade.

Antes de configurar MCP:

- documente a necessidade;
- defina permissoes minimas;
- registre riscos e dados acessados;
- nunca inclua segredos no repositorio;
- prefira configuracao local ou variaveis de ambiente para credenciais.

## Git

- Autor padrao do projeto: `Paulo Peixoto <paulo_henrique500@hotmail.com>`.
- Commits devem ser pequenos, revisaveis e no repositorio correto.
- Use mensagens claras, por exemplo `docs: add agentops foundation`.
- Antes de finalizar, confirme `git status --short --branch`.

## Code Review Rules

- Validar se a mudanca esta no repositorio correto.
- Validar se documentacao e codigo nao contradizem contratos existentes.
- Validar se dados sensiveis nao foram adicionados.
- Validar se as instrucoes agenticas continuam pequenas, claras e acionaveis.
- Para mudancas em backend ou frontend, exigir verificacao tecnica alem de leitura.
