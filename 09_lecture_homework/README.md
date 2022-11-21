# postgre_2022_10

**9-я лекция**

ВМ с ОС Ubuntu 22.04 LTS - создаю заново в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@84.201.165.136

Установил с нуля PostgreSQL 14.

Проверяю, что кластер PG запущен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

Поставил себе Midnight Commander, чтоб удобнее было ходить по каталогам. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo add-apt-repository universe
	artem@postgre-2022-10-new:~$ sudo apt update
	artem@postgre-2022-10-new:~$ sudo apt install mc
	
Запустил из-под sudo Midnight Commander. Командная строка:

	artem@postgre-2022-10-new:~$ sudo mc

Зашел в папку, где лежат конфигурационные файлы: /etc/postgresql/14/main
В файле postgresql.conf установил параметры контрольной точки раз в 30 секунд (checkpoint_timeout = 30s).	

Рестартовал кластер PG. Командная строка:
	
	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 main restart
	
Подготовил базу postgres к выполнению pgbench. Командная строка:	

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -i postgres
	
Вывод в консоль:

	done in 0.49 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.32 s, vacuum 0.06 s, primary keys 0.09 s).

C помощью pgbench подавал нагрузку - затем понял, что надо будет измерять объем журнальных файлов.
Поэтому сделал все заново.
Перед этим проверил проверил текущий LSN. Командная строка:

	postgres=# select pg_current_wal_insert_lsn();
	
Вывод в консоль:

	 pg_current_wal_insert_lsn
	---------------------------
	 0/21486D50
	(1 row)
	
Определил количество контрольных точек, выполненных до текущего момента. Командная строка:

	postgres=# select checkpoints_timed from pg_stat_bgwriter;
	
Вывод в консоль:

	 checkpoints_timed
	-------------------
				   203
	(1 row)

Затем уже в течение 10 минут (600 сек) подавал нагрузку на базу postgres. Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -P 1 -T 600 postgres
	
Вывод в консоль (итоговые строки):

	scaling factor: 1
	query mode: simple
	number of clients: 1
	number of threads: 1
	duration: 600 s
	number of transactions actually processed: 340138
	latency average = 1.764 ms
	latency stddev = 1.318 ms
	initial connection time = 2.436 ms
	tps = 566.898670 (without initial connection time)

Зашел в папку /var/lib/postgresql/14/main/pg_wal и посмотрел, что появились новые журнальные файлы.
Определил количество контрольных точек после выполнения бенчмарка - 231.
Текущий LSN теперь - 0/3A756FA0

Посчитал число байт между двумя позициями LSN в журнале предзаписи WAL - до бенчмарка и после. Командная строка:

	postgres=# select '0/3A756FA0'::pg_lsn - '0/21486D50'::pg_lsn as byte_size;

Вывод в консоль:

	 byte_size
	-----------
	 422380112
	(1 row)
	
Количество контрольных точек за 10 минут: 231 - 203 = 28. А должно быть 20, по идее. Не могу объяснить - почему так. Есть еще контрольные точки по требованию, например, по по достижению max_wal_size. И они тоже учитываются. Но как-то многовато.
Посчитал, сколько байт приходится на одну контрольную точку: 422380112 / 31 = 13625164,9 ~ 13305,83 КБ ~ 12,99 MB

Проверил данные статистики - чтобы ответить на вопрос: все ли контрольные точки выполнялись точно по расписанию?
Зашел в папку с логом  /var/log/postgresql/postgresql-14-main.log Однако не нашел там информации по контрольным точкам. Похоже, в лог эта информация не писалась.

Проверил параметр log_checkpoints (записывать информацию по контрольным точкам или нет). Командная строка:

	postgres=# show log_checkpoints;
	
Вывод в консоль:

	 log_checkpoints
	-----------------
	 off
	(1 row)	

К сожалению, параметр был выключен. И я не включил его перед бенчмарком. В описании ДЗ это также не было указано.
Не стал делать нагрузку заново. Можно рассмотреть вопрос теоретически.
Вообще, выполнение по времени контрольной точки связано еще с параметром checkpoint_completion_target.
Проверил параметр checkpoint_completion_target. Командная строка:

Вывод в консоль:

	 checkpoint_completion_target
	------------------------------
	 0.9
	(1 row)
	
Таким образом время для завершения процедуры контрольной точки должно быть: 0.9 * 30 сек = 27 сек. 
Получается, мои контрольные точки должны были выполняться через 27 секунд. Т.е. не точно по расписанию.

---	
Далее сравнивал tps в синхронном/асинхронном режиме.
Сначала сделал нагрузочное тестирование в синхронном режиме. Командная строка:	

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -i postgres
	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -P 1 -T 10 postgres
	
Вывод в консоль:

	progress: 1.0 s, 463.0 tps, lat 2.154 ms stddev 1.429
	progress: 2.0 s, 364.0 tps, lat 2.742 ms stddev 1.544
	progress: 3.0 s, 276.0 tps, lat 3.628 ms stddev 2.125
	progress: 4.0 s, 635.0 tps, lat 1.574 ms stddev 1.002
	progress: 5.0 s, 735.0 tps, lat 1.360 ms stddev 0.445
	progress: 6.0 s, 722.0 tps, lat 1.385 ms stddev 0.546
	progress: 7.0 s, 712.0 tps, lat 1.405 ms stddev 0.502
	progress: 8.0 s, 588.0 tps, lat 1.701 ms stddev 1.073
	progress: 9.0 s, 692.0 tps, lat 1.444 ms stddev 0.398
	progress: 10.0 s, 677.0 tps, lat 1.478 ms stddev 0.518	

Зашел из-под пользователя postgres в psql.	
Установил асинхронный режим. Командная строка:	

	artem@postgre-2022-10-new:~$ sudo -u postgres psql

	postgres=# ALTER SYSTEM SET synchronous_commit = off;
	
Рестартовал кластер PG (это обязательно после смены режима). Командная строка:
	
	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 main restart
	
Проверил, что режим асинхронный. Командная строка:

	postgres=# show synchronous_commit;
	
Вывод в консоль:

	 synchronous_commit
	--------------------
	 off
	(1 row)

Сделал нагрузочное тестирование в асинхронном режиме. Командная строка:	

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -i postgres
	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -P 1 -T 10 postgres
	
Вывод в консоль:

	progress: 1.0 s, 2785.0 tps, lat 0.358 ms stddev 0.033
	progress: 2.0 s, 2814.9 tps, lat 0.355 ms stddev 0.017
	progress: 3.0 s, 2779.1 tps, lat 0.360 ms stddev 0.016
	progress: 4.0 s, 2826.0 tps, lat 0.354 ms stddev 0.018
	progress: 5.0 s, 2788.9 tps, lat 0.359 ms stddev 0.019
	progress: 6.0 s, 2830.0 tps, lat 0.353 ms stddev 0.016
	progress: 7.0 s, 2815.0 tps, lat 0.355 ms stddev 0.014
	progress: 8.0 s, 2790.0 tps, lat 0.358 ms stddev 0.014
	progress: 9.0 s, 2776.1 tps, lat 0.360 ms stddev 0.023
	progress: 10.0 s, 2753.9 tps, lat 0.363 ms stddev 0.014
	
Tps в синхронном/асинхронном режиме существенно различаются. В асинхронном режиме прирост tps по сравнению с синхронным у меня примерно в 3,5 раза выше.
Потому что в асинхронном режиме сервер не ждет, пока записи из WAL будут сохранены на диск, чтобы сообщить клиенту об успешном завершении операции. 
Следовательно накладные расходы, связанные с данным процессом, не влияют на результат. А в синхронном режиме - влияют.

Выключил асинхронный режим. Командная строка:	

	artem@postgre-2022-10-new:~$ sudo -u postgres psql

	postgres=# ALTER SYSTEM SET synchronous_commit = on;
	
---	
Далее создал новый кластер PG с включенной контрольной суммой страниц. 
Для начала остановил первый кластер. Командная строка:	

	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 main stop
	
Проверяю, что кластер PG остановлен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:

	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

Теперь создаю новый кластер '14 double' с включенной контрольной суммой страниц. Командная строка:

	artem@postgre-2022-10-new:~$ sudo pg_createcluster 14 double -- --data-checksums

Вывод в консоли (последние строки):

	Ver Cluster Port Status Owner    Data directory                Log file
	14  double  5433 down   postgres /var/lib/postgresql/14/double /var/log/postgresql/postgresql-14-double.log
	
Видно, что все ок. Порт для подключения новый - 5433.
Запустил кластер '14 double' и подключился к нему. Командная строка:

	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 double start
	artem@postgre-2022-10-new:~$ sudo -u postgres psql -p 5433
	
Посмотрел параметр data-checksums. Командная строка:

	postgres=# show data_checksums;
	
Вывод в консоли:

	 data_checksums
	----------------
	 on
	(1 row)
	
Создал тестовую базу db_test и подключился к ней. Командная строка:
	
	postgres=# CREATE database db_test;
	postgres=# \c db_test;
	
Создал тестовую таблицу tbl_test и несколько строк в ней. Командная строка:

	db_test=# CREATE table tbl_test(id int, name varchar);
	db_test=# insert into tbl_test (id, name) values (1, 'a1'), (2, 'b2'), (3, 'c3'), (4, 'd4');
	db_test=# select * from tbl_test;
	
Вывод в консоли:
	
	 id | name
	----+------
	  1 | a1
	  2 | b2
	  3 | c3
	  4 | d4
	(4 rows)	
	
Нащел, где расположен файл таблицы tbl_test. Командная строка:

	db_test=# select pg_relation_filepath('tbl_test');

Вывод в консоли:

	 pg_relation_filepath
	----------------------
	 base/16384/16385
	(1 row)
	
Остановил кластер '14 double'. Командная строка:

	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 double stop
	
Запустил из-под sudo Midnight Commander. Прошел в папку /var/lib/postgresql/14/double/base/16384/ и в файле 16385 удалил несколько последних символов. 
	
Снова запустил кластер '14 double'. Командная строка:

	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 double start
	
Подключился к кластеру '14 double'. Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres psql -p 5433

Подключился к базе db_test и сделал селект из таблицы tbl_test. Командная строка:

	postgres=# \c db_test;
	db_test=# select * from tbl_test;

Вывод в консоли:
	
	 id | name
	----+------
	(0 rows)
	
В ДЗ написано, что должна быть ошибка, т.к. контрольные суммы файла таблицы не совпадают. Однако ошибки нет. Нет и данных в таблице.
Посмотрел значение параметра ignore_checksum_failure. Командная строка:

	db_test=# show ignore_checksum_failure;
	
Вывод в консоли:	
	
	 ignore_checksum_failure
	-------------------------
	 off
	(1 row)

Проверил, что будет, если поставить параметр ignore_checksum_failure равным true. Командная строка:

	db_test=# set ignore_checksum_failure = on;
		
Снова сделал селект из таблицы tbl_test. Командная строка:

	db_test=# select * from tbl_test;

Вывод в консоли:
	
	 id | name
	----+------
	(0 rows)
	
Ничего не изменилось. Не ясно, почему не было ошибки и почему пропали данные в таблице. На лекции этот момент не освещался. Как физически изменять файл таблицы, чтобы вызвать ошибку тоже не поясняли. Думаю, ошибка должна быть в любом случае, но что не так, не понимаю. 

Отключился от ВМ.