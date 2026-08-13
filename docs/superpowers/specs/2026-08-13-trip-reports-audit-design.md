# Trip Reports And Audit Design

## Objetivo

Definir a fase de relatorios de viagens e auditoria de revisao manual de carga do RIALMA TrackVision.

Esta fase transforma as viagens ja classificadas em evidencias exportaveis para operacao, conferencia e gestao. O relatorio deve permitir analise em planilha via CSV e emissao visual via PDF contendo dados da viagem, status de carga e imagens LPR/apoio quando existirem.

## Contexto Atual

Ja existem no projeto:

- backend Laravel parent com autenticacao Passport e permissoes Spatie;
- permissoes `captures.view`, `trips.manage` e `reports.view`;
- tabela e modelo `Trip`;
- tabela e modelo `TripEvent`;
- `CaptureEvent` com `load_status`;
- `MediaAsset` privado para `lpr_image` e `support_image`;
- endpoints admin de listagem/detalhe de viagens;
- endpoint autenticado para conteudo privado de imagem;
- tela Vue `/trips` para revisar carga por evento.

A fase anterior deixou relatorios formais e auditoria completa fora do escopo. Esta fase cobre esses dois pontos sem alterar a regra deterministica de classificacao de viagens.

## Decisao De Arquitetura

A solucao sera implementada como duas capacidades independentes no dominio de viagens:

1. **Auditoria de revisao de carga**: toda alteracao real de `TripEvent.load_status` feita por usuario admin gera um registro imutavel.
2. **Relatorios de viagens**: filtros validados geram CSV ou PDF a partir da mesma fonte de dados usada pela tela de viagens.

Controllers continuam magros. Form Requests validam filtros, periodo e permissao. Actions executam consultas, auditoria e montagem dos dados. Resources/DTOs expoem contratos estaveis. O frontend chama services especificos e nunca coloca token em query string.

## Escopo V1

Dentro do escopo:

- criar historico de alteracoes manuais de carga por `TripEvent`;
- registrar usuario, valor anterior, novo valor, data/hora e vinculos de viagem/captura;
- retornar historico no detalhe da viagem;
- exibir historico no painel lateral da tela `/trips`;
- gerar CSV de viagens filtradas;
- gerar PDF de viagens filtradas com imagens LPR/apoio embutidas quando disponiveis;
- adicionar botoes de exportacao CSV/PDF na tela `/trips`;
- proteger exportacoes com `reports.view`;
- cobrir auditoria, permissao, exportacao e frontend com testes.

Fora do escopo desta fase:

- envio automatico de relatorio por e-mail;
- agendamento recorrente de relatorios;
- dashboard em tempo real;
- relatorio por motorista, nota fiscal, balanca ou ordem de transporte;
- assinatura digital do PDF;
- armazenamento permanente de arquivos PDF/CSV gerados;
- edicao manual de historico de auditoria;
- classificacao automatica de carregado/vazio por visao computacional.

## Modelo De Auditoria

Criar tabela `trip_event_load_status_audits`.

Campos:

- `id`
- `uuid`
- `trip_event_id`
- `capture_event_id`
- `user_id`
- `old_load_status`
- `new_load_status`
- `changed_at`
- timestamps

Indices:

- `uuid` unico;
- indice por `trip_event_id`, `changed_at`;
- indice por `capture_event_id`;
- indice por `user_id`;
- indice por `changed_at`.

Regras:

- o registro de auditoria e imutavel do ponto de vista da aplicacao;
- auditoria e criada somente quando o valor novo for diferente do valor atual;
- `old_load_status` e `new_load_status` usam os valores do enum `LoadStatus`;
- `user_id` aponta para o usuario admin autenticado que fez a alteracao;
- `changed_at` deve ser definido pelo backend no momento da alteracao;
- se a atualizacao falhar, nenhum registro de auditoria deve ser persistido;
- a atualizacao de `TripEvent`, `CaptureEvent` e auditoria deve ocorrer na mesma transacao.

## Revisao Manual Com Auditoria

O endpoint existente de alteracao de carga continua sendo:

```text
PATCH /api/v1/admin/trip-events/{tripEvent}/load-status
```

Permissao:

```text
trips.manage
```

Payload:

```json
{
  "load_status": "loaded"
}
```

Fluxo:

1. Form Request valida permissao e enum `LoadStatus`.
2. Controller recebe `TripEvent`, usuario autenticado e Action.
3. Action abre transacao.
4. Action bloqueia ou recarrega o evento de forma consistente.
5. Se o valor for igual ao atual, retorna o evento sem criar auditoria.
6. Se o valor mudar, cria auditoria com valor anterior e novo.
7. Atualiza `trip_events.load_status`.
8. Atualiza `capture_events.load_status`.
9. Retorna `TripEventResource` atualizado com historico quando carregado.

## Relatorio CSV

Endpoint:

```text
GET /api/v1/admin/reports/trips.csv
```

Permissao:

```text
reports.view
```

Formato:

- resposta `text/csv; charset=UTF-8`;
- download com filename contendo data de geracao;
- uma linha por `TripEvent`;
- nao embutir bytes de imagem no CSV;
- nao expor path local de storage.

Colunas v1:

- `trip_uuid`
- `vehicle_plate`
- `vehicle_fleet_code`
- `location_name`
- `trip_status`
- `trip_opened_at`
- `trip_closed_at`
- `review_required_reason`
- `event_uuid`
- `event_direction`
- `event_occurred_at`
- `event_load_status`
- `capture_plate`
- `capture_plate_normalized`
- `lpr_image_available`
- `support_image_available`
- `last_load_reviewed_by`
- `last_load_reviewed_at`

## Relatorio PDF

Endpoint:

```text
GET /api/v1/admin/reports/trips.pdf
```

Permissao:

```text
reports.view
```

Formato:

- resposta `application/pdf`;
- download com filename contendo data de geracao;
- uma viagem por bloco visual;
- eventos ordenados por horario;
- imagem LPR e imagem de apoio exibidas lado a lado quando existirem;
- se uma imagem nao existir, mostrar texto curto de ausencia;
- nao expor path local de storage no HTML, Resource ou PDF final.

Conteudo por viagem:

- placa e codigo de frota;
- local;
- status da viagem;
- horario de abertura e fechamento;
- motivo de revisao quando existir;
- para cada evento: direcao, horario, carga, placa capturada, imagem LPR e imagem de apoio;
- resumo da ultima revisao de carga quando existir.

Implementacao tecnica recomendada:

- usar uma Action para montar a colecao de dados do relatorio;
- usar um renderer isolado para transformar dados em PDF;
- embutir imagens a partir do storage privado como `data:` URI;
- limitar tipos aceitos a imagens ja registradas como `MediaAsset`;
- nao buscar imagem por URL externa;
- considerar `dompdf/dompdf` como dependencia se o backend ainda nao tiver renderer PDF.

## Filtros De Relatorio

Os filtros devem reaproveitar os nomes ja usados em viagens quando possivel.

Filtros aceitos:

- `date_from` obrigatorio;
- `date_to` obrigatorio;
- `status`: `open`, `closed`, `needs_review`;
- `plate`;
- `vehicle_id`;
- `location_id`;
- `load_status`: `unknown`, `loaded`, `empty`, `needs_review`;
- `direction`: `outbound`, `inbound`, `unknown`.

Regras:

- `date_from` e `date_to` filtram por `trips.opened_at`;
- datas devem ser validadas como datas;
- `date_to` deve ser maior ou igual a `date_from`;
- PDF deve limitar intervalo a no maximo 31 dias;
- CSV deve limitar intervalo a no maximo 180 dias;
- PDF deve limitar o volume a `config('trackvision.reports_pdf_max_trips', 100)` viagens;
- CSV deve limitar o volume a `config('trackvision.reports_csv_max_rows', 5000)` linhas;
- quando o limite for excedido, retornar `422` com mensagem clara para refinar filtros.

## API Backend

Base path:

```text
/api/v1/admin
```

Novos endpoints:

```text
GET /reports/trips.csv
GET /reports/trips.pdf
```

Endpoints existentes alterados:

```text
GET /trips/{trip}
PATCH /trip-events/{tripEvent}/load-status
```

`GET /trips/{trip}` passa a incluir, nos eventos, uma colecao `load_status_audits` quando o detalhe carregar essa relacao.

`PATCH /trip-events/{tripEvent}/load-status` passa a receber o usuario autenticado na Action para registrar auditoria quando houver alteracao real de valor.

## Resources E Contratos

### TripEventLoadStatusAuditResource

Campos:

- `id`
- `uuid`
- `old_load_status`
- `new_load_status`
- `changed_at`
- `user`

`user` deve conter:

- `id`
- `uuid`
- `name`
- `email`

### TripEventResource

Adicionar:

- `load_status_audits` quando carregado.

Ordenacao:

- auditorias em ordem decrescente por `changed_at`.

## Frontend

### Tela `/trips`

Adicionar:

- filtro `date_from`;
- filtro `date_to`;
- botoes `CSV` e `PDF` visiveis somente com `reports.view`;
- estado de carregamento separado para exportacao CSV e PDF;
- mensagem de erro quando a exportacao falhar;
- historico de revisao de carga no detalhe de cada evento.

Comportamento:

- exportacoes usam os filtros atuais da tela;
- services baixam blob autenticado com Bearer token;
- frontend cria Object URL temporario para download;
- Object URL de download deve ser revogado apos o clique;
- token nunca deve ir em query string.

### Services E Tipos

Criar:

- `src/services/reportsService.ts`

Atualizar:

- `src/services/tripsService.ts` para incluir filtros de data/local/direction quando usados;
- `src/types/admin.ts` com `TripEventLoadStatusAudit`;
- `src/pages/TripsPage.vue` para filtros, exportacao e historico;
- testes da pagina e dos services.

## Seguranca

- `reports.view` e obrigatoria para CSV/PDF.
- `trips.manage` continua obrigatoria para alterar carga.
- Auditoria e criada pelo backend, nao pelo frontend.
- Auditoria nao pode ser editada ou apagada por endpoint admin v1.
- Exportacao nao deve revelar paths internos de storage.
- PDF nao deve carregar assets remotos.
- CSV nao deve incluir tokens ou URLs assinadas.
- A API deve validar periodo e volume antes de gerar PDF pesado.
- Controllers devem continuar magros.
- Validacao deve ficar em Form Requests.
- Regras de negocio devem ficar em Actions/Services.
- Eloquent deve carregar relacoes necessarias para evitar N+1.

## Testes Backend

Feature tests:

- alterar `load_status` cria auditoria com usuario, valor anterior e novo;
- alterar para o mesmo valor nao cria auditoria duplicada;
- falha de permissao `trips.manage` nao cria auditoria;
- detalhe da viagem retorna historico de auditoria ordenado;
- CSV exige `reports.view`;
- PDF exige `reports.view`;
- CSV exige `date_from` e `date_to`;
- PDF exige `date_from` e `date_to`;
- CSV retorna headers e linha por evento;
- PDF retorna `application/pdf` e conteudo nao vazio;
- limites de periodo/volume retornam `422`;
- relatorios nao expoem path local de storage.

Unit tests:

- Action de auditoria persiste old/new status de forma transacional;
- query de relatorio aplica filtros por placa, status, carga, direcao e periodo;
- mapper de linha CSV usa valores seguros quando imagens ou auditoria nao existem;
- renderer PDF ignora midia inexistente e nao tenta carregar URL externa.

## Testes Frontend

Vitest/component tests:

- botoes CSV/PDF aparecem somente com `reports.view`;
- exportacao CSV chama service com filtros atuais;
- exportacao PDF chama service com filtros atuais;
- erro de exportacao aparece sem perder a viagem selecionada;
- historico de auditoria e renderizado no detalhe;
- actions de carga continuam aparecendo somente com `trips.manage`;
- service de relatorio baixa blob com Authorization;
- service revoga Object URL depois do download.

## Documentacao

Atualizar backend:

- `docs/api-parent-admin.md` com endpoints de relatorio e auditoria de carga;
- `README.md` com nota operacional sobre permissao `reports.view`, limites e dependencia PDF.

Atualizar frontend:

- `README.md` com uso dos botoes CSV/PDF e historico de revisao.

Atualizar AgentOps se necessario:

- plano de implementacao em `docs/superpowers/plans/`.

## Criterios De Aceite

- Toda alteracao manual real de carga gera auditoria.
- Alteracao sem mudanca de valor nao gera duplicidade.
- Historico mostra quem alterou, quando alterou, valor anterior e novo.
- Usuario com `reports.view` consegue exportar CSV e PDF.
- Usuario sem `reports.view` nao consegue exportar.
- CSV contem uma linha por evento de viagem.
- PDF contem dados da viagem e imagens LPR/apoio quando disponiveis.
- Imagens privadas continuam sem path interno exposto.
- Relatorios respeitam filtros, periodo e limites de volume.
- Frontend baixa relatorios com Authorization header.
- Backend segue Controllers magros, Form Requests, Actions/Services e Resources.
