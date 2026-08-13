# Trip Classification And Load Review Design

## Objetivo

Definir a fase de classificacao de viagens e revisao manual de carga do RIALMA TrackVision.

Esta fase transforma `CaptureEvent` sincronizado no parent em viagens operacionais, identifica se cada captura representa ida, volta ou caso indefinido, e permite que o operador marque visualmente se o caminhao estava carregado ou vazio usando a imagem da camera de apoio.

## Contexto Atual

Ja existem no projeto:

- backend Laravel com perfis `parent` e `edge`;
- autenticacao Passport para usuarios e clientes edge;
- permissoes `captures.view`, `trips.manage` e `reports.view`;
- cadastros de usuarios, veiculos, locais, edge nodes, cameras e pares de cameras;
- `CaptureEvent` com `vehicle_id`, `camera_pair_id`, `event_time`, `direction`, `status` e `load_status`;
- `MediaAsset` com imagens privadas `lpr_image` e `support_image`;
- webhook Intelbras VIP 5460 LPR IA no edge;
- sincronizacao edge-to-parent idempotente de capturas e midias;
- frontend Vue com shell administrativo, rotas protegidas e services por dominio.

Esta fase deve usar capturas ja aceitas no parent como fonte de verdade para montar viagens. O edge continua responsavel por capturar e sincronizar; o parent passa a ser responsavel por agrupar, revisar e expor viagens.

## Decisao De Arquitetura

A classificacao v1 sera **deterministica e conservadora**.

O parent cria uma camada de dominio em cima das capturas:

- `Trip`: representa uma viagem de um veiculo em um local.
- `TripEvent`: representa uma captura anexada a uma viagem.

A direcao vem do `CaptureEvent.direction`, que por sua vez vem do `CameraPair.direction` configurado:

- `outbound`: abre uma viagem.
- `inbound`: fecha a viagem aberta mais recente do mesmo veiculo e local.
- `unknown`: cria um caso `needs_review`.

O sistema nao deve inferir carga automaticamente nesta fase. A carga deve ser marcada manualmente em `TripEvent` como `unknown`, `loaded`, `empty` ou `needs_review`.

## Escopo V1

Dentro do escopo:

- criar tabelas `trips` e `trip_events`;
- anexar capturas aceitas a viagens no parent;
- classificar viagens por veiculo, local e direcao do par de cameras;
- expor listagem e detalhe de viagens no admin API;
- expor imagem privada LPR/apoio por endpoint autenticado;
- permitir atualizar `load_status` de um `TripEvent`;
- refletir revisao de carga no `CaptureEvent.load_status`;
- adicionar tela Vue para revisao operacional de viagens e carga;
- cobrir classificacao, permissoes, idempotencia e revisao com testes.

Fora do escopo desta fase:

- relatorios formais e exportacao CSV/PDF;
- auditoria completa de alteracoes;
- classificacao automatica de carregado/vazio por visao computacional;
- correcao manual de direcao ou reprocessamento em massa;
- reclassificacao automatica quando a direcao de um camera pair antigo for editada;
- vinculo por motorista, ordem de transporte, nota fiscal ou balanca;
- estimativa de tempo de viagem, SLA, alertas ou dashboards em tempo real.

## Modelo De Dados

### trips

Campos:

- `id`
- `uuid`
- `vehicle_id`
- `location_id`
- `status`: `open`, `closed`, `needs_review`
- `opened_at`
- `closed_at`
- `review_required_reason`
- timestamps

Indices:

- `uuid` unico;
- indice por `vehicle_id`, `location_id`, `status`;
- indice por `opened_at`;
- indice por `closed_at`.

Regras:

- uma viagem pertence a um veiculo e a um local;
- `opened_at` deve usar o horario do primeiro evento da viagem;
- `closed_at` deve usar o horario do evento inbound que fechou a viagem;
- `review_required_reason` deve explicar casos ambiguos de forma curta e segura.

### trip_events

Campos:

- `id`
- `uuid`
- `trip_id`
- `capture_event_id`
- `direction`: `outbound`, `inbound`, `unknown`
- `load_status`: `unknown`, `loaded`, `empty`, `needs_review`
- `occurred_at`
- timestamps

Indices:

- `uuid` unico;
- `capture_event_id` unico;
- indice por `trip_id`, `occurred_at`;
- indice por `direction`;
- indice por `load_status`.

Regras:

- cada captura aceita pode gerar no maximo um `TripEvent`;
- `TripEvent.load_status` deve iniciar com o valor de `CaptureEvent.load_status`;
- ao atualizar `TripEvent.load_status`, atualizar tambem `CaptureEvent.load_status`;
- `TripEvent.direction` deve guardar a direcao usada na classificacao para preservar historico.

## Regras De Classificacao

A action principal recebe um `CaptureEvent` aceito e idempotente.

Pre-condicoes:

- `capture_events.status` deve ser `accepted`;
- `capture_events.vehicle_id` deve existir;
- `capture_events.camera_pair_id` deve existir;
- o local deve ser obtido por `captureEvent.cameraPair.location_id`;
- se ja existir `trip_events.capture_event_id`, retornar o evento existente sem duplicar.

### Evento outbound

1. Buscar a viagem `open` mais recente para o mesmo `vehicle_id` e `location_id`.
2. Se existir uma viagem aberta anterior, marcar essa viagem como `needs_review` com reason `new_outbound_before_inbound`.
3. Criar nova viagem `open`.
4. Criar `TripEvent` com direction `outbound`.
5. Definir `opened_at` com `capture_events.event_time`.

### Evento inbound

1. Buscar a viagem `open` mais recente para o mesmo `vehicle_id` e `location_id`.
2. Se existir, anexar o evento inbound a essa viagem.
3. Fechar a viagem com status `closed` e `closed_at` igual a `capture_events.event_time`.
4. Se nao existir viagem aberta, criar viagem `needs_review` com reason `inbound_without_outbound`.
5. Nesse caso, `opened_at` e `closed_at` usam o horario da captura inbound.

### Evento unknown

1. Criar viagem `needs_review`.
2. Criar `TripEvent` com direction `unknown`.
3. Usar reason `unknown_direction`.
4. Nao fechar ou alterar outra viagem aberta automaticamente.

### Ordem Ambigua

Se um inbound tiver horario anterior ao `opened_at` da viagem aberta encontrada:

- nao fechar a viagem aberta;
- criar uma nova viagem `needs_review`;
- usar reason `inbound_before_open_trip`;
- anexar o inbound nessa viagem de revisao.

## Revisao Manual De Carga

O operador revisa carga por `TripEvent`, nao por viagem inteira.

Valores aceitos:

- `unknown`
- `loaded`
- `empty`
- `needs_review`

Permissao:

- visualizar viagens e imagens: `captures.view`;
- alterar carga: `trips.manage`.

Efeito da atualizacao:

1. validar `load_status`;
2. atualizar `trip_events.load_status`;
3. atualizar `capture_events.load_status`;
4. retornar `TripEventResource` atualizado.

O frontend deve mostrar pelo menos a imagem de apoio quando existir, pois ela e a evidencia principal para carga. A imagem LPR tambem deve ficar disponivel para conferencia de placa.

## API Backend

Base path: `/api/v1/admin`

Todos os endpoints usam Passport bearer token de usuario e o middleware `EnsureUserAccessToken::using('admin:read')`.

### GET /trips

Permissao: `captures.view`

Filtros v1:

- `status`: `open`, `closed`, `needs_review`
- `vehicle_id`
- `location_id`
- `plate`
- `load_status`: filtra por eventos com o status informado
- `date_from`: filtra por `opened_at >= date_from`
- `date_to`: filtra por `opened_at <= date_to`

Resposta:

- paginada;
- ordenada por `opened_at desc`;
- inclui veiculo, local, status, horarios e resumo dos eventos.

### GET /trips/{trip}

Permissao: `captures.view`

Resposta:

- dados da viagem;
- veiculo;
- local;
- eventos ordenados por `occurred_at`;
- para cada evento: captura, placa, cameras, direcao, carga e referencias de media.

### PATCH /trip-events/{tripEvent}/load-status

Permissao: `trips.manage`

Payload:

```json
{
  "load_status": "loaded"
}
```

Resposta:

- `TripEventResource` atualizado.

### GET /media-assets/{mediaAsset}/content

Permissao: `captures.view`

Uso:

- o frontend chama esse endpoint via `fetch` com Bearer token;
- a resposta e um JPEG privado;
- o frontend cria um Object URL temporario para usar em `<img>`.

Regras:

- aceitar somente midias de capturas acessiveis pelo fluxo admin;
- retornar `404` se o arquivo nao existir no storage privado;
- retornar `403` se o usuario nao tiver permissao;
- nunca expor path local do storage em resposta JSON.

## Resources

### TripResource

Campos:

- `id`
- `uuid`
- `status`
- `opened_at`
- `closed_at`
- `review_required_reason`
- `vehicle`
- `location`
- `events_count`
- `current_load_status`
- `events` quando carregado em detalhe

`current_load_status` e derivado do evento mais recente com `load_status` diferente de `unknown`; se nao existir, retornar `unknown`.

### TripEventResource

Campos:

- `id`
- `uuid`
- `direction`
- `load_status`
- `occurred_at`
- `capture`
- `media`

`capture` deve conter:

- `id`
- `uuid`
- `plate`
- `plate_normalized`
- `event_time`
- `camera_pair`

`media` deve conter por kind:

- `id`
- `uuid`
- `kind`
- `content_type`
- `byte_size`
- `content_endpoint`

`content_endpoint` deve ser caminho relativo de API, por exemplo:

```text
/api/v1/admin/media-assets/{id}/content
```

## Frontend

### Rotas

Criar rota:

```text
/trips
```

Nome:

```text
trips
```

Permissao visual:

```text
captures.view
```

Adicionar item na sidebar:

- label: `Viagens`
- icone: usar `Route`, `Waypoints` ou icone equivalente do `lucide-vue-next`
- permission: `captures.view`

### Services E Tipos

Criar:

- `src/services/tripsService.ts`
- `src/services/mediaAssetsService.ts`

Adicionar tipos em `src/types/admin.ts`:

- `TripStatus`
- `TripEventDirection`
- `LoadStatus`
- `Trip`
- `TripEvent`
- `TripMediaAsset`

### Tela De Revisao

Criar `src/pages/TripsPage.vue`.

A tela deve ter:

- filtros por status, placa e carga;
- tabela de viagens com placa, veiculo, local, status, horario de abertura, horario de fechamento e carga atual;
- painel de detalhe ao selecionar uma viagem;
- eventos com direcao, horario, placa e carga;
- imagem LPR e imagem de apoio quando existirem;
- botoes para marcar `loaded`, `empty` e `needs_review` quando o usuario tiver `trips.manage`;
- estado loading, erro, vazio e sucesso.

O frontend nao deve colocar token em query string. Imagens privadas devem ser carregadas pelo service com Authorization header e convertidas em Object URL local. Object URLs devem ser revogadas quando a imagem sair da tela.

## Fluxos

### Fluxo Feliz

1. Edge captura placa em camera outbound.
2. Edge sincroniza captura com o parent.
3. Parent cria `CaptureEvent`, midias e `Trip open`.
4. Operador abre tela de viagens e ve evento outbound.
5. Edge captura a mesma placa em camera inbound.
6. Parent anexa inbound e fecha a viagem.
7. Operador marca carga do evento vendo a imagem de apoio.

### Sem Inbound

1. Outbound abre viagem.
2. Nenhum inbound chega.
3. Viagem continua `open`.
4. Relatorios futuros podem tratar viagem aberta separadamente.

### Inbound Sem Outbound

1. Inbound chega sem viagem aberta.
2. Parent cria viagem `needs_review`.
3. Tela mostra reason `inbound_without_outbound`.

### Direcao Desconhecida

1. Captura chega com direction `unknown`.
2. Parent cria viagem `needs_review`.
3. Tela mostra evento como revisao.

## Seguranca

- Controllers devem ficar magros.
- Validacao deve ficar em Form Requests.
- Regras de negocio devem ficar em Actions.
- Resources nao devem expor paths internos de storage.
- Imagens privadas devem exigir usuario autenticado e permissao `captures.view`.
- O frontend pode esconder botoes, mas o backend e autoridade final para `trips.manage`.
- Revisao de carga nao deve aceitar valores fora do enum `LoadStatus`.
- A API nao deve permitir atualizar carga de um evento que nao pertence a uma viagem.

## Testes Backend

Feature tests principais:

- outbound cria viagem `open`;
- inbound fecha viagem aberta do mesmo veiculo e local;
- inbound sem outbound cria viagem `needs_review`;
- unknown cria viagem `needs_review`;
- novo outbound antes de inbound marca viagem anterior como `needs_review`;
- reprocessar a mesma captura nao cria outro `TripEvent`;
- atualizar carga exige `trips.manage`;
- usuario com apenas `captures.view` consegue ver viagens, mas nao altera carga;
- endpoint de media exige `captures.view`;
- endpoint de media retorna JPEG privado quando arquivo existe;
- endpoint de media retorna `404` quando arquivo nao existe.

Unit tests:

- action de classificacao resolve direction `outbound`, `inbound` e `unknown`;
- derivacao de `current_load_status` prioriza o evento revisado mais recente.

## Testes Frontend

Vitest/component tests:

- rota `/trips` exige `captures.view`;
- sidebar mostra `Viagens` para usuario com `captures.view`;
- pagina renderiza loading, erro e vazio;
- pagina lista viagens retornadas pela API;
- detalhe renderiza eventos e estados de carga;
- botoes de carga aparecem somente com `trips.manage`;
- update de carga chama `tripsService.updateLoadStatus`;
- service de media busca blob com Authorization e libera Object URL no componente.

## Documentacao

Atualizar no backend:

- `docs/api-parent-admin.md` com endpoints de viagens, revisao de carga e media privada;
- `README.md` se houver novo comando operacional ou observacao de permissao.

Atualizar no frontend:

- `README.md` com a tela de viagens e permissao esperada.

## Criterios De Aceite

- Capturas aceitas no parent sao agrupadas em viagens.
- Viagens outbound/inbound sao classificadas de forma deterministica.
- Casos ambiguos ficam visiveis como `needs_review`.
- Operador consegue revisar carga pela imagem de apoio.
- Imagem LPR e imagem de apoio continuam privadas.
- Backend protege visualizacao e alteracao por permissao.
- Frontend oferece tela operacional usavel com estados de loading, erro, vazio e sucesso.
- Testes cobrem classificacao, permissao, idempotencia, media privada e revisao de carga.
