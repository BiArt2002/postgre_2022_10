# postgre_2022_10

**6-я лекция**

ВМ с ОС Ubuntu 22.04 LTS у меня уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@84.201.143.205

Установил PostgreSQL 14.

Проверяю, что кластер PG запущен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

Зашел из-под пользователя postgres в psql, командная строка: 
	
	artem@postgre-2022-10-new:~$ sudo -u postgres psql

Пароль почему-то не был запрошен. Вывод в консоли:
	
	postgres=#

Посмотрел список БД. Командная строка:
	
	postgres-# \l
	
Их три - postgres, template0, template1.

Создаю БД test, командная строка: 

	postgres=# CREATE DATABASE test;
	postgres=# \c test
	
Вывод в консоли:

	You are now connected to database "test" as user "postgres".
	test=#
	
Создаю произвольную таблицу tbl и в ней 3 произвольные записи. Командная строка:  

	test=# CREATE TABLE tbl (num serial, name varchar);
	test=# INSERT INTO tbl(name) VALUES ('x1');
	test=# INSERT INTO tbl(name) VALUES ('y2');
	test=# INSERT INTO tbl(name) VALUES ('z3');
	test=# SELECT * FROM tbl;

Вывод в консоли:

	num | name
	-----+------
	1 | x1
	2 | y2
	3 | z3
	(3 rows)
	
Вышел из psql обратно в командную строку ubuntu командой exit.

Остановливаю кластер PG. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo -u postgres pg_ctlcluster 14 main stop
	
Кластер остановился, в консоли предупреждение:

	Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
	sudo systemctl stop postgresql@14-main

Поставил себе Midnight Commander, чтоб удобнее было ходить по каталогам. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo add-apt-repository universe
	artem@postgre-2022-10-new:~$ sudo apt update
	artem@postgre-2022-10-new:~$ sudo apt install mc
	
Отключаюсь.

Остановил в облаке Яндекса ВМ.
Cоздал в облаке Яндекса новый standard persistent диск на 10 ГБ с именем vlm1. Раздел Compute Cloud/Диски
Присоединил этот диск к остановленной ВМ postgre-2022-10-new через меню в графическом интерфейсе раздела Compute Cloud/Диски.
Запустил в облаке Яндекса ВМ.
Подключился из PowerShell к ВМ в облаке Яндекса.

Далее следовал инструкциям Яндекса  https://cloud.yandex.ru/docs/compute/operations/vm-control/vm-attach-disk

Проверил, подключен ли диск как устройство и узнал его путь в системе. Командная строка:  

	artem@postgre-2022-10-new:~$ lsblk
	
Увидел диск vdb на 10 ГБ.
Разметил этот диск. Для этого создал на нем разделы с помощью утилиты fdisk. Командная строка:  

	artem@postgre-2022-10-new:~$ sudo fdisk /dev/vdb
	
Далее в меню программы fdisk делал по инструкции.
В итоге получил инфу по размеченному диску.
Вывод в консоли:

	Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors

	Device     Boot Start      End  Sectors Size Id Type
	/dev/vdb1        2048 20971519 20969472  10G 83 Linux

Отформатировал диск в файловую систему ext4 с помощью утилиты mkfs. Командная строка:

	artem@postgre-2022-10-new:~$ sudo mkfs.ext4 /dev/vdb1
	
Смонтировал раздел диска с помощью утилиты mount - раздел vdb1 в папку /mnt/vdb1. Командная строка:

	artem@postgre-2022-10-new:~$ sudo mkdir /mnt/vdb1
	artem@postgre-2022-10-new:~$ sudo mount /dev/vdb1 /mnt/vdb1
	
Проверил в Midnight Commander - диск примонтирован.
Сделал пользователя postgres владельцем /mnt/vdb1. Командная строка:

	artem@postgre-2022-10-new:~$ sudo chown -R postgres:postgres /mnt/vdb1
	
Перезагрузил инстанс ВМ и убедился, что диск остался примонтированным.
Проверил, что кластер PG запущен.
Перенес содержимое /var/lib/postgres/14 в /mnt/vdb1. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo mv /var/lib/postgresql/14 /mnt/vdb1

Проверил,  что в /mnt/vdb1 появился каталог 14.
Попытался запустить кластер PG. Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres pg_ctlcluster 14 main start
	
Вывод в консоли:
	
	Error: /var/lib/postgresql/14/main is not accessible or does not exist
	
Логично. Ведь /14/main теперь лежит тут: /mnt/vdb1/14/main

Стал смотреть конфигурационные файлы, раположенные в /etc/postgresql/14/main
В файле postgresql.conf с помощью встроенного редактора Midnight Commander попробовал изменил параметр data_directory 
со старого значения  /var/lib/postgresql/14/main на новое  /mnt/vdb1/14/main 
Сохранить изменения не удалось - запрещено.
Сделал с помощью редактора nano. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo nano /etc/postgresql/14/main/postgresql.conf

Параметр data_directory изменился в файле postgresql.conf после сохранения.	
Попытался запустить кластер PG. Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres pg_ctlcluster 14 main start
	
Вывод в консоли:

	Warning: the cluster will not be running as a systemd service. Consider using systemctl:
	sudo systemctl start postgresql@14-main
	Cluster is already running.	
	
Остановил PG. Командная строка: 	
	
	artem@postgre-2022-10-new:~$ sudo systemctl stop postgresql
	
Запустил PG. Командная строка: 	

	artem@postgre-2022-10-new:~$ sudo systemctl start postgresql
			
Зашел из-под пользователя postgres в psql, командная строка:
	
	artem@postgre-2022-10-new:~$ sudo -u postgres psql
	
Проверил значение каталога данных, командная строка:

	postgres=# SHOW data_directory;
	
Вывод в консоли:

	data_directory
	-------------------
	/mnt/vdb1/14/main
	(1 row)
	
Всё получилось, данные теперь в /mnt/vdb1/14/main
Перешел в свою базу test. Командная строка:

	postgres=# \c test
	
Проверил наличие в базе test своей ранее созданной таблицы tbl, в которой было 3 записи. Командная строка:  

	test=# SELECT * FROM tbl;
	
Вывод в консоли:

	num | name
	-----+------
	1 | x1
	2 | y2
	3 | z3
	(3 rows)
	
Таблица tbl с ее данными на месте.
Отключился от ВМ.