# Praktikum 6 & 7: Observability Terpadu (Centralized Logging + Resource Monitoring)

Praktikum ini menghubungkan **dua stack terpisah** yaitu Bab 6 (Centralized Logging dengan Fluent Bit & PostgreSQL) dan Bab 7 (Monitoring Resource dengan Prometheus, cAdvisor, Node Exporter & Grafana).

> **Inti:** *Monitoring* (Bab 7) memberi tahu **ada masalah**, *Logging* (Bab 6) membantu menemukan **penyebabnya**. Keduanya diintegrasikan lewat jaringan Docker agar log dari PostgreSQL Bab 6 bisa dibaca langsung oleh Grafana Bab 7.

## 1. Tujuan

Setelah menyelesaikan praktikum ini, mahasiswa diharapkan mampu:

- Menjalankan pipeline log `stdout/stderr` container ke Fluent Bit, lalu disimpan ke PostgreSQL (Bab 6)
- Menjalankan stack monitoring (Prometheus, Node Exporter, cAdvisor, Grafana) (Bab 7)
- Menghubungkan PostgreSQL Bab 6 ke jaringan monitoring Bab 7 menggunakan `scripts/connect-bab6.sh`
- Menambahkan PostgreSQL Bab 6 sebagai *data source* di Grafana untuk menampilkan log
- Melakukan korelasi waktu antara *metrics* (Prometheus) dan *log* (PostgreSQL) dalam satu dashboard
- Membuktikan mekanisme *buffering* Fluent Bit saat PostgreSQL tidak tersedia sementara
- Membedakan peran logging vs monitoring serta cara integrasinya

## 2. Struktur Folder

Sesuai arahan Cheatsheet, **kedua bab dibuat terpisah** di dalam direktori `docker-lab/`.

```text
~/docker-lab/
├── bab-6/        # Stack Logging (Fluent Bit + PostgreSQL + log-generator)
└── bab-7/        # Stack Monitoring (Prometheus + Node Exporter + cAdvisor + Grafana)
```

Kamu **tidak perlu** menggabungkan kedua `compose.yaml` menjadi satu. Yang digabung hanya **jaringan** (`bab7-monitoring-net`) dan **akses read-only** ke database Bab 6 melalui script `scripts/connect-bab6.sh`.

## 3. Persiapan Awal

Semua perintah dijalankan menggunakan Bash pada Linux/WSL 2.

```bash
mkdir -p ~/docker-lab
cd ~/docker-lab
```

## 4. Bab 6 – Centralized Logging dengan Fluent Bit dan PostgreSQL

Ikuti urutan **Kotak Perintah A – F** persis seperti di `cheatsheets/bab-06-cs.md`.

### Kotak Perintah A – Menyiapkan Direktori Bab 6

Buat struktur direktori Bab 6.

```bash
mkdir -p ~/docker-lab/bab-6/{init,fluent-bit,queries,scripts}
cd ~/docker-lab/bab-6
```

Struktur yang akan terbentuk:

```text
bab-6/
├── .env.example
├── .env
├── .gitignore
├── compose.yaml
├── fluent-bit/
│   └── fluent-bit.conf
├── init/
│   └── 01-init-logging.sh
├── queries/
│   └── analysis.sql
└── scripts/
    ├── generate-logs.sh
    └── test-buffering.sh
```

### Kotak Script 1 – File `.env.example`

> **Nama file:** `.env.example` | **Lokasi:** `~/docker-lab/bab-6/.env.example` | **Editor:** `nano .env.example`

```dotenv
POSTGRES_DB=labdb
POSTGRES_USER=labadmin
POSTGRES_PASSWORD=labadminpass123
FLUENTBIT_DB_USER=fluentbit_user
FLUENTBIT_DB_PASSWORD=fluentbitpass123
```

Salin menjadi `.env` dan atur permission.

```bash
cp .env.example .env
chmod 600 .env
```

> **Catatan:** File `.env` hanya untuk laboratorium. Jangan pernah commit ke repository.

### Kotak Script 2 – File `.gitignore`

> **Nama file:** `.gitignore` | **Lokasi:** `~/docker-lab/bab-6/.gitignore` | **Editor:** `nano .gitignore`

```gitignore
.env
*.log
reports/
```

### Kotak Script 3 – File `compose.yaml`

> **Nama file:** `compose.yaml` | **Lokasi:** `~/docker-lab/bab-6/compose.yaml` | **Editor:** `nano compose.yaml`

Gunakan spasi, **bukan tab**.

```yaml
services:
  postgres-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      FLUENTBIT_DB_USER: ${FLUENTBIT_DB_USER}
      FLUENTBIT_DB_PASSWORD: ${FLUENTBIT_DB_PASSWORD}
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    networks:
      - logging-net
    healthcheck:
      test:
        - CMD-SHELL
        - "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"
      interval: 5s
      timeout: 5s
      retries: 12
      start_period: 10s
    restart: unless-stopped

  fluent-bit:
    image: fluent/fluent-bit:4.2.8
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      FLUENTBIT_DB_USER: ${FLUENTBIT_DB_USER}
      FLUENTBIT_DB_PASSWORD: ${FLUENTBIT_DB_PASSWORD}
    volumes:
      - ./fluent-bit/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
      - flb-storage:/var/log/flb-storage
    ports:
      - "127.0.0.1:24224:24224"
      - "127.0.0.1:2020:2020"
    networks:
      - logging-net
    depends_on:
      postgres-db:
        condition: service_healthy
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  generator-auth:
    image: alpine:3.20
    command:
      - /bin/sh
      - -c
      - |
        i=1
        while [ "$$i" -le 20 ]; do
          printf '{"service":"auth","severity":"INFO","event":"login_test","sequence":%s}\n' "$$i"
          i=$$((i + 1))
          sleep 1
        done
    logging:
      driver: fluentd
      options:
        fluentd-address: "127.0.0.1:24224"
        fluentd-async: "true"
        tag: "docker.{{.Name}}"
    depends_on:
      - fluent-bit

  generator-api:
    image: alpine:3.20
    command:
      - /bin/sh
      - -c
      - |
        i=1
        while [ "$$i" -le 20 ]; do
          printf '{"service":"api","severity":"WARN","event":"latency_test","duration_ms":%s}\n' "$$((i * 10))"
          i=$$((i + 1))
          sleep 1
        done
    logging:
      driver: fluentd
      options:
        fluentd-address: "127.0.0.1:24224"
        fluentd-async: "true"
        tag: "docker.{{.Name}}"
    depends_on:
      - fluent-bit

  generator-worker:
    image: alpine:3.20
    command:
      - /bin/sh
      - -c
      - |
        i=1
        while [ "$$i" -le 20 ]; do
          printf '{"service":"worker","severity":"ERROR","event":"retry_test","attempt":%s}\n' "$$i"
          i=$$((i + 1))
          sleep 1
        done
    logging:
      driver: fluentd
      options:
        fluentd-address: "127.0.0.1:24224"
        fluentd-async: "true"
        tag: "docker.{{.Name}}"
    depends_on:
      - fluent-bit

volumes:
  pg-data:
  flb-storage:

networks:
  logging-net:
    driver: bridge
```

> **Penjelasan:** `fluentd-address` diarahkan ke `127.0.0.1:24224` karena Docker Engine yang menjalankan logging driver, bukan container generator. `fluentd-async: "true"` mencegah generator gagal startup jika collector belum siap.

### Kotak Script 4 – File `init/01-init-logging.sh`

> **Nama file:** `01-init-logging.sh` | **Lokasi:** `~/docker-lab/bab-6/init/01-init-logging.sh` | **Editor:** `nano init/01-init-logging.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

: "${POSTGRES_USER:?POSTGRES_USER belum diatur}"
: "${POSTGRES_DB:?POSTGRES_DB belum diatur}"
: "${FLUENTBIT_DB_USER:?FLUENTBIT_DB_USER belum diatur}"
: "${FLUENTBIT_DB_PASSWORD:?FLUENTBIT_DB_PASSWORD belum diatur}"

psql \
  --set=ON_ERROR_STOP=1 \
  --username "$POSTGRES_USER" \
  --dbname "$POSTGRES_DB" \
  --set=fb_user="$FLUENTBIT_DB_USER" \
  --set=fb_password="$FLUENTBIT_DB_PASSWORD" <<'SQL'
SELECT format(
    'CREATE ROLE %I LOGIN PASSWORD %L',
    :'fb_user',
    :'fb_password'
)
WHERE NOT EXISTS (
    SELECT 1 FROM pg_roles WHERE rolname = :'fb_user'
)
\gexec

CREATE SCHEMA IF NOT EXISTS logs;

CREATE TABLE IF NOT EXISTS logs.fluentbit (
    tag  TEXT NOT NULL,
    time TIMESTAMP WITHOUT TIME ZONE NOT NULL,
    data JSONB NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_fluentbit_time
    ON logs.fluentbit (time DESC);

CREATE INDEX IF NOT EXISTS idx_fluentbit_tag
    ON logs.fluentbit (tag);

CREATE INDEX IF NOT EXISTS idx_fluentbit_data_gin
    ON logs.fluentbit USING GIN (data);

CREATE OR REPLACE VIEW logs.recent_logs AS
SELECT
    time,
    tag,
    replace(data ->> 'container_name', '/', '') AS container_name,
    data ->> 'source' AS source,
    left(data ->> 'log', 200) AS log_preview
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 100;

SELECT format(
    'ALTER ROLE %I IN DATABASE %I SET search_path TO logs, public',
    :'fb_user',
    current_database()
)
\gexec

SELECT format('GRANT USAGE, CREATE ON SCHEMA logs TO %I', :'fb_user')
\gexec

SELECT format(
    'GRANT SELECT, INSERT ON TABLE logs.fluentbit TO %I',
    :'fb_user'
)
\gexec

SELECT format('GRANT SELECT ON TABLE logs.recent_logs TO %I', :'fb_user')
\gexec
SQL
```

### Kotak Script 5 – File `fluent-bit/fluent-bit.conf`

> **Nama file:** `fluent-bit.conf` | **Lokasi:** `~/docker-lab/bab-6/fluent-bit/fluent-bit.conf` | **Editor:** `nano fluent-bit/fluent-bit.conf`

```ini
[SERVICE]
    Flush                     1
    Daemon                    Off
    Log_Level                 info
    storage.path              /var/log/flb-storage
    storage.sync              normal
    storage.checksum          on
    storage.backlog.mem_limit 10M
    HTTP_Server               On
    HTTP_Listen               0.0.0.0
    HTTP_Port                 2020

[INPUT]
    Name          forward
    Listen        0.0.0.0
    Port          24224
    Mem_Buf_Limit 10M
    storage.type  filesystem

[OUTPUT]
    Name                     pgsql
    Match                    *
    Host                     postgres-db
    Port                     5432
    User                     ${FLUENTBIT_DB_USER}
    Password                 ${FLUENTBIT_DB_PASSWORD}
    Database                 ${POSTGRES_DB}
    Table                    fluentbit
    storage.total_limit_size 256M

[OUTPUT]
    Name   stdout
    Match  *
    Format json_lines
```

> **Penting:** Sesuai koreksi cheatsheet Bab 6, plugin PostgreSQL Fluent Bit 4.2.x **tidak** memakai `Schema` maupun `Timestamp_Key`. Schema `logs` diatur lewat `search_path` role `fluentbit_user`.

### Kotak Script 6 – File `queries/analysis.sql`

> **Nama file:** `analysis.sql` | **Lokasi:** `~/docker-lab/bab-6/queries/analysis.sql` | **Editor:** `nano queries/analysis.sql`

```sql
\pset pager off
\timing on

-- 1. Jumlah event dan rentang waktu
SELECT
    count(*) AS total_event,
    min(time) AS event_pertama,
    max(time) AS event_terakhir
FROM logs.fluentbit;

-- 2. Distribusi berdasarkan tag Docker
SELECT
    tag,
    count(*) AS total
FROM logs.fluentbit
GROUP BY tag
ORDER BY total DESC, tag;

-- 3. Metadata terstruktur dari Docker logging driver
SELECT
    time,
    tag,
    replace(data ->> 'container_name', '/', '') AS container_name,
    data ->> 'source' AS source,
    left(data ->> 'log', 160) AS log_preview
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 15;

-- 4. Pencarian severity ERROR pada field log
SELECT
    time,
    tag,
    data ->> 'log' AS log_message
FROM logs.fluentbit
WHERE data ->> 'log' ILIKE '%"severity":"ERROR"%'
ORDER BY time DESC;

-- 5. Ringkasan per menit
SELECT
    date_trunc('minute', time) AS minute_bucket,
    count(*) AS total
FROM logs.fluentbit
GROUP BY minute_bucket
ORDER BY minute_bucket DESC
LIMIT 10;

-- Contoh retensi (JANGAN dijalankan sembarangan)
-- DELETE FROM logs.fluentbit WHERE time < CURRENT_TIMESTAMP - INTERVAL '7 days';
```

### Kotak Script 7 – File `scripts/generate-logs.sh`

> **Nama file:** `generate-logs.sh` | **Lokasi:** `~/docker-lab/bab-6/scripts/generate-logs.sh` | **Editor:** `nano scripts/generate-logs.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

docker compose up -d postgres-db fluent-bit

for attempt in {1..30}; do
    if curl -fsS http://127.0.0.1:2020/api/v1/health >/dev/null; then
        break
    fi

    if [[ "$attempt" -eq 30 ]]; then
        echo "[FAIL] Endpoint kesehatan Fluent Bit belum siap." >&2
        exit 1
    fi
    sleep 2
done

docker compose up -d --force-recreate \
    generator-auth \
    generator-api \
    generator-worker

echo "[PASS] Tiga generator telah dijalankan. Tunggu sekitar 25 detik."
echo "[INFO] Pantau dengan: docker compose ps"
```

### Kotak Script 8 – File `scripts/test-buffering.sh`

> **Nama file:** `test-buffering.sh` | **Lokasi:** `~/docker-lab/bab-6/scripts/test-buffering.sh` | **Editor:** `nano scripts/test-buffering.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

count_rows() {
    docker compose exec -T postgres-db \
        psql \
        --username "$POSTGRES_USER" \
        --dbname "$POSTGRES_DB" \
        --tuples-only \
        --no-align \
        --command 'SELECT count(*) FROM logs.fluentbit;'
}

before="$(count_rows)"
echo "[INFO] Jumlah awal: $before"

docker compose stop postgres-db

docker compose up -d --force-recreate \
    generator-auth \
    generator-api \
    generator-worker

echo "[INFO] PostgreSQL berhenti; generator berjalan selama 25 detik."
sleep 25

docker compose start postgres-db

for attempt in {1..30}; do
    if docker compose exec -T postgres-db \
        pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB" >/dev/null 2>&1; then
        break
    fi

    if [[ "$attempt" -eq 30 ]]; then
        echo "[FAIL] PostgreSQL tidak kembali sehat." >&2
        exit 1
    fi
    sleep 2
done

for attempt in {1..60}; do
    after="$(count_rows)"
    if (( after > before )); then
        echo "[PASS] Jumlah sesudah pemulihan: $after"
        echo "[INFO] Kenaikan record: $((after - before))"
        exit 0
    fi
    sleep 2
done

echo "[FAIL] Tidak ada record baru setelah PostgreSQL pulih." >&2
echo "[INFO] Periksa: docker compose logs --tail 200 fluent-bit" >&2
exit 1
```

### Kotak Perintah B – Memberi Izin Eksekusi dan Memvalidasi Berkas

Pastikan kamu berada di `~/docker-lab/bab-6/`

```bash
chmod +x init/01-init-logging.sh
chmod +x scripts/generate-logs.sh scripts/test-buffering.sh

bash -n init/01-init-logging.sh
bash -n scripts/generate-logs.sh
bash -n scripts/test-buffering.sh

docker compose config --quiet
```

Validasi konfigurasi Fluent Bit:

```bash
docker compose run --rm --no-deps fluent-bit \
  --dry-run \
  --config=/fluent-bit/etc/fluent-bit.conf
```

Jika `--dry-run` tidak dikenali oleh image, cukup jalankan dan cek log:

```bash
docker compose up -d postgres-db fluent-bit
docker compose logs --tail 100 fluent-bit
```

### Kotak Perintah C – Menjalankan Pipeline dan Menghasilkan Log

```bash
./scripts/generate-logs.sh
sleep 25
docker compose ps --all
```

> **Catatan:** Generator akan `Exited (0)` setelah ~20 detik, ini **normal** karena prosesnya selesai. `postgres-db` dan `fluent-bit` harus tetap `Up`.

### Kotak Perintah D – Memeriksa Monitoring Fluent Bit

```bash
curl -fsS http://127.0.0.1:2020/api/v1/health
curl -fsS http://127.0.0.1:2020/api/v1/metrics
```

> **Penting:** Endpoint ini hanya bind ke `127.0.0.1`. Jangan publikasikan ke jaringan publik.

### Kotak Perintah E – Memverifikasi Record PostgreSQL

```bash
set -a
source .env
set +a

docker compose exec -T postgres-db \
  psql \
  --username "$POSTGRES_USER" \
  --dbname "$POSTGRES_DB" \
  --file /dev/stdin \
  < queries/analysis.sql
```

Cek view `logs.recent_logs`:

```bash
docker compose exec -T postgres-db \
  psql \
  --username "$POSTGRES_USER" \
  --dbname "$POSTGRES_DB" \
  --command 'TABLE logs.recent_logs;'
```

Verifikasi tipe data `JSONB`:

```bash
docker compose exec -T postgres-db \
  psql \
  --username "$POSTGRES_USER" \
  --dbname "$POSTGRES_DB" \
  --command "SELECT pg_typeof(data), count(*) FROM logs.fluentbit GROUP BY pg_typeof(data);"
```

### Kotak Perintah F – Menguji Buffering dan Pemulihan

Pastikan pipeline sudah menghasilkan data minimal dari Kotak Perintah C, lalu jalankan uji ini:

```bash
./scripts/test-buffering.sh
```

Pantau selama pengujian:

```bash
docker compose logs --tail 200 fluent-bit
docker compose volume ls
```

**Kriteria PASS:** Script menampilkan `[PASS] Jumlah sesudah pemulihan: ...` dengan kenaikan jumlah record. Artinya Fluent Bit berhasil menahan log ke *filesystem buffer* saat PostgreSQL dihentikan, lalu mengirimkannya kembali setelah PostgreSQL pulih.

## 5. Bab 7 – Monitoring Resource dengan Prometheus, cAdvisor, Node Exporter, dan Grafana

Setelah Bab 6 berjalan dan memiliki data di `logs.fluentbit`, lanjutkan ke Bab 7. **Jalankan di terminal/ direktori terpisah**.

### Persiapan Direktori Bab 7

```bash
mkdir -p ~/docker-lab/bab-7/{prometheus/rules,prometheus/tests,grafana/provisioning/datasources,grafana/provisioning/dashboards,grafana/dashboards,scripts,reports}
cd ~/docker-lab/bab-7
```

### Buat File Konfigurasi Bab 7

Buat file-file berikut persis seperti di `cheatsheets/bab-07-cs.md`:

- `.env.example` → copy ke `.env` dan `chmod 600 .env`
- `.gitignore`
- `compose.yaml`
- `prometheus/prometheus.yml`
- `prometheus/rules/recording-rules.yml`
- `prometheus/rules/alert-rules.yml`
- `prometheus/tests/rules.test.yml`
- `grafana/provisioning/datasources/datasources.yml`
- `grafana/provisioning/dashboards/provider.yml`
- `grafana/dashboards/bab-7-overview.json`
- `scripts/verify-stack.sh`
- `scripts/connect-bab6.sh`

Beri izin eksekusi:

```bash
chmod +x scripts/verify-stack.sh scripts/connect-bab6.sh
bash -n scripts/verify-stack.sh scripts/connect-bab6.sh
```

Validasi konfigurasi:

```bash
docker compose config --quiet
python3 -m json.tool grafana/dashboards/bab-7-overview.json >/dev/null
```

Validasi Prometheus config & rule test:

```bash
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus check config /etc/prometheus/prometheus.yml
docker compose run --rm --no-deps --entrypoint /bin/promtool prometheus test rules /etc/prometheus/tests/rules.test.yml
```

### Menjalankan Stack Bab 7

```bash
docker compose pull
docker compose up -d
docker compose ps
./scripts/verify-stack.sh
```

Akses:

- **Prometheus:** `http://127.0.0.1:9090`
- **Grafana:** `http://127.0.0.1:3000` (login sesuai `.env` Bab 7)

Coba beberapa query dasar di `Prometheus > Graph`:

```promql
up
instance:node_cpu_utilisation:ratio_rate5m * 100
instance:node_memory_utilisation:ratio * 100
topk(5, container:cpu_usage_cores:rate1m)
topk(5, container:memory_working_set_bytes:sum)
```

## 6. Integrasi Bab 6 – Bab 7

**PENTING:** Kedua stack harus dalam keadaan **running** (`bab-6/` dan `bab-7/`).

Jalankan dari **direktori Bab 7**:

```bash
cd ~/docker-lab/bab-7
./scripts/connect-bab6.sh
```

**Apa yang dilakukan script ini:**

1. Mencari container `postgres-db` milik stack Bab 6
2. Menghubungkan container tersebut ke jaringan `bab7-monitoring-net` dengan alias `postgres-db`
3. Membuat role read-only `grafana_reader` di PostgreSQL Bab 6 dengan hanya hak `SELECT` pada schema `logs`
4. Merestart Grafana agar datasource `PostgreSQL Logs Bab 6` bisa terhubung

Jika berhasil, akan muncul:

```text
[PASS] PostgreSQL Bab 6 terhubung dengan role read-only.
```

Buka Grafana → `Connections > Data sources`. Pastikan `Prometheus Bab 7` dan `PostgreSQL Logs Bab 6` berstatus **OK / Saved & tested**.

## 7. Korelasi Logs + Metrics di Grafana (Untuk Laporan)

Tujuannya adalah menunjukkan hubungan: **Metrics (Bab 7) mendeteksi ada masalah, Logs (Bab 6) menjelaskan penyebabnya**.

### Langkah Korelasi

1. Buka `http://127.0.0.1:3000` → Masuk ke dashboard `DevSecOps Lab > Bab 7 — Resource Monitoring`. Amati kondisi normal.
2. Buat beban untuk memicu *spike* (dari direktori Bab 7):

```bash
cd ~/docker-lab/bab-7
docker compose --profile load up -d load-generator
```

3. Tunggu 1–2 menit lalu amati kenaikan pada panel `Utilisasi CPU Host` / `Top 5 CPU Container`.
4. Buka `Grafana > Explore` → Pilih datasource **PostgreSQL Logs Bab 6**. Gunakan query berikut:

```sql
SELECT
    time AS "time",
    tag,
    replace(data ->> 'container_name', '/', '') AS container_name,
    data ->> 'source' AS source,
    data ->> 'log' AS log_message
FROM logs.fluentbit
WHERE $__timeFilter(time)
ORDER BY time DESC
LIMIT 200;
```

5. Atur *Time Range* di Explore sesuai periode saat beban berjalan (mis. `Last 10 minutes`). Perhatikan log yang muncul bersamaan dengan waktu *spike* di grafik CPU.
6. Hentikan beban:

```bash
docker compose --profile load stop load-generator
docker compose --profile load rm -f load-generator
```

**Ambil screenshot untuk bukti:** Grafik *spike* CPU (Prometheus/Grafana) + tabel log hasil query PostgreSQL di rentang waktu yang **sama**.

## 8. Tabel Verifikasi Bab 6 (PASS/FAIL)

| No | Skenario | Evidence | Status |
|---|---|---|---|
| 1 | Validasi Compose | `docker compose config --quiet` return code 0 | [ ] PASS / [ ] FAIL |
| 2 | PostgreSQL Healthy | `docker compose ps` menampilkan `postgres-db (healthy)` | [ ] PASS / [ ] FAIL |
| 3 | Fluent Bit Health | `curl http://127.0.0.1:2020/api/v1/health` sukses | [ ] PASS / [ ] FAIL |
| 4 | Tiga generator berjalan | Query `GROUP BY tag` menampilkan 3 tag (`docker.bab-6-generator-*`) | [ ] PASS / [ ] FAIL |
| 5 | Data tersimpan JSONB | `pg_typeof(data)` = `jsonb` | [ ] PASS / [ ] FAIL |
| 6 | View berfungsi | `TABLE logs.recent_logs;` menampilkan data | [ ] PASS / [ ] FAIL |
| 7 | Buffering & Recovery | `./scripts/test-buffering.sh` menampilkan `[PASS]` dengan kenaikan record | [ ] PASS / [ ] FAIL |
| 8 | Port dibatasi | PostgreSQL **tidak** dipublish ke host (hanya internal network) | [ ] PASS / [ ] FAIL |

## 9. Tabel Verifikasi Bab 7 + Integrasi (PASS/FAIL)

| No | Skenario | Evidence | Status |
|---|---|---|---|
| 1 | Validasi Rule | `promtool check config` & `promtool test rules` `SUCCESS` | [ ] PASS / [ ] FAIL |
| 2 | Stack Running | `docker compose ps` semua service `Up` | [ ] PASS / [ ] FAIL |
| 3 | Target UP | `http://127.0.0.1:9090/targets` semua berstatus `UP` | [ ] PASS / [ ] FAIL |
| 4 | Metrics Ada | Query PromQL menghasilkan nilai | [ ] PASS / [ ] FAIL |
| 5 | Grafana Provisioning | Dashboard `Bab 7 — Resource Monitoring` muncul otomatis | [ ] PASS / [ ] FAIL |
| 6 | Integrasi Berhasil | `./scripts/connect-bab6.sh` menampilkan `[PASS]` | [ ] PASS / [ ] FAIL |
| 7 | Datasource PostgreSQL OK | `PostgreSQL Logs Bab 6` status `Saved & tested` | [ ] PASS / [ ] FAIL |
| 8 | Korelasi Terbukti | Screenshot *spike* metrics + log di rentang waktu yang sama | [ ] PASS / [ ] FAIL |

## 10. Troubleshooting Singkat

| Gejala | Kemungkinan Penyebab | Solusi |
|---|---|---|
| Port `24224`/`2020`/`9090`/`3000` bentrok | Stack lain masih berjalan | Hentikan stack lama atau ganti port sesuai kebutuhan |
| Fluent Bit koneksi gagal ke postgres-db | Belum healthy / kredensial beda | Tunggu `postgres-db` `healthy`, cek `.env` Bab 6 |
| Generator tidak muncul di log | `fluentd-address` salah / collector belum ready | Pastikan pakai `127.0.0.1:24224`, jalankan `./scripts/generate-logs.sh` lagi |
| Target Prometheus DOWN | Nama service salah di `prometheus.yml` | Sesuai cheatsheet Bab 7, pakai nama service internal |
| `connect-bab6.sh` gagal | Jaringan `bab7-monitoring-net` belum ada / container Bab 6 tidak ditemukan | Jalankan Bab 7 dulu (`docker compose up -d`), pastikan `postgres-db` Bab 6 running |
| PostgreSQL datasource gagal di Grafana | Belum terhubung / role salah | Jalankan ulang `./scripts/connect-bab6.sh` dari `bab-7/`, restart Grafana |
| cAdvisor minim metrik (WSL2/Windows) | Batasan Docker Desktop VM | Wajar untuk lab ini, catat di laporan sesuai catatan cheatsheet |

## 11. Cleanup

Hentikan per stack secara terpisah. **Jangan hapus volume** kecuali benar-benar ingin reset total.

```bash
# Stop Bab 7
cd ~/docker-lab/bab-7
docker compose --profile load down --remove-orphans

# Stop Bab 6
cd ~/docker-lab/bab-6
docker compose down --remove-orphans
```

Jika ingin **reset TOTAL** (akan menghapus semua data log, buffer Fluent Bit, metrics Prometheus & data Grafana):

```bash
cd ~/docker-lab/bab-7
docker compose --profile load down --volumes --remove-orphans

cd ~/docker-lab/bab-6
docker compose down --volumes --remove-orphans
```

## 12. Checklist Laporan

- [ ] **Bab 6 A-F**: Screenshot setiap tahap (persiapan direktori, buat file, `generate-logs.sh`, cek health Fluent Bit, verifikasi PostgreSQL, `test-buffering.sh`)
- [ ] **Bukti Buffering**: Screenshot output `./scripts/test-buffering.sh` yang menunjukkan `[PASS]` + kenaikan jumlah record
- [ ] **Bab 7**: Screenshot `docker compose ps`, Prometheus Targets (semua UP), dashboard `Bab 7 — Resource Monitoring`
- [ ] **Integrasi**: Screenshot jalannya `./scripts/connect-bab6.sh` `[PASS]` dan datasource `PostgreSQL Logs Bab 6` berstatus OK
- [ ] **Korelasi**: Screenshot **grafik spike CPU/Memory** bersamaan dengan **tabel log dari PostgreSQL Logs Bab 6** pada rentang waktu yang sama (`$__timeFilter(time)`)
- [ ] **Analisis**: Jelaskan perbedaan *logging* vs *monitoring*, kenapa Bab 6–Bab 7 tetap terpisah tapi bisa terhubung, serta apa yang bisa disimpulkan dari hasil korelasi
- [ ] **Troubleshooting**: Catat kendala yang muncul (jika ada) beserta cara mengatasinya
