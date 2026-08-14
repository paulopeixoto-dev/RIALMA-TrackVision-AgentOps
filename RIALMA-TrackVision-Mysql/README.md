# RIALMA TrackVision MySQL

Ambiente local de MySQL para desenvolvimento do TrackVision.

## Comandos

```bash
docker compose --env-file .env up -d
docker compose --env-file .env ps
docker compose --env-file .env logs mysql
docker compose --env-file .env down
```

## Dados Locais

- Host: `127.0.0.1`
- Porta: `3307`
- Banco: `trackvision`
- Usuario: `trackvision`

As senhas locais ficam em `.env`, que e ignorado pelo Git.
