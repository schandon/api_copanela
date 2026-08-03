# Visão Geral

Para utilização do `docker-compose.yml` faça a configuração do arquivo `.env` na raiz de `/infra/postgres` (copie de `.env.docker_example`).

Variáveis obrigatórias para o script `migrate-split-type-os.sh`: `POSTGRES_USERNAME`, `POSTGRES_PASSWORD`, `POSTGRES_DATABASE`, `POSTGRES_HOST` e `POSTGRES_LOCAL_PORT` (ou `DATABASE_URL` completa). Em desenvolvimento local com Docker, use `POSTGRES_HOST=127.0.0.1`; em staging/produção (`--no-docker`), defina o hostname ou IP do servidor PostgreSQL.

Para a versão em que o banco ainda não separou `os` de `type_os`, utilize `migrate-split-type-os.sh` — o script respeita exclusivamente as variáveis de `infra/postgres/.env`.

### Banco Local

```bash
./infra/postgres/migrate-split-type-os.sh
```

### Da raiz do repo

```bash
./infra/postgres/migrate-split-type-os.sh --dry-run
```

### Staging/produção (banco remoto)

```bash
./infra/postgres/migrate-split-type-os.sh --no-docker
```
