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



1. согласно пошаговой интсрукции создаю новый кластер командой (имя кластера содержит месяц занятий и номер занятия)
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

2. подклчюение к СУБД
```
sudo -u postgres psql -p 5434
```


3. создал БД
``
CREATE DATABASE testdb\gx
``
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

17. 18. 19. идей нет


26. 27. пользователем testread при попытке получить данные из таблицы t1 БД testdb так же permission denied


выполнил повторно, т.к. таблицу ранее все-таки удалял, это новый объект
```
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly \g
testdb=# GRANT SELECT on ALL TABLES IN SCHEMA testnm TO readonly \g

```
после этого пользователь readonly получил значения из таблицы t1


37. 38. 39. 40. 41. 42.команда не выполнилась, т.к. согласно шпаргалки убрал право создания для схемы public для роли public
а так же убрал все неявные права на БД testdb для роли Public
```
REVOKE CREATE on SCHEMA public FROM public; 
REVOKE ALL on DATABASE testdb FROM public; 
```

