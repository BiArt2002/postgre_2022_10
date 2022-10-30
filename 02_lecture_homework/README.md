# postgre_2022_10

**2-я лекция**

Создал ВМ с именем postgre-2022-10 в облаке Яндекса.
Характеристики ВМ: платформа Intel Ice Lake, 2 vCPU, 50% vCPU, RAM 4 ГБ, HDD 15 ГБ, ОС Ubuntu 22.04 LTS.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Законнектился к ВМ с помощью PuTTY (в ней же заранее сгенерил SSH).

Добавил репозиторий PostgreSQL и некотрые пакеты, следуя инструкциям на сайте  https://computingforgeeks.com/install-postgresql-14-on-ubuntu-jammy-jellyfish/

Установил на ВМ PostgreSQL 14, командная строка:

	artem@postgre-2022-10:~$ sudo apt install postgresql-14

Убедился, что кластер стартовал, командная строка:

	artem@postgre-2022-10:~$ pg_lsclusters
	
Вывод в консоли:

	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

Дополнительно поставил PostgreSQL 15, командная строка:

	artem@postgre-2022-10:~$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15

Убедился, что кластер стартовал, командная строка:

	artem@postgre-2022-10:~$ pg_lsclusters
	
Вывод в консоли:

	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
	15  main    5433 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

Посмотрел, какой метод шифрования пароля у PostgreSQL 14 и 15, командная строка:

	artem@postgre-2022-10:~$ sudo cat /etc/postgresql/14/main/pg_hba.conf
	artem@postgre-2022-10:~$ sudo cat /etc/postgresql/15/main/pg_hba.conf
	
Убедился, что у обоих PG метод шифрования scram-sha-256

Стал тестить ssh, сразу не получилось. Пришлось сначала установить google-cloud-cli. Командная строка:

	artem@postgre-2022-10:~$ sudo snap install google-cloud-cli --classic
	
Опять начал тестить ssh, не получилось. Вывод в консоли:

	WARNING: Could not open the configuration file: [/home/artem/.config/gcloud/configurations/config_default].
	ERROR: (gcloud.beta.compute.instances.create) You do not currently have an active account selected.
	
Понял, что не смогу протестить ssh (как в ДЗ), т.к. облако у меня не в Google, а в Яндексе.

Далее выполнил, командная строка:

	artem@postgre-2022-10:~$ ssh artem@51.250.23.36

Вывод в консоли:
	
	The authenticity of host '51.250.23.36 (51.250.23.36)' can't be established.
	ED25519 key fingerprint is SHA256:bB/aUqyGY9maQOzzuZ+S1EZdvOXBMzFy4RKluHEwAWM.
	This key is not known by any other names
	Are you sure you want to continue connecting (yes/no/[fingerprint])?
	
Ответил yes. Вывод в консоли:

	Warning: Permanently added '51.250.23.36' (ED25519) to the list of known hosts.
	artem@51.250.23.36: Permission denied (publickey).
	
Поставил себе на компьютер Яндекс CLI.
В командной строке Виндовс написал:

	ssh artem@51.250.23.36
	
Вывод в консоли:
	
	The authenticity of host '51.250.23.36 (51.250.23.36)' can't be established.
	ECDSA key fingerprint is SHA256:Ahip0y9fjS53YJYf3Z2quFve+OsS1LvmOqKwMNkBeVM.
	Are you sure you want to continue connecting (yes/no/[fingerprint])?

Ответил yes. Вывод в консоли:

	Warning: Permanently added '51.250.23.36' (ECDSA) to the list of known hosts.
	artem@51.250.23.36: Permission denied (publickey).
	
Итого - ssh, сгенерированный PuTTY не очень-то подходит, надо было генерировать с помощью Яндекс CLI (как в инструкции на сайте Яндекс).
Способ изменить ssh на ВМ в документации Яндекса не нашел. В изменении настроек ВМ тоже нет смены ssh.

Подключился к ВМ вторым инстансом PuTTY. Перед этим останавливал ВМ (хотел менять настройки, не смог), и у ВМ изменился публичный IPv4.

Посмотрел на кластера, командная строка:

	artem@postgre-2022-10:~$ pg_lsclusters
		
Вывод в консоли:

	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
	15  main    5433 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
	
Остановил и удалил в первой консоли PG 14, командная строка:

	artem@postgre-2022-10:~$ sudo pg_ctlcluster 14 main stop
	artem@postgre-2022-10:~$ sudo pg_dropcluster 14 main
	
Во второй консоли посомтрел, что остался только PG 15.

	Ver Cluster Port Status Owner    Data directory              Log file
	15  main    5433 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

Создал PG 14 под пользователем postgres, командная строка:

	artem@postgre-2022-10:~$ sudo -u postgres pg_createcluster 14 main

Вывод в консоли:

	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

Запустил PG 14, командная строка: 

	artem@postgre-2022-10:~$ sudo pg_ctlcluster 14 main start

Убедился, что PG 14 стартовал. Status - online.

Выполнил команду, командная строка: 
	
	artem@postgre-2022-10:~$ sudo -u postgres psql
	
Вывод в консоли:

	could not change directory to "/home/artem": Permission denied
	psql (15.0 (Ubuntu 15.0-1.pgdg22.04+1), server 14.5 (Ubuntu 14.5-2.pgdg22.04+2))

Создал табличку для тестов, командная строка: 

	postgres=# CREATE DATABASE iso;
	postgres=# \c iso
	
Вывод в консоли:

	psql (15.0 (Ubuntu 15.0-1.pgdg22.04+1), server 14.5 (Ubuntu 14.5-2.pgdg22.04+2))
	You are now connected to database "iso" as user "postgres".
	
Посмотрел список БД, командная строка: 

	iso=# \l
	
Вывод в консоли:
                                                 	List of databases
   	Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
	-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 	iso       | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 	postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 	template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
     	      |          |          |             |             |            |                 | postgres=CTc/postgres
	 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
       	    |          |          |             |             |            |                 | postgres=CTc/postgres

Выбрал текущую базу, командная строка: 

	iso=# SELECT current_database();
		
Создал таблицу, вставил 2 записи, выбрал все записи из таблицы, командная строка:  

	iso=# CREATE TABLE test (i serial, amount int);
	iso=# INSERT INTO test(amount) VALUES (100);
	iso=# INSERT INTO test(amount) VALUES (500);
	iso=# SELECT * FROM test;
	
Вывод в консоли:
		
	i | amount
	---+--------
	1 |    100
	2 |    500
	(2 rows)

Посмотрел, что с автокоммитом и отключил его, командная строка:  
	
	iso=# \echo :AUTOCOMMIT
	iso=# \set AUTOCOMMIT OFF

Посмотрел, уровень изоляции транзакций, командная строка:   

	iso=# show transaction isolation level;

Вывод в консоли:
		
	transaction_isolation
	-----------------------
	read committed
	(1 row)
	
Последовательно установил разыне уровни изоляции, командная строка: 

	iso=*# set transaction isolation level read committed;
	iso=*# set transaction isolation level repeatable read;
	iso=*# set transaction isolation level serializable;
	
Посмотрел номер транзакции, командная строка: 
	
	iso=*# SELECT txid_current();

Вывод в консоли:

	txid_current
	--------------
			737
Включил автокоммит и опять посмотрел номер транзакции, командная строка: 

	iso=*# \set AUTOCOMMIT ON
	iso=*# SELECT txid_current();
	
Вывод в консоли:

	txid_current
	--------------
			737

Выполнил запрос, командная строка: 

	iso=*# SELECT * FROM test;
	iso=*# commit;
	
Посмотрел номер транзакции, он увеличился на 1 - стал 738

Выполнил запрос, командная строка:

	iso=# SELECT * FROM pg_stat_activity;
	
Вывод в консоли был очень большой, не записываю.

Стал тестировать в 1-й консоли TRANSACTION ISOLATION LEVEL READ COMMITTED, командная строка:

	iso=*# set transaction isolation level read committed;	
	iso=# BEGIN;
	iso=*# SELECT * FROM test;
	
Вывод в консоли:		

	i | amount
	---+--------
	1 |    100
	2 |    500

Запустил 2-ю консоль, выполнил запросы (один из них - апдейт таблицы), командная строка:

	artem@postgre-2022-10:~$ sudo -u postgres psql
	postgres=# \c iso
	iso=# BEGIN;
	iso=*# UPDATE test set amount = 555 WHERE i = 1;
	iso=*# COMMIT;
	iso=# SELECT * FROM test;

Вывод во 2-й консоли для селекта после апдейта:

	i | amount
	---+--------
	2 |    500
	1 |    555
	
Сделал такой же селект в 1-й консоли,  командная строка:	
Вижу, что данные в таблице в 1-й консоли совпадают с данными во 2-й консоли, т.к. был коммит и уровень изоляции READ COMMITTED. 

В 1-й консоли установил уровень изоляции для транзакции REPEATABLE READ, сделал селект, командная строка:

	iso=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
	iso=*# SELECT * FROM test;
	
Убедился, данные прежние. Суммы - 500 и 555.

Во 2-й консоли так же установил уровень изоляции для транзакции REPEATABLE READ, вставил строку в таблицу, командная строка:
	
	iso=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
	iso=*# INSERT INTO test VALUES (777);
	iso=*# COMMIT;

Во 2-й консоли сделал селект.
Вывод в консоли:

	i  | amount
	-----+--------
	2 |    500
	1 |    555
	777 |
	(3 rows)
  
Но в 1-й консоли селект дает прежние данные, т.к. уровень изоляции REPEATABLE READ.

	i | amount
	---+--------
	2 |    500
	1 |    555

В 1-й консоли создал новую тестовую таблицу, командная строка:

	iso=*# DROP TABLE IF EXISTS testS;
	iso=*# CREATE TABLE testS (i int, amount int);
	iso=*# INSERT INTO TESTS VALUES (1,10), (1,20), (2,100), (2,200);
	iso=*# COMMIT;
	
В 1-й консоли установил уровень изоляции для транзакции SERIALIZABLE, сделал селект и инсерт, командная строка:

	iso=# BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
	iso=*# SELECT sum(amount) FROM testS WHERE i = 1;
	iso=*# INSERT INTO testS VALUES (2,30);
	
Сумма по селекту была 30	
	
Во 2-й консоли установил уровень изоляции для транзакции SERIALIZABLE, сделал селект и другой инсерт, командная строка:
	iso=# BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
	iso=*# SELECT sum(amount) FROM testS WHERE i = 2;
	iso=*# INSERT INTO testS VALUES (1,300);

Сумма по селекту была 300

В 1-й консоли сделал коммит изменений. Всё ок.

Во 1-й консоли сделал коммит изменений. Получил ошибку, т.к. уровень изоляции SERIALIZABLE.

	ERROR:  could not serialize access due to read/write dependencies among transactions
	DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
	HINT:  The transaction might succeed if retried.
	
Вышел из PG, команда exit	
