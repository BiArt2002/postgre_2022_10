# postgre_2022_10

**10-я лекция**

ВМ с ОС Ubuntu 22.04 LTS - уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@51.250.103.196

PostgreSQL 14 уже есть на ВМ.

Проверяю, что кластер PG запущен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory                Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main   /var/log/postgresql/postgresql-14-main.log

Midnight Commander уже установлен - чтоб удобнее было ходить по каталогам.

---

Настроил сервер PG так, чтобы в журнал сообщений писалась информация о блокировках, удерживаемых более 200 миллисекунд.

Запустил из-под sudo Midnight Commander. Командная строка:

	artem@postgre-2022-10-new:~$ sudo mc
	
Зашел в папку, где лежат конфигурационные файлы: /etc/postgresql/14/main
Установил в конфигурационном файле postgresql.conf параметр deadlock_timeout = 200ms

Зашел из-под пользователя postgres в psql.
Перешел в базу postgres. 	
Включил параметр log_lock_waits. Командная строка:	

	artem@postgre-2022-10-new:~$ sudo -u postgres psql
	postgres=# \c postgres;
	postgres=# ALTER SYSTEM SET log_lock_waits = on;
	
Перечитал конфигурацию сервера PG, чтобы параметры применились. Командная строка:	

	postgres=# SELECT pg_reload_conf();

Убедился, что значение параметров deadlock_timeout и log_lock_waits изменилось. Командная строка:

	postgres=# SHOW deadlock_timeout;
	postgres=# SHOW log_lock_waits;
	
Вывод в консоли (по порядку):

	 deadlock_timeout
	------------------
	 200ms
	

	 log_lock_waits
	----------------
	 on

Создал таблицу accounts и в ней три записи. Командная строка: 	
	
	postgres=# CREATE TABLE accounts(acc_no integer PRIMARY KEY, amount numeric);	
	postgres=# INSERT INTO accounts VALUES (1, 1000.00), (2, 2000.00), (3, 3000.00);

Открыл второе окно с подключением к ВМ.
Зашел из-под пользователя postgres в psql.
Перешел в базу postgres. 
 
Создал ситуацию, при которой в журнале появятся сообщения о блокировках, удерживаемых более 200 миллисекунд.
Начал транзакцию для 1-й сессии. Командная строка: 	

	postgres=# BEGIN;
	postgres=# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;

Начал транзакцию для 2-й сессии (она будет ждать 1-ю транзакцию). Командная строка: 	

	postgres=# BEGIN;
	postgres=# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

Для 1-й транзакции подождал 1 секунду и закоммитил.

	postgres=# SELECT pg_sleep(1);
	postgres=# COMMIT;
	
Теперь 2-я транзакция тоже может завершиться после 1-й. Коммичу 2-ю транзакцию.
	
	postgres=# COMMIT;

Посмотрел в журнале последние 10 записей. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo tail -n 10 /var/log/postgresql/postgresql-14-main.log
			
Вывод в консоли (по блокировкам):	

	2022-11-23 21:19:49.930 UTC [1803] postgres@postgres LOG:  process 1803 still waiting for ShareLock on transaction 667499 after 200.105 ms
	2022-11-23 21:19:49.930 UTC [1803] postgres@postgres DETAIL:  Process holding the lock: 1568. Wait queue: 1803.
	2022-11-23 21:19:49.930 UTC [1803] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "accounts"
	2022-11-23 21:19:49.930 UTC [1803] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
	2022-11-23 21:20:44.398 UTC [1803] postgres@postgres LOG:  process 1803 acquired ShareLock on transaction 667499 after 54668.281 ms
	2022-11-23 21:20:44.398 UTC [1803] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "accounts"
	2022-11-23 21:20:44.398 UTC [1803] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
		
Видно, что блокировки были (более 200 миллисекунд, как и требовалось). Информация о них писалась в журнал. 

---

Открыл третье окно с подключением к ВМ.
Зашел из-под пользователя postgres в psql.
Перешел в базу postgres. 

Для начала построил представление (view) над pg_locks для удобства анализа блокировок. Командная строка:

	postgres=# CREATE VIEW locks_v AS
	SELECT pid,
		   locktype,
		   CASE locktype
			 WHEN 'relation' THEN relation::regclass::text
			 WHEN 'transactionid' THEN transactionid::text
			 WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
		   END AS lockid,
		   mode,
		   granted
	FROM pg_locks
	WHERE locktype in ('relation','transactionid','tuple')
	AND (locktype != 'relation' OR relation = 'accounts'::regclass);

Смоделировал ситуацию обновления одной и той же строки в таблице accounts тремя командами UPDATE в трех разных сеансах.

Начал транзакцию для 1-й сессии. Сначала посмотрел id транзакции и pid. Командная строка: 	
	
	postgres=# BEGIN;
	postgres=*# SELECT txid_current(), pg_backend_pid();
	
Вывод в консоли:

	 txid_current | pg_backend_pid
	--------------+----------------
	   667505 |           1031
	
Делаю update строки в таблице accounts. Командная строка: 

	postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

Смотрю, что с блокировками. Командная строка: 

	postgres=*# SELECT * FROM locks_v WHERE pid = 1031;
	
Вывод в консоли:

	 pid  |   locktype    |  lockid  |       mode       | granted
	------+---------------+----------+------------------+---------
	 1031 | relation      | accounts | RowExclusiveLock | t
	 1031 | transactionid | 667505   | ExclusiveLock    | t
 
Итак, транзакция в 1-й сессии держит исключительную блокировку строки на строку таблицы accounts и блокировку собственного номера транзакции 667505.


Начал транзакцию для 2-й сессии. Так же посмотрел id транзакции и pid. Командная строка: 
	
	postgres=# BEGIN;
	postgres=*# SELECT txid_current(), pg_backend_pid();
	
Вывод в консоли:

	 txid_current | pg_backend_pid
	--------------+----------------
		   667506 |           1183
	   
Делаю update той же строки в таблице accounts, что и в предыдущей транзакции. Командная строка:

	postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
	
В 1-й сессии смотрю, что с блокировками. Командная строка: 	
	
	postgres=*# SELECT * FROM locks_v WHERE pid = 1183;
	
Вывод в консоли:

	 pid  |   locktype    |   lockid   |       mode       | granted
	------+---------------+------------+------------------+---------
	 1183 | relation      | accounts   | RowExclusiveLock | t
	 1183 | transactionid | 667506     | ExclusiveLock    | t
	 1183 | transactionid | 667505     | ShareLock        | f
	 1183 | tuple         | accounts:5 | ExclusiveLock    | t
 	 
Так же вижу исключительную блокировку строки в таблице accounts. Так же есть блокировка номера каждой транзакции - 667505 и 667506.
Вижу, что режим  блокировки первой транзакции № 667505 изменился на ShareLock (защита от совместных изменений), т.к. вторая транзакция № 667506 хочет обновить ту же строку. Теперь вторая транзакция ждет первую. 
Плюс транзакция № 667506 ждет получение исключительной блокировки типа tuple (блокировка версии строки) для обновляемой строки.


Начал транзакцию для 3-й сессии. Так же посмотрел id транзакции и pid. Командная строка: 
	
	postgres=# BEGIN;
	postgres=*# SELECT txid_current(), pg_backend_pid();
	
Вывод в консоли:

	 txid_current | pg_backend_pid
	--------------+----------------
		   667507 |           1652

Делаю update всё той же строки в таблице accounts, что и в предыдущих двух транзакциях. Командная строка:

	postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
	
В 1-й сессии смотрю, что с блокировками. Командная строка: 	
	
	postgres=*# SELECT * FROM locks_v WHERE pid = 1652;
	
Вывод в консоли:

	 pid  |   locktype    |   lockid   |       mode       | granted
	------+---------------+------------+------------------+---------
	 1652 | relation      | accounts   | RowExclusiveLock | t
	 1652 | tuple         | accounts:5 | ExclusiveLock    | f
	 1652 | transactionid | 667507     | ExclusiveLock    | t

Третья транзакция № 667507 попыталась захватить исключительную блокировку типа tuple (блокировка версии строки) и повисла уже на этом шаге. То же самое будет и с четвертой транзакцией, которая захочет обновить ту же строку в таблице accounts. И с пятой, и т.д.

Кто кого ждет можно увидеть в представлении pg_stat_activity, добавив информацию о блокирующих процессах. Командная строка:

	postgres=*# SELECT pid, wait_event_type, wait_event, pg_blocking_pids(pid)
	FROM pg_stat_activity
	WHERE backend_type = 'client backend' ORDER BY pid;

Вывод в консоли:

	 pid  | wait_event_type |  wait_event   | pg_blocking_pids
	------+-----------------+---------------+------------------
	 1031 |                 |               | {}
	 1183 | Lock            | transactionid | {1031}
	 1652 | Lock            | tuple         | {1183}
	 
Вижу, что вторая транзакция ждет первую, а третья ждет вторую. При этом первая транзакция удерживает блокировку версии строки. Получилась очередь.	 
	 
В 1-й сессии делаю коммит. Смотрю, что с блокировками. Командная строка:	 
	 
	 postgres=*# COMMIT;
	 postgres=*# SELECT * FROM locks_v WHERE pid = 1031;
	 
Вывод в консоли:
	 	 
	 pid | locktype | lockid | mode | granted
	-----+----------+--------+------+---------
	(0 rows)	 
	 
Первая транзакция более ничего не блокирует. Что со второй?  Командная строка:	 
	 
	postgres=*# SELECT * FROM locks_v WHERE pid = 1183;
	
Вывод в консоли:	 
	 
	 pid  |   locktype    |   lockid    |       mode       | granted
	------+---------------+-------------+------------------+---------
	 1183 | relation      | accounts    | RowExclusiveLock | t
	 1183 | transactionid | 667506      | ExclusiveLock    | t
	 1183 | tuple         | accounts:10 | ExclusiveLock    | t
	 1183 | transactionid | 667507      | ShareLock        | f
	 
А вторая транзакция № 667506 теперь ждет третью № 667507. Т.к. третья после коммита первой транзакции успела захватить блокировку раньше, чем вторая.
Что с третьей транзакцией?

	postgres=*# SELECT * FROM locks_v WHERE pid = 1652;
	
Вывод в консоли:
	
	 pid  |   locktype    |  lockid  |       mode       | granted
	------+---------------+----------+------------------+---------
	 1652 | relation      | accounts | RowExclusiveLock | t
	 1652 | transactionid | 667507   | ExclusiveLock    | t

Транзакция в 3-й сессии держит исключительную блокировку на строку таблицы accounts и блокировку собственного номера транзакции 667507.
В 3-й сессии делаю коммит.
Теперь вторая транзакция никого не должна ждать. Посмотрим, так ли это?  Командная строка:

	postgres=*# SELECT * FROM locks_v WHERE pid = 1183;
	
Вывод в консоли:
		
	 pid  |   locktype    |  lockid  |       mode       | granted
	------+---------------+----------+------------------+---------
	 1183 | relation      | accounts | RowExclusiveLock | t
	 1183 | transactionid | 667506   | ExclusiveLock    | t
	 
Транзакция в 2-й сессии держит исключительную блокировку на строку таблицы accounts и блокировку собственного номера транзакции 667506.	 
Во 2-й сессии делаю коммит и заканчиваю исследовать ситуацию обновления одной и той же строки тремя командами UPDATE. 	 
 
---

Воспроизвел взаимоблокировку трех транзакций.
Параметр deadlock_timeout я уже установил ранее = 200ms. Параметр log_lock_waits также был включен.

В транзакции для 1-й сессии добавляем на первый счет 1 руб. Командная строка:

	postgres=# BEGIN;
	postgres=*# UPDATE accounts SET amount = amount + 1.00 WHERE acc_no = 1;

В транзакции для 2-й сессии добавляем на второй счет 2 руб. Командная строка:

	postgres=# BEGIN;
	postgres=*# UPDATE accounts SET amount = amount + 2.00 WHERE acc_no = 2;
	
В транзакции для 3-й сессии добавляем на третий счет 3 руб. Командная строка:	

	postgres=# BEGIN;
	postgres=*# UPDATE accounts SET amount = amount + 3.00 WHERE acc_no = 3;
	
Теперь будем снимать со счетов ранее добавленные суммы, но в других сессиях, чем добавляли (при этом транзакции пока не завершены).
	
В транзакции для 1-й сессии снимем со второго счета 2 руб. Командная строка: 		
	
	postgres=*# UPDATE accounts SET amount = amount - 2.00 WHERE acc_no = 2;
	
Запрос не выполнился, он ждет.
В транзакции для 2-й сессии снимем с третьего счета 3 руб. Командная строка: 		
	
	postgres=*# UPDATE accounts SET amount = amount - 3.00 WHERE acc_no = 3;	
	
Запрос тоже не выполнился, он ждет.
И наконец, в транзакции для 3-й сессии снимем с первого счета 1 руб. Командная строка: 
	
	postgres=*# UPDATE accounts SET amount = amount - 1.00 WHERE acc_no = 1;

В 3-й сессии сразу получил вывод в консоль о дедлоке:

	ERROR:  deadlock detected
	DETAIL:  Process 1296 waits for ShareLock on transaction 667517; blocked by process 1163.
	Process 1163 waits for ShareLock on transaction 667518; blocked by process 1204.
	Process 1204 waits for ShareLock on transaction 667519; blocked by process 1296.
	HINT:  See server log for query details.
	CONTEXT:  while updating tuple (0,22) in relation "accounts"

Коммит в 3-й сессии не проходит. В консоли пишется: ROLLBACK.
Селект из таблицы accounts дает начальные значения сумм на счетах: 1000, 2000, 3000 руб.
Сделал rollback в других двух сессиях.

А можно ли разобраться в ситуации взаимоблокировки трех транзакций постфактум, изучая журнал сообщений?
Посмотрим, что писалось в журнал, последние 30 записей. Командная строка:

	artem@postgre-2022-10-new:~$ sudo tail -n 30 /var/log/postgresql/postgresql-14-main.log
	
Вывод в консоль:

	2022-11-27 21:06:38.688 UTC [1163] postgres@postgres LOG:  process 1163 still waiting for ShareLock on transaction 667518 after 200.148 ms
	2022-11-27 21:06:38.688 UTC [1163] postgres@postgres DETAIL:  Process holding the lock: 1204. Wait queue: 1163.
	2022-11-27 21:06:38.688 UTC [1163] postgres@postgres CONTEXT:  while updating tuple (0,23) in relation "accounts"
	2022-11-27 21:06:38.688 UTC [1163] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount - 2.00 WHERE acc_no = 2;
	2022-11-27 21:07:26.085 UTC [1204] postgres@postgres LOG:  process 1204 still waiting for ShareLock on transaction 667519 after 200.138 ms
	2022-11-27 21:07:26.085 UTC [1204] postgres@postgres DETAIL:  Process holding the lock: 1296. Wait queue: 1204.
	2022-11-27 21:07:26.085 UTC [1204] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "accounts"
	2022-11-27 21:07:26.085 UTC [1204] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount - 3.00 WHERE acc_no = 3;
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres LOG:  process 1296 detected deadlock while waiting for ShareLock on transaction 667517 after 200.121 ms
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres DETAIL:  Process holding the lock: 1163. Wait queue: .
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres CONTEXT:  while updating tuple (0,22) in relation "accounts"
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount - 1.00 WHERE acc_no = 1;
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres ERROR:  deadlock detected
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres DETAIL:  Process 1296 waits for ShareLock on transaction 667517; blocked by process 1163.
			Process 1163 waits for ShareLock on transaction 667518; blocked by process 1204.
			Process 1204 waits for ShareLock on transaction 667519; blocked by process 1296.
			Process 1296: UPDATE accounts SET amount = amount - 1.00 WHERE acc_no = 1;
			Process 1163: UPDATE accounts SET amount = amount - 2.00 WHERE acc_no = 2;
			Process 1204: UPDATE accounts SET amount = amount - 3.00 WHERE acc_no = 3;
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres HINT:  See server log for query details.
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres CONTEXT:  while updating tuple (0,22) in relation "accounts"
	2022-11-27 21:10:57.729 UTC [1296] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount - 1.00 WHERE acc_no = 1;
	2022-11-27 21:10:57.729 UTC [1204] postgres@postgres LOG:  process 1204 acquired ShareLock on transaction 667519 after 211844.342 ms
	2022-11-27 21:10:57.729 UTC [1204] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "accounts"
	2022-11-27 21:10:57.729 UTC [1204] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount - 3.00 WHERE acc_no = 3;
	2022-11-27 21:16:34.472 UTC [1204] postgres@postgres ERROR:  syntax error at or near "roolback" at character 1
	2022-11-27 21:16:34.472 UTC [1204] postgres@postgres STATEMENT:  roolback;
	2022-11-27 21:16:34.473 UTC [1163] postgres@postgres LOG:  process 1163 acquired ShareLock on transaction 667518 after 595984.427 ms
	2022-11-27 21:16:34.473 UTC [1163] postgres@postgres CONTEXT:  while updating tuple (0,23) in relation "accounts"
	2022-11-27 21:16:34.473 UTC [1163] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount - 2.00 WHERE acc_no = 2;

В журнале виден дедлок. ERROR:  deadlock detected. И детализаци по дедлоку. Значит постфактум разобраться в ситуации взаимоблокировки можно.

Отключился от ВМ.