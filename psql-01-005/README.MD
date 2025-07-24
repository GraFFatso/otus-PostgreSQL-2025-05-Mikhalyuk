## Занятие 5. ЛОгический уровень PostgreSQL
### Преподаватель Антон Золотов
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


### Цель занятия

иметь представление об логическом урстройстве Postgresql;\
рассмотреть работу утилиты psql;\
понять, как PostgreSQL работакет с данным на логическом уровне.


### Выполнение ДЗ

из предудущего ДЗ используется ubuntu,\
при запуске ВМ не запустилась из-за поврежденной файловой системы,\
запуск ВМ в режиме обслуживания,\

исправление ошибок файловой системы командой
```
fsck /dev/mapper/ub...
```
*повезло, ВМ запустилась (повреждение ФС из-за отключений электроэнергии, ИБП нет)



* согласно пошаговой интсрукции создаю новый кластер командой (имя кластера содержит месяц занятий и номер занятия)
```
sudo pg_createcluster 15 task_01_005
```
т.к.  в предыдущих заданиях я задал параметры, с которыми будет инициализироватьсяновый кластер через содержимое файла
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
15  task_01_005 5434 down   postgres /var/lib/pg_ext_10/15/task_01_005 /var/log/postgresql/postgresql-15-task_01_005.log
```

запуск кластера
```
sudo pg_ctlcluster 15 task_01_005
```

подклчюение к СУБД
```
sudo -u postgres psql -p 5434
```


создал БД
``
CREATE DATABASE testdb\gx
``
*переключился на созданную БД и создал таблицу
```
\c tesdb \g
```

создал схему командой
```
CREATE SCHEMA testnm \g
```

создал таблицу t1 с одной колонкой c1 типа integer
```
testdb=# CREATE TABLE t1 (c1 integer)\g
```

вставлена строка со значением c1=1
```
testdb=# INSERT INTO t1 values (1)\g

```

создана роль командой
```
testdb=# CREATE ROLE readonly \g

```

роли readonly выдано право подключаться к БД, использвать схему и право выполнять SELECT последовательностью команд
```
testdb=# GRANT CONNECT on DATABASE testdb to readonly \g
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly \g
testdb=# GRANT SELECT on ALL TABLES IN SCHEMA testnm TO readonly \g

```





otus01004=# \d+
List of relations
Schema |    Name    | Type  |  Owner   | Persistence | Access method |    Size    | Description
--------+------------+-------+----------+-------------+---------------+------------+-------------
public | some_talbe | table | postgres | permanent   | heap          | 8192 bytes |
                                   (1 row)
otus01004=# INSERT  INTO some_talbe VALUES ('firts_text')\gx
INSERT 0 1
otus01004=# SELECT * FROM some_talbe \gx
-[ RECORD 1 ]+-----------
some_columnt | firts_text
```
* остановил кластер
```
sudo pg_ctlcluster 15 main stop
sudo pg_ctlcluster 16 first stop
```
* остановлен контейнер docker и служба docker, т.к. на этом ДЗ не нужна
```
sudo docker stop 
sudo systemctl docker stop
sudo systemctl docker disable
```
* выключена ВМ ubuntu
* добавлен диск размером 10ГБ на HDD
* включена ВМ, проверка наличия диска
```
lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
fd0                         2:0    1    4K  0 disk
sda                         8:0    0   10G  0 disk
sdb                         8:16   0  127G  0 disk
├─sdb1                      8:17   0    1M  0 part
├─sdb2                      8:18   0    2G  0 part /boot
└─sdb3                      8:19   0  125G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0 62.5G  0 lvm  /
```
> блочное устройытво sda размером 10ГБ существует

* создание первисчного раздела 
```
sudo fdisk /dev/sda
```
n - создать новый раздел\
p - первичный раздел\
w - записать изменения и выйти из утилиты

* форматирование созданного раздела
```
sudo mkfs /dev/sda1
```
* при поиске UID раздела увидел, что отформатировался раздел в ext2, желательно ext4
```
lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
NAME                      FSTYPE      LABEL UUID                                   MOUNTPOINT
fd0
sda
└─sda1                    ext2              4f398715-5717-4264-84ff-170e8c6dbc83
sdb
├─sdb1
├─sdb2                    ext4              24a2333e-4d26-4c75-9bd1-8054e820bcf2   /boot
└─sdb3                    LVM2_member       eKCLNI-h3PL-kpwC-K51x-k8MJ-vYqs-v7Sxxw
  └─ubuntu--vg-ubuntu--lv ext4              16200388-af5e-4392-a83d-e67ef50fc83f   /
```
* отформатировал в нужную файловую систему
```
sudo mkfs.ext4 /dev/sda1
```

тогда у раздела получился другой UID

```
lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
NAME                      FSTYPE      LABEL UUID                                   MOUNTPOINT
fd0
sda
└─sda1                    ext4              2fcb8a2d-12f4-4e4c-b822-893909f953ab

```

* т.к. существующие каталоги кластеров располагаются в директории _/var/lib/postgresql_,\
то создаю новую директорию и в fstab прописываю путь мониторования к этой директории

```
sudo mkdir /var/lib/pg_ext_10
```

* в /etc/fstab добавил строку
```
LABEL=PGSQL_ext_part /var/lib/pg_ext_10 ext4 defaults 0 2
```
> ошибка, т.к. я LABEL не задал для раздела, тогда в mainetance режиме закомментировал строку мониторвоания и перезагрузившись в нормальном режиме задал верную строку монтирования раздела

```
UUID=2fcb8a2d-12f4-4e4c-b822-893909f953ab /var/lib/pg_ext_10 ext4 defaults 0 2
```

* после перезагрузки
```
lsblk -fs
NAME                  FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
fd0
sda1                  ext4        1.0            2fcb8a2d-12f4-4e4c-b822-893909f953ab      9.2G     0% /var/lib/pg_ext_10
```
* скопировал каталог 15 версии кластер main
` sudo cp /var/lib/postgresql/15 /var/lib/pg_ext_10/`
 
 * назначил владельцем каталога рекурскивно пользователя postgres
 
` sudo chown postgres:postgres -R /var/lib/pg_ext_10/`

* запустив кластер 15 main узнал путь к конфиг файлу этого кластера, затем остановил кластер
```
ps aux | grep 'post'
postgres     901  0.0  0.3 216732 30000 ?        Ss   12:25   0:00 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
```

* командой выяснил как выглядит строка, в которой необходимо изменить путь к каталогу кластера
```
sudo grep -R '/var/lib/' /etc/postgresql/15/ --color
/etc/postgresql/15/main/postgresql.conf:data_directory = '/var/lib/postgresql/15/main'          # use data in another directory
```

* командой изменил путь к каталогу кластера СУБД 15 main
```
sudo sed -i.bak 's/var\/lib\/postgresql\/15\/main/var\/lib\/pg_ext_10/g' /etc/postgresql/15/main/postgresql.conf
```

* ошибся, указав не полный путь к каталогу кластера, изменил занчение командой, заменив / на #, чтобы не заниматься экранированием слэша
```
sudo sed -i.bak2 's#var/lib/pg_ext_10#var/lib/pg_ext_10/15/main#g' /etc/postgresql/15/main/postgresql.conf
```

* успешное подключение командой и отображение созданной БД и записи...

`sudo -u postgres psql -p 5434`
```
\l
\c otus01004
\d
```

### задание со сзвездочкой
раздел с каталогом кластера переключить от ВМ ubuntu к ВМ centos, запустить кластер и увидеть имеющиеся в СУБД БД и записи

* в centos при установке "ванильного" postgreslq 15 создался каталог /var/lib/pgsql/15/data - не нужен
* инициализирую кластер с настройками по-умолчанию командой тоже не нужно для запуска СУБД

`sudo -u postgres /usr/pgsql-15/bin/initdb -D /var/lib/pgsql/15/data/`

* удалил содерджимое проинициализировавшегося каталога
`rm -rf /var/lib/pgsql/15/data/*`

* в ВМ ubuntu добавил локаль, которая сущесвтует в centos

```
 sudo locale-gen ru_RU.utf8
 reboot postgresql
```
* конфиг файлы кластера 
```
pg_hba.cong
posgresql.conf
```
из каталога 
`/etc/postgres/15/main`

скопировал в каталог `/var/lib/pg_ext_10/settings`

* в ВМ ubuntu в файле `/etc/fstab` закомментировал строку монтирования диска, выклчюил ВМ ubuntu, отключил диск от ВМ
* ВМ centos выключил, подключил диск с кластером postgresql 15, включил ВМ

```
 lsblk -fs
 
 sdd1        ext4              2fcb8a2d-12f4-4e4c-b822-893909f953ab
 └─sdd
```
по имени устройства смонтировал диск в одноименную папку
`mount /dev/sdd1 /var/lib/pg_ext_10`

* поменял владельца подкаталогов и файлов на пользователя centos
`chown postgres:postgres -R  /var/lib/pgsql/15/data/`

* из каталога settings в каталог кластера скопировал два конфиг файла, которых достаточно для запуска кластера
* изменил локаль с C.UTF-8 на ru_RU.UTF-8 в конфиг файле 
```
sed -i 's/C.UTF-8/ru_RU.UTF-8/g' /var/lib/pg_ext_10/15/main/postgresql.conf
```

* отключил ssl, т.к. не принес сертификаты Ubuntu
```
sed -i 's/ssl = on/#ssl = on/g' /var/lib/pg_ext_10/15/main/postgresql.conf
```


* запустил службу командой
```
sudo -u postgres /usr/pgsql-15/bin/pg_ctl  -D /var/lib/pg_ext_10/15/main/ stop
```

* были предупреждения, избавился командой  для каждой из БД
```
 ALTER DATABASE template1 REFRESH COLLATION VERSION \gx
 ```

* базы данных существуют, данные в таблицах есть





