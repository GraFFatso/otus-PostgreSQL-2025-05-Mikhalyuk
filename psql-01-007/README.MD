## Занятие 7. MVCC, vacuum и autovacuum
### Преподаватель Виктор Коробков
### Проверяющий ДЗ Игорь Тоескин

## подготовительные работы, вводные

использую "стенд" с hyper-v, ~~i3-6100 +40gb RAM~~\
конфигурация гипервизора изменена на i7-3770, 30GB RAM\
создаю ВМ по необходимости\
к различным хостам подключаюсь через mobaxterm

### Подготовтительные работы
* для ВМ выделил 2 ядра, 8гб ОЗУ, статический MAC, чтобы не сбивался IP

* установил ubuntu-22.05.5 live -server без GUI

* установка доп. необходимго ПО в ubuntu

```sudo apt update && sudo apt upgrade```\
```sudo apt install mc openssh-server openssh-client unzip wget sysstat whiptail```

команда _iostat_ в пакете _sysstat_\
_whiptail_ необходим для диалогов установки postgresql\
команда _netstat_ в пакете _net-tools_\
установилось новое ядро Linux 5.15.0

* в ВМ ubuntu 22.04 установлены пакеты 15, 16 и 17 версий postgresql

## Цель занятия

понять работу механизма многоверсионности в POstgreSQL\
Знать и уметь использовать vacuum и autovacuum;\
разобраться, есть ли проблемы с наличием или отсутствием autovacuum.


## Выполнение ДЗ

из предудущего ДЗ используется ubuntu


*  согласно пошаговой инструкции создаю новый кластер командой (имя кластера содержит месяц занятий и номер занятия)
```
sudo pg_createcluster 15 task_01_007
```
т.к.  в предыдущих заданиях я задал параметры, с которыми будет инициализироваться новый кластер через содержимое файла
```
/etc/postgresql-common/createcluster.d/001.conf
```
файл "принимается в работу" согласно параметров по-умолчанию в файле
```
cat /etc/postgresql-common/createcluster.conf
```
содержимое настроечного файла
```
start_conf = 'manual'
initdb_options = '--encoding=UTF8 --data-checksums --icu-locale=ru --locale-provider=icu  --lc-collate=ru_RU.UTF-8  --lc-ctype=ru_RU.UTF-8'
```

создался необходимый кластер
```
Ver Cluster     Port Status Owner    Data directory                    Log file
15  task_01_007 5436 down   postgres /var/lib/pg_ext_10/15/task_01_007 /var/log/postgresql/postgresql-15-task_01_007.log
```

запуск кластера
```
sudo pg_ctlcluster 15 task_01_007
```

* подключение к СУБД
```
sudo -u postgres psql -p 5436
```

* создана БД
```
CREATE DATABASE task_01_007 \g
```
* инициализировать pgbench

```
pgbench -i task_01_007 -p 5436 -U postgres -h 127.0.0.1
done in 2.06 s (drop tables 0.00 s, create tables 0.22 s, client-side generate 0.73 s, vacuum 0.60 s, primary keys 0.51 s).
```
т.к. создана БД, то инициализирую тест для созданной БД
так же задаю порт необходимого кластера, имя пользователя которым буду подключаться к БД кластера, предварительно в pg_hba.conf добавив строку
```
host all all 127.0.0.1 trust
```
* тест запущен тест командой
```
 pgbench -c 8 -P 6 -T 60   -p 5436 -U postgres -h 127.0.0.1 task_01_007
 
```
результат выполнения команды:
```
starting vacuum...end.
progress: 6.0 s, 21.8 tps, lat 350.061 ms stddev 220.969, 0 failed
progress: 12.0 s, 15.5 tps, lat 511.660 ms stddev 256.096, 0 failed
progress: 18.0 s, 16.5 tps, lat 481.497 ms stddev 267.272, 0 failed
progress: 24.0 s, 15.0 tps, lat 540.619 ms stddev 339.608, 0 failed
progress: 30.0 s, 13.2 tps, lat 578.246 ms stddev 432.171, 0 failed
progress: 36.0 s, 16.3 tps, lat 506.958 ms stddev 311.169, 0 failed
progress: 42.0 s, 18.3 tps, lat 441.826 ms stddev 241.554, 0 failed
progress: 48.0 s, 15.2 tps, lat 525.453 ms stddev 277.492, 0 failed
progress: 54.0 s, 22.8 tps, lat 356.833 ms stddev 215.767, 0 failed
progress: 60.0 s, 24.2 tps, lat 328.966 ms stddev 202.329, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 1081
number of failed transactions: 0 (0.000%)
latency average = 444.678 ms
latency stddev = 285.469 ms
initial connection time = 54.350 ms
tps = 17.949766 (without initial connection time)

```

* Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
командой создаю конфиг файл, который применится при запуске кластера
так же меняю владельца этого конфиг файла

```
sudo echo 'max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB' | sudo tee /etc/postgresql/15/task_01_007/conf.d/123.conf
sudo chown postgres:postgres /etc/postgresql/15/task_01_007/conf.d/123.conf
```

* перезапуск кластера, проверка, что параметры применились 

```
postgres=# SHOW work_mem \g
 work_mem
 ----------
  6553kB
  (1 row)
```

* повторный запуск теста с примененными настроками дал такой результат:

```
starting vacuum...end.
progress: 6.0 s, 23.3 tps, lat 327.903 ms stddev 248.790, 0 failed
progress: 12.0 s, 16.5 tps, lat 484.650 ms stddev 240.128, 0 failed
progress: 18.0 s, 16.0 tps, lat 495.572 ms stddev 262.148, 0 failed
progress: 24.0 s, 14.5 tps, lat 540.659 ms stddev 291.640, 0 failed
progress: 30.0 s, 16.8 tps, lat 452.927 ms stddev 248.005, 0 failed
progress: 36.0 s, 14.2 tps, lat 592.558 ms stddev 375.150, 0 failed
progress: 42.0 s, 16.8 tps, lat 480.807 ms stddev 276.046, 0 failed
progress: 48.0 s, 20.2 tps, lat 408.048 ms stddev 262.348, 0 failed
progress: 54.0 s, 24.5 tps, lat 323.355 ms stddev 192.592, 0 failed
progress: 60.0 s, 19.2 tps, lat 399.532 ms stddev 216.052, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 1100
number of failed transactions: 0 (0.000%)
latency average = 437.245 ms
latency stddev = 273.106 ms
initial connection time = 51.866 ms
tps = 18.248992 (without initial connection time)


```



* Что изменилось и почему?
мало что изменилось, т.к. диск HDD и параметры random_page_cost =4 для HDD\
проверяю, что все кластеры остановлены\
```
pg_lsclusters
Ver Cluster     Port Status Owner     Data directory                    Log file
15  main        5433 down   <unknown> /var/lib/pg_ext_10/15/main        /var/log/postgresql/postgresql-15-main.log
15  task_01_005 5434 down   postgres  /var/lib/pg_ext_10/15/task_01_005 /var/log/postgresql/postgresql-15-task_01_005.log
15  task_01_007 5436 down   postgres  /var/lib/pg_ext_10/15/task_01_007 /var/log/postgresql/postgresql-15-task_01_007.log
15  task_02_006 5435 down   postgres  /var/lib/pg_ext_10/15/task_02_006 /var/log/postgresql/postgresql-15-task_02_006.log
16  first       5432 down   postgres  /var/lib/postgresql/16/first      /var/log/postgresql/postgresql-16-first.log
```

подключаю раздел к ВМ с NVME диска\
вычисляю имя подключенного устройства, создаю раздел, форматирую ФС, определаю UUID форматированного раздела, временно монтирую в каталог /mnt, перемещаю все каталоги папки "15" на этот раздел, отключаю раздел и временно монтирую в каталог кластеров
```
lsblk -fs
sudo fdisk /dev/sdb
sudo mount  UUID="24229417-5e72-4044-9c39-9c62eff40ecc" /mnt
sudo mv /var/lib/pg_ext_10/15 /mnt
sudo umount /mnt
sudo mount  UUID="24229417-5e72-4044-9c39-9c62eff40ecc" /var/lib/pg_ext_10/
```

чтобы конфиг не применялся, переименовываю файл командой, перезапускаю кластер и повторяю тест
```
sudo mv /etc/postgresql/15/task_01_007/conf.d/123.conf /etc/postgresql/15/task_01_007/conf.d/124.not_use
```
получил результат:

```
starting vacuum...end.
progress: 6.0 s, 693.2 tps, lat 11.430 ms stddev 5.963, 0 failed
progress: 12.0 s, 699.0 tps, lat 11.446 ms stddev 5.814, 0 failed
progress: 18.0 s, 617.2 tps, lat 12.961 ms stddev 7.875, 0 failed
progress: 24.0 s, 661.3 tps, lat 12.096 ms stddev 6.548, 0 failed
progress: 30.0 s, 656.5 tps, lat 12.185 ms stddev 6.527, 0 failed
progress: 36.0 s, 624.7 tps, lat 12.801 ms stddev 7.739, 0 failed
progress: 42.0 s, 648.2 tps, lat 12.347 ms stddev 10.733, 0 failed
progress: 48.0 s, 657.3 tps, lat 12.167 ms stddev 6.431, 0 failed
progress: 54.0 s, 664.0 tps, lat 12.039 ms stddev 6.211, 0 failed
progress: 60.0 s, 676.5 tps, lat 11.833 ms stddev 5.958, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 39595
number of failed transactions: 0 (0.000%)
latency average = 12.114 ms
latency stddev = 7.107 ms
initial connection time = 50.419 ms
tps = ~~660.230827~~ (without initial connection time)

```


####################################################################################

Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
Посмотреть размер файла с таблицей
5 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
Подождать некоторое время, проверяя, пришел ли автовакуум
5 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть размер файла с таблицей
Отключить Автовакуум на конкретной таблице
10 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть размер файла с таблицей
Объясните полученный результат
Не забудьте включить автовакуум)
####################################################################################




4. переключился на созданную БД
```
\c tesdb \g
```

5. создал схему командой
```
CREATE SCHEMA testnm \g
```

6. создал таблицу t1 с одной колонкой c1 типа integer
```
testdb=# CREATE TABLE t1 (c1 integer)\g
```

7. вставлена строка со значением c1=1
```
testdb=# INSERT INTO t1 values (1)\g

```

8. создана роль командой
```
testdb=# CREATE ROLE readonly \g

```

9. 10. 11. роли readonly выдано право подключаться к БД, использвать схему и право выполнять SELECT последовательностью команд
```
testdb=# GRANT CONNECT on DATABASE testdb to readonly \g
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly \g
testdb=# GRANT SELECT on ALL TABLES IN SCHEMA testnm TO readonly \g

```


12. 13. создан пользователь и назначена роль командами
```
testdb=# CREATE USER testread WITH PASSWORD 'test123' \g
testdb=# GRANT readonly TO testread  \g

```

14. подключиться к БД testread пользователем testdb невозможно, т.к. в файле pg_hba.conf нет разрешающего правила\
командой в операционной системе открыл на редактирование файл
```
sudo mcedit /etc/postgresql/15/task_01_005/pg_hba.conf
```

добавил строку, разрешение подключения пользователя readonly к БД test с паролем
```
local   testdb          readonly                                md5
```

перезапуск кластера и возможно подключиться к БД командой с явным указанием необходимости ввода пароля, имени БД, к какой подключаться 
```
sudo pg_ctlcluster 15 task_01_005 restart
psql -U testread -p 5434 -W -d testdb -h 127.0.0.1
```


15. выполнить ```SELECT * FROM t1 \g  ``` т.к. таблица testdb создана по-умолчанию в схеме public, на эту схему у роли readonly прав доступа нет

16,20,22,23,24,25.  отключиться от СУБД пользователем testread, подключиться пользователем postgres, ~~удалить таблицу t1, создать таблицу в нужной схеме~~ так было написано в шпаргалке, но я изменил схему для таблицы командой

```
\q
ub-cent@sad:~$ sudo -u postgres psql -p 5434
postgres=# \c testdb
testdb=# ALTER TABLE public.t1 SET SCHEMA testnm
testdb=#\dt public.*   --пусто
testdb=#\dt testnm.*   --отображает нашу таблицу
```

17,18,19. идей нет


26,27. пользователем testread при попытке получить данные из таблицы t1 БД testdb так же permission denied


выполнил повторно, т.к. таблицу ранее все-таки удалял, это новый объект
```
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly \g
testdb=# GRANT SELECT on ALL TABLES IN SCHEMA testnm TO readonly \g

```
после этого пользователь readonly получил значения из таблицы t1


37,38,39,40,41,42. команда не выполнилась, т.к. согласно шпаргалки убрал право создания для схемы public для роли public
а так же убрал все неявные права на БД testdb для роли Public
```
REVOKE CREATE on SCHEMA public FROM public; 
REVOKE ALL on DATABASE testdb FROM public; 
```

