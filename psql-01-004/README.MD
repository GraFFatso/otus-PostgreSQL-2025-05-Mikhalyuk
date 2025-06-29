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
