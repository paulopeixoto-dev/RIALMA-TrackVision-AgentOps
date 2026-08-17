# NVR Support Image Recovery

## Objetivo

Este runbook define como operar a recuperacao de imagem da camera de apoio pelo
NVR no RIALMA TrackVision.

A camera LPR continua sendo a origem do evento de placa. O NVR e usado como
fonte historica para buscar um frame JPEG da camera de apoio quando a imagem de
apoio nao foi capturada em tempo real.

## O Que Deve Ficar Rodando

- MySQL local do edge.
- Backend Laravel edge.
- Listener LPR Intelbras ou webhook configurado para o endpoint do edge.
- Comando recorrente de recuperacao:

```bash
php artisan edge:recover-support-images --missing-only --limit=50
```

- Sincronizacao edge-to-parent quando houver internet.

## Dados De Campo Obrigatorios

- IP, porta e protocolo do NVR.
- Usuario de integracao somente leitura do NVR.
- Senha do usuario de integracao, registrada apenas no backend.
- Canal do NVR para cada camera de apoio.
- Confirmacao de gravacao ativa no canal.
- Retencao minima esperada no NVR.
- NTP alinhado entre LPR, camera de apoio, NVR, edge e parent.
- Template HTTP validado para recuperar JPEG historico no firmware instalado.

## Configuracao No Sistema

1. Cadastre o NVR em `Gravadores/NVRs`.
2. Configure host, porta, protocolo, autenticacao e credencial.
3. Cadastre o mapeamento da camera de apoio para o canal do NVR.
4. Ajuste `target_offset_seconds` quando o melhor frame estiver alguns segundos
   depois do evento da LPR.
5. Ajuste `search_window_seconds` para tolerar pequena diferenca de relogio.

## Variaveis De Ambiente

```text
TRACKVISION_NVR_DEFAULT_SEARCH_WINDOW_SECONDS=5
TRACKVISION_NVR_DEFAULT_TARGET_OFFSET_SECONDS=2
TRACKVISION_NVR_RECOVERY_RETRY_MINUTES=10
TRACKVISION_NVR_RECOVERY_MAX_ATTEMPTS=5
TRACKVISION_NVR_TIMEOUT_SECONDS=15
TRACKVISION_NVR_INTELBRAS_FRAME_ENDPOINT_TEMPLATE=
```

`TRACKVISION_NVR_INTELBRAS_FRAME_ENDPOINT_TEMPLATE` deve permanecer vazio ate o
endpoint de frame historico ser validado no NVR instalado em campo.

Placeholders suportados pelo template:

- `{channel}`
- `{stream}`
- `{timestamp_iso}`
- `{timestamp_local}`
- `{unix_timestamp}`
- `{search_window_seconds}`

## Fluxo Operacional

1. A VIP 5460 LPR IA identifica a placa.
2. O edge cria o `CaptureEvent`.
3. O edge tenta salvar a imagem LPR.
4. O edge tenta snapshot imediato da camera de apoio.
5. Se a imagem de apoio nao existir, o edge cria uma tentativa de recuperacao.
6. O comando `edge:recover-support-images` busca o frame no NVR.
7. Ao recuperar a imagem, o edge salva `media_assets.kind = support_image`.
8. A captura volta para a outbox como `pending`.
9. Quando houver internet, a sincronizacao envia a captura com a nova imagem ao parent.

## Estados De Recuperacao

- `pending_configuration`: a camera de apoio ainda nao tem NVR/canal configurado.
- `pending`: tentativa pronta para processamento.
- `running`: tentativa em processamento.
- `recovered`: imagem de apoio recuperada.
- `not_found`: o NVR nao retornou imagem na janela configurada.
- `failed`: houve falha tecnica; o sistema tenta novamente ate o limite configurado.

## Teste De Campo

1. Confirme que o edge acessa o NVR pela rede local.
2. Confirme que o canal da camera de apoio esta gravando.
3. Passe um veiculo pela LPR.
4. Abra `Viagens` e selecione a viagem.
5. Se a imagem de apoio estiver ausente, clique em `Recuperar apoio`.
6. Rode:

```bash
php artisan edge:recover-support-images --missing-only --limit=10
```

7. Reabra a viagem e confirme a imagem de apoio.
8. Exporte o PDF e confirme que a imagem recuperada aparece no relatorio.

## Rotinas Criticas

- O listener/webhook da LPR deve estar ativo para capturar novas passagens.
- O comando de recuperacao deve rodar de forma recorrente no edge.
- O processo de sync edge-to-parent deve rodar quando houver internet.
- O NVR deve manter gravacao e relogio sincronizado.

## Seguranca

- Use usuario somente leitura no NVR.
- Nao registre senha, header de autorizacao, URL com credencial ou payload binario
  em logs.
- O parent nunca acessa o NVR.
- As imagens continuam privadas e sao servidas apenas pelos endpoints autenticados
  do backend.
