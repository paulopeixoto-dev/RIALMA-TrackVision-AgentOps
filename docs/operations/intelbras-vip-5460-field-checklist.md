# Intelbras VIP 5460 LPR IA Field Checklist

## Dados Necessarios

- IP local do backend edge.
- Porta HTTP ou HTTPS do backend edge.
- UUID do camera pair cadastrado no TrackVision.
- Usuario Basic Auth dedicado para upload Intelbras.
- Senha Basic Auth dedicada para upload Intelbras.
- IP, porta, canal, usuario e senha da VIP 5460 LPR IA.
- IP, porta, canal, usuario e senha da camera de apoio.

## Configuracao Recomendada

- Habilitar `PictureHttpUpload` quando disponivel.
- Evento: `TrafficJunction`.
- Destino: `/api/v1/edge/intelbras/camera-pairs/{camera_pair_uuid}/lpr-events`.
- Autenticacao: Basic Auth dedicado.

## Validacao

- Passar um veiculo cadastrado pela LPR.
- Confirmar `capture_events.status=accepted`.
- Confirmar uma midia `lpr_image`.
- Confirmar uma midia `support_image`.
- Desligar internet do local e repetir passagem.
- Confirmar que o evento fica em `edge_outbox_messages.status=pending`.
