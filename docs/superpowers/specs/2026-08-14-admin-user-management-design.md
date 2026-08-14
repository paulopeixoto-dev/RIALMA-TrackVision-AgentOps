# Admin User Management Design

## Objetivo

Completar o controle administrativo de usuarios do RIALMA TrackVision, permitindo que
administradores criem, editem, ativem/desativem e redefinam usuarios, alem de
atribuir roles existentes a cada usuario.

Esta fase transforma a area de seguranca de uma listagem operacional em um fluxo de
gestao real, sem abrir ainda edicao dinamica de roles e permissoes.

## Escopo

Esta fase toca dois repositorios:

- `RIALMA-TrackVision-Backend`: API Laravel para escrita de usuarios e roles do
  usuario.
- `RIALMA-TrackVision-Frontend`: telas Vue para criar, editar, ativar/desativar,
  resetar senha e atribuir roles.

O AgentOps registra apenas esta especificacao e o plano de implementacao.

Funcionalidades desta fase:

- listar usuarios com roles;
- criar usuario com nome, email, senha inicial, confirmacao de senha, status e roles;
- editar nome, email, status e roles;
- redefinir senha de usuario;
- impedir que um admin remova seu proprio acesso administrativo de forma acidental;
- manter roles e permissoes como catalogo somente leitura;
- exibir feedback claro de loading, erro, validacao, vazio e sucesso no frontend.

Fora do escopo:

- criar, renomear ou excluir roles pela tela;
- criar ou excluir permissoes pela tela;
- politicas de convite por email;
- recuperacao de senha publica;
- MFA;
- auditoria historica detalhada de alteracoes de usuario;
- tela de perfil do usuario logado.

## Decisao Principal

A fase deve implementar **CRUD de usuarios + atribuicao de roles existentes**.
Roles e permissoes continuam controladas por seed/catalogo do backend.

Motivo:

- roles/permissoes impactam seguranca global do sistema;
- o catalogo atual ja representa o modelo operacional do TrackVision;
- permitir editar roles dinamicamente antes de uma politica de auditoria e revisao
  aumenta risco de bloqueio administrativo ou permissao excessiva;
- usuarios precisam de gestao diaria mais cedo que roles customizadas.

## Regras De Seguranca

Permissao exigida:

- toda escrita de usuario exige `users.manage`;
- listagem de usuarios continua exigindo `users.manage`;
- listagem de roles deve ser permitida para `users.manage` ou `permissions.manage`,
  porque a tela de usuarios precisa do catalogo de roles para atribuicao;
- listagem de permissoes continua exigindo `permissions.manage`.

Protecoes obrigatorias:

- token de usuario continua validado por Passport;
- autorizacao final continua no backend, nunca apenas no frontend;
- senha nunca e retornada em Resource;
- senha e sempre armazenada com hash Laravel;
- roles recebidas na API devem existir e usar `guard_name = api`;
- usuarios inativos nao devem conseguir login;
- o usuario logado nao pode desativar a si mesmo;
- o usuario logado nao pode remover de si mesmo a ultima role que preserve acesso
  administrativo quando isso deixaria a conta sem `users.manage`.

Na primeira versao, a protecao contra auto-bloqueio deve ser simples e previsivel:

- se o alvo da edicao for o proprio usuario autenticado, a API rejeita `is_active =
  false`;
- se o alvo da edicao for o proprio usuario autenticado, a API rejeita uma lista de
  roles que remova a permissao efetiva `users.manage`.

## Backend

O backend deve seguir `docs/backend-laravel-guidelines.md`:

- Controllers magros;
- Form Requests para validacao/autorizacao;
- Actions para regras de criacao, atualizacao, reset de senha e sync de roles;
- Resources para respostas;
- Eloquent com eager loading de roles para evitar N+1.

### Modelo De Usuario

Campos usados pela API:

- `id`
- `uuid`
- `name`
- `email`
- `is_active`
- `roles`
- `created_at`
- `updated_at`

O campo `is_active` deve ser adicionado ao model `User`.

Regras:

- `is_active` padrao `true`;
- login so permite usuario ativo;
- email continua unico;
- senha e obrigatoria na criacao;
- senha e opcional na edicao geral;
- reset de senha e endpoint separado para evitar troca acidental junto de edicao comum.

### Endpoints

Usuarios:

- `GET /api/v1/admin/users`
- `POST /api/v1/admin/users`
- `GET /api/v1/admin/users/{user}`
- `PATCH /api/v1/admin/users/{user}`
- `PATCH /api/v1/admin/users/{user}/password`
- `DELETE /api/v1/admin/users/{user}`

Semantica:

- `DELETE` nao apaga fisicamente: marca `is_active = false`;
- `PATCH /password` altera apenas senha;
- `POST` e `PATCH` aceitam `roles` como array de nomes de roles existentes;
- respostas usam `UserResource`.

Payload de criacao:

```json
{
  "name": "Operador Patio",
  "email": "operador@example.com",
  "password": "senha-segura",
  "password_confirmation": "senha-segura",
  "is_active": true,
  "roles": ["operator"]
}
```

Payload de edicao:

```json
{
  "name": "Operador Patio 01",
  "email": "operador01@example.com",
  "is_active": true,
  "roles": ["operator", "auditor"]
}
```

Payload de reset de senha:

```json
{
  "password": "nova-senha-segura",
  "password_confirmation": "nova-senha-segura"
}
```

### Validacoes

Criacao:

- `name`: obrigatorio, string, max 255;
- `email`: obrigatorio, email valido, unico;
- `password`: obrigatorio, confirmado, minimo 8;
- `is_active`: boolean opcional, padrao true;
- `roles`: array opcional;
- `roles.*`: string, role existente com guard `api`.

Edicao:

- `name`: obrigatorio, string, max 255;
- `email`: obrigatorio, email valido, unico ignorando o usuario atual;
- `is_active`: boolean obrigatorio;
- `roles`: array obrigatorio;
- `roles.*`: string, role existente com guard `api`.

Reset de senha:

- `password`: obrigatorio, confirmado, minimo 8.

### Actions

Actions esperadas:

- `CreateUserAction`
- `UpdateUserAction`
- `ResetUserPasswordAction`
- `DeactivateUserAction`
- `SyncUserRolesAction`

Responsabilidades:

- `CreateUserAction`: cria usuario, aplica hash na senha, aplica `is_active` e delega
  roles ao sync.
- `UpdateUserAction`: altera dados basicos e delega roles ao sync.
- `ResetUserPasswordAction`: altera somente senha.
- `DeactivateUserAction`: aplica desativacao logica.
- `SyncUserRolesAction`: valida roles `api`, protege auto-bloqueio e chama
  `syncRoles`.

### Resources

`UserResource` deve expor:

- `id`
- `uuid`
- `name`
- `email`
- `is_active`
- `roles`: nomes das roles;
- `permissions`: permissoes efetivas quando o usuario carregado for o usuario atual
  ou quando o backend ja tiver essa relacao disponivel sem consulta extra;
- `created_at`
- `updated_at`

## Frontend

O frontend deve seguir `docs/frontend-vue-guidelines.md`:

- Vue 3 com `<script setup>`;
- services para API;
- componentes pequenos;
- loading, erro, vazio e sucesso;
- backend como autoridade final de permissao.

### Tela Usuarios

A tela `Usuarios` deve passar de somente leitura para gestao.

Elementos:

- tabela com nome, email, roles e status;
- botao `Novo usuario`;
- acao `Editar`;
- acao `Redefinir senha`;
- acao `Desativar` para usuarios ativos;
- acao `Reativar` para usuarios inativos, feita por edicao de status;
- modal/formulario de usuario;
- modal/formulario de senha.

Campos do formulario de usuario:

- nome;
- email;
- ativo/inativo;
- roles com selecao multipla baseada em roles retornadas pela API.

Campos do formulario de criacao:

- nome;
- email;
- senha;
- confirmacao de senha;
- ativo/inativo;
- roles.

Campos do formulario de reset de senha:

- nova senha;
- confirmacao de senha.

### Roles E Permissoes

As paginas `Roles` e `Permissoes` continuam somente leitura nesta fase.

Elas devem permanecer uteis como catalogo:

- roles exibem permissoes vinculadas;
- permissoes exibem nomes disponiveis;
- nao devem mostrar botoes de criar/editar/excluir.

Para suportar o formulario de usuarios, o endpoint de roles pode ser acessado por
usuarios com `users.manage`. A pagina de permissoes continua restrita a
`permissions.manage`.

### UX De Seguranca

Comportamentos esperados:

- usuario sem permissao ve rota bloqueada ou alerta de acesso negado;
- erro `422` mostra mensagens nos campos do formulario;
- erro `403` mostra mensagem de autorizacao;
- sucesso de criacao/edicao/reset/desativacao mostra alerta curto;
- depois de salvar, a listagem recarrega;
- formularios nao devem apagar dados digitados quando houver erro de validacao.

## Contratos De API Frontend

`usersService` deve expor:

- `list()`;
- `create(payload)`;
- `show(id)`;
- `update(id, payload)`;
- `resetPassword(id, payload)`;
- `deactivate(id)`.

Tipos esperados:

- `User`;
- `UserPayload`;
- `ResetUserPasswordPayload`;
- `Role`.

O frontend deve enviar roles como array de nomes, nao ids, para manter o contrato
estavel com o catalogo Spatie.

## Testes

Backend:

- criar usuario com roles retorna usuario ativo e roles;
- criar usuario rejeita email duplicado;
- criar usuario rejeita role inexistente;
- usuario inativo nao consegue login;
- editar usuario altera roles;
- reset de senha permite login com nova senha e rejeita senha antiga;
- delete/desativacao marca `is_active = false`;
- usuario nao pode desativar a si mesmo;
- usuario nao pode remover de si mesmo acesso efetivo a `users.manage`;
- usuario sem `users.manage` nao cria/edita/desativa usuario.

Frontend:

- `usersService.create` envia payload correto;
- `usersService.update` envia roles como nomes;
- `usersService.resetPassword` chama endpoint correto;
- `UsersPage` renderiza usuarios com status e roles;
- `UsersPage` abre modal de criacao;
- `UsersPage` mostra erros de validacao;
- `UsersPage` chama reset de senha;
- `UsersPage` chama desativacao e recarrega lista.

Verificacoes obrigatorias:

- Backend: `composer validate`, `php artisan test`, `git diff --check`;
- Frontend: `npm test -- --run`, `npm run build`, `git diff --check`.

## Documentacao

Backend:

- atualizar `docs/api-parent-admin.md` com endpoints de usuarios;
- atualizar `README.md` se novas variaveis ou comandos forem necessarios.

Frontend:

- atualizar `README.md` com resumo da gestao de usuarios, caso a tela mude o fluxo
  operacional.

AgentOps:

- manter esta spec e criar plano de implementacao correspondente antes de codar.

## Criterios De Aceite

- Administrador com `users.manage` cria usuario com roles existentes.
- Administrador com `users.manage` edita nome, email, status e roles.
- Administrador com `users.manage` redefine senha de outro usuario.
- Usuario inativo nao autentica.
- API impede auto-desativacao.
- API impede que o proprio admin remova seu acesso efetivo a `users.manage`.
- Roles/permissoes permanecem somente leitura.
- Frontend mostra status, roles e feedback de validacao.
- Testes backend e frontend passam.
- Codigo segue os padroes Laravel e Vue documentados no AgentOps.
