# Padroes do Backend Laravel

Este guia define o padrao obrigatorio para demandas de backend do RIALMA TrackVision.
Quando uma tarefa envolver API, banco, regra de negocio, integracao ou autenticacao,
o agente deve seguir estas regras antes de propor ou implementar qualquer mudanca.

## Principio central

O backend deve ser previsivel, testavel e facil de evoluir. Para isso:

- Controllers ficam magros.
- Validacao e autorizacao de entrada ficam em Form Requests.
- Eloquent e usado com atencao a performance e clareza.
- Logica de negocio fica em Services ou Actions.
- Mudancas relevantes incluem testes e documentacao de contrato.

## Controllers magros

Controllers devem coordenar o fluxo HTTP, nao executar regra de negocio.

Um Controller pode:

- receber uma Form Request ou Request simples quando nao houver validacao relevante;
- chamar uma Action ou Service;
- retornar Resource, JSON response, redirect ou status HTTP;
- aplicar autorizacao simples quando ela nao pertencer a Form Request, Policy ou Gate.

Um Controller nao deve:

- concentrar regras de negocio;
- montar consultas complexas diretamente;
- executar multiplas operacoes de dominio com condicionais longas;
- manipular transacoes de banco extensas;
- chamar integracoes externas diretamente;
- validar arrays grandes com `$request->validate()` quando uma Form Request for mais clara.

## Form Requests

Use Form Requests para toda entrada que tenha regras de validacao, autorizacao ou
mensagens especificas.

Padrao esperado:

- criar requests em `app/Http/Requests`;
- nomear por intencao, como `StoreVehicleRequest`, `UpdateCameraRequest` ou
  `CreateTrackingSessionRequest`;
- manter `authorize()` explicito;
- manter `rules()` declarativo;
- usar `validated()` ou DTO/array validado ao chamar Services ou Actions;
- colocar mensagens customizadas somente quando melhorarem a resposta da API.

Evite espalhar validacao entre Controller, Service e Model. A entrada HTTP deve ser
validada antes de chegar na logica de negocio.

## Services e Actions

Use Services ou Actions para isolar casos de uso e regras de negocio.

Use Action quando:

- o caso de uso tem uma intencao unica e clara;
- a classe representa uma operacao, como `CreateVehicleAction`;
- a execucao pode ser exposta por `execute()`, `handle()` ou `__invoke()`.

Use Service quando:

- existe um conjunto coeso de operacoes relacionadas;
- a classe encapsula integracao externa, regra de dominio ou orquestracao reutilizavel;
- faz sentido manter estado ou dependencias compartilhadas no construtor.

Services e Actions devem:

- receber dependencias por injecao;
- ser testaveis sem depender diretamente do ciclo HTTP;
- controlar transacoes quando alterarem varios registros;
- encapsular integracoes externas atras de interfaces quando houver risco de troca;
- retornar objetos, Models, DTOs ou resultados claros para o Controller.

## Eloquent eficiente

Use Eloquent como ORM principal, mas com cuidado para evitar consultas desnecessarias.

Padroes esperados:

- definir relacionamentos nos Models;
- usar eager loading (`with`, `load`, `loadMissing`) quando a resposta acessar relacoes;
- evitar N+1 em listas, dashboards e relatorios;
- usar scopes para filtros recorrentes;
- selecionar colunas quando o payload completo nao for necessario;
- usar pagination em listagens;
- usar casts para tipos de dados relevantes;
- proteger atribuicao em massa com `fillable` ou `guarded` conforme padrao do projeto;
- manter queries complexas em Services, Actions, Query Objects ou scopes nomeados.

Evite consultas duplicadas em loops e acesso preguicoso a relacoes em respostas com
muitos registros.

## Estrutura sugerida

```text
app/
|-- Actions/
|   `-- Domain/
|-- Services/
|   `-- Domain/
|-- Http/
|   |-- Controllers/
|   |-- Requests/
|   `-- Resources/
|-- Models/
`-- Policies/
```

A estrutura final deve respeitar o que existir no backend quando o Laravel for
iniciado. Se o projeto ja tiver um padrao local, siga o padrao local e atualize este
guia quando necessario.

## Fluxo esperado em endpoints

```text
Route -> Controller -> Form Request -> Service/Action -> Eloquent -> Resource/Response
```

Para endpoints de escrita:

1. Route aponta para Controller.
2. Controller recebe Form Request.
3. Form Request valida e autoriza a entrada.
4. Controller chama Service ou Action com dados validados.
5. Service ou Action executa regra de negocio e persistencia.
6. Controller retorna Resource, JSON ou status HTTP adequado.

## Testes

Demandas de backend devem considerar:

- Feature tests para endpoints e contratos HTTP;
- testes de Form Requests quando houver regras criticas ou autorizacao;
- testes unitarios para Services ou Actions com regra de negocio;
- testes de banco com factories quando houver persistencia;
- mocks/fakes para integracoes externas.

Quando uma verificacao nao puder ser rodada, o agente deve explicar o motivo e dizer
qual comando deveria ser usado.

## Checklist para agentes

Antes de finalizar uma demanda de backend Laravel, confirme:

- [ ] Controllers continuam magros.
- [ ] Entradas relevantes usam Form Requests.
- [ ] Regras de negocio estao em Services ou Actions.
- [ ] Consultas Eloquent evitam N+1 e loops com query acidental.
- [ ] Listagens usam paginacao quando necessario.
- [ ] Contratos de API estao documentados quando mudarem.
- [ ] Testes cobrem regras, validacoes ou endpoints alterados.
- [ ] `git status --short --branch` foi conferido no repositorio correto.

## Referencias oficiais

- Laravel Controllers: https://laravel.com/docs/13.x/controllers
- Laravel Validation e Form Requests: https://laravel.com/docs/13.x/validation
- Laravel Eloquent Relationships: https://laravel.com/docs/13.x/eloquent-relationships
- Laravel Service Container: https://laravel.com/docs/13.x/container
