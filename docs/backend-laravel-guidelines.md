# Padroes do Backend Laravel

Este guia define o padrao obrigatorio para demandas de backend do RIALMA TrackVision.
Quando uma tarefa envolver API, banco, regra de negocio, integracao ou autenticacao,
o agente deve seguir estas regras antes de propor ou implementar qualquer mudanca.

## Principio central

O backend deve ser previsivel, testavel e facil de evoluir. Para isso:

- Use uma versao estavel e suportada do Laravel.
- Controllers ficam magros.
- Validacao e autorizacao de entrada ficam em Form Requests.
- Eloquent e usado com atencao a performance e clareza.
- Logica de negocio fica em Services ou Actions.
- APIs usam contratos estaveis, Resources e versionamento quando necessario.
- Configuracao da aplicacao usa arquivos de `config/` e leitura por `config()`.
- Mudancas relevantes incluem testes e documentacao de contrato.

## Versao, ferramentas e scaffolding

Antes de iniciar ou alterar o backend:

- prefira uma versao estavel, suportada e documentada do Laravel;
- use Artisan para gerar artefatos padrao, como Controllers, Requests, Models,
  Migrations, Policies e Resources;
- prefira ferramentas oficiais ou amplamente adotadas no ecossistema Laravel antes
  de adicionar dependencias novas;
- evite criar estrutura customizada quando a estrutura padrao do Laravel resolver;
- registre no AgentOps qualquer decisao que afete arquitetura, autenticacao,
  integracoes, filas, cache ou banco.

Comandos como `php artisan make:request`, `php artisan make:controller`,
`php artisan make:model -m`, `php artisan make:resource` e `php artisan test` devem
ser preferidos a scaffolding manual repetitivo.

## Controllers magros

Controllers devem coordenar o fluxo HTTP, nao executar regra de negocio.

Um Controller pode:

- receber uma Form Request ou Request simples quando nao houver validacao relevante;
- chamar uma Action ou Service;
- retornar Resource, JSON response, redirect ou status HTTP;
- aplicar autorizacao simples quando ela nao pertencer a Form Request, Policy ou Gate.
- depender de classes via injecao de dependencias do container.

Um Controller nao deve:

- concentrar regras de negocio;
- montar consultas complexas diretamente;
- executar multiplas operacoes de dominio com condicionais longas;
- manipular transacoes de banco extensas;
- chamar integracoes externas diretamente;
- instanciar Services, Actions ou clients externos com `new` quando a injecao de
  dependencias resolver;
- validar arrays grandes com `$request->validate()` quando uma Form Request for mais clara.

Para CRUD simples, use Resource Controllers quando isso deixar rotas e metodos mais
previsiveis. Para uma operacao unica ou complexa, prefira Controller invokable ou
Action explicita.

Como regra pratica, se um metodo de Controller comeca a passar de 10 a 15 linhas ou
mistura validacao, consulta, regra e resposta, extraia a regra para Service ou Action.

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

Passe para Services e Actions somente dados validados. Quando usar mass assignment,
combine `$request->validated()` com `fillable` ou `guarded` bem definido no Model.

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
- obedecer ao principio de responsabilidade unica;
- controlar transacoes quando alterarem varios registros;
- encapsular integracoes externas atras de interfaces quando houver risco de troca;
- retornar objetos, Models, DTOs ou resultados claros para o Controller.

Evite Services genericos demais. Se uma classe vira um agrupamento de metodos sem
coesao, divida por caso de uso, dominio ou Action especifica.

## Design de API

APIs devem ser pensadas para evoluir sem quebrar o frontend ou integracoes futuras.

Padroes esperados:

- usar prefixo de versao quando houver contrato publico ou risco de evolucao
  incompativel, por exemplo `/api/v1`;
- manter autenticacao, permissao e rate limiting em middleware, Policies ou Gates
  quando a regra for recorrente;
- retornar dados por API Resources ou Resource Collections em vez de expor Models crus;
- padronizar formato de erro, status HTTP e payloads de sucesso;
- documentar endpoints, parametros, payloads, respostas e codigos de erro quando
  um contrato mudar;
- evitar campos acidentais em respostas JSON;
- manter compatibilidade de contrato ou registrar breaking change explicitamente.

## Eloquent eficiente

Use Eloquent como ORM principal, mas com cuidado para evitar consultas desnecessarias.

Padroes esperados:

- definir relacionamentos nos Models;
- usar eager loading (`with`, `load`, `loadMissing`) quando a resposta acessar relacoes;
- evitar N+1 em listas, dashboards e relatorios;
- usar scopes para filtros recorrentes;
- selecionar colunas quando o payload completo nao for necessario;
- usar pagination em listagens;
- usar `chunk`, `lazy`, cursors ou filas para processamento de grandes volumes;
- usar casts para tipos de dados relevantes;
- proteger atribuicao em massa com `fillable` ou `guarded` conforme padrao do projeto;
- manter queries complexas em Services, Actions, Query Objects ou scopes nomeados.

Evite consultas duplicadas em loops e acesso preguicoso a relacoes em respostas com
muitos registros.

Prefira Eloquent e Query Builder a SQL bruto. SQL bruto so deve ser usado quando houver
ganho claro, necessidade tecnica ou expressividade insuficiente nas APIs do Laravel, e
deve vir com comentario ou teste que proteja o comportamento.

## Cache

Cache deve ser uma estrategia explicita, nao um remendo escondido.

Use cache quando:

- o dado e lido com frequencia e muda pouco;
- a consulta e cara;
- a resposta depende de integracao externa lenta;
- dashboards ou listas repetem agregacoes.

Ao adicionar cache:

- defina chave, TTL e regra de invalidacao;
- use tags quando o driver e o caso de uso suportarem invalidacao granular;
- evite cachear dados sensiveis sem necessidade;
- documente o impacto em consistencia quando houver risco de dado desatualizado;
- use helpers como `Cache::remember()` quando simplificarem o padrao read-through.

## HTTP externo e integracoes

Toda chamada HTTP externa deve ter limites claros.

Padroes esperados:

- configurar timeout;
- considerar retry com backoff quando a integracao for instavel e idempotente;
- encapsular chamadas externas em Service, client dedicado ou adapter;
- validar e normalizar respostas antes de chegar ao Controller;
- usar fakes/mocks nos testes;
- registrar falhas relevantes com contexto seguro, sem vazar segredo.

Controllers nao devem chamar `Http::get()`, `Http::post()` ou SDK externo diretamente
em fluxos de negocio.

## Configuracao e ambiente

Nunca leia `.env` diretamente em codigo de aplicacao fora de arquivos de `config/`.

Padrao esperado:

- valores de ambiente entram em arquivos de `config/` com `env()`;
- codigo de aplicacao le configuracoes com `config()`;
- segredos ficam fora do Git;
- `.env.example` deve listar chaves necessarias sem valores reais;
- ao adicionar uma integracao, documente variaveis, finalidade e valor esperado.

Esse padrao protege o uso de config cache e evita comportamento diferente entre
ambientes.

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

## Convencoes de nome

Siga as convencoes do Laravel e PSR sempre que nao houver padrao local mais especifico.

Padroes esperados:

- Models no singular, como `Vehicle`;
- Controllers com sufixo `Controller`, como `VehicleController`;
- Form Requests por acao, como `StoreVehicleRequest`;
- Resources com sufixo `Resource`, como `VehicleResource`;
- tabelas no plural em `snake_case`;
- colunas e chaves estrangeiras em `snake_case`, como `vehicle_id`;
- relacionamentos `hasOne` e `belongsTo` no singular;
- relacionamentos `hasMany`, `belongsToMany` e similares no plural;
- metodos em `camelCase`;
- migrations com nome claro e `down()` implementado.

Nao invente convencoes novas sem registrar a decisao.

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

Para endpoints de leitura:

1. Route aponta para Controller.
2. Controller chama Query Object, Service, Action ou Model scope conforme complexidade.
3. Consulta usa filtros validados, eager loading e pagination quando necessario.
4. Controller retorna Resource ou Resource Collection.

## Migrations

Toda migration deve ser reversivel sempre que tecnicamente possivel.

Padroes esperados:

- implementar `down()` para desfazer o que `up()` aplicou;
- evitar alteracoes destrutivas sem plano de migracao;
- usar nomes claros para tabelas, colunas, indices e constraints;
- documentar risco quando uma migration exigir backfill, downtime ou rotina manual;
- usar seeds/factories para dados de teste ou bootstrap, sem dados sensiveis reais.

## Testes

Demandas de backend devem considerar:

- Feature tests para endpoints e contratos HTTP;
- testes de Form Requests quando houver regras criticas ou autorizacao;
- testes unitarios para Services ou Actions com regra de negocio;
- testes de banco com factories quando houver persistencia;
- mocks/fakes para integracoes externas.
- padrao Arrange-Act-Assert para manter testes legiveis;
- `RefreshDatabase` ou estrategia equivalente para isolar estado de banco em testes.

Quando uma verificacao nao puder ser rodada, o agente deve explicar o motivo e dizer
qual comando deveria ser usado.

## Checklist para agentes

Antes de finalizar uma demanda de backend Laravel, confirme:

- [ ] Controllers continuam magros.
- [ ] Entradas relevantes usam Form Requests.
- [ ] Regras de negocio estao em Services ou Actions.
- [ ] Dependencias sao injetadas em vez de instanciadas diretamente.
- [ ] APIs usam Resources quando retornam Models ou colecoes.
- [ ] Versionamento de API foi considerado para contratos publicos.
- [ ] Consultas Eloquent evitam N+1 e loops com query acidental.
- [ ] Listagens usam paginacao quando necessario.
- [ ] Processamentos grandes usam chunk/lazy/fila quando aplicavel.
- [ ] Chamadas HTTP externas tem timeout e ficam fora do Controller.
- [ ] Codigo de aplicacao usa `config()`, nao `env()`.
- [ ] Cache novo tem chave, TTL e invalidacao definidos.
- [ ] Migrations tem `down()` quando reversao for possivel.
- [ ] Nomes seguem convencoes Laravel/PSR ou padrao local documentado.
- [ ] Contratos de API estao documentados quando mudarem.
- [ ] Testes cobrem regras, validacoes ou endpoints alterados.
- [ ] `git status --short --branch` foi conferido no repositorio correto.

## Referencias oficiais

- Laravel Controllers: https://laravel.com/docs/13.x/controllers
- Laravel Validation e Form Requests: https://laravel.com/docs/13.x/validation
- Laravel Eloquent Relationships: https://laravel.com/docs/13.x/eloquent-relationships
- Laravel Service Container: https://laravel.com/docs/13.x/container

## Referencias complementares

- ButterCMS, "19 Laravel best practices for developers in 2026":
  https://buttercms.com/blog/laravel-best-practices/
