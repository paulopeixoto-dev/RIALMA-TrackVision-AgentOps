# NVR Support Image Recovery

## Objetivo

Este runbook define como operar a recuperacao de imagem da camera de apoio pelo
NVR no RIALMA TrackVision.

A camera LPR continua sendo a origem do evento de placa. O NVR e usado como
fonte historica para buscar um frame JPEG da camera de apoio quando a imagem de
apoio nao foi capturada em tempo real.

O processamento do evento LPR e nao bloqueante em relacao ao NVR: a captura e o
`CaptureEvent` continuam sendo processados mesmo quando o NVR esta indisponivel,
nao configurado ou fora da retencao necessaria. Nesses casos, a imagem de apoio
pode permanecer ausente e a recuperacao fica registrada para tratamento posterior.

## O Que Deve Ficar Rodando

- MySQL local do edge.
- Backend Laravel edge.
- Listener LPR Intelbras ou webhook configurado para o endpoint do edge.
- Comando recorrente de recuperacao:

```bash
php artisan edge:recover-support-images --missing-only --limit=50
```

- Sincronizacao edge-to-parent quando houver internet.

## Rotina Recorrente Recomendada

O comando de recuperacao deve rodar em intervalo igual ou menor que
`TRACKVISION_NVR_RECOVERY_RETRY_MINUTES`. Com o valor padrao de 10 minutos,
configure a rotina para executar a cada 5 minutos.

Em Linux, use cron ou systemd timer supervisionado:

```cron
*/5 * * * * cd /caminho/do/backend && php artisan edge:recover-support-images --missing-only --limit=50 >> storage/logs/nvr-recovery.log 2>&1
```

Em Windows, use o Agendador de Tarefas com reinicio automatico em falha e log
redirecionado para arquivo. A rotina precisa voltar apos reinicio do edge.

Validacao obrigatoria da rotina:

- confirmar ultima execucao no log;
- confirmar codigo de saida sem erro;
- confirmar que tentativas `pending` mudam para `recovered`, `not_found` ou
  `failed`;
- confirmar que o agendamento continua ativo apos reiniciar o edge.

## Dados De Campo Obrigatorios

- IP, porta e protocolo do NVR.
- Usuario de integracao somente leitura do NVR.
- Senha do usuario de integracao, registrada apenas no backend.
- Canal do NVR para cada camera de apoio.
- Confirmacao de gravacao ativa no canal.
- Retencao minima esperada no NVR.
- NTP alinhado entre LPR, camera de apoio, NVR, edge e parent.
- Template HTTP validado para recuperar JPEG historico no firmware instalado.
- Timezone do NVR igual ao timezone do edge, ou endpoint usando timestamp com offset
  ou Unix timestamp.

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

Enquanto esse template estiver vazio, a recuperacao historica nao deve ser tratada
como habilitada. O cliente nao consulta o NVR sem template validado; se uma tentativa
for processada nessa condicao, ela pode terminar como `not_found` e exigir nova
solicitacao de recuperacao depois que o template correto for configurado.

Placeholders suportados pelo template:

- `{channel}`
- `{stream}`
- `{timestamp_iso}`
- `{timestamp_local}`
- `{unix_timestamp}`
- `{search_window_seconds}`

Prefira `{unix_timestamp}` ou `{timestamp_iso}` quando o endpoint do NVR suportar
timezone/offset. Use `{timestamp_local}` somente quando o NVR e o edge estiverem no
mesmo timezone e isso tiver sido validado com uma captura real.

Somente endpoints historicos de frame ou APIs suportadas pelo firmware do NVR
podem ser usados. E proibido fazer scraping da interface web do NVR.

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

`not_found` deve ser interpretado junto com os pre-requisitos. Se o template HTTP
ainda nao foi validado, configure o template primeiro e solicite nova recuperacao.
Se o template foi validado, investigue canal, retencao, timezone e janela de busca.

## Teste De Campo

1. Confirme que o edge acessa o NVR pela rede local.
2. Confirme que o canal da camera de apoio esta gravando.
3. Passe um veiculo pela LPR.
4. Confirme que o `CaptureEvent` da passagem foi criado no edge e identifique a
   captura correspondente.
5. Antes de qualquer recuperacao, confirme se a captura ja possui
   `support_image`; registre explicitamente a presenca ou a ausencia inicial.
6. Confirme que o template HTTP esta configurado e validado. Se ainda nao estiver,
   configure-o antes de processar a tentativa.
7. Aguarde a rotina agendada ou execute a mesma verificacao operacionalmente:

```bash
php artisan edge:recover-support-images --missing-only --limit=10
```

8. Reabra a viagem e confirme o resultado da tentativa: `support_image` recuperada
   quando houver frame disponivel, ou estado de ausencia/falha quando o NVR estiver
   sem dados.
9. Somente se ainda for necessario, como acao separada do operador, abra
   `Viagens` e clique em `Recuperar apoio`. Registre essa acao manual
   separadamente e nao a use para validar o comportamento da rotina agendada.
10. Exporte o PDF e confirme que a imagem de apoio aparece no relatorio quando
   tiver sido recuperada.

## Rotinas Criticas

- O listener/webhook da LPR deve estar ativo para capturar novas passagens.
- O comando de recuperacao deve rodar de forma recorrente no edge.
- O processo de sync edge-to-parent deve rodar quando houver internet.
- O NVR deve manter gravacao e relogio sincronizado.

## Seguranca

- Use usuario somente leitura no NVR.
- Armazene as credenciais com Laravel encrypted casts no backend.
- Nao registre senha, header de autorizacao, URL com credencial ou payload binario
  em logs.
- Credenciais nunca podem aparecer em `Resources`, payloads de bootstrap ou bundles
  do frontend.
- O parent nunca acessa o NVR.
- As imagens continuam privadas e sao servidas apenas pelos endpoints autenticados
  do backend.
