
# Установка PostgreSQL 18.0 из tar-архива
Инструкция описывает установку PostgreSQL версии 18.0 из официального tar-архива

!!! tip "Как пользоваться инструкцией"
    Команды можно копировать и выполнять по порядку сверху вниз. :slightly_smiling_face:

!!! info "Особенности установки"
    - PostgreSQL устанавливается в отдельную директорию `/app/postgresql/18.0`.
    - Такая схема позволяет устанавливать рядом несколько версий PostgreSQL.

## Назначение схемы установки

При установке PostgreSQL в отдельную версионную директорию можно держать на одном сервере несколько независимых экземпляров PostgreSQL разных версий.

Например:

```text
/app/postgresql/16.0
/app/postgresql/17.0
/app/postgresql/18.0
```

!!! warning "Важно"
    У каждого экземпляра должен быть собственный `port`, а также отдельный `systemd` unit.

## 1. Установка зависимостей

```bash
dnf install wget tar gcc zlib-devel readline-devel make libicu libicu-devel bison flex perl docbook-dtds docbook-style-xsl openssl lz4 lz4-devel openssl-devel libxml2-devel libxslt libxslt-devel python3-devel bzip2 systemd-devel -y
```

## 2. Создание пользователя postgres

```bash
useradd -m -s /bin/bash postgres
passwd postgres
```

## 3. Создание директорий

```bash
mkdir -p /app/postgresql/18.0
mkdir -p /app/postgresql/18.0/pg_home
mkdir -p /app/postgresql/18.0/pg_build
mkdir -p /app/postgresql/18.0/pg_dbcluster_1/data
mkdir -p /app/postgresql/18.0/pg_dbcluster_1/log
mkdir -p /app/postgresql/18.0/pg_wal_archive
```

## 4. Назначение прав на директории

```bash
chown -R postgres:postgres /app/postgresql
```

## 5. Переключение на пользователя postgres

```bash
su - postgres
```

## 6. Скачивание архива PostgreSQL

```bash
cd /app/postgresql/18.0
wget https://ftp.postgresql.org/pub/source/v18.0/postgresql-18.0.tar.gz
tar -xvf postgresql-18.0.tar.gz -C /app/postgresql/18.0/pg_build --strip-components=1
```

## 7. Сборка PostgreSQL

```bash
cd /app/postgresql/18.0/pg_build
./configure --prefix=/app/postgresql/18.0/pg_home --enable-debug --with-python --with-lz4 --with-openssl --with-libxml --with-libxslt --with-systemd --with-icu
make -j$(nproc) world-bin
make check-world
make install-world-bin
```

## 8. Инициализация кластера

```bash
/app/postgresql/18.0/pg_home/bin/initdb -D /app/postgresql/18.0/pg_dbcluster_1/data/ -k -U postgres -W --auth-local=scram-sha-256 --auth-host=scram-sha-256
```

## 9. Настройка основного конфигурационного файла postgresql.conf

```bash
vim /app/postgresql/18.0/pg_dbcluster_1/data/postgresql.conf
```

В конец файла `postgresql.conf` добавляем подключение дополнительного конфигурационного файла:

```ini
include_if_exists = 'db_conf.conf'
```

## 10. Создание дополнительного конфигурационного файла db_conf.conf

```bash
vim /app/postgresql/18.0/pg_dbcluster_1/data/db_conf.conf
```

```ini
port = 5432
listen_addresses = '*'
logging_collector = on
log_directory = '/app/postgresql/18.0/pg_dbcluster_1/log'
log_filename = '%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_truncate_on_rotation = on
log_error_verbosity = VERBOSE
log_line_prefix = '%m [%p] db=%d,user=%u,app=%a,client=%h '
cluster_name = 'cluster_1'
unix_socket_directories = '/tmp'
```

## 11. Настройка файла клиентской аутентификации pg_hba.conf

```bash
vim /app/postgresql/18.0/pg_dbcluster_1/data/pg_hba.conf
```

```ini
local all all scram-sha-256
host all all 127.0.0.1/32 scram-sha-256
host all all ::1/128 scram-sha-256
host all all 0.0.0.0/0 scram-sha-256
```

## 12. Создание файла переменных окружения `env`

```bash
vim /home/postgres/pgsql_18_0.env
```

```bash
export PG_VERSION=18.0
export PG_BASE=/app/postgresql/$PG_VERSION
export PG_HOME=$PG_BASE/pg_home

export PGDATA=$PG_BASE/pg_dbcluster_1/data
export PG_LOG=$PG_BASE/pg_dbcluster_1/log
export PG_WAL_ARCHIVE=$PG_BASE/pg_wal_archive

export PGUSER=postgres
export PGPASSWORD='Qwerty123!'
export PGDATABASE=postgres
export PGPORT=5432
export PGHOST=/tmp

export LD_LIBRARY_PATH=$PG_HOME/lib

case ":$PATH:" in
  *":$PG_HOME/bin:"*) ;;
  *) export PATH="$PG_HOME/bin:$PATH" ;;
esac
```

`PG_VERSION`, `PG_BASE`, `PG_HOME`, `PG_LOG`, `PG_WAL_ARCHIVE` — пользовательские переменные для удобства.
`PGDATA`, `PGUSER`, `PGDATABASE`, `PGPORT`, `PGHOST` - используются PostgreSQL-клиентами и утилитами.

## 13. Ручной запуск кластера и проверка

```bash
source /home/postgres/pgsql_18_0.env

pg_ctl -D "$PGDATA" start
pg_ctl -D "$PGDATA" status
pg_isready -p "$PGPORT"
psql -p "$PGPORT" -U "$PGUSER" -d "$PGDATABASE" -c "select version();"

pg_ctl -D "$PGDATA" stop -m fast
```

## 14. Создание systemd unit-файла

Создание unit-файла выполняется из под `root`.

```bash
vim /etc/systemd/system/postgresql-18.service
```

```ini
[Unit]
Description=PostgreSQL 18.0
After=network.target

[Service]
Type=forking
User=postgres
Group=postgres

Environment=PGDATA=/app/postgresql/18.0/pg_dbcluster_1/data
Environment=PG_HOME=/app/postgresql/18.0/pg_home
Environment=LD_LIBRARY_PATH=/app/postgresql/18.0/pg_home/lib
Environment=PATH=/app/postgresql/18.0/pg_home/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin

ExecStart=/app/postgresql/18.0/pg_home/bin/pg_ctl start -D ${PGDATA} -s -w -t 300
ExecStop=/app/postgresql/18.0/pg_home/bin/pg_ctl stop -D ${PGDATA} -s -w -t 300 -m fast
ExecReload=/app/postgresql/18.0/pg_home/bin/pg_ctl reload -D ${PGDATA} -s

TimeoutSec=300
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Регистрируем unit-файл и включаем сервис в автозагрузку

```bash
systemctl daemon-reload
systemctl enable postgresql-18.service
```

Запускаем сервис

```bash
systemctl start postgresql-18.service
systemctl status postgresql-18.service
journalctl -u postgresql-18.service -n 100 --no-pager
```

## 15. Подключение к PostgreSQL

```bash
su - postgres
source /home/postgres/pgsql_18_0.env
psql -p "$PGPORT" -U "$PGUSER" -d "$PGDATABASE"
```