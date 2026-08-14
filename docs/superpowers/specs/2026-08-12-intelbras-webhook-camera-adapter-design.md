# Intelbras Webhook Camera Adapter Design

## Objetivo

Definir a integracao v1 entre o backend edge do RIALMA TrackVision e o par de cameras Intelbras usado em campo:

- camera LPR: Intelbras VIP 5460 LPR IA;
- camera de apoio: camera IP Intelbras compativel com snapshot HTTP padrao;
- servidor edge local: Laravel em perfil `edge`, operando mesmo sem internet;
- servidor parent: recebe os eventos depois por sincronizacao idempotente.

A decisao aprovada e usar uma arquitetura **webhook-first com fallback por assinatura de eventos**.

## Referencias Tecnicas

- Produto Intelbras VIP 5460 LPR IA:
  `https://www.intelbras.com/pt-br/camera-ip-com-leitura-automatica-de-placas-vip-5460-lpr-ia`
- Intelbras HTTP API V3.35:
  `https://botminio.apps.intelbras.com.br/sdk-api/HTTP%20API%20V3.35_Intelbras.pdf`

Pontos relevantes da API HTTP V3.35:

- `PictureHttpUpload` permite configurar upload ativo de imagem e evento para um servidor.
- `EventHttpUpload` permite configurar upload ativo de evento para um servidor, sem imagem.
- `snapshot.cgi` permite capturar JPEG de uma camera IP por HTTP.
- `snapManager.cgi?action=attachFileProc` permite assinar eventos e snapshots por stream multipart.
- `TrafficJunction` e o evento ANPR/LPR.
- O campo de placa aparece como `TrafficCar.PlateNumber` em payloads de trafego inteligente.

## Decisao De Arquitetura

O edge deve receber eventos da VIP 5460 LPR IA por webhook sempre que o firmware em campo permitir:

1. Priorizar `PictureHttpUpload`, porque pode entregar metadados do evento junto com a imagem LPR.
2. Aceitar `EventHttpUpload`, porque entrega o evento por `POST` JSON, mas sem imagem.
3. Quando o webhook vier sem imagem LPR, capturar snapshot imediato da propria LPR como fallback visual.
4. Sempre capturar snapshot da camera de apoio quando a placa pertencer a um veiculo ativo cadastrado.
5. Manter assinatura por `snapManager.cgi?action=attachFileProc` como fallback operacional quando o webhook nao puder ser configurado no equipamento ou firmware.

Essa decisao evita depender de internet no local, reduz latencia na passagem do caminhao e ainda deixa um caminho de contingencia para variacao de firmware.

## Escopo V1

Dentro do escopo:

- Endpoint local no backend edge para receber eventos Intelbras.
- Parser unico para payloads `PictureHttpUpload`, `EventHttpUpload` e `attachFileProc`.
- Captura HTTP de snapshot da camera de apoio.
- Captura HTTP de snapshot LPR quando o webhook nao trouxer imagem.
- Normalizacao de placa antes de consultar veiculos ativos.
- Persistencia local de evento, midias e outbox para sincronizacao posterior.
- Idempotencia para evitar duplicidade de passagem.
- Testes unitarios do parser e testes de Feature do webhook.
- Documentacao de configuracao de campo.

Fora do escopo desta fase:

- Classificacao automatica de carregado/vazio por visao computacional.
- Sincronizacao parent completa, que fica na fase de edge-to-parent sync.
- UI de revisao de viagens e relatorios.
- Controle automatico de configuracao da camera via API; no v1 o tecnico pode configurar o upload na interface ou ferramenta da Intelbras seguindo checklist.

## Contratos HTTP Do Edge

Endpoint recomendado:

```text
POST /api/v1/edge/intelbras/camera-pairs/{cameraPair:uuid}/lpr-events
```

Esse endpoint existe somente no perfil `edge`.

Autenticacao v1:

- credenciais Basic Auth dedicadas para uploads Intelbras, configuradas no edge;
- credenciais diferentes das credenciais administrativas da camera;
- opcionalmente restringir acesso por rede local/firewall;
- nunca registrar usuario, senha ou Authorization header em logs.
- Digest Auth fica reservado para chamadas que o edge faz para a camera; se o firmware em campo exigir Digest Auth tambem no upload ativo, isso deve virar ajuste explicito antes do piloto.

Configuracao esperada na VIP 5460 LPR IA:

- modo preferencial: `PictureHttpUpload`;
- modo aceito: `EventHttpUpload`;
- servidor destino: IP local do backend edge;
- porta: porta HTTP/HTTPS local do backend edge;
- caminho: `/api/v1/edge/intelbras/camera-pairs/{camera_pair_uuid}/lpr-events`;
- tipo de autenticacao: Basic Auth;
- tipo de evento: `TrafficJunction`.

Resposta do edge:

```text
200 OK
OK
```

O edge deve responder `OK` para eventos validos. Quando `TRACKVISION_EDGE_AUTO_REGISTER_UNKNOWN_VEHICLES=true`, uma placa legivel ainda nao cadastrada vira um veiculo ativo auto cadastrado e segue o fluxo operacional. Eventos sem placa legivel continuam sem inventar veiculo; se forem ignorados, tambem devem responder `OK` para evitar reenvios infinitos da camera.

## Fluxo De Dados

1. Caminhao passa pela VIP 5460 LPR IA.
2. A camera gera evento `TrafficJunction`.
3. A camera faz `POST` para o endpoint local do backend edge.
4. `IntelbrasLprWebhookController` recebe a requisicao e delega o processamento.
5. `IntelbrasWebhookParser` identifica codigo do evento, acao, horario, pista/faixa, placa, payload bruto e imagens anexadas.
6. `HandleIntelbrasLprWebhookAction` valida que o evento pertence ao par de cameras informado.
7. A placa e normalizada no mesmo padrao de `vehicles.plate_normalized`.
8. Se a placa legivel nao estiver em um veiculo ativo e `TRACKVISION_EDGE_AUTO_REGISTER_UNKNOWN_VEHICLES=true`, o edge cria um veiculo ativo com descricao `Auto cadastrado via LPR`.
9. Se o veiculo estiver resolvido, inclusive por auto cadastro, o edge captura snapshot da camera de apoio quando o par tiver apoio.
10. Se o webhook nao trouxer imagem LPR, o edge captura snapshot da camera LPR.
11. O edge salva as imagens como midias privadas locais.
12. O edge cria `capture_event` e `edge_outbox_message` com chave idempotente.
13. Quando houver internet, a fila/outbox sincroniza o evento com o parent.

## Componentes Backend

Controller:

- `Api/V1/Edge/IntelbrasLprWebhookController`
  - controller invokable e magro;
  - valida somente o fluxo HTTP;
  - chama Action;
  - retorna `OK`.

Request:

- `IntelbrasLprWebhookRequest`
  - valida autenticacao do upload;
  - valida existencia e atividade do camera pair;
  - nao tenta interpretar todo payload flexivel da Intelbras.

Actions:

- `HandleIntelbrasLprWebhookAction`
  - orquestra recebimento do webhook;
  - chama parser;
  - cria chave idempotente;
  - chama o fluxo local de captura.
- `ProcessLprEventAction`
  - fica responsavel por regra de negocio de placa ativa, midias e outbox.

Services:

- `IntelbrasHttpClient`
  - encapsula HTTP externo para cameras;
  - suporta timeout obrigatorio;
  - suporta autenticacao Digest para chamadas do edge para a camera;
  - nunca vaza senha em exception ou log.
- `IntelbrasSnapshotClient`
  - usa `GET /cgi-bin/snapshot.cgi?channel={channel}&type=0`;
  - retorna bytes JPEG, content type, hash e horario de captura.
- `IntelbrasWebhookParser`
  - interpreta JSON de `EventHttpUpload`;
  - interpreta multipart de `PictureHttpUpload`;
  - interpreta formato key-value/multipart do `attachFileProc`;
  - normaliza `TrafficCar.PlateNumber` para DTO comum.
- `IntelbrasEventStreamClient`
  - assina `snapManager.cgi?action=attachFileProc`;
  - filtra `TrafficJunction`;
  - reconecta com backoff;
  - usado somente quando webhook nao for viavel.

Data objects:

- `TrafficCaptureData`
  - `event_code`
  - `action`
  - `plate_number`
  - `plate_normalized`
  - `event_time`
  - `lane`
  - `group_id`
  - `index_in_group`
  - `raw_payload`
  - `lpr_image_bytes`
  - `lpr_image_content_type`
- `SnapshotData`
  - `bytes`
  - `content_type`
  - `sha256_hash`
  - `captured_at`
  - `source_camera_uuid`

## Configuracao

Adicionar em `config/trackvision.php`:

```php
'intelbras' => [
    'webhook_username' => env('TRACKVISION_INTELBRAS_WEBHOOK_USERNAME'),
    'webhook_password' => env('TRACKVISION_INTELBRAS_WEBHOOK_PASSWORD'),
    'event_code' => env('TRACKVISION_INTELBRAS_LPR_EVENT_CODE', 'TrafficJunction'),
    'snapshot_type' => (int) env('TRACKVISION_INTELBRAS_SNAPSHOT_TYPE', 0),
    'event_stream_heartbeat_seconds' => (int) env('TRACKVISION_INTELBRAS_EVENT_STREAM_HEARTBEAT_SECONDS', 5),
    'event_dedupe_window_seconds' => (int) env('TRACKVISION_INTELBRAS_EVENT_DEDUPE_WINDOW_SECONDS', 30),
],
```

Adicionar na `.env.example`, sem valores reais:

```text
TRACKVISION_INTELBRAS_WEBHOOK_USERNAME=
TRACKVISION_INTELBRAS_WEBHOOK_PASSWORD=
TRACKVISION_INTELBRAS_LPR_EVENT_CODE=TrafficJunction
TRACKVISION_INTELBRAS_SNAPSHOT_TYPE=0
TRACKVISION_INTELBRAS_EVENT_STREAM_HEARTBEAT_SECONDS=5
TRACKVISION_INTELBRAS_EVENT_DEDUPE_WINDOW_SECONDS=30
```

## Idempotencia

A chave idempotente do evento deve ser estavel e calculada com dados do evento:

```text
intelbras:{edge_node_uuid}:{camera_pair_uuid}:{plate_normalized}:{event_time_or_pts}:{group_id}:{index_in_group}:{payload_hash}
```

Quando algum campo nao existir, usar o melhor substituto disponivel e incluir hash do payload bruto. O objetivo e evitar duplicidade quando a camera reenviar o mesmo evento ou quando o edge reiniciar durante processamento.

## Erros E Contingencia

- Payload invalido: retornar erro controlado e registrar log seguro.
- Evento diferente de `TrafficJunction`: responder `OK` e ignorar.
- Placa ausente: registrar como evento invalido/ignorado, sem criar viagem.
- Placa desconhecida: nao enviar para fluxo parent de viagens; armazenar somente se configurado.
- Snapshot da camera de apoio falhou: criar captura com status `failed_support_capture` e enfileirar para revisao, preservando imagem LPR quando existir.
- Webhook sem imagem LPR: tentar snapshot da LPR.
- Camera LPR indisponivel para webhook: usar comando de fallback por assinatura.
- Parent offline: persistir localmente e sincronizar depois via outbox.

## Seguranca

- O endpoint webhook deve existir apenas no perfil `edge`.
- Credenciais do upload Intelbras devem ser separadas das credenciais admin das cameras.
- O edge deve aceitar upload apenas de camera pair ativo.
- Logs nao podem conter senha, token, Authorization header nem payload binario completo.
- Imagens devem ser gravadas em storage privado.
- O parent deve aceitar captura sincronizada de placa auto cadastrada pelo edge, criando o veiculo central pelo `vehicle_uuid` enviado no batch quando ainda nao existir.
- Eventos sem placa legivel nao devem criar veiculo nem viagem automaticamente.
- O fallback por assinatura deve usar timeout, reconexao com backoff e credenciais criptografadas do model `Camera`.

## Testes

Unitarios:

- parser de `EventHttpUpload` JSON com `TrafficCar.PlateNumber`;
- parser de `PictureHttpUpload` multipart com imagem JPEG;
- parser de `attachFileProc` key-value com `TrafficJunction`;
- normalizacao de placa;
- geracao de hash e DTO de snapshot.

Feature:

- webhook com placa cadastrada e ativa cria captura local;
- webhook com placa desconhecida e auto cadastro ligado cria veiculo, captura aceita, outbox e viagem no parent apos sync;
- webhook sem placa legivel responde `OK` quando ignorado e nao cria viagem parent;
- webhook sem imagem LPR captura snapshot da LPR;
- webhook sempre captura snapshot da camera de apoio para veiculo ativo;
- duplicidade de evento nao cria captura duplicada;
- credencial invalida rejeita upload;
- rota nao fica disponivel no perfil `parent`.

Integracao de campo:

- configurar `PictureHttpUpload` na VIP 5460 LPR IA e validar recebimento;
- configurar `EventHttpUpload` e validar fallback de snapshot LPR;
- desligar internet, passar veiculo cadastrado e confirmar captura local;
- religar internet e confirmar sincronizacao posterior com o parent.

## Criterios De Aceite

- A estrategia oficial do projeto e webhook-first.
- A VIP 5460 LPR IA pode usar `PictureHttpUpload` quando disponivel.
- O sistema aceita `EventHttpUpload` sem imagem e compensa com snapshot LPR.
- O fallback `attachFileProc` fica documentado e implementavel.
- Toda captura de veiculo ativo inclui imagem da camera de apoio ou status explicito de falha.
- O processamento nao depende de Controller para regra de negocio.
- O edge continua funcional sem internet.
- As credenciais e imagens sao tratadas como dados sensiveis.
