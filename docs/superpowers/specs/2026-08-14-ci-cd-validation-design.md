# CI/CD Validation Design

## Objetivo

Adicionar validacao automatica no GitHub Actions para os repositorios backend e
frontend do RIALMA TrackVision, garantindo que cada `push` e `pull_request` para
`main` rode os comandos basicos de qualidade antes de novas fases do projeto.

Esta etapa cria CI de validacao. Deploy automatico, provisionamento de servidor e
publicacao de artefatos de producao ficam fora do escopo.

## Escopo

Repositorios afetados:

- `RIALMA-TrackVision-Backend`
- `RIALMA-TrackVision-Frontend`

AgentOps deve registrar esta decisao e o plano de implementacao correspondente.

Funcionalidades desta fase:

- criar workflow GitHub Actions para backend;
- criar workflow GitHub Actions para frontend;
- validar que o backend roda `composer validate`, Pint e testes Laravel;
- validar que o frontend roda ESLint, Vitest e build Vite;
- usar cache de dependencias para npm e Composer;
- manter permissoes do `GITHUB_TOKEN` restritas a leitura de conteudo.

Fora do escopo:

- deploy automatico;
- ambientes GitHub Environments;
- secrets de producao;
- build/push de imagens Docker;
- publicacao de assets;
- Playwright/E2E em CI;
- Dependabot.

## Decisao Principal

Implementar dois workflows independentes, um por repositorio, com gatilhos em
`push` e `pull_request` para `main`.

Motivo:

- backend e frontend sao repositorios separados;
- cada repositorio deve conseguir provar sua propria saude sem depender do outro;
- falhas ficam mais faceis de diagnosticar;
- deploy real ainda depende de decisoes de infraestrutura e homologacao de campo.

## Referencias Tecnicas

- `actions/setup-node` oferece cache integrado para gerenciadores como npm.
- `shivammathur/setup-php` suporta configurar PHP 8.4 e Composer no GitHub Actions.
- GitHub Actions permite cache de dependencias por chave baseada nos lockfiles.

## Backend CI

Arquivo esperado:

- `.github/workflows/backend-ci.yml`

Gatilhos:

- `push` para `main`;
- `pull_request` para `main`.

Ambiente:

- `ubuntu-latest`;
- PHP `8.4`;
- Composer v2;
- coverage desabilitado.

Passos:

1. checkout do repositorio;
2. setup PHP com extensoes necessarias para Laravel e SQLite de teste;
3. cache do Composer usando `composer.lock`;
4. `composer validate --strict`;
5. `composer install --prefer-dist --no-progress --no-interaction`;
6. preparar `.env` a partir de `.env.example`;
7. gerar `APP_KEY`;
8. rodar `vendor/bin/pint --test`;
9. rodar `php artisan test`.

Observacao: antes de ativar Pint no CI, a implementacao deve corrigir qualquer
desvio de estilo ja existente no backend.

## Frontend CI

Arquivo esperado:

- `.github/workflows/frontend-ci.yml`

Gatilhos:

- `push` para `main`;
- `pull_request` para `main`.

Ambiente:

- `ubuntu-latest`;
- Node.js `24`;
- npm via `package-lock.json`.

Passos:

1. checkout do repositorio;
2. setup Node com cache npm baseado em `package-lock.json`;
3. `npm ci`;
4. `npm run lint`;
5. `npm test -- --run`;
6. `npm run build`.

## Seguranca

Os workflows devem usar:

```yaml
permissions:
  contents: read
```

Nenhum workflow desta fase deve exigir secrets. Se algum passo precisar de segredo,
isso indica que a necessidade pertence a uma fase futura de deploy/homologacao.

## Tratamento De Falhas

Falha esperada deve parar o job com status vermelho:

- dependencia nao instala;
- Composer invalido;
- Pint encontra divergencia;
- teste Laravel falha;
- ESLint falha;
- Vitest falha;
- build Vite/TypeScript falha.

Os workflows nao devem mascarar falhas com `continue-on-error`.

## Documentacao

Backend:

- atualizar `README.md` com o resumo do workflow e comandos locais equivalentes.

Frontend:

- atualizar `README.md` com o resumo do workflow e comandos locais equivalentes.

AgentOps:

- manter esta spec e criar plano de implementacao antes de codar.

## Criterios De Aceite

- Backend possui workflow CI versionado.
- Frontend possui workflow CI versionado.
- Backend CI executa Composer validate, Pint e testes Laravel.
- Frontend CI executa ESLint, Vitest e build.
- Workflows rodam em `push` e `pull_request` para `main`.
- Workflows usam permissoes minimas (`contents: read`).
- Nenhum secret e necessario nesta fase.
- Comandos equivalentes passam localmente antes do push.
