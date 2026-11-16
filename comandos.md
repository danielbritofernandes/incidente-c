# Comandos executados — Cenário C

Caminho A (Docker). Diretório de trabalho: `/home/daniel/Documentos/pos/lab1`.
Horários do host em America/Sao_Paulo (UTC−3). O relógio do PostgreSQL está em UTC.

## Preparação do ambiente

### 14:59:11 — conferir papéis

```bash
docker compose exec -T primary psql -U postgres -c "SELECT pg_is_in_recovery() AS primario_em_recovery, inet_server_port() AS porta;"
docker compose exec -T replica psql -U postgres -c "SELECT pg_is_in_recovery() AS replica_em_recovery, inet_server_port() AS porta;"
docker compose exec -T primary psql -U postgres -c "SELECT application_name, client_addr, state, sync_state, replay_lag FROM pg_stat_replication;"
```

Resultado: primário fora de recovery, réplica em recovery, replicação `streaming` / `async`.

### 14:59:12 — dados iniciais

```bash
docker compose exec -T primary psql -U postgres -v ON_ERROR_STOP=1 \
  -c "CREATE TABLE clientes (id serial PRIMARY KEY, nome text, email text);" \
  -c "INSERT INTO clientes (nome, email) SELECT 'cliente ' || g, 'c' || g || '@loja.com' FROM generate_series(1, 1000) g;" \
  -c "CREATE TABLE eventos (id bigserial PRIMARY KEY, ts timestamptz DEFAULT now());" \
  -c "SELECT count(*) AS clientes_primario FROM clientes;"
docker compose exec -T replica psql -U postgres -c "SELECT pg_is_in_recovery(), count(*) AS clientes FROM clientes;"
```

Resultado: 1000 clientes no primário e na réplica.

### 14:59:12 — a sentença do enunciado é válida (teste revertido)

```bash
docker compose exec -T primary psql -U postgres -v ON_ERROR_STOP=1 \
  -c "BEGIN; INSERT INTO eventos SELECT FROM generate_series(1, 2); ROLLBACK;"
```

Resultado: `INSERT 0 2` e `ROLLBACK`. As duas linhas não ficaram na tabela. O `nextval` da sequence não volta atrás: os ids 1 e 2 foram consumidos.

## Reprodução do split-brain

### 14:59:46 — T_promocao (T0)

```bash
date '+%Y-%m-%d %H:%M:%S %Z'    # 2026-11-16 14:59:46 -03
date +%s                        # 1794851986
docker compose exec -T primary psql -U postgres -tA -c "SELECT clock_timestamp();"
# 2026-11-16 17:59:46.564105+00
```

### 14:59:46 — promover a réplica com o primário ainda no ar

```bash
docker compose exec -T replica psql -U postgres -c "SELECT timeline_id, checkpoint_lsn FROM pg_control_checkpoint();"
docker compose exec -T primary psql -U postgres -c "SELECT timeline_id, checkpoint_lsn FROM pg_control_checkpoint();"
docker compose exec -T replica psql -U postgres -c "SELECT pg_promote();"
```

Resultado: `pg_promote = t`.

### 14:59:47 — confirmar os dois primários

```bash
docker compose exec -T replica psql -U postgres -c "SELECT pg_is_in_recovery() AS novo_primario_em_recovery, clock_timestamp();"
docker compose exec -T primary psql -U postgres -c "SELECT pg_is_in_recovery() AS primario_antigo_em_recovery, clock_timestamp();"
docker compose exec -T primary psql -U postgres -c "SELECT count(*) AS replicas_conectadas FROM pg_stat_replication;"
```

Resultado: `pg_is_in_recovery = f` nas portas 5443 e 5442. Nenhuma réplica conectada ao primário antigo.

### 14:59:48 — escritas divergentes

```bash
docker compose exec -T primary psql -U postgres -c "INSERT INTO eventos SELECT FROM generate_series(1, 10);"
docker compose exec -T replica psql -U postgres -c "INSERT INTO eventos SELECT FROM generate_series(1, 7);"
```

Resultado: `INSERT 0 10` no primário antigo e `INSERT 0 7` no novo primário.

### 14:59:48 — comparar os dois lados

```bash
docker compose exec -T primary psql -U postgres -c "SELECT 'primario_antigo_5442' AS lado, max(id), count(*) FROM eventos;"
docker compose exec -T replica psql -U postgres -c "SELECT 'novo_primario_5443' AS lado, max(id), count(*) FROM eventos;"
docker compose exec -T primary psql -U postgres -c "SELECT id, ts FROM eventos ORDER BY id;"
docker compose exec -T replica psql -U postgres -c "SELECT id, ts FROM eventos ORDER BY id;"
```

Resultado: antigo `max(id)=12`, `count=10` (ids 3 a 12). Novo `max(id)=40`, `count=7` (ids 34 a 40). Nenhum id em comum.

### 14:59:49 — amostra das linhas posteriores a T_promocao

```bash
docker compose exec -T primary psql -U postgres -c "SELECT count(*) AS divergentes_antigo FROM eventos WHERE ts > '2026-11-16 17:59:46.564105+00';"
docker compose exec -T replica psql -U postgres -c "SELECT count(*) AS divergentes_novo FROM eventos WHERE ts > '2026-11-16 17:59:46.564105+00';"
docker compose exec -T primary psql -U postgres -c "SELECT * FROM eventos WHERE ts > '2026-11-16 17:59:46.564105+00' ORDER BY id LIMIT 5;"
docker compose exec -T replica psql -U postgres -c "SELECT * FROM eventos WHERE ts > '2026-11-16 17:59:46.564105+00' ORDER BY id LIMIT 5;"
```

Resultado: 10 linhas só no antigo, 7 só no novo.

### 14:59:49 — as mesmas contagens pelas portas publicadas

```bash
docker run --rm --network host -e PGPASSWORD=postgres postgres:16 \
  psql -h 127.0.0.1 -p 5442 -U postgres -d postgres \
  -c "SELECT '5442' AS porta, pg_is_in_recovery() AS em_recovery, count(*) AS eventos, max(id) AS max_id FROM eventos;"
docker run --rm --network host -e PGPASSWORD=postgres postgres:16 \
  psql -h 127.0.0.1 -p 5443 -U postgres -d postgres \
  -c "SELECT '5443' AS porta, pg_is_in_recovery() AS em_recovery, count(*) AS eventos, max(id) AS max_id FROM eventos;"
```

Resultado: 5442 recovery false, 10 eventos, max id 12. 5443 recovery false, 7 eventos, max id 40.

### 15:01:10 — sequence, timeline e exportação (ainda com os dois primários no ar)

```bash
docker compose exec -T primary psql -U postgres -c "SELECT last_value, log_cnt, is_called FROM eventos_id_seq;"
docker compose exec -T primary psql -U postgres -c "SELECT pg_walfile_name(pg_current_wal_lsn()) AS wal;"
docker compose exec -T replica psql -U postgres -c "SELECT last_value, log_cnt, is_called FROM eventos_id_seq;"
docker compose exec -T replica psql -U postgres -c "SELECT pg_walfile_name(pg_current_wal_lsn()) AS wal;"
docker compose exec -T primary psql -U postgres -c "\copy (SELECT id, ts FROM eventos ORDER BY id) TO STDOUT WITH CSV HEADER"
docker compose exec -T replica psql -U postgres -c "\copy (SELECT id, ts FROM eventos ORDER BY id) TO STDOUT WITH CSV HEADER"
```

Resultado: primário antigo na timeline 1 (`000000010000000000000003`), sequence em 12 com `log_cnt=21`. Novo primário na timeline 2 (`000000020000000000000003`), sequence em 40 com `log_cnt=26`. CSVs em `evidencias/`.

## Correção: isolar o primário antigo

### 15:02:28 — fencing

```bash
docker compose stop primary
docker compose ps
docker run --rm --network host -e PGPASSWORD=postgres postgres:16 \
  psql -h 127.0.0.1 -p 5442 -U postgres -d postgres -c "SELECT 1;"
```

Resultado: container `lab1-primary-1` parado. A porta 5442 recusou a conexão.

### 15:02:29 — o único primário aceita escrita (T1)

```bash
docker run --rm --network host -e PGPASSWORD=postgres postgres:16 \
  psql -h 127.0.0.1 -p 5443 -U postgres -d postgres \
  -c "SELECT pg_is_in_recovery() AS em_recovery, count(*) AS eventos, max(id) AS max_id FROM eventos;"
docker compose exec -T replica psql -U postgres -v ON_ERROR_STOP=1 \
  -c "CREATE TABLE prova_escrita (id int, ts timestamptz DEFAULT now());" \
  -c "INSERT INTO prova_escrita (id) VALUES (1) RETURNING id, ts;" \
  -c "SELECT count(*) AS clientes FROM clientes;"
date '+%Y-%m-%d %H:%M:%S %Z'    # 2026-11-16 15:02:29 -03
date +%s                        # 1794852149
```

Resultado: 5443 fora de recovery, 7 eventos, 1000 clientes, `INSERT` em `prova_escrita` confirmado. RTO = 1794852149 − 1794851986 = 163 s.
