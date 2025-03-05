# OTUS PRO Homework 46 Postgres

## Домашняя работа 46: Postgres: Backup + Репликация

### Домашнее задание:
**1. Подготовка рабочего места**   
**2. В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp**

Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:

| bookmaker   |   
| competition |   
| market      |   
| odds        |   
| outcome     |   
   
Настроить GTID репликацию   
 
---
## Выполнение задания:
### 1. Подготовка рабочего места:
Выполнение домашнего задания предполагает, что на компьютере установлен Vagrant+VirtualBox   
**[Как установить Vagrant на Debian 12](https://github.com/avlikh/Install_Vagrant_Debian12/blob/main/README.md)**   

Развернем Vagrant-стенд:
  - Создайте папку с проектом и зайдите в нее (например: /opt/otus/mysql):
```
mkdir -p /opt/otus/mysql ; cd /opt/otus/mysql
```
  - Клонируете проект с Github, набрав команду:
```
apt update -y && apt install git -y ; git clone https://github.com/avlikh/Otus_pro_44.git .
```
  - Запустите проект из папки, в которую склонировали проект (в нашем примере /opt/otus/mysql):
```
vagrant up
```
**Результатом выполнения команды vagrant up станет:** 
* создание 2-х виртуальных машин **mysqlsrs** и **mysqlrep** с установленными MySQL серверами;
* на MySQL source (mysqlsrs) загружен дамп таблицы 'bet';
* настроена GTID репликация с MySQL source (mysqlsrs) на MySQL replica (mysqlrep), при этом таблицы bet.events_on_demand и bet.v_same_event исключены из репликации.
---
### 2. Домашнее задание было выполнено при помощи Vagrant+Ansible.
   
Проверим что стенд коректно работает и условия домашнего задания выполнены.

Зайдем на мастер (mysqlsrs) и повысим полномочия до root: 
```   
vagrant ssh mysqlsrs   
```
```
sudo -i
```

Зайдем в mysql:   
```
mysql
```
Выведим список баз данных:
```
show databases;
```
<details>
<summary> результат выполнения команды: </summary>

```
+--------------------+
| Database           |
+--------------------+
| bet                |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)
```
</details>

Выберим базу 'bet'
```
use bet;
```

Посмотрим таблицы базы данных:

```
show tables;
```
<details>
<summary> результат выполнения команды: </summary>

```
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
7 rows in set (0.01 sec)
```
</details>
   
**Видим, что база 'bet' содержит 7 таблиц.**
   
Зайдем на реплику (mysqlrep) и повысим полномочия до root: 
```   
vagrant ssh mysqlrep   
```
```
sudo -i
```

Зайдем в mysql:   
```
mysql
```
Выведим список баз данных:
```
show databases;
```
<details>
<summary> результат выполнения команды: </summary>

```
+--------------------+
| Database           |
+--------------------+
| bet                |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)
```
</details>

Выберим базу 'bet'
```
use bet;
```

Посмотрим таблицы базы данных:

```
show tables;
```
<details>
<summary> результат выполнения команды: </summary>

```
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
5 rows in set (0.00 sec)
```
</details>

**Видим, что база 'bet' содержит 5 таблиц.**   
**Таблицы 'events_on_demand' и 'v_same_event' отсутствуют т.к. исключены из репликации**

Посмотрим статус реплики:
 
```
show replica status\G
```
<details>
<summary> результат выполнения команды: </summary>

```
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 192.168.57.10
                  Source_User: repl
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000001
          Read_Source_Log_Pos: 93814
               Relay_Log_File: relay-log-server.000002
                Relay_Log_Pos: 94030
        Relay_Source_Log_File: mysql-bin.000001
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Source_Log_Pos: 93814
              Relay_Log_Space: 94241
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Source_SSL_Allowed: No
           Source_SSL_CA_File:
           Source_SSL_CA_Path:
              Source_SSL_Cert:
            Source_SSL_Cipher:
               Source_SSL_Key:
        Seconds_Behind_Source: 0
Source_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Source_Server_Id: 1
                  Source_UUID: d29bf739-f350-11ef-bd99-0800278dc04d
             Source_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Source_Retry_Count: 86400
                  Source_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Source_SSL_Crl:
           Source_SSL_Crlpath:
           Retrieved_Gtid_Set: d29bf739-f350-11ef-bd99-0800278dc04d:1-39
            Executed_Gtid_Set: d29bf739-f350-11ef-bd99-0800278dc04d:1-39
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Source_TLS_Version:
       Source_public_key_path:
        Get_Source_public_key: 1
            Network_Namespace:
1 row in set (0.00 sec)
```
</details>

**Видим что репликация работает корректно.**

P.S.   

<details>
<summary> Конфиг MySQL source (master): </summary>

```
[mysqld]
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
datadir		= /var/lib/mysql
log-error	= /var/log/mysql/error.log

bind-address    = 0.0.0.0
server-id	= 1
log-bin		= mysql-bin
binlog_format	= row
gtid-mode=ON
enforce-gtid-consistency
log-replica-updates
```
</details>

<details>
<summary> Конфиг MySQL replica (slave): </summary>

```
[mysqld]
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
datadir		= /var/lib/mysql
log-error	= /var/log/mysql/error.log

bind-address    = 0.0.0.0
server-id       = 2
log-bin         = mysql-bin
relay-log	= relay-log-server
read-only	= ON
gtid-mode=ON
enforce-gtid-consistency
log-replica-updates
replicate-ignore-table=bet.events_on_demand
replicate-ignore-table=bet.v_same_event
```
</details>
