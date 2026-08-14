# Vuestic Admin Frontend Migration Design

## Objetivo

Migrar o frontend do RIALMA TrackVision para usar o Vuestic Admin como referencia
visual e estrutural do painel administrativo, preservando as regras de negocio,
rotas, permissoes, services, stores e testes ja existentes no projeto.

Esta migracao deve deixar o frontend com aparencia e comportamento de sistema
administrativo profissional, responsivo e consistente, sem importar paginas demo ou
funcionalidades genericas que nao pertencem ao TrackVision.

## Escopo

Esta fase pertence ao repositorio `RIALMA-TrackVision-Frontend` e deve seguir:

- `docs/frontend-vue-guidelines.md`;
- Vue 3 com Composition API e `<script setup>`;
- Vue Router com meta fields para autenticacao e permissao;
- Pinia para sessao e estado compartilhado;
- services isolados para integracao com API;
- componentes pequenos, nomeados e testaveis.

Funcionalidades dentro do escopo:

- instalar e configurar `vuestic-ui`;
- registrar o plugin Vuestic no `src/main.ts`;
- importar os estilos oficiais do Vuestic UI;
- criar configuracao global de tema para identidade TrackVision;
- adaptar o layout administrativo para uma estrutura inspirada no Vuestic Admin;
- substituir os componentes base proprios por wrappers sobre Vuestic UI;
- migrar login, dashboard e paginas administrativas para o novo padrao visual;
- manter controles de permissao existentes;
- manter integracao com backend Laravel existente;
- atualizar testes impactados pela mudanca de markup.

Fora do escopo:

- trocar backend ou contratos de API;
- adicionar paginas demo do Vuestic Admin;
- adicionar analytics, i18n, Storybook, GTM ou graficos sem necessidade de dominio;
- reescrever services que ja funcionam;
- mudar estrategia de autenticacao;
- implementar captura Intelbras, relatorios ou dashboard em tempo real nesta fase.

## Fontes Oficiais

As decisoes desta spec usam como referencia:

- Vuestic Admin: `https://github.com/epicmaxco/vuestic-admin`;
- Vuestic UI installation: `https://ui.vuestic.dev/getting-started/installation/`;
- Vuestic UI: `https://ui.vuestic.dev/`.

O Vuestic Admin oficial e baseado em Vue 3, Vite, Pinia, Tailwind CSS e Vuestic UI.
Para este projeto, a migracao deve usar Vuestic UI diretamente dentro do frontend
existente, evitando substituir o repositorio por um scaffold completo.

## Decisao Arquitetural

Usar uma migracao incremental e preservadora:

1. manter a aplicacao TrackVision atual como fonte de verdade de dominio;
2. adicionar Vuestic UI como camada visual;
3. adaptar layout, base components e paginas para os componentes Vuestic;
4. deixar o projeto preparado para adotar mais padroes do Vuestic Admin em fases
   futuras quando fizer sentido.

Esta abordagem reduz risco porque o frontend atual ja possui autenticacao, guards,
permissoes, services, testes e telas administrativas funcionais.

## Stack Esperada

- Vue 3;
- Vite;
- TypeScript;
- Vue Router;
- Pinia;
- Vuestic UI;
- lucide-vue-next enquanto os icones existentes forem mais claros para o dominio;
- Vitest e Vue Test Utils;
- Playwright para smoke test quando o ambiente local estiver disponivel.

Nao adicionar Tailwind nesta fase. O Vuestic Admin usa Tailwind, mas o frontend atual
nao depende dele. Incluir Tailwind agora aumentaria superficie de migracao sem
beneficio direto para as telas existentes.

## Experiencia Visual

O frontend deve parecer uma ferramenta operacional de monitoramento e administracao,
nao uma landing page.

Direcao visual:

- sidebar administrativa com marca TrackVision, grupos de navegacao e links por
  permissao;
- topbar compacta com usuario logado, estado do sistema e acao de logout;
- conteudo principal com largura fluida, boa densidade e leitura rapida;
- cards Vuestic apenas para paineis e blocos de conteudo reais;
- tabelas Vuestic para cadastros;
- modais Vuestic para criacao e edicao;
- alerts/notificacoes Vuestic para erros e sucesso;
- formularios com labels claros, validacao visivel e botoes consistentes;
- layout responsivo com sidebar adaptada para telas menores;
- tema claro como padrao inicial.

Evitar:

- paginas demo do template;
- herois e textos promocionais;
- cards aninhados sem necessidade;
- fundos decorativos;
- paleta de uma unica cor dominante;
- texto instrucional visivel explicando como a interface funciona.

## Tema TrackVision

A configuracao global do Vuestic deve declarar cores semanticamente:

- primary: azul operacional para acoes principais;
- secondary: neutro escuro para navegacao e estrutura;
- success: verde para operacoes concluidas;
- warning: amarelo/ambar para alerta operacional;
- danger: vermelho/laranja para erro ou acao destrutiva;
- background: cinza claro para areas de trabalho;
- text: tons neutros com contraste adequado.

O tema deve ficar em arquivo proprio, por exemplo:

```text
src/app/vuestic.ts
```

Este arquivo deve exportar uma configuracao usada por `createVuestic`.

## Arquitetura Frontend

Estrutura alvo:

```text
src/
|-- app/
|   |-- config.ts
|   `-- vuestic.ts
|-- components/
|   |-- base/
|   |   |-- BaseAlert.vue
|   |   |-- BaseButton.vue
|   |   |-- BaseInput.vue
|   |   |-- BaseModal.vue
|   |   |-- BaseSelect.vue
|   |   `-- BaseTable.vue
|   `-- navigation/
|       |-- TheSidebar.vue
|       `-- TheTopbar.vue
|-- layouts/
|   `-- AdminLayout.vue
|-- pages/
|-- router/
|-- services/
|-- stores/
|-- styles/
|   `-- main.css
|-- types/
|-- App.vue
`-- main.ts
```

Os wrappers `Base*` continuam existindo para proteger o dominio TrackVision da API
direta do componente visual. Assim, futuras trocas ou ajustes globais ficam
centralizados.

## Componentes Base

### BaseButton

Deve encapsular `VaButton`.

Entradas mantidas:

- `variant: 'primary' | 'secondary' | 'ghost' | 'danger'`;
- `type: 'button' | 'submit' | 'reset'`;
- `disabled`;
- `loading`.

Mapeamento:

- `primary`: `color="primary"`;
- `secondary`: `color="secondary"` ou `preset="secondary"`;
- `ghost`: `preset="plain"`;
- `danger`: `color="danger"`.

### BaseInput

Deve encapsular `VaInput`.

Preservar:

- `v-model`;
- `label`;
- `type`;
- `autocomplete`;
- `name`;
- `error`;
- `disabled`.

### BaseSelect

Deve encapsular `VaSelect`.

Preservar:

- `v-model`;
- `label`;
- `options`;
- `optionLabel`;
- `optionValue`;
- `error`;
- `disabled`.

### BaseAlert

Deve encapsular `VaAlert`.

Mapear variantes:

- `success`;
- `warning`;
- `error`;
- `info`.

### BaseModal

Deve encapsular `VaModal`.

Preservar:

- `open`;
- `title`;
- evento `close`;
- slot de conteudo.

O modal deve manter a responsabilidade de pagina: forms continuam fora do componente.

### BaseTable

Deve encapsular `VaDataTable` quando a API de slots atender as telas existentes.
Se algum slot ficar mais complexo do que o necessario, pode manter uma tabela HTML
com classes Vuestic nesta fase, desde que a aparencia fique integrada ao tema.

Preservar:

- `columns`;
- `rows`;
- `loading`;
- `emptyText`;
- slot `row`.

## Layout Administrativo

`AdminLayout.vue` deve assumir a estrutura principal do painel:

- sidebar;
- topbar;
- area de conteudo;
- responsividade para mobile.

`TheSidebar.vue` deve manter o filtro por permissao usando `authStore.can`.

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

`TheTopbar.vue` deve manter:

- nome do usuario;
- texto curto da area administrativa;
- botao de sair;
- sem expor dados sensiveis.

## Login

`LoginPage.vue` deve migrar para uma tela com componentes Vuestic:

- `VaCard` ou estrutura equivalente para formulario;
- `BaseInput` para email;
- `BaseInput` para senha;
- `BaseButton` para submit;
- `BaseAlert` para erro.

O fluxo de login nao muda:

1. enviar email e senha para `authStore.login`;
2. guardar token e usuario pelo store existente;
3. redirecionar para `redirect` ou `/dashboard`;
4. exibir mensagem para `401`, `422` e erro generico.

## Paginas Administrativas

As paginas existentes devem manter sua regra de dominio e receber apenas migracao de
componentes/markup visual.

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

Cada pagina deve continuar tratando:

- loading;
- erro;
- vazio;
- sucesso quando houver escrita;
- permissao negada pelo guard ou pela API.

## Contratos Que Nao Podem Quebrar

- `VITE_API_BASE_URL` continua sendo a URL base da API;
- login continua em `POST /api/v1/auth/login`;
- logout continua em `POST /api/v1/auth/logout`;
- usuario atual continua vindo de `GET /api/v1/me`;
- rotas existentes mantem os mesmos nomes;
- meta permissions seguem iguais;
- services seguem retornando os tipos atuais;
- testes podem mudar a forma de localizar markup, mas nao o comportamento esperado.

## Testes

Atualizar e manter verdes:

- `src/App.test.ts`;
- `src/router/routerGuard.test.ts`;
- `src/stores/authStore.test.ts`;
- `src/components/navigation/TheSidebar.test.ts`;
- testes de paginas com BaseTable, BaseModal, BaseButton ou BaseInput;
- testes de forms que dependem de inputs/selects.

Adicionar ou ajustar testes para garantir:

- `main.ts` registra Vuestic sem quebrar montagem;
- login renderiza campos e chama fluxo existente;
- sidebar continua filtrando links por permissao;
- `BaseButton` preserva estados `loading` e `disabled`;
- `BaseModal` emite `close`;
- tabela renderiza loading, vazio e linhas.

Verificacoes obrigatorias:

- `npm run lint`;
- `npm run test`;
- `npm run build`;
- smoke visual no navegador local em `http://127.0.0.1:5173`.

## Plano De Rollout

1. instalar `vuestic-ui`;
2. criar configuracao global em `src/app/vuestic.ts`;
3. registrar Vuestic em `src/main.ts`;
4. substituir wrappers `Base*`;
5. migrar `AdminLayout`, `TheSidebar` e `TheTopbar`;
6. migrar login e dashboard;
7. migrar paginas administrativas por dominio;
8. atualizar CSS global para conviver com Vuestic;
9. atualizar testes;
10. rodar lint, testes, build e smoke no navegador.

## Criterios De Aceite

- frontend abre localmente sem erro fatal;
- tela de login usa padrao Vuestic;
- usuario admin consegue logar;
- dashboard abre apos login;
- sidebar mostra apenas links permitidos;
- paginas administrativas existentes continuam acessiveis;
- formularios continuam enviando os mesmos payloads;
- modais, alerts, tabelas e botoes seguem um padrao visual consistente;
- `npm run lint`, `npm run test` e `npm run build` passam;
- nenhum segredo novo e exposto no frontend;
- nao foram importadas paginas demo do Vuestic Admin.

## Decisoes

- A migracao usa Vuestic UI diretamente em vez de substituir o projeto pelo scaffold
  completo do Vuestic Admin.
- Wrappers `Base*` permanecem para manter baixo acoplamento com a biblioteca visual.
- Tailwind fica fora desta fase para reduzir risco e tamanho da mudanca.
- Rotas, permissoes, services e stores existentes sao preservados.
- A migracao e visual/estrutural, nao uma reescrita do dominio.
