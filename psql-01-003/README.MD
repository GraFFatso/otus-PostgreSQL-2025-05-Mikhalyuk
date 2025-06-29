## Занятие 3. Установка PostgreSQL
### Преподаватель Ростислав Михеев
## подготовительные работы, вводные

использую "стенд" с hyper-v, i3-6100 +40gb RAM\
создаю ВМ по необходимости\
к различным хостам подключаюсь через mobaxterm

### Подготовтительные работы
* для ВМ выделил 2 ядра, 8гб ОЗУ, статический MAC, чтобы не сбивался IP

* установил ubuntu-22.05.5 live -server без GUI

* забыл имя УЗ

загрузился в режим recovery, увидел имя пользователя, смог влогиниваться не переустанавливая ОС

* установка доп. необходимго ПО

```sudo apt update && sudo apt upgrade```\
```sudo apt install mc openssh-server openssh-client unzip wget sysstat whiptail```

команда _iostat_ в пакете _sysstat_\
_whiptail_ необходим для диалогов установки postgresql\
установилось новое ядро Linux 5.15.0

* далее в соответствии с методичкой, приложенной к занятию\
включен ssh, сгенерированы открытый и закрытый ключи\
сохранил публичный ключ в файл на win машине\
для поключения к VM ubuntu добавил ключ для подключения\
но "не прокатило", курс не по  linux, поэтому оставил на потом

### Цель занятия

развернуть кластер PostgreSQL различными способами;\
понимать возможности и ограничения каждого способа;\
выбрать наиболее оптимальный способ в зависимости от задачи;\
обновить кластер на новую версию.

### Выполнение ДЗ

* добавил репозиторий и ключ подписи пакетов для установки различных версий postgreSQL командами

```sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list```\
```wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add```\
```sudo apt update```

_разбил на строки, чтобы мне понятнее было, что делает последовательность команд_

* поиск доступных для установки версий:

```apt search postgresql-15,16,17,18```

пакеты доступны

* установил 15 версию postgresql

```sudo apt install postgresql-15```

во время установки увидел необходимсть установки _whiptail_

```------------------------------
Creating config file /etc/postgresql-common/createcluster.conf with new version
Building PostgreSQL dictionaries from installed myspell/hunspell packages...
Removing obsolete dictionary files:
Created symlink /etc/systemd/system/multi-user.target.wants/postgresql.service → /lib/systemd/system/postgresql.service.
Setting up postgresql-15 (15.13-1.pgdg22.04+1) ...
debconf: unable to initialize frontend: Dialog
debconf: (No usable dialog-like program is installed, so the dialog based frontend cannot be used. at /usr/share/perl5/Debconf/FrontEnd/Dialog.pm line 78.)
debconf: falling back to frontend: Readline
Creating new PostgreSQL cluster 15/main ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Processing triggers for libc-bin (2.35-0ubuntu3.10) ...
debconf: unable to initialize frontend: Dialog
debconf: (No usable dialog-like program is installed, so the dialog based frontend cannot be used. at /usr/share/perl5/Debconf/FrontEnd/Dialog.pm line 78.)

------------------------------
```

* удалил инсталляцию

```sudo apt remove postgresql-15```

* (доставил пакет whiplist) и потом установил еще раз 15 postgreSQL

```sudo apt install postgresql-15```

_предупреждений никаких не было во время установки_\
при установке автоматом инициализировался кластер с настройками по-умолчанию из файла\
```/etc/postgresql-common/createcluster.conf```

* установил пакеты 16 и 17, чтобы 15 main обновить до 17 версии

```sudo apt install  postgresql-16 postgresql-17```

* т.к. в конфиг-файле есть описание

``Directory for additional createcluster confiп\
include_dir '/etc/postgresql-common/createcluster.d'1
``
* то создал файл, понимая, что настройки из этого файла будут переопределять настройки каждого из вновь инициализируемых кластеров ```001.conf``` (т.к. пока не знаю какие именно настройки будут там) с таким содержимым:

``start_conf = 'manual'
initdb_options = '--encoding=UTF8 --data-checksums --icu-locale=ru --locale-provider=icu --set=work_mem=200MB --set=log_timezone=Europe/Moscow  --set=timezone=Europe/Moscow --set=listen_addresses=*'
``

тогда кластер инициализировался с нужными параметрами:

``sudo grep -R 'work_mem' /etc/postgresql/``
```/etc/postgresql/16/first/postgresql.conf:work_mem = 200MB                       # min 64kB
/etc/postgresql/15/main/postgresql.conf:#work_mem = 4MB                         # min 64kB
```

``sudo grep -R 'listen_addresses' /etc/postgresql/ --color``
```/etc/postgresql/16/first/postgresql.conf:listen_addresses = '*'                 # what IP address(es) to listen on;
/etc/postgresql/15/main/postgresql.conf:#listen_addresses = 'localhost'         # what IP address(es) to listen on;
```
 
``sudo grep -R 'timezone' /etc/postgresql/ ``
```/etc/postgresql/16/first/postgresql.conf:log_timezone = 'Europe/Moscow'
/etc/postgresql/16/first/postgresql.conf:timezone = 'Europe/Moscow'
/etc/postgresql/15/main/postgresql.conf:log_timezone = 'Etc/UTC'
/etc/postgresql/15/main/postgresql.conf:timezone = 'Etc/UTC'
/etc/postgresql/15/main/postgresql.conf:#timezone_abbreviations = 'Default'     # Select the set of available time zone
```

* добавил разрешение подключаться из локальной сети к СУБД

``echo 'host all all 192.168.70.0/24 scram-sha-256' | sudo tee -a   /etc/postgresql/15/main/pg_hba.conf``

``sudo pg_ctlcluster start 15 main``

* задал пароль для пользователя СУБД postgres

``sudo -u postgres psql``
``\password   #12345``
``\q``


*  все еще нeвозможно подключиться из pgadmin\
  проверка возможности покдлючения из powershell\
`` Test-NetConnection -ComputerName 192.168.70.12 -RemotePort 5432``

* через psql внес изменения в работающий кластер

``ALTER SYSTEM SET listen_addresses = '*'\gx``

* перезапустил кластер командой

``sudo pg_ctlcluster restart 15 main``\
затем успешное подключение
   
* создал БД в кластере 15 версии

``sudo -u postgres psql
   CRATE DATABASE otus \g``

* насоздалвал в БД объектов

``sudo -u postgres pgbench -i --scale=10  --port=5432 otus``

* обновил кластер с 15 на 17 версию командой

``sudo -u postgres pg_upgradecluster -v 17 15 main /var/lib/postgresql/17/upgraded --rename=upgraded_15_to_17``

* в pg_admin увидел, что версия кластера поменялась

* затем удалил обновленный кластер

``sudo pg_ctlcluster stop 17 upgraded_15_to_17``
``sudo pg_dropcluster 17 upgraded_15_to_17``
``sudo pg_ctlcluster start 15 main ``

подключение неуспешное, т.к. при обновлении для 15 main поменялдся порт с 5432 на 5434, а порт 5432 был "передан" кластеру, обновленному с 15 на 17

*  номера портов зарегистрированных кластеров увидел командой\
``pg_lsclusters``
     
* в pgadmin изменил порт подключения и смог подключиться к необновленному master 15 на 5434


---------------------------------------------------------------------------------------------------------------

* по инструкции
https://docs.docker.com/engine/install/ubuntu/

* добавил репозиторий docker, установил необходимые пакеты, создал сеть для docker командой

``sudo docker network create pg-net``

* командой скачались необходимые пакеты и создался контейнер с необходимыми параметрами

``sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15``

> в предыдущих итерациях ДЗ пароль для УЗ был другой\
> путь к каталогу в команде /var/lib/postgresql/data, был /var/lib/postgresql/<version>/<cluster_name>


т.к. имеющиеся кластеры работают не на 5432, то СУБД запустилась и работает на порту 5432
```pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
15  main    5434 down   postgres /var/lib/postgresql/15/main  /var/log/postgresql/postgresql-15-main.log
16  first   5433 down   postgres /var/lib/postgresql/16/first /var/log/postgresql/postgresql-16-first.log
```
* в СУБД контейнера создал БД, с именем отличным от ранее созданной
``sudo docker exec -it pg-server bash``

``CREATE DATABASE otus23 \gx``

* к кластеру возможно подключиться командой 
`` psql -p 5432 -h 127.0.0.1 -U postgres``
 увидел созданную БД

* остановил контейнер, нашел в каталоге /var/lib/docker/containers/... каталог контейнера кластера, изменил номера портов, чтобы понимать, который из параметров, за что отвечает.
* реализовал таким образом, чтобы внутри контейнера использовался такой же номер порта, который хадействован на одном из кластеров, но маппинг портов из контейнера в хост на другой порт, таким образом могут работать все три кластера(два на хосте и один в контейнере)

``sudo docker ps -a``
``CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS                   PORTS                                         NAMES
 58c244557bfb   postgres:15   "docker-entrypoint.s…"   2 hours ago   Up 3 minutes             0.0.0.0:5441->5432/tcp, [::]:5441->5432/tcp   pg-server
 3e75f399a18a   hello-world   "/hello"                 2 hours ago   Exited (0) 2 hours ago                                                 laughing_tu
``
``netstat -ntlp
 (Not all processes could be identified, non-owned process info
  will not be shown, you would have to be root to see it all.)
  Active Internet connections (only servers)
  Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
  tcp        0      0 0.0.0.0:5441            0.0.0.0:*               LISTEN      -
  tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
  tcp        0      0 127.0.0.1:6010          0.0.0.0:*               LISTEN      -
  tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
  tcp6       0      0 :::5441                 :::*                    LISTEN      -
  tcp6       0      0 :::22                   :::*                    LISTEN      -
  tcp6       0      0 ::1:6010                :::*                    LISTEN      -
``

``sudo docker inspect  58c244557bfb``

`` ...
    "Ports": {
            "5432/tcp": [
            {
            "HostIp": "0.0.0.0",
            "HostPort": "5441"
            },
            {
            "HostIp": "::",
            "HostPort": "5441"
            }
            ]
            },
...``
               в pgadmin изменил порт, смог подключиться к кластеру, увидел созданную БД