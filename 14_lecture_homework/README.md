# postgre_2022_10

**14-я лекция**

ВМ с ОС Ubuntu 22.04 LTS - уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@51.250.97.147

PostgreSQL 14 уже есть на ВМ.

Преподаватель сказал, что можно сделать в ДЗ репликацию не на трех разных ВМ, а на одной, если взять разные кластеры PG.
Мне удобнее делать на одной ВМ. Поэтому сделал себе еще два кластера PG: "14 main2" и "14 main3".

	artem@postgre-2022-10-new:~$ sudo pg_createcluster 14 main2
	artem@postgre-2022-10-new:~$ sudo pg_createcluster 14 main3
	
Вывод в консоли (последние строки):

	Ver Cluster Port Status Owner    Data directory               Log file
	14  main2   5433 down   postgres /var/lib/postgresql/14/main2 /var/log/postgresql/postgresql-14-main2.log
	14  main3   5434 down   postgres /var/lib/postgresql/14/main3 /var/log/postgresql/postgresql-14-main3.log
	
Видно, что все ок. Порты для подключения в новых кластерах отличаются от первого кластера: 5433, 5434.
Запустил кластеры "14 main2" и "14 main3". Командная строка:

	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 main2 start
	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 main3 start
	
Проверил, что все кластеры PG запущены, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory               Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main  /var/log/postgresql/postgresql-14-main.log
	14  main2   5433 online postgres /var/lib/postgresql/14/main2 /var/log/postgresql/postgresql-14-main2.log
	14  main3   5434 online postgres /var/lib/postgresql/14/main3 /var/log/postgresql/postgresql-14-main3.log

---

Открыл три консоли.
Подключился к первому кластеру PG (второму, третьему). Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres psql	
	artem@postgre-2022-10-new:~$ sudo -u postgres psql -p 5433	
	artem@postgre-2022-10-new:~$ sudo -u postgres psql -p 5434	

Во всех кластерах зашел из-под пользователя postgres в psql.
Задал логическую репликацию у всех трех кластеров PG. Командная строка:

	postgres=# ALTER SYSTEM SET wal_level = logical;

После изменения параметра wal_level надо рестартовать все кластеры. Командная строка (для первого кластера):

	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 main restart
	
Проверил в первом кластере, что параметр wal_level изменился. Командная строка:

	postgres=# show wal_level;
	
Вывод в консоли:

	 wal_level
	-----------
	 logical
 
Создал в базе postgres таблицы tbl_test для записи, tbl_test2 для запросов на чтение. Командная строка:

	postgres=# CREATE table tbl_test(id int, name varchar);
	postgres=# CREATE table tbl_test2(id int, num int, caption varchar);
	
Таблицу tbl_test сразу заполнил значениями, чтоб поехали позже на реплику (второй и третий кластер). Командная строка:

	postgres=# insert into tbl_test select  
	generate_series(1,10) as id,
	md5(random()::text)::char(10) as name;
	
Посмотрел, что в таблице tbl_test. Командная строка:  

	postgres=# select * from tbl_test;
	
Вывод в консоли:

	 id |    name
	----+------------
	  1 | 6ed783394c
	  2 | efa5d379fd
	  3 | aa712c033a
	  4 | 59c0436d62
	  5 | af5bd63255
	  6 | b57e8b0eda
	  7 | 27c5b9ac50
	  8 | c189e5046b
	  9 | 320e3f25a2
	 10 | 0f94bfbff0
	(10 rows)

Во втором кластере в базе postgres создал таблицы tbl_test для запросов на чтение, а tbl_test2 для записи. Всё технически то же самое.
Таблицу tbl_test2 сразу заполнил значениями, чтоб поехали позже на реплику (первый и третий кластер).

	postgres=# insert into tbl_test2 select  
	generate_series(1,10) as id, generate_series(101,110) as num,
	md5(random()::text)::char(10) as caption;

Посмотрел, что в таблице tbl_test2. Командная строка:  

	postgres=# select * from tbl_test2;
	
Вывод в консоли:

	 id | num |  caption
	----+-----+------------
	  1 | 101 | cf54be6a84
	  2 | 102 | a0f4fb94ed
	  3 | 103 | 9985829fc0
	  4 | 104 | 449dbeb939
	  5 | 105 | 3f434f88ba
	  6 | 106 | 8b03371b6d
	  7 | 107 | 27cbbecb96
	  8 | 108 | b68af08705
	  9 | 109 | 7853ef88d5
	 10 | 110 | 06ba579535
	(10 rows)

В третьем кластере в базе postgres создал таблицы tbl_test и tbl_test2 для запросов на чтение. Это будут реплики для таблиц первого и второго кластера.

---

В первом кластере создал публикацию для таблицы tbl_test. Командная строка: 

	postgres=# CREATE PUBLICATION test_pub FOR TABLE tbl_test;

Можно просмотреть публикацию. Командная строка: 

	postgres=# \dRp+
	
Вывод в консоли:
	
								Publication test_pub
	  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
	----------+------------+---------+---------+---------+-----------+----------
	 postgres | f          | t       | t       | t       | t         | f
	Tables:
		"public.tbl_test"

Задал пароль (111). Командная строка:

	\password 

В втором кластере таким же образом создал публикацию для таблицы tbl_test2. Командная строка: 

	postgres=# CREATE PUBLICATION test_pub FOR TABLE tbl_test2;
		
Таблица tbl_test пока пустая.

	postgres=# select * from tbl_test;

Вывод в консоли:
	
	 id | name
	----+------
	(0 rows)

Далее создал подписку на таблицу tbl_test из первого кластера (с копированием прежних данных). Командная строка:   

	postgres=# CREATE SUBSCRIPTION test_sub 
	CONNECTION 'host=localhost port=5432 user=postgres password=111 dbname=postgres' 
	PUBLICATION test_pub WITH (copy_data = true);

В таблице tbl_test сразу же появились данные (реплика с первого кластера).

	postgres=# select * from tbl_test;
	
Вывод в консоли:
	
	 id |    name
	----+------------
	  1 | 6ed783394c
	  2 | efa5d379fd
	  3 | aa712c033a
	  4 | 59c0436d62
	  5 | af5bd63255
	  6 | b57e8b0eda
	  7 | 27c5b9ac50
	  8 | c189e5046b
	  9 | 320e3f25a2
	 10 | 0f94bfbff0
	(10 rows)
	
В первом кластере создал подписку на таблицу tbl_test2 из второго кластера (с копированием прежних данных). Командная строка:  

	postgres=# CREATE SUBSCRIPTION test_sub 
	CONNECTION 'host=localhost port=5433 user=postgres password=111 dbname=postgres' 
	PUBLICATION test_pub WITH (copy_data = true);

В таблице tbl_test2 тоже сразу же появились данные (реплика со второго кластера).

	postgres=# select * from tbl_test2;
	
Вывод в консоли:

	 id | num |  caption
	----+-----+------------
	  1 | 101 | cf54be6a84
	  2 | 102 | a0f4fb94ed
	  3 | 103 | 9985829fc0
	  4 | 104 | 449dbeb939
	  5 | 105 | 3f434f88ba
	  6 | 106 | 8b03371b6d
	  7 | 107 | 27cbbecb96
	  8 | 108 | b68af08705
	  9 | 109 | 7853ef88d5
	 10 | 110 | 06ba579535
	(10 rows)

В третьем кластере создал две подписки - на таблицу tbl_test из первого кластера и таблицу tbl_test2 из второго кластера (обе с копированием прежних данных).
Сделал другие имена двух репликационных слотов, т.к. с прежними не удалось - ошибка, что уже существуют. Командная строка:  

	postgres=# CREATE SUBSCRIPTION pg3_test_sub 
	CONNECTION 'host=localhost port=5432 user=postgres password=111 dbname=postgres' 
	PUBLICATION test_pub WITH (copy_data = true);

	postgres=# CREATE SUBSCRIPTION pg4_test_sub 
	CONNECTION 'host=localhost port=5433 user=postgres password=111 dbname=postgres' 
	PUBLICATION test_pub WITH (copy_data = true);

В таблице tbl_test сразу же появились данные (реплика с первого кластера).
В таблице tbl_test2 сразу же появились данные (реплика со второго кластера).

Теперь добавлю две новые строки в таблицу tbl_test, которые была предназначена для записи в первом кластере, и проверю, как пройдет реплика. Командная строка:  

	postgres=# insert into tbl_test (id, name) values (11, 'aaaaaa'), (12, 'bbbbbb');
	
Вывод в консоли:

	INSERT 0 2

Делаю селект из tbl_test во втором кластере. Командная строка:  

	postgres=# select * from tbl_test where id > 10;
	
Вывод в консоли:

	 id |  name
	----+--------
	 11 | aaaaaa
	 12 | bbbbbb

Две новые записи среплицировались.	

Посмотрел, какие есть подписки во втором кластере (одна, конечно). Командная строка:

	postgres=# \dRs+

Вывод в консоли:

																		List of subscriptions
	   Name   |  Owner   | Enabled | Publication | Binary | Streaming | Synchronous commit |                              Conninfo
	----------+----------+---------+-------------+--------+-----------+--------------------+---------------------------------------------------------------------
	 test_sub | postgres | t       | {test_pub}  | f      | f         | off                | host=localhost port=5432 user=postgres password=111 dbname=postgres
	(1 row) 
	
Проверил состояние этой подписки на втором кластере. Командная строка:  

	postgres=# postgres=# SELECT * FROM pg_stat_subscription \gx
	
Вывод в консоли:

	-[ RECORD 1 ]---------+------------------------------
	subid                 | 16396
	subname               | test_sub
	pid                   | 786
	relid                 |
	received_lsn          | 0/173D258
	last_msg_send_time    | 2022-12-08 21:52:06.096939+00
	last_msg_receipt_time | 2022-12-08 21:52:06.096984+00
	latest_end_lsn        | 0/173D258
	latest_end_time       | 2022-12-08 21:52:06.096939+00

Проверил всё то же самое в третьем кластере. Всё работает.

Отключился от ВМ.