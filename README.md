# OTUS PRO Homework 46 Postgres

## Домашняя работа 46: Postgres: Backup + Репликация

### Домашнее задание:
**1. Подготовить рабочее место**
**2. Настроить hot_standby репликацию с использованием слотов**   
**3. Настроить правильное резервное копирование**  

---
## Выполнение задания:
### 1. Подготовка рабочего места:
Выполнение домашнего задания предполагает, что на компьютере установлен Vagrant+VirtualBox   
**[Как установить Vagrant на Debian 12](https://github.com/avlikh/Install_Vagrant_Debian12/blob/main/README.md)**   

Развернем Vagrant-стенд:
  - Создайте папку с проектом и зайдите в нее (например: /opt/otus/postgresql):
```
mkdir -p /opt/otus/postgresql ; cd /opt/otus/postgresql
```
  - Клонируете проект с Github, набрав команду:
```
apt update -y && apt install git -y ; git clone https://github.com/avlikh/Otus_pro_46.git .
```
  - Запустите проект из папки, в которую склонировали проект (в нашем примере /opt/otus/mysql):
```
vagrant up
```
**Результатом выполнения команды vagrant up станет:** 
* создание 3-х виртуальных машин:
* **node1** - на нем развернут сервер Postgres (postgres source)
* **node2** - на нем развернута hot_standby реплика Postgres c сервера node1
* **barman** - на нем развернут backup Postgres (средствами barman) с сервера node1
---
### 2. Домашнее задание было выполнено при помощи Vagrant+Ansible.
   
Проверим что стенд коректно работает и условия домашнего задания выполнены.

**2. Для начала проверим что репликация работает:**

**2.1.** Зайдем на сервер node1 (postgres source) и повысим полномочия до root: 
```   
vagrant ssh node1   
```
```
sudo -i
```
Создадим тестовую базу **replica_test** в кластере Postgres сервера node1:
```
sudo -u postgres psql
```
```
CREATE DATABASE replica_test;
```
```
\l
```
<details>
<summary> результат выполнения команды: </summary>

```
CREATE DATABASE
                                               List of databases
     Name     |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
--------------+----------+----------+---------+---------+------------+-----------------+-----------------------
 otus         | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 postgres     | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 replica_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
              |          |          |         |         |            |                 | postgres=CTc/postgres
 template1    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
              |          |          |         |         |            |                 | postgres=CTc/postgres
(5 rows)
```
</details>

Видим что база успешно создалась.   

**2.2.** Проверим, что база данных **replica_test** присутствует на сервере node2

Зайдем на сервер node2 (postgres replica) и повысим полномочия до root: 
```   
vagrant ssh node1   
```
```
sudo -i
```
Посмотрим на список баз:
```
sudo -u postgres psql
```
```
\l
```
<details>
<summary> результат выполнения команды: </summary>

```
                                               List of databases
     Name     |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
--------------+----------+----------+---------+---------+------------+-----------------+-----------------------
 otus         | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 postgres     | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 replica_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
              |          |          |         |         |            |                 | postgres=CTc/postgres
 template1    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
              |          |          |         |         |            |                 | postgres=CTc/postgres
(5 rows)
```
</details>

Видим что база данных **replica_test** среплицировалась на node2 (postgres replica).   
   
**Итак, репликация работает.**    
    
    
**2. Теперь приступим про проверке резервного копирования postgres node1 через barman:**
   
Зайдем на сервер barman (сервер резервного копирования postgres) и повысим полномочия до root: 
```   
vagrant ssh node1   
```
```
sudo -i
``` 
Зайдем в пользователя barman: 
```
su - barman
``` 
Проверим репликацию:
```
psql -h 192.168.57.11 -U barman -c "IDENTIFY_SYSTEM" replication=1
```
<details>
<summary> результат выполнения команды: </summary>

```
      systemid       | timeline |  xlogpos  | dbname
---------------------+----------+-----------+--------
 7478331311638356174 |        1 | 0/38A31B8 |
(1 row)
```
</details>

Приступим к созданию резервной копии кластера postgre node1:
```
barman switch-wal node1
```
<details>
<summary> результат выполнения команды: </summary>

```
The WAL file 000000010000000000000003 has been closed on server 'node1'
```
</details>

```
barman cron
```
<details>
<summary> результат выполнения команды: </summary>

```
Starting WAL archiving for server node1
Starting streaming archiver for server node1
```
</details>

```
barman check node1
```
<details>
<summary> результат выполнения команды: </summary>

```
Server node1:
        PostgreSQL: OK
        superuser or standard user with backup privileges: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
        backup minimum size: OK (0 B)
        wal maximum age: OK (no last_wal_maximum_age provided)
        wal size: OK (0 B)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK (no system Id stored on disk)
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archiver errors: OK

```
</details>

По результату выполнения команды `barman check node1` видим только 2 предупреждения:
*  backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
*  minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
Данные предупреждения свидетельствуют о том что резервные копии с сервера node1 пока еще не делались.
    
Игнорируем предупреждения и запускаем создание резервной копии node1:
```
barman backup node1
``` 
<details>
<summary> результат выполнения команды: </summary>

```
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20250305T151133
Backup start at LSN: 0/6000148 (000000010000000000000006, 00000148)
Starting backup copy via pg_basebackup for 20250305T151133
WARNING: pg_basebackup does not copy the PostgreSQL configuration files that reside outside PGDATA. Please manually backup the following files:
        /etc/postgresql/15/main/postgresql.conf
        /etc/postgresql/15/main/pg_hba.conf
        /etc/postgresql/15/main/pg_ident.conf

Copy done (time: less than one second)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
        000000010000000000000004 from server node1 has been removed
        000000010000000000000005 from server node1 has been removed
Backup size: 36.5 MiB
Backup end at LSN: 0/8000000 (000000010000000000000007, 00000000)
Backup completed (start time: 2025-03-05 15:11:33.619439, elapsed time: less than one second)
Processing xlog segments from streaming for node1
        000000010000000000000006
        000000010000000000000007
```
</details>

Видим что резервная копия успешно создалась.
* Примечание: Barman предупредил нас что конфиги postgres node1 не были сохранены. Стоит позаботится о сохранении их вручную, либо поставить бэкап конфигов на cron.
    
    
**2. Теперь приступим про проверке восстановления postgres node1 с сервера barman:**    
    
Зайдем на сервер node1 и удалим базы **replica_test** и **otus**
```
sudo -u postgres psql
```
```
DROP DATABASE otus;
```
```
DROP DATABASE replica_test;
```
```
\l
```
<details>
<summary> результат выполнения команды: </summary>

```
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
(3 rows)
```
</details>
Видим что обе базы удалены из кластера postgres node1   
   
Зайдем на сервер node2 и проверим удалились ли базы **replica_test** и **otus** с него
```
sudo -u postgres psql
```
```
\l
```   
<details>
<summary> результат выполнения команды: </summary>

```
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
(3 rows)
```
</details>
Видим, что базы удалились.   
   
   
**Восстановим кластер postgres node1 с сервера barman:**

Зайдем на сервер barman и проверим посмотрим список резервных копий node1:
```
su -u barman
```
```
barman list-backup node1
```
`node1 20250305T151133 - Wed Mar  5 12:11:34 2025 - Size: 36.5 MiB - WAL Size: 0 B`   
   
Выполним восстановление:
```
barman recover node1 20250305T151133 /var/lib/postgresql/15/main/ --remote-ssh-comman "ssh postgres@192.168.57.11"
```

<details>
<summary> результат выполнения команды: </summary>

```
The authenticity of host '192.168.57.11 (192.168.57.11)' can't be established.
ED25519 key fingerprint is SHA256:X/+Bp798rl0xNTO4CyZpu7E6xK/9U3PWU2VGElDBOvY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Starting remote restore for server node1 using backup 20250305T151133
Destination directory: /var/lib/postgresql/15/main/
Remote command: ssh postgres@192.168.57.11
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

WARNING
The following configuration files have not been saved during backup, hence they have not been restored.
You need to manually restore them in order to start the recovered PostgreSQL instance:

    postgresql.conf
    pg_hba.conf
    pg_ident.conf

Recovery completed (start time: 2025-03-05 15:45:23.063142+00:00, elapsed time: 9 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```
</details>
Видим, что восстановление выполнено успешно.    
     
    
Зайдем на сервер node1, перезапустим postgres и проверим наличие ранее удаленных баз **replica_test** и **otus**
```
systemctl restart postgresql
```
```
sudo -u postgres psql
```
```
\l
```
<details>
<summary> результат выполнения команды: </summary>

```
                                               List of databases
     Name     |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
--------------+----------+----------+---------+---------+------------+-----------------+-----------------------
 otus         | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 postgres     | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 replica_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
              |          |          |         |         |            |                 | postgres=CTc/postgres
 template1    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
              |          |          |         |         |            |                 | postgres=CTc/postgres
(5 rows)
```
</details>
Видим что базы **replica_test** и **otus** восстановились из резервной копии.   
    
    
Проверим среплицировались ли изменения с node1 на node2, после воосстановления кластера postgres.    
    
Зайдем на сервер node2  и посмотрим список баз:
```
sudo -u postgres psql
```
```
\l
```
<details>
<summary> результат выполнения команды: </summary>

```
                                             List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
           |          |          |         |         |            |                 | postgres=CTc/postgres
(3 rows)
```
</details>
    
Видим что баз данных **replica_test** и **otus** нет на кластере node2 (postgres replica).   
     
В чем может быть прична?    
Скорее всего, после восстановления из физического быкапа измениись pg_wal и это привело с нарушению репликации.    
    
**Нальем реплику с node1 в node2. Для этого:**
    
Остановим postgres:

```
systemctl stop postgresql
```

Удалим датафайлы postgres: 
```
rm -rf /var/lib/postgresql/15/main/*
```

Зальем перлику с node1 в node2:
```
PGPASSWORD='Otus2022!' pg_basebackup -h 192.168.57.11 -U replication -p 5432 -D /var/lib/postgresql/15/main/ -R -P
```
`38207/38207 kB (100%), 1/1 tablespace`    
    
Сменим вледельца папки: /var/lib/postgresql/15/main
```
chown -R postgres:postgres /var/lib/postgresql/15/main
```
     
Запустим сервис postgres:
```
systemctl start postgresql
```
     
Посмотрим на базы данных в класетер postgres node1:

```
sudo -u postgres psql
```
```
\l
```
<details>
<summary> результат выполнения команды: </summary>

```
                                               List of databases
     Name     |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
--------------+----------+----------+---------+---------+------------+-----------------+-----------------------
 otus         | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 postgres     | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 replica_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
 template0    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
              |          |          |         |         |            |                 | postgres=CTc/postgres
 template1    | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
              |          |          |         |         |            |                 | postgres=CTc/postgres
(5 rows)
```
</details>
     
Видим, что все базы успешно среплицировались с кластера postgres node1

---

P.S.    
**Конфиги node1/node2:**
<details>
<summary> /etc/postgresql/15/main/postgresql.conf </summary>

```
data_directory = '/var/lib/postgresql/15/main'
hba_file = '/etc/postgresql/15/main/pg_hba.conf'
ident_file = '/etc/postgresql/15/main/pg_ident.conf'
external_pid_file = '/var/run/postgresql/15-main.pid'
listen_addresses = 'localhost, 192.168.57.11'
port = 5432
max_connections = 100
unix_socket_directories = '/var/run/postgresql'
password_encryption = scram-sha-256
ssl = on
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
shared_buffers = 128MB
dynamic_shared_memory_type = posix
wal_level = replica
max_wal_size = 1GB
min_wal_size = 80MB
max_wal_senders = 3
max_replication_slots = 3
hot_standby = on
hot_standby_feedback = on
log_directory = 'log'
log_filename = 'postgresql-%a.log'
log_rotation_age = 1d
log_rotation_size = 0
log_truncate_on_rotation = on
log_line_prefix = '%m [%p] '
log_timezone = 'UTC+3'
cluster_name = '15/main'
datestyle = 'iso, mdy'
timezone = 'UTC+3'
lc_messages = 'C.UTF-8'
lc_monetary = 'C.UTF-8'                 # locale for monetary formatting
lc_numeric = 'C.UTF-8'                  # locale for number formatting
lc_time = 'C.UTF-8'                     # locale for time formatting
default_text_search_config = 'pg_catalog.english'
include_dir = 'conf.d'
```
</details>

<details>
<summary> /etc/postgresql/15/main/pg_hba.conf </summary>

```
# PostgreSQL Client Authentication Configuration File
# ===================================================
#
# Refer to the "Client Authentication" section in the PostgreSQL
# documentation for a complete description of this file.  A short
# synopsis follows.
#
# This file controls: which hosts are allowed to connect, how clients
# are authenticated, which PostgreSQL user names they can use, which
# databases they can access.  Records take one of these forms:
#
# local         DATABASE  USER  METHOD  [OPTIONS]
# host          DATABASE  USER  ADDRESS  METHOD  [OPTIONS]
# hostssl       DATABASE  USER  ADDRESS  METHOD  [OPTIONS]
# hostnossl     DATABASE  USER  ADDRESS  METHOD  [OPTIONS]
# hostgssenc    DATABASE  USER  ADDRESS  METHOD  [OPTIONS]
# hostnogssenc  DATABASE  USER  ADDRESS  METHOD  [OPTIONS]
#
# (The uppercase items must be replaced by actual values.)
#
# The first field is the connection type:
# - "local" is a Unix-domain socket
# - "host" is a TCP/IP socket (encrypted or not)
# - "hostssl" is a TCP/IP socket that is SSL-encrypted
# - "hostnossl" is a TCP/IP socket that is not SSL-encrypted
# - "hostgssenc" is a TCP/IP socket that is GSSAPI-encrypted
# - "hostnogssenc" is a TCP/IP socket that is not GSSAPI-encrypted
#
# DATABASE can be "all", "sameuser", "samerole", "replication", a
# database name, or a comma-separated list thereof. The "all"
# keyword does not match "replication". Access to replication
# must be enabled in a separate record (see example below).
#
# USER can be "all", a user name, a group name prefixed with "+", or a
# comma-separated list thereof.  In both the DATABASE and USER fields
# you can also write a file name prefixed with "@" to include names
# from a separate file.
#
# ADDRESS specifies the set of hosts the record matches.  It can be a
# host name, or it is made up of an IP address and a CIDR mask that is
# an integer (between 0 and 32 (IPv4) or 128 (IPv6) inclusive) that
# specifies the number of significant bits in the mask.  A host name
# that starts with a dot (.) matches a suffix of the actual host name.
# Alternatively, you can write an IP address and netmask in separate
# columns to specify the set of hosts.  Instead of a CIDR-address, you
# can write "samehost" to match any of the server's own IP addresses,
# or "samenet" to match any address in any subnet that the server is
# directly connected to.
#
# METHOD can be "trust", "reject", "md5", "password", "scram-sha-256",
# "gss", "sspi", "ident", "peer", "pam", "ldap", "radius" or "cert".
# Note that "password" sends passwords in clear text; "md5" or
# "scram-sha-256" are preferred since they send encrypted passwords.
#
# OPTIONS are a set of options for the authentication in the format
# NAME=VALUE.  The available options depend on the different
# authentication methods -- refer to the "Client Authentication"
# section in the documentation for a list of which options are
# available for which authentication methods.
#
# Database and user names containing spaces, commas, quotes and other
# special characters must be quoted.  Quoting one of the keywords
# "all", "sameuser", "samerole" or "replication" makes the name lose
# its special character, and just match a database or username with
# that name.
#
# This file is read on server startup and when the server receives a
# SIGHUP signal.  If you edit the file on a running system, you have to
# SIGHUP the server for the changes to take effect, run "pg_ctl reload",
# or execute "SELECT pg_reload_conf()".
#
# Put your actual configuration here
# ----------------------------------
#
# If you want to allow non-local connections, you need to add more
# "host" records.  In that case you will also need to make PostgreSQL
# listen on a non-local interface via the listen_addresses
# configuration parameter, or via the -i or -h command line switches.




# DO NOT DISABLE!
# If you change this first entry you will need to make sure that the
# database superuser can access the database using some other method.
# Noninteractive access to all databases is required during automatic
# maintenance (custom daily cronjobs, replication, and similar tasks).
#
# Database administrative login by Unix domain socket
local   all             all                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
host    replication  replication        192.168.57.11/32         scram-sha-256
host    replication  replication        192.168.57.12/32         scram-sha-256
host    all   barman    192.168.57.13/32    scram-sha-256
host    replication   barman    192.168.57.13/32    scram-sha-256

```
</details>
    



**Конфиги barman:**
<details>
<summary> /etc/barman.conf </summary>

```
[barman]
#Указываем каталог, в котором будут храниться бекапы
barman_home = /var/lib/barman
#Указываем каталог, в котором будут храниться файлы конфигурации бекапов
configuration_files_directory = /etc/barman.d
#пользователь, от которого будет запускаться barman
barman_user = barman
#расположение файла с логами
log_file = /var/log/barman/barman.log
#Используемый тип сжатия
compression = gzip
#Используемый метод бекапа
backup_method = rsync
archiver = on
retention_policy = REDUNDANCY 3
immediate_checkpoint = true
#Глубина архива
last_backup_maximum_age = 4 DAYS
minimum_redundancy = 1
```
</details>

<details>
<summary> /etc/barman.d/node1.conf </summary>

```
[node1]
#Описание задания
description = "backup node1"
#Команда подключения к хосту node1
ssh_command = ssh postgres@192.168.57.11
#Команда для подключения к postgres-серверу
conninfo = host=192.168.57.11 user=barman port=5432 dbname=postgres
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 7 days
wal_retention_policy = main
streaming_archiver=on
#Указание префикса, который будет использоваться как $PATH на хосте node1
path_prefix = /usr/lib/postgresql
#настройки слота
create_slot = auto
slot_name = node1
#Команда для потоковой передачи от postgres-сервера
streaming_conninfo = host=192.168.57.11 user=barman
#Тип выполняемого бекапа
backup_method = postgres
archiver = off
```
</details>
