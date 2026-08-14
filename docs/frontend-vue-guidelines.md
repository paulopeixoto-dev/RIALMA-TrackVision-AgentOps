# Padroes do Frontend Vue

Este guia define o padrao obrigatorio para demandas de frontend do RIALMA TrackVision.
Quando uma tarefa envolver telas, componentes, rotas, estado, consumo de API,
responsividade ou experiencia do usuario, o agente deve seguir estas regras antes de
propor ou implementar qualquer mudanca.

## Principio central

O frontend deve ser previsivel, acessivel, performatico e facil de evoluir. Para isso:

- Vue 3 e o padrao do projeto.
- Use Composition API e Single-File Components.
- Prefira `<script setup>` quando o projeto estiver com Vue 3 e Vite.
- Componentes devem ser pequenos, nomeados com clareza e ter uma responsabilidade.
- Logica reutilizavel deve ir para composables, services ou stores, nao para templates.
- Estado compartilhado deve usar Pinia quando estado local ou composables nao bastarem.
- Integracao com API deve ficar isolada em clients/services.
- Toda tela deve considerar estados de loading, erro, vazio e sucesso.
- Mudancas relevantes devem incluir testes e documentacao de contrato visual/API.
- O template visual padrao e Vuestic Admin com Vuestic UI. Componentes `Va*`,
  layouts, tokens e padroes do template devem ser a primeira escolha em telas,
  formularios, navegacao, modais, tabelas e acoes.

## Stack esperada

A stack deve respeitar o que existir no repositorio frontend. Se o frontend ainda nao
tiver sido iniciado, use como referencia:

- Vue 3;
- Vite;
- TypeScript quando a base do projeto permitir;
- Vue Router para rotas;
- Pinia para estado compartilhado;
- Vitest e Vue Test Utils para testes unitarios e de componentes;
- Playwright ou Cypress para fluxos criticos end-to-end quando necessario.
- Vuestic Admin como referencia de layout administrativo e Vuestic UI como biblioteca
  primaria de componentes visuais.

Nao adicionar bibliotecas grandes sem avaliar impacto no bundle, manutencao e real
necessidade.

Nao criar framework visual paralelo ao Vuestic. Se o Vuestic entregar o componente ou
padrao de interacao, use-o primeiro.

## Estrutura sugerida

```text
src/
|-- app/
|-- assets/
|-- components/
|-- composables/
|-- layouts/
|-- pages/
|-- router/
|-- services/
|-- stores/
|-- styles/
|-- types/
`-- utils/
```

A estrutura final deve seguir o padrao real do frontend quando ele existir. Se a
estrutura mudar, atualize este guia.

## Componentes

Componentes devem ser pequenos, previsiveis e reutilizaveis quando fizer sentido.

Padroes esperados:

- nomes de componentes sempre com duas ou mais palavras, exceto `App`;
- nomes em PascalCase em imports e arquivos `.vue`, como `VehicleStatusCard.vue`;
- nomes completos em vez de abreviacoes obscuras;
- props declaradas explicitamente, com tipos e obrigatoriedade quando aplicavel;
- emits declarados explicitamente;
- eventos customizados emitidos e escutados em kebab-case, como `close-window`;
- props declaradas em camelCase no script e usadas em kebab-case no template, mantendo
  consistencia no projeto;
- templates com `v-for` sempre usando `:key` unica e estavel;
- nunca usar `v-if` e `v-for` no mesmo elemento;
- expressoes no template devem ser simples; logica de formatacao, filtro ou calculo
  deve ir para computed, composable ou metodo nomeado;
- estilos escopados ou organizados pelo padrao global do projeto;
- componentes de tela devem orquestrar, componentes reutilizaveis devem focar UI.
- componentes de tela, layout, formulario e navegacao devem usar `VaButton`,
  `VaInput`, `VaSelect`, `VaCard`, `VaModal`, `VaAlert`, `VaDataTable`, `VaLayout`
  e demais componentes Vuestic diretamente quando houver equivalente.
- componentes de dominio podem envolver componentes Vuestic para acoplar regra do
  TrackVision, mas nao devem esconder um componente Vuestic simples sem ganho claro.

Um componente nao deve:

- concentrar regra de negocio complexa;
- chamar multiplos endpoints diretamente sem service;
- acessar estado global sem necessidade;
- misturar fetch, transformacao, validacao visual e layout em um unico arquivo grande;
- receber dados indefinidos sem tratar loading, erro ou vazio.
- recriar visualmente componentes que o Vuestic ja fornece.
- usar HTML cru para botoes, campos, selects, modais, alerts ou tabelas quando houver
  componente Vuestic adequado.

## Padrao Vuestic Admin

Vuestic Admin e o template de referencia do frontend TrackVision. Toda demanda de
tela deve consultar primeiro os padroes ja aplicados no projeto e, quando necessario,
as referencias oficiais do Vuestic.

Regras obrigatorias:

- priorizar componentes `Va*` nas telas e formularios;
- priorizar `VaLayout`, `VaNavbar`, `VaSidebar`/navegacao equivalente, `VaCard`,
  `VaForm`, `VaInput`, `VaSelect`, `VaButton`, `VaModal`, `VaAlert` e `VaDataTable`
  antes de criar alternativa propria;
- preservar classes locais somente para densidade, espacamento, responsividade e
  necessidades especificas do dominio;
- evitar wrappers genericos como substitutos de componentes Vuestic simples;
- documentar qualquer excecao quando um componente proprio for inevitavel;
- manter testes de contrato para impedir regressao para componentes visuais proprios.

Wrappers ou componentes `Base*` existentes so devem permanecer quando forem
composicoes temporarias, compatibilidade legado ou quando adicionarem comportamento
real que o Vuestic nao entrega diretamente. Codigo novo ou tela alterada deve migrar
para Vuestic direto sempre que isso for seguro.

## Convencoes de nomes de componentes

Use nomes para revelar o papel do componente na aplicacao.

Padroes esperados:

- componentes reutilizaveis e genericos de UI devem usar prefixo `Base`, como
  `BaseButton.vue`, `BaseModal.vue` ou `BaseInput.vue`;
- componentes usados uma unica vez por pagina ou layout devem usar prefixo `The`,
  como `TheHeader.vue`, `TheSidebar.vue` ou `TheFooter.vue`;
- componentes acoplados a um dominio devem carregar o dominio no nome, como
  `VehicleMapPanel.vue` ou `CameraStatusBadge.vue`;
- evite nomes vagos como `Card.vue`, `List.vue`, `Modal.vue` ou `Item.vue`;
- escolha PascalCase para arquivos e imports e mantenha essa convencao em todo o repo.

Nao registre componentes globais automaticamente sem necessidade. Se houver registro
global para componentes base, documente o padrao e mantenha apenas componentes
realmente genericos nessa categoria.

## Composition API e composables

Use composables para logica reutilizavel de UI, estado local reativo, browser APIs,
timers, observers, filtros, formatacao com estado ou padroes de interacao.

Padroes esperados:

- nomear composables com prefixo `use`, como `useVehicleFilters`;
- manter composables em `src/composables`;
- retornar estado e funcoes com nomes claros;
- aceitar refs, getters ou valores simples quando isso aumentar reutilizacao;
- limpar listeners, timers e subscriptions no ciclo de vida apropriado;
- testar composables com Vitest quando tiverem regra propria.

Nao use composable como deposito generico de qualquer codigo. Se a logica fala com API,
prefira service/client. Se representa estado compartilhado, prefira Pinia.

## Options API

O padrao preferido e Composition API com `<script setup>`, mas se Options API aparecer
em codigo legado ou em uma decisao especifica, siga estes cuidados:

- `data` deve sempre ser uma funcao que retorna um novo objeto;
- nao chame a mesma rotina em `created` e em `watch` para simular inicializacao;
- quando um watcher precisar rodar na criacao, use `immediate: true`;
- mantenha computed para valores derivados e methods para acoes;
- preserve a ordem recomendada das options descrita neste guia.

## Estado com Pinia

Use Pinia quando o estado precisar ser compartilhado entre paginas, sobreviver a troca
de componentes, ou representar sessao, permissoes, filtros globais, preferencias ou
dados de dominio reutilizados.

Padroes esperados:

- stores em `src/stores`;
- state para dados;
- getters para valores derivados;
- actions para mudancas e operacoes assincronas;
- evitar stores que conhecem detalhes visuais demais;
- evitar ciclos entre stores;
- testar stores quando houver regra, transformacao ou fluxo assincrono relevante.

Estado local simples deve ficar no componente.

## Rotas e navegacao

Use Vue Router para navegacao de aplicacao.

Padroes esperados:

- rotas organizadas por dominio ou area;
- lazy loading para componentes de rota;
- guards para autenticacao e permissoes recorrentes;
- meta fields para titulo, permissao ou layout quando isso simplificar a composicao;
- tratamento claro de pagina nao encontrada e acesso negado;
- manter nomes de rotas estaveis quando usados em links programaticos.

Nao coloque regra de negocio pesada em guards. Guards devem decidir navegacao, nao
executar fluxo de dominio.

## Integracao com API

Chamadas HTTP devem ficar fora de componentes sempre que houver regra, reutilizacao ou
tratamento padronizado.

Padroes esperados:

- clients/services em `src/services`;
- base URL via `import.meta.env`;
- tipos ou interfaces para payloads e respostas quando usar TypeScript;
- tratamento central de erro, timeout e autenticacao quando possivel;
- normalizacao de resposta perto do service, nao espalhada em componentes;
- components recebem dados prontos para renderizar;
- contratos de API devem alinhar com o backend Laravel e seus Resources.

Cada tela que consome API deve considerar:

- loading;
- erro recuperavel;
- vazio;
- sucesso;
- permissao negada quando aplicavel.

## Configuracao e ambiente

No frontend Vite, variaveis expostas ao browser usam `import.meta.env`.

Padroes esperados:

- somente variaveis com prefixo `VITE_` entram no bundle do frontend;
- nunca colocar segredos, tokens privados, senhas ou chaves sensiveis em `VITE_*`;
- converter tipos vindos de env, pois valores chegam como string;
- documentar variaveis em `.env.example` sem valores reais;
- usar um modulo de configuracao do frontend quando muitas variaveis forem usadas.

Segredos pertencem ao backend, nao ao frontend.

## Performance

Performance deve ser pensada desde o design da tela.

Padroes esperados:

- lazy loading de rotas;
- evitar dependencias pesadas sem medicao;
- preferir imports tree-shakable;
- manter props estaveis em listas grandes;
- virtualizar listas grandes quando houver muitos itens;
- evitar computed que cria objetos novos sem necessidade;
- usar `v-once` ou `v-memo` somente quando houver ganho claro e comentario breve;
- evitar reatividade profunda em estruturas grandes e imutaveis quando isso pesar.

Para dashboards, mapas, tabelas grandes ou telas em tempo real, defina estrategia de
paginacao, filtros, debounce, polling, cache local ou streaming antes de implementar.

## Seguranca

Nunca usar conteudo nao confiavel como template Vue.

Padroes esperados:

- evitar `v-html`;
- se `v-html` for inevitavel, exigir conteudo sanitizado e justificar a decisao;
- URLs fornecidas por usuario devem ser sanitizadas no backend antes de persistir;
- nao expor segredos no bundle;
- nao confiar em permissao apenas no frontend;
- tratar XSS, open redirect e dados sensiveis em conjunto com o backend.

O frontend pode esconder ou desabilitar acoes, mas a autoridade final de permissao
fica no backend.

## Regras recomendadas do Vue Style Guide

O projeto deve seguir as regras recomendadas oficiais do Vue para consistencia.

Use shorthands de diretivas de forma consistente:

- `:` para `v-bind`;
- `@` para `v-on`;
- `#` para `v-slot`.

O padrao preferido do projeto e usar shorthands em templates Vue. Se o time escolher
nao usar shorthands, registre a decisao e aplique em todo o repositorio.

Em componentes Options API, quando usados, mantenha a ordem recomendada:

1. `name`
2. `compilerOptions`
3. `components` e `directives`
4. `extends`, `mixins`, `provide` e `inject`
5. `inheritAttrs`, `props`, `emits` e `expose`
6. `setup`
7. `data` e `computed`
8. `watch` e lifecycle hooks
9. `methods`
10. `template` ou `render`

Em templates, ordene atributos de forma consistente:

1. `is`
2. `v-for`
3. `v-if`, `v-else-if`, `v-else`, `v-show`, `v-cloak`
4. `v-pre`, `v-once`
5. `id`
6. `ref`, `key`
7. `v-model`
8. outros atributos e bindings
9. eventos
10. `v-html` e `v-text`

Use linhas em branco entre blocos multi-line quando isso melhorar leitura. Em Single
File Components, mantenha ordem consistente dos blocos. O padrao preferido do projeto e:

```vue
<script setup>
</script>

<template>
</template>

<style scoped>
</style>
```

Se o projeto adotar outro padrao consistente, registre a decisao neste guia.

## Testes

Demandas de frontend devem considerar:

- testes unitarios para utils, services e composables com regra propria;
- testes de componente para renderizacao, props, emits e interacoes;
- testes de store quando houver estado compartilhado com regra;
- testes E2E para fluxos criticos, login, permissoes, dashboards e integracoes;
- mocks/fakes para API em testes unitarios e de componente;
- fixtures pequenas e legiveis.

Use Vitest para unidades e componentes headless. Use Vue Test Utils para montar
componentes. Use Playwright ou Cypress quando o comportamento depender do browser real
ou de fluxo ponta a ponta.

## Checklist para agentes

Antes de finalizar uma demanda de frontend Vue, confirme:

- [ ] Componentes tem responsabilidade clara.
- [ ] Telas, layouts, formularios e navegacao priorizam componentes `Va*` do Vuestic.
- [ ] Nao foi criado wrapper generico ou HTML cru quando havia componente Vuestic
      equivalente.
- [ ] Componentes single-instance usam prefixo `The`.
- [ ] Props e emits estao explicitamente definidos.
- [ ] Eventos customizados usam kebab-case.
- [ ] Props usam camelCase no script e kebab-case no template.
- [ ] `v-for` usa `:key` unica e estavel.
- [ ] Nao ha `v-if` e `v-for` no mesmo elemento.
- [ ] Templates nao contem expressoes complexas que deveriam ser computed/composable.
- [ ] Shorthands de diretivas sao usados de forma consistente.
- [ ] Logica reutilizavel saiu de componentes para composables, services ou stores.
- [ ] Chamadas de API ficam em services/clients.
- [ ] Telas tratam loading, erro, vazio e sucesso.
- [ ] Rotas protegidas usam guards ou meta fields quando aplicavel.
- [ ] Variaveis de ambiente nao expoem segredos.
- [ ] Performance foi considerada para rotas, dependencias e listas grandes.
- [ ] `v-html` foi evitado ou justificado com sanitizacao.
- [ ] Regras recomendadas do Vue Style Guide foram seguidas.
- [ ] Testes cobrem regras, componentes ou fluxos alterados.
- [ ] Testes de contrato do template Vuestic seguem passando quando aplicavel.
- [ ] `git status --short --branch` foi conferido no repositorio correto.

## Referencias oficiais

- Vue Style Guide - Essential: https://vuejs.org/style-guide/rules-essential
- Vue Style Guide - Strongly Recommended:
  https://vuejs.org/style-guide/rules-strongly-recommended.html
- Vue Style Guide - Recommended:
  https://vuejs.org/style-guide/rules-recommended.html
- Vue Composables: https://vuejs.org/guide/reusability/composables.html
- Vue State Management: https://vuejs.org/guide/scaling-up/state-management
- Vue Testing: https://vuejs.org/guide/scaling-up/testing.html
- Vue Performance: https://vuejs.org/guide/best-practices/performance
- Vue Security: https://vuejs.org/guide/best-practices/security.html
- Vite Env Variables: https://vite.dev/guide/env-and-mode
- Vuestic Admin: https://github.com/epicmaxco/vuestic-admin
- Vuestic UI: https://ui.vuestic.dev/

## Referencias complementares

- ButterCMS, "19 Laravel best practices for developers in 2026":
  https://buttercms.com/blog/laravel-best-practices/
- LearnVue, "12 VueJS Best Practices for Pro Developers":
  https://learnvue.co/articles/vue-best-practices
