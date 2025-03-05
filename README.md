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
Видим что базы **replica_test** и **otus** востсановились из резервной копии
