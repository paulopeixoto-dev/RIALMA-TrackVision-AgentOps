# Parent Admin Domain Design

## Objetivo

Construir a base administrativa do servidor pai do RIALMA TrackVision para cadastrar
veiculos, locais, edge nodes, cameras Intelbras e pares de cameras. Esta fase prepara
o sistema para captura LPR, imagem de apoio e sincronizacao edge-parent, mas nao
implementa ainda captura, integracao Intelbras, outbox, viagens ou relatorios.

## Escopo

Esta fase pertence ao backend em `RIALMA-TrackVision-Backend` e deve seguir o guia
`docs/backend-laravel-guidelines.md`.

Entidades da fase:

- `Vehicle`: cadastro de caminhao/veiculo autorizado.
- `Location`: local fisico onde existe um conjunto de cameras.
- `EdgeNode`: backend local instalado em um local fisico.
- `Camera`: camera Intelbras cadastrada como `lpr` ou `support`.
- `CameraPair`: par operacional com uma camera LPR e uma camera de apoio.

Fora do escopo:

- consumo real da API/event stream Intelbras;
- captura de imagem;
- armazenamento de midia;
- sincronizacao offline;
- classificacao de viagem;
- revisao de carga;
- frontend.

## Arquitetura

A fase deve expor endpoints administrativos versionados em `/api/v1/admin/*`,
protegidos por Passport e Spatie Permission. Passport autentica o token de usuario e
Spatie autoriza a acao por permissao.

Controllers devem ser magros. Form Requests validam entrada e autorizacao basica.
Actions executam persistencia e regras de negocio que nao pertencem ao HTTP, como
normalizacao de placa, criptografia de credenciais e validacao do par de cameras.
Resources serializam as respostas sem expor campos sensiveis.

## Modelo De Dados

### Vehicles

Campos esperados:

- `id`
- `uuid`
- `plate`
- `plate_normalized`
- `fleet_code`
- `description`
- `is_active`
- timestamps

Regras:

- `plate_normalized` remove espacos, hifens e caracteres nao alfanumericos e converte
  para maiusculo.
- `plate_normalized` e unico.
- somente veiculos ativos entram em captura, sincronizacao e relatorios futuros.

### Locations

Campos esperados:

- `id`
- `uuid`
- `name`
- `description`
- `is_active`
- timestamps

Regras:

- local representa a area fisica onde ficam o edge node e as cameras.
- locais inativos continuam historicos, mas nao devem ser usados em novos pares.

### Edge Nodes

Campos esperados:

- `id`
- `uuid`
- `location_id`
- `name`
- `description`
- `status`
- `last_seen_at`
- `is_active`
- timestamps

Regras:

- cada edge node pertence a um local.
- `status` inicia como `offline`.
- `last_seen_at` sera atualizado em fase futura pelo heartbeat.

### Cameras

Campos esperados:

- `id`
- `uuid`
- `location_id`
- `edge_node_id`
- `name`
- `type`
- `vendor`
- `host`
- `port`
- `channel`
- `username`
- `password_encrypted`
- `is_active`
- timestamps

Regras:

- `type` aceita `lpr` ou `support`.
- `vendor` inicia com `intelbras`.
- senha deve ser criptografada ao salvar.
- senha nunca deve aparecer em Resource ou listagem.
- cameras inativas nao podem compor novos pares.

### Camera Pairs

Campos esperados:

- `id`
- `uuid`
- `location_id`
- `edge_node_id`
- `name`
- `lpr_camera_id`
- `support_camera_id`
- `direction`
- `is_active`
- timestamps

Regras:

- um par deve ter exatamente uma camera `lpr` e uma camera `support`.
- as duas cameras devem pertencer ao mesmo local e ao mesmo edge node do par.
- `direction` aceita `outbound`, `inbound` ou `unknown`.
- direction define a direcao inicial de captura ate existir regra mais sofisticada de
  classificacao de viagem.

## API

Endpoints da fase:

- `GET /api/v1/admin/vehicles`
- `POST /api/v1/admin/vehicles`
- `GET /api/v1/admin/vehicles/{vehicle}`
- `PATCH /api/v1/admin/vehicles/{vehicle}`
- `DELETE /api/v1/admin/vehicles/{vehicle}`
- `GET /api/v1/admin/locations`
- `POST /api/v1/admin/locations`
- `GET /api/v1/admin/locations/{location}`
- `PATCH /api/v1/admin/locations/{location}`
- `DELETE /api/v1/admin/locations/{location}`
- `GET /api/v1/admin/edge-nodes`
- `POST /api/v1/admin/edge-nodes`
- `GET /api/v1/admin/edge-nodes/{edgeNode}`
- `PATCH /api/v1/admin/edge-nodes/{edgeNode}`
- `DELETE /api/v1/admin/edge-nodes/{edgeNode}`
- `GET /api/v1/admin/cameras`
- `POST /api/v1/admin/cameras`
- `GET /api/v1/admin/cameras/{camera}`
- `PATCH /api/v1/admin/cameras/{camera}`
- `DELETE /api/v1/admin/cameras/{camera}`
- `GET /api/v1/admin/camera-pairs`
- `POST /api/v1/admin/camera-pairs`
- `GET /api/v1/admin/camera-pairs/{cameraPair}`
- `PATCH /api/v1/admin/camera-pairs/{cameraPair}`
- `DELETE /api/v1/admin/camera-pairs/{cameraPair}`

Listagens devem usar paginacao Laravel. Respostas devem usar Resources.

## Permissoes

Permissoes existentes usadas nesta fase:

- `vehicles.manage`: criar, alterar, listar, visualizar e remover veiculos.
- `cameras.manage`: criar, alterar, listar, visualizar e remover locais, edge nodes,
  cameras e pares de cameras.

`super_admin` possui todas as permissoes por seed e por `Gate::before`. `operator`
pode usar estas permissoes conforme o catalogo atual, mas nao pode gerenciar usuarios
ou permissoes.

## Erros E Validacoes

Validacoes devem retornar `422` para entrada invalida:

- placa duplicada apos normalizacao;
- camera com `type` invalido;
- camera com `vendor` invalido;
- par com duas cameras do mesmo tipo;
- par com cameras de outro local ou outro edge node;
- par usando camera inativa;
- relacionamentos inexistentes.

Autorizacao deve retornar `403` quando o usuario autenticado nao tiver permissao
Spatie. Token ausente ou invalido deve retornar `401`.

## Testes

Feature tests obrigatorios:

- criar veiculo normaliza placa e impede duplicidade.
- usuario sem `vehicles.manage` nao cria veiculo.
- criar local e edge node com relacao correta.
- criar camera criptografa senha e nao retorna senha na API.
- criar camera pair com LPR + apoio retorna sucesso.
- camera pair com duas LPR retorna `422`.
- camera pair com cameras de locais diferentes retorna `422`.
- usuario sem `cameras.manage` nao acessa cameras ou pares.

Verificacoes obrigatorias:

- `php artisan test --filter=Admin`
- `php artisan test`
- `composer validate`
- `vendor\bin\pint --test`

## Decisoes

- Exclusao nesta fase deve ser soft delete quando o model puder impactar historico
  futuro de capturas e viagens. `DELETE` administrativo marca registros como removidos
  sem apagar fisicamente.
- Credenciais de camera entram no banco criptografadas com cast `encrypted`.
- A primeira versao nao testa conectividade real da camera no CRUD. Isso fica para a
  fase Intelbras adapter.
- O cadastro aceita IP, hostname ou DNS em `host`; validacao de reachability fica fora
  desta fase.
