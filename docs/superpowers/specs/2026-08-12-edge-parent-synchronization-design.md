# Edge Parent Synchronization Design

## Objetivo

Definir a sincronizacao v1 entre o backend `edge` local e o backend `parent` central do RIALMA TrackVision.

O edge deve operar sem internet, acumular capturas em outbox local e sincronizar com o parent quando a conexao voltar. O parent deve receber capturas de forma idempotente, preservar imagens LPR e apoio, expor estado de saude do edge e fornecer configuracao atual para o edge.

## Contexto Atual

Ja existem no backend:

- autenticacao Passport;
- scopes OAuth para clientes edge;
- modelo `EdgeNode` com `uuid`, `status`, `last_seen_at` e `is_active`;
- modelos administrativos de `Vehicle`, `Camera`, `CameraPair`;
- modelos locais de captura `CaptureEvent`, `MediaAsset` e `EdgeOutboxMessage`;
- webhook Intelbras local que cria capturas e outbox `capture.created`;
- endpoint provisorio `GET /api/v1/edge/bootstrap` retornando arrays vazios.

Esta fase substitui o bootstrap provisorio e adiciona os contratos reais de sync.

## Decisao De Arquitetura

A sincronizacao v1 usa **batch multipart idempotente**.

O edge executa um comando agendado:

```text
php artisan edge:sync-parent
```

Esse comando:

1. autentica no parent usando OAuth client credentials;
2. chama `GET /api/v1/edge/bootstrap` para baixar veiculos ativos e configuracao de cameras;
3. chama `POST /api/v1/edge/heartbeat` para registrar saude local;
4. envia outbox pendente para `POST /api/v1/edge/captures/batch`;
5. marca mensagens como `synced` somente quando o parent confirmar cada item.

O parent deduplica por `idempotency_key`, nao por placa ou horario. Isso permite reenvio seguro apos falha de rede, reinicio do edge ou resposta HTTP perdida.

## Escopo V1

Dentro do escopo:

- bootstrap real com veiculos ativos, cameras ativas e camera pairs ativos do `edge_node`;
- heartbeat do edge com status e timestamp;
- batch ingest de capturas com metadados JSON e arquivos de imagem multipart;
- persistencia parent de `CaptureEvent` e `MediaAsset`;
- tabela `edge_sync_batches` no parent para auditoria minima de lotes;
- tabela `edge_sync_state` no edge para cursor e ultimo sync;
- comando `edge:sync-parent`;
- retry com backoff para outbox;
- testes de idempotencia e comportamento offline.

Fora do escopo desta fase:

- tela operacional de sync no frontend;
- classificacao de viagens;
- exportacao de relatorios;
- upload em partes para arquivos muito grandes;
- resolucao automatica de conflito quando a configuracao local foi editada no edge;
- assinatura ou criptografia de payload adicional alem de HTTPS e OAuth.

## Autenticacao E Autorizacao

Usar Passport client credentials.

Scopes:

- `edge:read` para bootstrap;
- `edge:write` para heartbeat;
- `captures:write` para ingestao de capturas.

O parent deve identificar o `EdgeNode` pelo `TRACKVISION_EDGE_NODE_UUID` enviado em header e validar que:

- o cliente OAuth tem o scope requerido;
- o `edge_node_uuid` existe;
- o edge node esta ativo;
- cameras e camera pairs recebidos pertencem ao edge node autenticado;
- veiculos recebidos existem no cadastro global e estao ativos.

Header obrigatorio em chamadas edge-parent:

```text
X-TrackVision-Edge-Node: {edge_node_uuid}
```

O token OAuth prova a identidade da aplicacao edge. O header seleciona qual edge node esta enviando dados. Em v1, a associacao cliente OAuth -> edge node deve ser validada por configuracao/registro do edge node antes de producao.

## Endpoints Parent

### GET /api/v1/edge/bootstrap

Scope: `edge:read`

Retorna:

```json
{
  "data": {
    "edge_node": {
      "uuid": "edge-uuid",
      "name": "Portaria Principal",
      "status": "online"
    },
    "vehicles": [
      {
        "uuid": "vehicle-uuid",
        "plate": "ABC-1D23",
        "plate_normalized": "ABC1D23",
        "fleet_code": "TRUCK-001",
        "is_active": true,
        "updated_at": "2026-08-12T15:00:00Z"
      }
    ],
    "cameras": [
      {
        "uuid": "camera-uuid",
        "type": "lpr",
        "vendor": "intelbras",
        "host": "192.168.10.10",
        "port": 80,
        "channel": 1,
        "is_active": true
      }
    ],
    "camera_pairs": [
      {
        "uuid": "pair-uuid",
        "lpr_camera_uuid": "lpr-camera-uuid",
        "support_camera_uuid": "support-camera-uuid",
        "direction": "outbound",
        "is_active": true
      }
    ],
    "sync": {
      "server_time": "2026-08-12T15:00:00Z",
      "batch_size": 100
    }
  }
}
```

Camera passwords nao devem ser retornadas no bootstrap.

### POST /api/v1/edge/heartbeat

Scope: `edge:write`

Payload:

```json
{
  "status": "online",
  "local_time": "2026-08-12T15:00:00Z",
  "pending_outbox_count": 12,
  "last_capture_at": "2026-08-12T14:55:00Z",
  "disk_free_mb": 51200
}
```

Efeito:

- atualiza `edge_nodes.status`;
- atualiza `edge_nodes.last_seen_at`;
- pode registrar payload bruto seguro em `edge_sync_state` ou futura tabela de auditoria.

### POST /api/v1/edge/captures/batch

Scope: `captures:write`

Content-Type:

```text
multipart/form-data
```

Partes:

- `metadata`: JSON com lote e itens;
- `files[{capture_uuid}][lpr_image]`: JPEG opcional;
- `files[{capture_uuid}][support_image]`: JPEG opcional.

Metadata:

```json
{
  "batch_uuid": "batch-uuid",
  "sent_at": "2026-08-12T15:00:00Z",
  "items": [
    {
      "capture_uuid": "capture-uuid",
      "idempotency_key": "intelbras:edge:pair:ABC1D23:...",
      "vehicle_uuid": "vehicle-uuid",
      "camera_pair_uuid": "pair-uuid",
      "lpr_camera_uuid": "lpr-camera-uuid",
      "support_camera_uuid": "support-camera-uuid",
      "plate": "ABC-1D23",
      "plate_normalized": "ABC1D23",
      "event_code": "TrafficJunction",
      "action": "Pulse",
      "event_time": "2026-08-12T14:55:00Z",
      "lane": 1,
      "direction": "outbound",
      "status": "accepted",
      "load_status": "unknown",
      "raw_payload": {
        "Code": "TrafficJunction"
      },
      "media": [
        {
          "kind": "lpr_image",
          "field": "files[capture-uuid][lpr_image]",
          "sha256_hash": "hash",
          "content_type": "image/jpeg",
          "byte_size": 123456
        },
        {
          "kind": "support_image",
          "field": "files[capture-uuid][support_image]",
          "sha256_hash": "hash",
          "content_type": "image/jpeg",
          "byte_size": 123456
        }
      ]
    }
  ]
}
```

Resposta:

```json
{
  "data": {
    "batch_uuid": "batch-uuid",
    "accepted": [
      {
        "capture_uuid": "capture-uuid",
        "idempotency_key": "intelbras:edge:pair:ABC1D23:...",
        "status": "accepted"
      }
    ],
    "duplicates": [],
    "rejected": []
  }
}
```

O edge marca outbox como `synced` quando o item aparecer em `accepted` ou `duplicates`.

## Modelo De Dados

Parent:

- reutiliza `capture_events`;
- reutiliza `media_assets`;
- cria `edge_sync_batches`.

Edge:

- reutiliza `edge_outbox_messages`;
- cria `edge_sync_state`.

`edge_sync_batches` deve guardar:

- `uuid`;
- `edge_node_id`;
- `status`: `received`, `partial`, `failed`;
- `items_count`;
- `accepted_count`;
- `duplicate_count`;
- `rejected_count`;
- `received_at`;
- `summary`;

`edge_sync_state` deve guardar:

- `edge_node_uuid`;
- `last_bootstrap_at`;
- `last_push_at`;
- `last_successful_push_at`;
- `last_error`;
- `bootstrap_payload_hash`;

## Fluxo De Bootstrap

O edge chama bootstrap antes de empurrar outbox.

No v1, bootstrap e usado para manter consistencia operacional e validar que o edge esta configurado corretamente. A implementacao pode inicialmente armazenar o hash do payload e o horario em `edge_sync_state`; a replicacao local completa dos cadastros pode ser implementada em passo incremental se os testes exigirem.

## Fluxo De Push

1. Buscar `edge_outbox_messages` com `status=pending` e `available_at <= now()`.
2. Limitar por `config('trackvision.sync_batch_size')`.
3. Montar metadata JSON a partir do payload da outbox e do `CaptureEvent`.
4. Anexar arquivos de `media_assets` existentes em storage privado.
5. Enviar multipart para o parent.
6. Para itens `accepted` ou `duplicates`, marcar mensagem como `synced`.
7. Para itens `rejected`, marcar como `failed` com `last_error`.
8. Para falha HTTP/rede, incrementar `attempts`, preencher `last_error`, calcular `available_at` com backoff e manter `status=pending`.

Backoff v1:

```text
min(60 minutos, 2 ^ attempts minutos)
```

## Idempotencia

O parent deve usar `capture_events.idempotency_key` como chave unica.

Regras:

- se `idempotency_key` nao existe, criar captura e midias;
- se `idempotency_key` ja existe, retornar item como `duplicate`;
- se metadata e arquivo divergem para uma chave existente, manter a captura existente e registrar rejeicao/auditoria leve para investigacao futura;
- reenviar o mesmo batch nao pode duplicar `capture_events`, `media_assets` ou registros de outbox.

## Midias

As imagens continuam privadas.

No parent, `media_assets.path` deve ser regravado para um caminho do parent, por exemplo:

```text
captures/{capture_uuid}/{kind}/{sha256_hash}.jpg
```

O parent deve validar:

- content type permitido: `image/jpeg`;
- hash SHA-256 informado bate com bytes recebidos;
- tamanho informado bate com bytes recebidos;
- tipo da midia e `lpr_image` ou `support_image`.

## Erros E Respostas

- Edge sem scope: `403`.
- Edge node inexistente/inativo: `403` ou `404`, sem vazar detalhes sensiveis.
- Batch malformado: `422`.
- Arquivo ausente para item que declarou media: item entra em `rejected`.
- Falha parcial: resposta HTTP `200` com itens em `accepted`, `duplicates` e `rejected`.
- Falha total por payload invalido: `422`.
- Falha de rede no edge: sem alterar itens para `synced`.

## Seguranca

- Nunca enviar passwords de cameras no bootstrap.
- Nunca registrar tokens OAuth ou Authorization headers.
- Imagens devem ficar em storage privado.
- Parent deve validar que `camera_pair_uuid` e cameras pertencem ao edge node autenticado.
- Parent deve aceitar capturas de veiculos ativos ja cadastrados e tambem auto cadastrar veiculo ausente quando o edge enviar captura aceita com `vehicle_uuid`, `plate` e `plate_normalized`.
- Como o cadastro atual de veiculos e global por placa, o vinculo veiculo-edge fica fora do v1 e pode virar uma tabela de escopo por local/edge quando a operacao exigir.
- O edge nunca deve apagar captura local apos sync nesta fase; retencao fica para fase operacional.

## Testes

Feature tests principais:

- bootstrap retorna veiculos ativos e camera pairs do edge autenticado;
- bootstrap nao retorna senha de camera;
- heartbeat atualiza status e `last_seen_at`;
- batch cria captura e midias no parent;
- reenvio do mesmo batch retorna duplicate e nao duplica captura;
- falha de parent mantem outbox pending e agenda retry;
- resposta accepted/duplicate marca outbox como synced;
- item rejected marca outbox failed com erro.

Unit tests:

- serializer do batch monta metadata com campos corretos;
- parser de resposta do batch separa accepted, duplicates e rejected;
- backoff calcula proxima tentativa com limite de 60 minutos.

## Criterios De Aceite

- Edge consegue sincronizar capturas pendentes depois de uma queda de internet.
- Parent deduplica por `idempotency_key`.
- Imagens LPR e apoio chegam ao parent sem base64.
- Heartbeat deixa o parent saber quando o edge foi visto pela ultima vez.
- Bootstrap substitui o endpoint provisorio vazio.
- Outbox so muda para `synced` apos confirmacao do parent.
- Testes cobrem sync feliz, duplicidade e falha de rede.
