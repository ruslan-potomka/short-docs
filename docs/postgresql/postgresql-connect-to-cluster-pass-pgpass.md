# Подключение к PostgreSQL через `.pgpass`

Инструкция описывает настройку подключения к PostgreSQL без ручного ввода пароля с использованием файла `.pgpass`.

!!! tip "Как пользоваться инструкцией"
    Команды можно копировать и выполнять по порядку сверху вниз. :slightly_smiling_face:

!!! info "Особенности подключения"
    - Файл `.pgpass` используется PostgreSQL-клиентами для автоматической подстановки пароля.
    - Пароль не нужно хранить в переменной окружения `PGPASSWORD`.
    - Файл `.pgpass` должен иметь права `600`, иначе PostgreSQL может его проигнорировать.

## Назначение `.pgpass`

Файл `.pgpass` нужен для подключения к PostgreSQL без интерактивного ввода пароля. `.pgpass` создаётся в домашней директории пользователя, от которого выполняется подключение.

В этой инструкции подключение выполняется от пользователя `postgres`, поэтому файл создаётся здесь:

```text
/home/postgres/.pgpass
```

## 1. Создание файла `.pgpass`

```bash
vim /home/postgres/.pgpass
```

Добавляем строку подключения:

```text
/tmp:5432:postgres:postgres:Qwerty123!
```

Формат строки `.pgpass`:

```text
hostname:port:database:username:password
```

Где:

```text
/tmp        — путь к Unix socket PostgreSQL
5432        — порт PostgreSQL
postgres    — имя базы данных
postgres    — имя пользователя PostgreSQL
Qwerty123!  — пароль пользователя postgres
```

## 2. Назначение прав на файл `.pgpass`

Файл `.pgpass` должен быть доступен только владельцу.

```bash
chmod 600 /home/postgres/.pgpass
```

Проверяем права:

```bash
ls -la /home/postgres/.pgpass
```

Ожидаемый результат:

```text
-rw-------. 1 postgres postgres 42 Jun  1 12:00 /home/postgres/.pgpass
```

Если файл был создан под `root`, нужно дополнительно назначить владельца:

```bash
chown postgres:postgres /home/postgres/.pgpass
chmod 600 /home/postgres/.pgpass
```

## 3. Примеры `.pgpass` и проверки подключения через Unix socket и TCP/IP

Ниже два варианта подключения: через Unix socket и через TCP/IP.
Выбери тот вариант, который используется в твоём окружении.

### Вариант 1. Подключение через Unix socket

Строка в `.pgpass`:

```text
/tmp:5432:postgres:postgres:Qwerty123!
```

Проверка подключения:

```bash
psql -h /tmp -p 5432 -U postgres -d postgres
```

Тестовый SQL-запрос:

```bash
psql -h /tmp -p 5432 -U postgres -d postgres -c "select version();"
```

### Вариант 2. Подключение через TCP/IP

Если подключение выполняется через TCP/IP, например через `192.168.32.41`, строка в `.pgpass` будет такой:

```text
192.168.32.41:5432:postgres:postgres:Qwerty123!
```

Проверка подключения:

```bash
psql -h 192.168.32.41 -p 5432 -U postgres -d postgres
```

Тестовый SQL-запрос:

```bash
psql -h 192.168.32.41 -p 5432 -U postgres -d postgres -c "select version();"
```

Если `.pgpass` настроен правильно, PostgreSQL подключится без запроса пароля.

## 4. Пример `.pgpass` для подключения к конкретной базе

Например, есть база `app_db` и пользователь `app_user`.

Строка в `.pgpass`:

```text
/tmp:5432:app_db:app_user:Qwerty123!
```

Проверка подключения:

```bash
psql -h /tmp -p 5432 -U app_user -d app_db
```

## 5. Пример `.pgpass` с wildcard

В `.pgpass` можно использовать символ `*`.

Пример:

```text
192.168.32.41:5432:*:postgres:Qwerty123!
```

Это означает:

```text
192.168.32.41 — ip сервера
5432          — порт 5432
*             — любая база данных
postgres      — пользователь postgres
Qwerty123!    — пароль пользователя postgres
```
