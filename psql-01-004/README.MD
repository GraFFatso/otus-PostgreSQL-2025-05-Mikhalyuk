## Занятие 4. Установка PostgreSQL
### Преподаватель Антон Золотов
### Проверяющий ДЗ Игорь Тоескин

## подготовительные работы, вводные

использую "стенд" с hyper-v, i3-6100 +40gb RAM\
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

* ВМ centos была установлена давно, установлена там СУБД postgresPRO  1c 14 версии на отдельный диск


``df``
```
/dev/sdb1               309503808     48354680 245404156           17% /var/lib/pgpro/1c-14
```
демон остановлен и отключен

```
systemctl disable postgrespro-1c-14
systemctl stop postgrespro-1c-14
```


* в ВМ ubuntu 22.04 установлены пакеты 15, 16 и 17 версий postgresql

* в ВМ  centos 7 установлены пакеты 15.13 версии postgresql, т.к. для centos 7 это самая высокая поддерживаемая версия

>https://www.postgresql.org/download/linux/redhat/


### Цель занятия

создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему\
переносить содержимое базы данных PostgreSQL на дополнительный диск\
переносить содержимое БД PostgreSQL между виртуальными машинами

### Выполнение ДЗ

*из предудущего ДЗ используется ubuntu, кластер 15 версии

* т.к. кластер 15 версии работает на нестандартном порту, то подключаться к СУБД командой
```
sudo -u postgres psql -p 5434
```
создал еще одну БД (номер месяца+номерДЗ)
``
CREATE DATABASE otus01004\gx
``
*переключился на созданную БД и создал таблицу
```
\c otus01004 \gx
otus01004=# CREATE TABLE some_talbe (some_columnt text)\gx
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
```
sudo fdisk /dev/sda
```
n - создать новый раздел\
p - первичный раздел\
w - записать изменения и выйти из утилиты

```
sudo mkfs /dev/sda1
```

