# Intelbras VIP 5460 LPR IA Field Checklist

## Dados Necessarios

- IP local do backend edge.
- Porta HTTP ou HTTPS do backend edge.
- UUID do camera pair cadastrado no TrackVision.
- Usuario Basic Auth dedicado para upload Intelbras.
- Senha Basic Auth dedicada para upload Intelbras.
- IP, porta, canal, usuario e senha da VIP 5460 LPR IA.
- IP, porta, canal, usuario e senha da camera de apoio.
- IP, porta, protocolo, usuario somente leitura e senha do NVR.
- Canal do NVR correspondente a cada camera de apoio.
- Confirmacao de que cada canal do NVR mapeado esta ativamente gravando.
- Retencao minima esperada no NVR.

## Configuracao Recomendada

- Habilitar `PictureHttpUpload` quando disponivel.
- Evento: `TrafficJunction`.
- Destino: `/api/v1/edge/intelbras/camera-pairs/{camera_pair_uuid}/lpr-events`.
- Autenticacao: Basic Auth dedicado.
- Para capturar todos os veiculos com placa legivel, deixar `TRACKVISION_EDGE_AUTO_REGISTER_UNKNOWN_VEHICLES=true` no backend edge.
- Manter LPR, camera de apoio, NVR, backend edge e parent sincronizados por NTP.
- Cadastrar o NVR em `Gravadores/NVRs`.
- Mapear cada camera de apoio para seu canal do NVR.
- Validar em campo o template HTTP de recuperacao historica do NVR antes de
  preencher `TRACKVISION_NVR_INTELBRAS_FRAME_ENDPOINT_TEMPLATE`.

## Fallback Sem Webhook

Quando a camera nao estiver enviando push/webhook para o edge, manter o listener
local ativo no backend edge:

```bash
php artisan edge:listen-intelbras {camera_pair_uuid}
```

Esse processo assina `snapManager.cgi?action=attachFileProc`, recebe eventos
`TrafficJunction`, cria capturas locais, salva imagem LPR por snapshot quando o
evento vier sem imagem e anexa capturas aceitas a viagens. Em campo, executar
como servico supervisionado para reiniciar automaticamente em queda de energia ou
reinicio da maquina.

## Validacao

- Passar um veiculo cadastrado pela LPR.
- Confirmar que o canal do NVR mapeado para a camera de apoio esta ativamente gravando.
- Confirmar `capture_events.status=accepted`.
- Confirmar uma midia `lpr_image`.
- Confirmar uma midia `support_image` quando o par tiver camera de apoio.
- Quando a imagem de apoio estiver ausente, confirmar uma tentativa em
  `capture_media_recovery_attempts`.
- Rodar `php artisan edge:recover-support-images --missing-only --limit=10`.
- Confirmar que uma recuperacao bem sucedida cria `media_assets.kind=support_image`
  e reabre `edge_outbox_messages.status=pending`.
- Confirmar que a tela `Viagens` lista a passagem e abre o detalhe com imagem LPR.
- Confirmar que a tela `Viagens` mostra o estado de recuperacao da imagem de apoio
  e permite solicitar `Recuperar apoio` para usuarios com permissao.
- Passar um veiculo ainda nao cadastrado.
- Confirmar que o veiculo foi criado com descricao `Auto cadastrado via LPR`.
- Confirmar que a captura do veiculo novo foi aceita e entrou na outbox.
- Desligar internet do local e repetir passagem.
- Confirmar que o evento fica em `edge_outbox_messages.status=pending`.
