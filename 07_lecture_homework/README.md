# postgre_2022_10

**7-я лекция**

ВМ с ОС Ubuntu 22.04 LTS у меня уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@84.201.162.154

Установил PostgreSQL 14.

Проверяю, что кластер PG запущен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory    Log file
	14  main    5432 online postgres /mnt/vdb1/14/main /var/log/postgresql/postgresql-14-main.log

Зашел из-под пользователя postgres в psql, командная строка: 
	
	artem@postgre-2022-10-new:~$ sudo -u postgres psql
	
Пароль почему-то не был запрошен. Вывод в консоли:
	
	postgres=#

Создал БД testdb и зашел под пользователем postgres, командная строка: 

	postgres=# CREATE DATABASE testdb;
	postgres=# \c testdb
	
Вывод в консоли:

	You are now connected to database "testdb" as user "postgres".
	testdb=#
	
Создал новую схему testnm, командная строка: 

	testdb=# CREATE SCHEMA testnm;
	
Вывод в консоли:

	CREATE SCHEMA
	testdb=#

Создал таблицу t1 с одной колонкой c1 типа integer. Вставил строку со значением c1 = 1. Командная строка:  

	testdb=# CREATE TABLE t1 (c1 integer);
	testdb=# INSERT INTO t1 (c1) VALUES (1);
	testdb=# SELECT * FROM t1;

Вывод в консоли:

	c1
	----
	1
	(1 row)	
	
Создал новую роль readonly. Командная строка:  

	testdb=# CREATE role readonly;
	
Дал роли readonly право на подключение к базе данных testdb. Командная строка:  

	testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
	
Дал роли readonly право на использование схемы testnm. Командная строка: 

	testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
	
Дал роли readonly право на select для всех таблиц схемы testnm. Командная строка:
	
	testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
	
Создал пользователя testread с паролем test123. Командная строка:

	testdb=# CREATE USER testread PASSWORD 'test123';
	
Назначил роль readonly пользователю testread. Командная строка:

	testdb=# GRANT readonly TO testread;
	
Открыл второе подключение из PowerShell к ВМ в облаке Яндекса.
Зашел под пользователем testread в базу данных testdb. Командная строка:

	postgres=# \c testdb testread
	
Не получилось.
Вывод в консоли:
	
	connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
	Previous connection kept
	
Зашел иначе с подключением по сети. Командная строка:

	artem@postgre-2022-10-new:~$ psql -h 127.0.0.1 -U testread -d testdb -W
	
Сделал select из таблицы t1, получил ошибку. Командная строка:

	testdb=> select * from t1;
	
Вывод в консоли:

	ERROR:  permission denied for table t1
	
Логично. Таблица t1 создалась в схеме public, а вовсе не в схеме testnm, на которую давали права. На схему public прав нет.
Если бы наша схема testnm совпадала по названию с нашим юзером, то таблица t1 создалась в схеме testnm (когда create table без явного указания схемы). Но нет. 
Посмотрел список  таблиц в базе testdb, таблица t1 принадлежит схеме public. Командная строка: 

	testdb=> \dt
	
Вывод в консоли:

			List of relations
	Schema | Name | Type  |  Owner
	--------+------+-------+----------
	public | t1   | table | postgres
	(1 row)
	
Если бы таблицу t1 изначально создали в схеме testnm (указав схему явно), то было бы ОК.

Вернулся в первое окне подключения из PowerShell к ВМ в облаке Яндекса.
Удалил таблицу t1. Командная строка:

	testdb=# DROP TABLE t1;
	
Создал в схемы testnm таблицу t1 с одной колонкой c1 типа integer. Вставил строку со значением c1 = 1. Командная строка: 

	testdb=# CREATE TABLE testnm.t1 (c1 integer);
	testdb=# INSERT INTO testnm.t1 (c1) VALUES (100);
	testdb=# SELECT * FROM testnm.t1;

Вывод в консоли:

	 c1
	-----
	100
	(1 row)
	
Вновь зашел под пользователем testread в базу данных testdb (второе окно с подключением). Командная строка:

	artem@postgre-2022-10-new:~$ psql -h 127.0.0.1 -U testread -d testdb -W

Сделал select из таблицы t1, получил ошибку. Командная строка:

	testdb=> select * from testnm.t1;

Вывод в консоли:

	ERROR:  permission denied for table t1
	 
Странно. Попытался снова дать роли readonly право на select для всех таблиц схемы testnm (первое окно с подключением). Командная строка:

	testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;

Во втором окне с подключением вышел из базы testdb, опять вошел под пользователем testread.
Сделал select из таблицы t1, получил ошибку. Командная строка:

	testdb=> select * from testnm.t1;
	
Вывод в консоли:

	 c1
	-----
	100
	(1 row)

Получилось. Значит, когда таблицу t1 дропнули и вновь воздали, права "потерялись"? Видимо, так.	

Создал таблицу t2 с одной колонкой c1 типа integer. Вставил строку со значением c1 = 2. Командная строка:  

	testdb=# CREATE TABLE t2 (c1 integer);
	testdb=# INSERT INTO t2 (c1) VALUES (2);
	testdb=# SELECT * FROM t2;
	
Вывод в консоли:

	c1
	----
	2
	(1 row)
	
Все ок. Однако прав на создание таблиц и insert в них под ролью readonly нет. Только на селект.
Можно проверить права самого юзера testread. Командная строка:   

	SELECT * from information_schema.table_privileges WHERE grantee = 'testread';
	
Вижу, что в схеме public для юзера testread есть права на создание, вставку, удаление, апдейт... 
А ведь таблица t2 создалась в схеме public. Посмотрим. Командная строка: 

	testdb=> \dt

			List of relations
	Schema | Name | Type  |  Owner
	--------+------+-------+----------
	public | t2   | table | testread
	(1 row)	
	
Так и есть. Надо бы убрать права для роли public в схеме public. Т.к. роль public дается каждому новому юзеру в любой базе, если юзер может подключатсья к этой базе.
Вернулся к первому окну с подключением. Отзываю права. Командная строка:

	testdb=# REVOKE CREATE ON SCHEMA public FROM public;
	testdb=# REVOKE ALL ON DATABASE testdb FROM public;

Вернулся ко второму окну с подключением.
Пробую создать таблицу t3 с одной колонкой c1 типа integer. Командная строка:  
	
	testdb=> CREATE TABLE t3 (c1 integer);

Вывод в консоли:

	ERROR:  permission denied for schema public
	
Прав на создание в схеме public у роли public больше нет.
	
Пробую вставить в созданную ранее таблицу t2 новую строку со значением c1 = 3. Командная строка:  

	testdb=> INSERT INTO t2 (c1) VALUES (3);
	testdb=> SELECT * FROM t2;
	
Вывод в консоли:
	
	 c1
	----
	2
	3
	(2 rows)
	
Права на использование схемы public остались у роли public, которая также есть у юзера testread. Поэтому все ок.						

Отключился от ВМ.