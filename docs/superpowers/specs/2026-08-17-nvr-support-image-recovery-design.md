# NVR Support Image Recovery Design

## Objetivo

Definir a estrategia oficial para recuperar a imagem historica da camera de apoio usando o NVR quando o snapshot em tempo real nao existir, falhar ou precisar ser reprocessado.

O TrackVision deve continuar usando a VIP 5460 LPR IA como fonte primaria do evento de placa. O NVR passa a ser a fonte primaria para recuperar imagens historicas da camera de apoio por horario.

## Contexto Atual

Ja existem no projeto:

- backend Laravel com perfis `edge` e `parent`;
- camera LPR Intelbras VIP 5460 LPR IA integrada por webhook ou listener `attachFileProc`;
- `ProcessLprEventAction` que cria `CaptureEvent`, salva imagem LPR e tenta snapshot imediato da camera de apoio;
- `CaptureEvent.status = failed_support_capture` quando a captura da camera de apoio falha;
- `MediaAsset` privado com kinds `lpr_image` e `support_image`;
- sincronizacao edge-to-parent por outbox idempotente;
- tela de viagens e relatorios que dependem da imagem de apoio para revisar carga.

O problema a resolver e a lacuna historica: se a captura de apoio em tempo real falhar, a imagem ainda pode existir na gravacao do NVR. O sistema deve buscar esse frame depois e anexar ao `CaptureEvent` correto.

## Referencias Tecnicas

- Produto Intelbras VIP 5460 LPR IA:
  `https://www.intelbras.com/pt-br/camera-ip-com-leitura-automatica-de-placas-vip-5460-lpr-ia`
- Datasheet VIP 5460 LPR IA:
  `https://backend.intelbras.com/sites/default/files/2025-12/Datasheet%20-%20VIP%205460%20LPR%20IA%20-%20V14.pdf`
- Intelbras HTTP API V3.35, ja referenciada na spec do adapter Intelbras:
  `https://botminio.apps.intelbras.com.br/sdk-api/HTTP%20API%20V3.35_Intelbras.pdf`

Pontos confirmados do produto:

- a VIP 5460 LPR IA gera eventos e relatorios de placas;
- a camera armazena informacoes como horario, data, placa, cor, marca, tipo, direcao e sentido;
- o produto suporta HTTP/HTTPS, RTSP, ONVIF, FTP/SFTP e micro-SD;
- a foto e JPEG.

Ponto a validar em campo antes da implementacao final:

- endpoint exato do NVR Intelbras disponivel no firmware instalado para buscar frame ou trecho de gravacao por canal e horario.

## Decisao De Arquitetura

A arquitetura aprovada e:

- LPR e a fonte do evento operacional.
- NVR e a fonte de imagem historica da camera de apoio.
- Camera de apoio continua podendo ser consultada por snapshot em tempo real.
- Se o snapshot em tempo real nao gerar `support_image`, o edge agenda recuperacao pelo NVR.
- Uma rotina recorrente tambem varre capturas antigas sem `support_image` e tenta preencher a evidencia faltante.

Essa decisao evita depender de cartao SD em cada camera de apoio e concentra a recuperacao historica no equipamento que ja grava multiplos canais.

## Escopo V1

Dentro do escopo:

- cadastrar NVR/gravador como fonte de gravacao no edge;
- mapear camera de apoio para canal do NVR;
- configurar offset e janela de busca por camera pair;
- recuperar frame de apoio por horario do evento LPR;
- anexar a imagem recuperada como `media_assets.kind = support_image`;
- re-enfileirar captura na outbox quando uma imagem nova for anexada no edge;
- expor estado de recuperacao para operacao e auditoria;
- documentar checklist de campo para NTP, canais e usuario somente leitura.

Fora do escopo da V1:

- substituir o webhook/listener LPR por leitura do NVR;
- classificar automaticamente carregado/vazio por visao computacional;
- baixar ou armazenar video completo no TrackVision;
- apagar arquivos do NVR;
- administrar configuracoes internas do NVR automaticamente;
- recuperar imagem historica de apoio quando o NVR nao tiver gravacao do canal.

## Modelo De Dados

### recording_devices

Representa NVR/DVR usado como fonte de gravacao.

Campos:

- `id`
- `uuid`
- `edge_node_id`
- `location_id`
- `name`
- `vendor`: inicialmente `intelbras`
- `host`
- `port`
- `username`
- `password_encrypted`
- `is_active`
- timestamps
- soft deletes

Regras:

- senha deve usar cast `encrypted`;
- usuario recomendado deve ser somente leitura;
- credenciais nunca aparecem em resources, logs ou bootstrap do parent.

### camera_recording_sources

Mapeia uma camera do TrackVision para um canal do NVR.

Campos:

- `id`
- `uuid`
- `camera_id`
- `recording_device_id`
- `channel`
- `stream`: `main` ou `sub`
- `target_offset_seconds`
- `search_window_seconds`
- `is_active`
- timestamps

Regras:

- `camera_id` deve apontar para camera de apoio quando usado como evidencia de carga;
- `target_offset_seconds` ajusta o momento ideal do frame em relacao ao horario LPR;
- `search_window_seconds` define tolerancia para achar gravacao quando ha drift de relogio;
- deve existir no maximo uma fonte ativa por camera para V1.

### capture_media_recovery_attempts

Registra tentativas de recuperacao de midia faltante.

Campos:

- `id`
- `uuid`
- `capture_event_id`
- `media_kind`: inicialmente `support_image`
- `recording_device_id`
- `camera_id`
- `target_time`
- `status`: `pending`, `running`, `recovered`, `not_found`, `failed`
- `attempts`
- `last_error`
- `next_attempt_at`
- timestamps

Regras:

- uma captura nao deve ter duas tentativas pendentes para o mesmo kind;
- `last_error` deve ser sanitizado e nao pode conter senha, URL com credencial ou payload binario;
- tentativas `recovered` nao devem ser reprocessadas.

## Fluxo De Dados Em Tempo Real

1. Caminhao passa na LPR.
2. O edge recebe evento `TrafficJunction`.
3. `ProcessLprEventAction` cria ou reutiliza o `CaptureEvent`.
4. O edge salva imagem LPR quando disponivel ou captura snapshot da LPR.
5. O edge tenta snapshot imediato da camera de apoio quando ela existe.
6. Se o snapshot de apoio funcionar, salva `support_image`.
7. Se falhar ou nao houver `support_image`, o edge cria `capture_media_recovery_attempts.status = pending`.
8. Um comando agendado busca o frame no NVR e salva `support_image`.
9. Ao anexar nova midia no edge, a captura volta para outbox para sincronizar a imagem faltante com o parent.

## Fluxo De Recuperacao Retroativa

Comando recomendado:

```bash
php artisan edge:recover-support-images {cameraPairUuid?} --from=2026-08-17T00:00:00 --to=2026-08-17T23:59:59 --missing-only
```

Comportamento:

1. Selecionar capturas aceitas ou `failed_support_capture` dentro do periodo.
2. Filtrar capturas sem `media_assets.kind = support_image`.
3. Resolver a camera de apoio pelo `camera_pair`.
4. Resolver o canal do NVR via `camera_recording_sources`.
5. Calcular `target_time = capture_event.event_time + target_offset_seconds`.
6. Procurar gravacao no NVR dentro de `search_window_seconds`.
7. Baixar frame JPEG ou baixar trecho curto e extrair frame, conforme capacidade do NVR.
8. Salvar `support_image` privado.
9. Atualizar tentativa como `recovered`, `not_found` ou `failed`.
10. Reenfileirar sincronizacao quando a imagem for recuperada.

## Componentes Backend

Controllers:

- controllers admin para CRUD de `recording_devices`;
- controllers admin para vincular camera de apoio ao canal do NVR;
- controllers devem permanecer magros e delegar para Actions.

Requests:

- Form Requests para criar/editar NVR;
- Form Requests para mapear camera/canal;
- validacao de host, porta, canal, offset e janela de busca.

Actions:

- `CreateRecordingDeviceAction`
- `UpdateRecordingDeviceAction`
- `MapCameraRecordingSourceAction`
- `ScheduleSupportImageRecoveryAction`
- `RecoverSupportImageFromNvrAction`
- `RecoverMissingSupportImagesAction`

Services:

- `RecordingPlaybackClient`
  - interface de dominio para buscar imagem por canal e horario.
- `IntelbrasNvrPlaybackClient`
  - implementacao Intelbras.
  - usa timeout obrigatorio e autenticacao Digest quando necessario.
  - deve suportar descoberta de capacidade do NVR no piloto.
- `SupportImageRecoverySelector`
  - seleciona capturas candidatas sem `support_image`.
- `RecordingFrameExtractor`
  - encapsula a estrategia de obter JPEG quando o NVR entregar video ou stream em vez de foto direta.

Data objects:

- `RecordingFrameRequestData`
  - `recording_device_uuid`
  - `camera_uuid`
  - `channel`
  - `target_time`
  - `search_window_seconds`
  - `stream`
- `RecordingFrameData`
  - `bytes`
  - `content_type`
  - `sha256_hash`
  - `captured_at`
  - `source_recording_device_uuid`
  - `source_channel`

## Estrategias De Busca No NVR

A implementacao deve ser preparada para tres estrategias, selecionando a primeira suportada pelo NVR em campo:

1. **Frame por horario**
   - Melhor opcao.
   - O NVR retorna uma imagem JPEG de um canal em um timestamp.

2. **Trecho curto por horario**
   - O NVR retorna um clip curto.
   - O edge extrai um frame localmente no `target_time`.

3. **Playback RTSP por horario**
   - Opcao de contingencia quando a API HTTP nao fornece foto direta.
   - Exige cuidado com timeout e dependencia de extrator de frame.

Nao se deve implementar scraping da interface web do NVR como caminho oficial.

## Configuracao

Adicionar em `config/trackvision.php`:

```php
'nvr' => [
    'default_search_window_seconds' => (int) env('TRACKVISION_NVR_DEFAULT_SEARCH_WINDOW_SECONDS', 5),
    'default_target_offset_seconds' => (int) env('TRACKVISION_NVR_DEFAULT_TARGET_OFFSET_SECONDS', 2),
    'recovery_retry_minutes' => (int) env('TRACKVISION_NVR_RECOVERY_RETRY_MINUTES', 10),
    'max_attempts' => (int) env('TRACKVISION_NVR_RECOVERY_MAX_ATTEMPTS', 5),
],
```

Adicionar na `.env.example`, sem valores reais:

```text
TRACKVISION_NVR_DEFAULT_SEARCH_WINDOW_SECONDS=5
TRACKVISION_NVR_DEFAULT_TARGET_OFFSET_SECONDS=2
TRACKVISION_NVR_RECOVERY_RETRY_MINUTES=10
TRACKVISION_NVR_RECOVERY_MAX_ATTEMPTS=5
```

## Sincronizacao De Horario

NTP e requisito operacional.

Todos devem apontar para a mesma fonte de horario:

- camera LPR;
- camera de apoio;
- NVR;
- servidor edge;
- servidor parent.

Se o NVR estiver fora de sincronia, o sistema pode buscar a imagem errada. A UI deve permitir configurar `target_offset_seconds` por camera de apoio ou camera pair para compensar diferenca operacional pequena.

## Idempotencia

A recuperacao nao pode duplicar midia.

Regras:

- se ja existir `support_image` para a captura, pular por padrao;
- se o hash da imagem recuperada ja existir para a mesma captura/kind, reutilizar ou ignorar;
- uma tentativa `pending` por `capture_event_id + media_kind` deve ser unica;
- reprocessamento manual deve exigir flag explicita, por exemplo `--force`.

## Erros E Contingencia

- NVR sem conexao: tentativa vira `failed`, com `next_attempt_at` preenchido.
- Canal sem gravacao no periodo: tentativa vira `not_found`.
- Credencial invalida: registrar erro sanitizado e nao tentar em loop agressivo.
- NVR retorna video mas extrator nao esta disponivel: tentativa vira `failed` com motivo operacional claro.
- Horario fora da retencao do NVR: tentativa vira `not_found`.
- Camera pair sem apoio: nao criar tentativa.
- Camera de apoio sem mapeamento de NVR: marcar como pendente de configuracao, sem derrubar captura LPR.

## Seguranca

- NVR deve usar usuario de integracao somente leitura.
- Senha do NVR deve ser criptografada no banco.
- Logs nao podem conter senha, Authorization header, URL com credencial nem payload binario.
- Recuperacao roda apenas no edge da rede local.
- Parent recebe somente a midia privada ja anexada via outbox; parent nao deve acessar NVR diretamente.
- API admin deve proteger cadastro de NVR com `cameras.manage`.
- Visualizacao de imagens continua protegida por `captures.view`.

## Frontend

Adicionar, dentro do padrao Vuestic Admin:

- pagina ou aba de `Gravadores/NVRs`;
- formulario de NVR com senha write-only;
- mapeamento de cameras de apoio para canal do NVR;
- campos de offset e janela de busca;
- indicador em viagens para `support_image` ausente, recuperada ou pendente;
- acao manual para solicitar recuperacao de imagem de apoio quando o usuario tiver permissao adequada.

O frontend deve priorizar componentes Vuestic conforme diretriz vigente do projeto.

## Documentacao Operacional

Atualizar checklist de campo com:

- IP, porta, usuario e senha do NVR;
- canal do NVR para cada camera de apoio;
- confirmacao de gravacao ativa no canal;
- retencao minima desejada;
- sincronizacao NTP;
- usuario somente leitura;
- teste de recuperacao usando uma captura real.

## Testes Backend

Feature tests:

- captura aceita sem `support_image` agenda tentativa de recuperacao;
- camera pair sem apoio nao agenda tentativa;
- camera de apoio sem NVR mapeado registra pendencia controlada;
- comando recupera imagem do NVR e cria `support_image`;
- comando nao duplica midia quando executado duas vezes;
- recuperacao reenfileira outbox quando uma nova midia e anexada;
- usuario sem `cameras.manage` nao cadastra NVR.

Unit tests:

- calculo de `target_time` com offset;
- selecao de capturas sem `support_image`;
- parser/cliente Intelbras NVR para resposta de sucesso;
- tratamento de `not_found`, timeout e credencial invalida;
- sanitizacao de erro antes de persistir `last_error`.

## Testes Frontend

- pagina lista NVRs usando Vuestic;
- formulario omite senha quando vazia em edicao;
- mapeamento mostra cameras de apoio e canais;
- tela de viagens mostra estado de imagem de apoio pendente;
- acao manual de recuperar imagem chama service correto;
- botoes respeitam permissoes.

## Criterios De Aceite

- LPR continua sendo a fonte do evento de placa.
- NVR e usado para recuperar imagem historica de apoio por canal e horario.
- Capturas sem `support_image` podem ser preenchidas depois sem duplicar midia.
- Parent nao precisa acessar NVR.
- Fluxo funciona sem internet no local, desde que edge tenha acesso ao NVR.
- Falhas de NVR ficam visiveis e reprocessaveis.
- Credenciais e imagens seguem privadas.
- Relatorios passam a aproveitar `support_image` recuperada quando disponivel.
