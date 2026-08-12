# Frontend Admin Shell Design

## Objetivo

Construir o primeiro frontend operacional do RIALMA TrackVision com Vue 3, Vite e
TypeScript. Esta fase entrega login, sessao autenticada, layout administrativo,
navegacao por permissao e telas administrativas conectadas aos endpoints ja
disponiveis no backend.

## Escopo

Esta fase pertence ao repositorio `RIALMA-TrackVision-Frontend` e deve seguir o guia
`docs/frontend-vue-guidelines.md`.

Funcionalidades da fase:

- login administrativo via `POST /api/v1/auth/login`;
- logout via `POST /api/v1/auth/logout`;
- recuperacao de usuario atual via `GET /api/v1/me`;
- layout administrativo com sidebar, topbar e area de conteudo;
- navegacao protegida por autenticacao e permissao;
- listagem de usuarios, roles e permissoes;
- CRUD de veiculos;
- CRUD de locais;
- CRUD de edge nodes;
- CRUD de cameras;
- CRUD de pares de cameras.

Fora do escopo:

- captura Intelbras;
- dashboard em tempo real;
- imagens de LPR ou camera de apoio;
- revisao de carga;
- viagens;
- relatorios;
- criacao/edicao de usuarios, roles e permissoes, porque o backend atual expoe
  apenas endpoints de listagem para estes recursos.

## Stack

- Vue 3
- Vite
- TypeScript
- Vue Router
- Pinia
- Vitest
- Vue Test Utils
- Playwright para smoke test do fluxo de login quando a aplicacao estiver rodando

Nao adicionar bibliotecas grandes de UI nesta fase. A interface deve ser criada com
CSS proprio e componentes base pequenos. Icones podem usar `lucide-vue-next` por ser
leve e consistente com a orientacao de UI do projeto.

## Experiencia Visual

O frontend deve parecer uma ferramenta operacional de escritorio/portaria, nao uma
landing page. O primeiro viewport deve ser uma tela de login simples e direta. Depois
do login, a aplicacao usa um layout administrativo denso, escaneavel e previsivel.

Direcao visual:

- sidebar fixa em desktop e recolhivel em telas menores;
- topbar compacta com nome do usuario e acao de logout;
- area principal com titulo da pagina, acoes primarias e tabela;
- modais ou paineis laterais para criar e editar registros;
- estados claros de loading, erro, vazio e sucesso;
- cores contidas, sem paleta de uma unica cor dominante;
- sem hero, marketing copy ou cards decorativos aninhados;
- formularios com labels claros e feedback de validacao.

## Arquitetura Frontend

Estrutura esperada:

```text
src/
|-- app/
|   `-- config.ts
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
|   |-- LoginPage.vue
|   |-- DashboardPage.vue
|   |-- UsersPage.vue
|   |-- RolesPage.vue
|   |-- PermissionsPage.vue
|   |-- VehiclesPage.vue
|   |-- LocationsPage.vue
|   |-- EdgeNodesPage.vue
|   |-- CamerasPage.vue
|   |-- CameraPairsPage.vue
|   |-- ForbiddenPage.vue
|   `-- NotFoundPage.vue
|-- router/
|   `-- index.ts
|-- services/
|   |-- apiClient.ts
|   |-- authService.ts
|   |-- cameraPairsService.ts
|   |-- camerasService.ts
|   |-- edgeNodesService.ts
|   |-- locationsService.ts
|   |-- permissionsService.ts
|   |-- rolesService.ts
|   |-- usersService.ts
|   `-- vehiclesService.ts
|-- stores/
|   `-- authStore.ts
|-- styles/
|   `-- main.css
|-- types/
|   |-- api.ts
|   |-- auth.ts
|   `-- admin.ts
|-- utils/
|   `-- formatters.ts
|-- App.vue
`-- main.ts
```

## Configuracao

Variaveis:

- `VITE_API_BASE_URL`: URL base do backend, exemplo local `http://localhost:8000/api/v1`.

Nenhum segredo deve existir no frontend. Token de acesso e salvo em `localStorage`
para persistir sessao inicial. Em fase futura, se houver politica mais rigida de
seguranca, o armazenamento pode migrar para cookie httpOnly controlado pelo backend.

## Contratos De API

Autenticacao:

- `POST /auth/login`
- `POST /auth/logout`
- `GET /me`

Admin:

- `GET /admin/users`
- `GET /admin/roles`
- `GET /admin/permissions`
- `GET|POST|PATCH|DELETE /admin/vehicles`
- `GET|POST|PATCH|DELETE /admin/locations`
- `GET|POST|PATCH|DELETE /admin/edge-nodes`
- `GET|POST|PATCH|DELETE /admin/cameras`
- `GET|POST|PATCH|DELETE /admin/camera-pairs`

O client deve entender o formato de Resource do Laravel:

- colecoes paginadas em `data` com `links` e `meta`;
- itens em `data`;
- erros de validacao em `errors`;
- erro de autenticacao `401`;
- erro de autorizacao `403`.

## Permissoes

O frontend nao e a autoridade de permissao; o backend continua sendo decisivo.
Mesmo assim, a navegacao deve esconder links indisponiveis para reduzir ruido.

Mapeamento:

- `users.manage`: Usuarios;
- `permissions.manage`: Roles e Permissoes;
- `vehicles.manage`: Veiculos;
- `cameras.manage`: Locais, Edge Nodes, Cameras e Pares de Cameras.

Como o endpoint de login atual retorna `user` sem roles carregadas, a fase deve chamar
`GET /me` apos login e usar roles quando existirem. Para permissao de navegacao, a
primeira versao pode carregar roles/permissoes disponiveis apenas para usuario com
acesso permitido e tratar `403` como bloqueio visual. Se o backend passar a retornar
permissoes efetivas do usuario, o store deve usar esse campo diretamente.

## Telas

### Login

Campos:

- email;
- senha.

Estados:

- enviando;
- credenciais invalidas;
- erro inesperado;
- redirecionamento para dashboard apos sucesso.

### Dashboard

Tela inicial simples com atalhos para os modulos permitidos e resumo estatico dos
cadastros carregados quando houver acesso. Nao deve prometer metricas em tempo real.

### Usuarios, Roles E Permissoes

Listagens somente leitura nesta fase:

- usuarios: nome, email e roles quando o backend enviar;
- roles: nome e permissoes;
- permissoes: nome.

### Veiculos

CRUD completo:

- placa;
- codigo de frota;
- descricao;
- ativo/inativo.

Exibir `plate_normalized` como campo gerado pelo backend.

### Locais

CRUD completo:

- nome;
- descricao;
- ativo/inativo.

### Edge Nodes

CRUD completo:

- local;
- nome;
- descricao;
- ativo/inativo.

Exibir `status` e `last_seen_at` como leitura.

### Cameras

CRUD completo:

- local;
- edge node;
- nome;
- tipo `lpr` ou `support`;
- fabricante `intelbras`;
- host;
- porta;
- canal;
- usuario;
- senha.

Senha e campo write-only. A tela nunca deve tentar exibir senha atual.

### Pares De Cameras

CRUD completo:

- local;
- edge node;
- camera LPR;
- camera de apoio;
- direcao `outbound`, `inbound` ou `unknown`;
- ativo/inativo.

A UI deve filtrar cameras pelo local/edge node selecionado quando houver dados
carregados. O backend ainda valida a regra final.

## Tratamento De Erros

O `apiClient` deve normalizar erros em um formato unico:

- `status`;
- `message`;
- `errors` para campos;
- `isUnauthorized`;
- `isForbidden`.

Comportamento:

- `401`: limpar sessao e voltar para login;
- `403`: mostrar tela ou alerta de acesso negado;
- `422`: mostrar erros de campo no formulario;
- erro de rede: mostrar alerta recuperavel.

## Testes

Testes obrigatorios desta fase:

- `authService.login` envia credenciais e normaliza resposta;
- `authStore.login` salva token e usuario;
- router guard redireciona usuario nao autenticado para login;
- router guard bloqueia rota sem permissao conhecida;
- `VehiclesPage` renderiza tabela com dados paginados;
- `CameraForm` nao exibe senha existente e envia senha somente quando preenchida.

Verificacoes obrigatorias:

- `npm run lint`
- `npm run test`
- `npm run build`

Se Playwright for configurado nesta fase, incluir smoke test de login com API mockada
ou ambiente local documentado.

## Decisoes

- A fase usa componentes base proprios em vez de framework visual grande.
- CRUDs administrativos podem compartilhar padroes de tabela/form/modal, mas a primeira
  implementacao deve evitar abstracao generica demais.
- Usuario, roles e permissoes ficam somente leitura ate o backend expor escrita.
- A UI deve tratar `403` como fluxo esperado, porque permissoes reais sao garantidas no
  backend.
- O frontend inicia sem dados mockados em runtime; mocks entram apenas em testes.
