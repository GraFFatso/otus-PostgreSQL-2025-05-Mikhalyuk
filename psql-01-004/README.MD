## Занятие 4. Установка PostgreSQL
### Преподаватель АНтон Золотов
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
установилось новое ядро Linux 5.15.0

* 

* в ВМ ubuntu 22.04 установлены пакеты 15, 16 и 17 версий postgresql

* в ВМ  centos 7 установлены пакеты 15.13 версии postgresql, т.к. для centos 7 это самая высокая поддерживаемая версия

>https://www.postgresql.org/download/linux/redhat/


### Цель занятия

создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему\
переносить содержимое базы данных PostgreSQL на дополнительный диск\
переносить содержимое БД PostgreSQL между виртуальными машинами

### Выполнение ДЗ


     оздал БД в кластере 15 версии
     sudo -u postgres psql CRATE DATABASE otus \g
     
     насоздалвал в БД объектов
     sudo -u postgres pgbench -i --scale=10  --port=5432 otus