# Vuestic Admin Template Shell Design

## Objetivo

Evoluir a migracao Vuestic ja aplicada no frontend para que a tela de login e
todo o painel administrativo usem um padrao visual mais proximo do Vuestic Admin,
preservando o dominio TrackVision, rotas, permissoes, services, stores e contratos
com o backend.

Esta etapa nao substitui o projeto pelo scaffold oficial. Ela adapta o shell do
TrackVision para consumir os padroes de layout, autenticacao, navegacao e
superficies administrativas do Vuestic Admin dentro da arquitetura existente.

## Contexto

O frontend ja possui:

- Vue 3, Vite, TypeScript, Pinia e Vue Router;
- `vuestic-ui` instalado e registrado;
- tema global em `src/app/vuestic.ts`;
- alguns componentes `Base*` legados ainda existentes no codigo;
- paginas administrativas funcionais;
- login integrado ao backend Laravel;
- controle de permissoes por rota e sidebar;
- testes Vitest para stores, services, componentes e paginas.

A lacuna atual e visual/estrutural: a aplicacao ja usa componentes Vuestic, mas
o login, a sidebar, a topbar e a composicao das paginas ainda parecem um shell
customizado simples, nao um template administrativo completo.

## Fontes De Referencia

As decisoes usam como referencia:

- Vuestic Admin oficial: `https://github.com/epicmaxco/vuestic-admin`;
- Vuestic Admin demo/site: `https://admin.vuestic.dev/`;
- Vuestic UI docs: `https://ui.vuestic.dev/`.

O Vuestic Admin oficial usa uma separacao clara entre layout autenticado
(`AppLayout`) e layout de autenticacao (`AuthLayout`), alem de `VaLayout`,
sidebar responsiva, navbar e formularios Vuestic. O TrackVision deve absorver
essa direcao sem importar paginas demo nem funcionalidades genericas.

## Decisao Recomendada

Usar a abordagem incremental:

1. manter o frontend TrackVision como fonte de verdade;
2. criar/adaptar um layout de autenticacao inspirado em `AuthLayout`;
3. adaptar `AdminLayout` para usar `VaLayout` como estrutura principal;
4. adaptar `TheSidebar` para uma navegacao Vuestic com estado responsivo;
5. adaptar `TheTopbar` para uma navbar operacional Vuestic;
6. revisar todas as paginas para usar cabecalho, cards, tabelas, formularios,
   alerts e modais de forma consistente;
7. priorizar componentes `Va*` diretamente nas telas quando houver equivalente no
   Vuestic UI;
8. manter o CSS global apenas para tokens, densidade, responsividade e pequenos
   ajustes de dominio.

Esta decisao evita uma reescrita ampla e reduz risco para autenticacao,
permissoes e CRUDs existentes.

## Escopo

Dentro do escopo:

- tela de login com composicao de template administrativo;
- shell autenticado com `VaLayout`;
- sidebar responsiva/recolhivel;
- topbar com identidade operacional, usuario logado e logout;
- padronizacao de headers, cards, tabelas e areas vazias das paginas;
- estados visuais de loading, erro, vazio e sucesso;
- ajustes em testes impactados por markup e layout;
- smoke local no navegador em `http://127.0.0.1:5173`.

Fora do escopo:

- trocar contratos da API;
- trocar autenticacao;
- adicionar funcionalidades demo do Vuestic Admin;
- implementar graficos novos sem dados reais;
- adicionar Tailwind ao projeto;
- mudar regras de permissao;
- refatorar services que nao sejam impactados pela UI.

## Login

`LoginPage.vue` deve passar a representar uma tela de autenticacao de template,
com:

- layout dividido em desktop, com painel de marca/contexto e formulario;
- layout compacto em mobile;
- `VaForm` como raiz do formulario;
- `VaInput` para email e senha;
- controle para mostrar/ocultar senha se couber sem complexidade;
- `VaAlert` para erros de credencial, validacao e falha generica;
- `VaButton` com largura total e estado de carregamento;
- textos curtos e operacionais, sem conteudo promocional;
- preservacao do fluxo atual de `authStore.login` e redirect.

O login deve continuar aceitando:

- `email`;
- `password`;
- query `redirect`;
- retorno para `/dashboard` quando nao houver redirect.

## Layout Autenticado

`AdminLayout.vue` deve usar `VaLayout` para organizar:

- slot superior com topbar;
- slot lateral com sidebar;
- conteudo principal;
- overlay/recolhimento em telas menores;
- colapso automatico apos navegacao em mobile.

O layout deve ser denso e operacional, adequado para uso repetido em portaria,
controle de viagens e administracao.

## Sidebar

`TheSidebar.vue` deve manter `authStore.can` como regra de exibicao dos links.

Esperado:

- marca RIALMA TrackVision no topo;
- grupo principal de navegacao;
- icones existentes do `lucide-vue-next` enquanto fizerem sentido no dominio;
- item ativo claro;
- botao de recolher/expandir em desktop;
- fechamento automatico em mobile apos clique;
- links preservando os nomes de rotas atuais.

Links esperados:

- Dashboard;
- Usuarios;
- Roles;
- Permissoes;
- Veiculos;
- Viagens;
- Locais;
- Edge Nodes;
- Cameras;
- Pares de Cameras.

## Topbar

`TheTopbar.vue` deve funcionar como uma navbar administrativa:

- titulo curto da area atual ou contexto "Painel administrativo";
- indicador textual de ambiente operacional;
- usuario logado;
- botao de logout;
- acao de abrir/fechar menu em telas menores;
- sem expor token, permissoes detalhadas ou dados sensiveis.

## Paginas Administrativas

Todas as paginas existentes devem seguir o mesmo padrao:

- `page-section` como container principal;
- `page-header` com eyebrow, titulo e acoes;
- `VaCard` para bloco de tabela, formulario ou resumo;
- componentes Vuestic diretos para botoes, campos, selects e demais controles
  nativos do template;
- componentes de composicao existentes apenas quando adicionarem comportamento de
  dominio ainda nao coberto por um componente Vuestic direto;
- textos vazios objetivos;
- botoes alinhados e com variantes consistentes.

Paginas impactadas:

- `DashboardPage.vue`;
- `UsersPage.vue`;
- `RolesPage.vue`;
- `PermissionsPage.vue`;
- `VehiclesPage.vue`;
- `TripsPage.vue`;
- `LocationsPage.vue`;
- `EdgeNodesPage.vue`;
- `CamerasPage.vue`;
- `CameraPairsPage.vue`;
- `ForbiddenPage.vue`;
- `NotFoundPage.vue`.

## CSS E Tema

`src/styles/main.css` deve ficar responsavel por:

- tokens de layout do TrackVision;
- largura e padding do conteudo;
- responsividade do shell;
- pequenos ajustes sobre Vuestic quando necessario;
- classes compartilhadas de pagina, tabela e formulario.

Evitar:

- recriar componentes que o Vuestic ja entrega;
- cards dentro de cards;
- fundos decorativos sem funcao operacional;
- paleta de uma cor so;
- textos instrucionais sobre como usar a interface.

## Testes

Seguir TDD para alteracoes de comportamento/markup relevante.

Testes esperados:

- login renderiza uma tela de autenticacao com estrutura Vuestic e campos;
- login continua chamando `authStore.login` e redirecionando;
- sidebar filtra links por permissao;
- sidebar/topbar expoem controles esperados;
- layout autenticado monta com `VaLayout`;
- paginas principais continuam renderizando titulos, tabelas e acoes;
- testes existentes seguem verdes apos ajustes.

Verificacoes obrigatorias:

- `npm run lint`;
- `npm run test -- --reporter=dot`;
- `npm run build`;
- smoke HTTP da tela de login e dashboard;
- abrir painel local no navegador integrado quando possivel.

## Criterios De Aceite

- login parece parte do template administrativo Vuestic;
- shell autenticado usa estrutura Vuestic de layout;
- todas as telas seguem padrao visual consistente;
- regras de permissao continuam funcionando;
- CRUDs existentes continuam usando os mesmos payloads;
- usuario admin local consegue logar;
- dashboard e paginas administrativas abrem apos login;
- frontend roda em `http://127.0.0.1:5173`;
- backend continua em `http://127.0.0.1:8000`;
- lint, testes e build passam.

## Riscos E Mitigacoes

- Risco: mudar markup e quebrar testes existentes.
  Mitigacao: ajustar testes para comportamento visivel, nao detalhes frageis.

- Risco: acoplar demais o dominio ao Vuestic.
  Mitigacao: manter services/stores como fonte de regra e permitir componentes de
  dominio somente quando composicao Vuestic direta nao bastar.

- Risco: layout mobile ficar dificil de operar.
  Mitigacao: usar `VaLayout` com overlay/recolhimento e validar viewport menor.

- Risco: aumento de bundle.
  Mitigacao: aceitar nesta fase e avaliar code splitting/tree-shaking em etapa
  posterior.

## Decisoes

- A implementacao deve adaptar o template ao TrackVision, nao importar demos.
- O template Vuestic Admin/Vuestic UI e a prioridade visual do frontend.
- Telas alteradas devem preferir componentes `Va*` diretamente antes de wrappers
  genericos ou HTML cru.
- Componentes `Base*` nao sao o padrao atual para telas; se permanecerem no codigo,
  devem ser tratados como legado ate migracao segura.
- `VaLayout` sera usado no shell autenticado.
- Login tera experiencia propria inspirada em `AuthLayout`.
- Tailwind continua fora desta fase.
- A branch principal do frontend e `main`, mesmo quando o usuario chamar de master.
