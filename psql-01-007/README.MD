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
...initial connection time = 54.350 ms
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
...
initial connection time = 51.866 ms
tps = 18.248992 (without initial connection time)
```



* **Что изменилось и почему?**\
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

подключаю раздел к ВМ с NVME диска.\
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
tps = 223.897541 (without initial connection time)

```

затем применяю конфиг файл с настройками, получаю другой результат:
```
...
number of transactions actually processed: 39595
number of failed transactions: 0 (0.000%)
latency average = 12.114 ms
latency stddev = 7.107 ms
initial connection time = 50.419 ms
tps = 660.230827 (without initial connection time)
```

изменение значения effective_io_concurrency с значения по-умолчанию в 1 на 2 позволяет СУБД выполнять большее число параллельных вводов/выводов\
так же изменение значения shared_buffers c дефолтного значения в 128 МБ на 1гб позволяет СУБД использовать больше памяти для буферов




* **Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк**
командами создал таблицу, вставил 1млн строк, получил размер(в байтах) таблицы на диске порследовательностью команд:
```
CREATE TABLE IF NOT EXISTS table_task (test_column integer) \g
INSERT INTO table_task (test_column) VALUES (generate_series(1,1000000))\g
SELECT pg_relation_size('table_task')\g
 pg_relation_size
 ------------------
          36249600
```



* **5 раз обновить все строчки и добавить к каждой строчке любой символ**
обновлял сожержимое ячеек строк командами типа\
т.к. создал таблицу со столбцом типа integer, то "добавление символа" реализовал умножением на чило, большее 10
```
UPDATE   table_task SET test_column =floor(random()*9) \g
UPDATE   table_task SET test_column =floor(random()*91) \g
```

проверяя, что меняется сождержимое ячеек запросом
```
 SELECT * FROM table_task LIMIT 10 \g
  test_column
  -------------
        18335
        19241
        9316
        86346
        9954
        83058
        7038
        79446
        25764
        50111
```










* **Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум**
проверял колво мертвых строк запросом
```
SELECT n_dead_tup  FROM pg_stat_user_tables WHERE relname = 'table_task' \gx
```


* **Подождать некоторое время, проверяя, пришел ли автовакуум**
зачастую не успевал увидеть кол-во мертвых строк
приходилось принудительно пару раз "UPDATE " выполнять, чтобы увидеть колво "мертвых строк"

####################################################################################
* **5 раз обновить все строчки и добавить к каждой строчке любой символ\
Посмотреть размер файла с таблицей**
размер файла таблицы на диске увеличился до 108 мб с 34мб

* **Отключить Автовакуум на конкретной таблице**


10 раз обновить все строчки и добавить к каждой строчке любой символ
Посмотреть размер файла с таблицей
Объясните полученный результат
Не забудьте включить автовакуум)
####################################################################################



