
# Установка MySQL 8.0.46 из tar-архива
Инструкция описывает установку MySQL версии 8.0.46 из официального tar-архива

!!! tip "Как пользоваться инструкцией"
    Команды можно копировать и выполнять по порядку сверху вниз. :slightly_smiling_face:

!!! info "Особенности установки"
    - MySQL устанавливается в отдельную директорию `/app/mysql/8.0.46`.
    - Такая схема позволяет устанавливать рядом несколько версий MySQL.
    - Каждая версия может иметь собственные директории `data`, `binlog`, `relaylog`, `log`, `tmp`, `run`, `config`.
    - Для каждой версии можно использовать отдельный файл конфигурации `my.cnf` и отдельный `systemd` unit.
    - Сервис запускается от системного пользователя `mysql`.

## Назначение схемы установки

При установке MySQL в отдельную версионную директорию можно держать на одном сервере несколько независимых экземпляров MySQL разных версий.

Например:

```text
/app/mysql/8.0.35
/app/mysql/8.0.46
/app/mysql/8.4.0
```

!!! warning "Важно"
    У каждого экземпляра должны быть собственные значения параметров, например: `port`, `socket`, `datadir`, `server_id`, а также отдельный `systemd` unit.

## 1. Установка зависимостей

```bash
dnf install -y libaio numactl-libs tar wget xz
```

## 2. Создание системного пользователя

```bash
useradd -r -s /sbin/nologin mysql
```

## 3. Создание директорий

```bash
mkdir -p /app/mysql/8.0.46/{server,data,binlog,relaylog,log,tmp,run,config}
```

## 4. Скачивание архива MySQL

```bash
cd /app/mysql/8.0.46
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz
sha256sum mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz
```

!!! warning "Проверка архива"
    Значение `sha256sum` должно совпадать с checksum, указанным на официальном сайте MySQL.

После проверки checksum распаковываем архив в директорию `/app/mysql/8.0.46/server`

```bash
tar -xvf mysql-8.0.46-linux-glibc2.28-x86_64.tar.xz -C /app/mysql/8.0.46/server --strip-components=1
```
## 5. Создание файла конфигурации

```bash
vim /app/mysql/8.0.46/config/my.cnf
```

```ini
[client]
port = 3306
socket = /app/mysql/8.0.46/run/mysql.sock

[mysql]
socket = /app/mysql/8.0.46/run/mysql.sock

[mysqld]
server_id = 1
user = mysql
port = 3306
bind_address = 0.0.0.0
skip_name_resolve = ON

socket = /app/mysql/8.0.46/run/mysql.sock
pid-file = /app/mysql/8.0.46/run/mysql.pid

basedir = /app/mysql/8.0.46/server
datadir = /app/mysql/8.0.46/data
tmpdir = /app/mysql/8.0.46/tmp

character-set-server = utf8mb4
collation-server = utf8mb4_0900_ai_ci

default_storage_engine = InnoDB
disabled_storage_engines = MyISAM,BLACKHOLE,FEDERATED,ARCHIVE

log-error = /app/mysql/8.0.46/log/mysql.err

log_bin = /app/mysql/8.0.46/binlog/mysql-bin
log_bin_index = /app/mysql/8.0.46/binlog/mysql-bin.index
log_replica_updates = ON

relay_log = /app/mysql/8.0.46/relaylog/mysql-relay-bin
relay_log_index = /app/mysql/8.0.46/relaylog/mysql-relay-bin.index
relay_log_recovery = ON

gtid_mode = ON
enforce_gtid_consistency = ON

innodb_flush_log_at_trx_commit = 1
sync_binlog = 1

source_verify_checksum = ON
replica_sql_verify_checksum = ON
binlog_checksum = CRC32
```

## 6. Назначение прав

```bash
chown -R mysql:mysql /app/mysql/8.0.46
chmod 750 /app/mysql/8.0.46
```

## 7. Создание файла окружения `env`

Создаем файл окружения в директории `/root`

```bash
vim /root/mysql_8_0_46.env
```

```bash
export MYSQL_VERSION=8.0.46
export MYSQL_BASE=/app/mysql/8.0.46
export MYSQL_HOME=/app/mysql/8.0.46/server
export MYSQL_CNF=/app/mysql/8.0.46/config/my.cnf
export MYSQL_SOCK=/app/mysql/8.0.46/run/mysql.sock
export MYSQL_PORT=3306

case ":$PATH:" in
  *":$MYSQL_HOME/bin:"*) ;;
  *) export PATH="$MYSQL_HOME/bin:$PATH" ;;
esac
```

Загружаем переменные окружения

```bash
source /root/mysql_8_0_46.env
```

## 8. Инициализация data-директории

!!! warning "Важно"
    Перед инициализацией директория `/app/mysql/8.0.46/data` должна быть пустой.

```bash
/app/mysql/8.0.46/server/bin/mysqld --defaults-file=/app/mysql/8.0.46/config/my.cnf --initialize --user=mysql --console
```

!!! note "Временный пароль root"
    После инициализации MySQL запишет временный пароль пользователя `root`
    в error log.

```bash
sudo grep 'temporary password' /app/mysql/8.0.46/log/mysql.err
```

## 9. Первый тестовый запуск MySQL

```bash
/app/mysql/8.0.46/server/bin/mysqld_safe --defaults-file=/app/mysql/8.0.46/config/my.cnf --user=mysql &
```

## 10. Подключаемся к базе данных

```bash
mysql -u root -p -S /app/mysql/8.0.46/run/mysql.sock
```

!!! note "Временный пароль"
    Используйте временный пароль пользователя `root`, который был получен после инициализации.

## 11. Смена пароля пользователя root
После первого подключения необходимо заменить временный пароль пользователя `root` на постоянный

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Qwerty123!';
```

## 12. Ручная остановка MySQL

```bash
/app/mysql/8.0.46/server/bin/mysqladmin -uroot -p -S /app/mysql/8.0.46/run/mysql.sock shutdown
```

## 13. Создание systemd unit-файла

```bash
vim /etc/systemd/system/mysql_8_0_46.service
```

```ini
[Unit]
Description=MySQL 8.0.46 Server
After=network.target

[Service]
Type=simple
User=mysql
Group=mysql

ExecStart=/app/mysql/8.0.46/server/bin/mysqld_safe --defaults-file=/app/mysql/8.0.46/config/my.cnf --user=mysql

PIDFile=/app/mysql/8.0.46/run/mysql.pid

LimitNOFILE=65535
Restart=on-failure
RestartSec=5

PrivateTmp=false

[Install]
WantedBy=multi-user.target
```

Регистрируем unit-файл и включаем сервис в автозагрузку
```bash
sudo systemctl daemon-reload
sudo systemctl enable mysql_8_0_46.service
```

Запускаем сервис
```bash
systemctl start mysql_8_0_46.service
systemctl status mysql_8_0_46.service
```

## 14. Проверка работы MySQL

```bash
systemctl status mysql_8_0_46.service
ss -lntp | grep 3306
mysql -u root -p -S /app/mysql/8.0.46/run/mysql.sock
```

```sql
SHOW VARIABLES LIKE 'version';
```