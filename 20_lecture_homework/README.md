# postgre_2022_10

**20-я лекция**

ВМ с ОС Ubuntu 22.04 LTS - уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@51.250.26.232

Установил с нуля PostgreSQL 14.

Проверяю, что кластер PG запущен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
	
Midnight Commander уже установлен - чтоб удобнее было ходить по каталогам.

---	

Попробовал скачать архив c большой демо-базой flights   https://postgrespro.com/education/demodb   Командная строка:

	artem@postgre-2022-10-new:~$ wget https://edu.postgrespro.ru/demo-big-en.zip
	
Ошибка 404 Not Found.

Скачал маленькую базу. Командная строка:

	artem@postgre-2022-10-new:~$ wget https://edu.postgrespro.ru/demo-small.zip
	
Вывод в консоли:

	HTTP request sent, awaiting response... 200 OK
	Length: 22187733 (21M) [application/zip]
	Saving to: ‘demo-small.zip’
	
Установил архиватор. Командная строка:

	artem@postgre-2022-10-new:~$ sudo apt install unzip

Распаковал архив. Командная строка:

	artem@postgre-2022-10-new:~$ unzip demo-small.zip

Вывод в консоли:

	Archive:  demo-small.zip
	  inflating: demo-small-20170815.sql

Открыл в редакторе Nano файл demo-small-20170815.sql. Командная строка:

	artem@postgre-2022-10-new:~$ sudo nano demo-small-20170815.sql

Подправил в нем создание таблицы flights, чтобы секционировать ее по "Отправление по расписанию", поле scheduled_departure.

	CREATE TABLE flights (
		flight_id integer NOT NULL,
		flight_no character(6) NOT NULL,
		scheduled_departure timestamp with time zone NOT NULL,
		scheduled_arrival timestamp with time zone NOT NULL,
		departure_airport character(3) NOT NULL,
		arrival_airport character(3) NOT NULL,
		status character varying(20) NOT NULL,
		aircraft_code character(3) NOT NULL,
		actual_departure timestamp with time zone,
		actual_arrival timestamp with time zone,
		CONSTRAINT flights_check CHECK ((scheduled_arrival > scheduled_departure)),
		CONSTRAINT flights_check1 CHECK (((actual_arrival IS NULL) OR ((actual_departure IS NOT NULL) AND (actual_arrival IS NOT NULL) AND (actual_arrival > actual_departure)))),
		CONSTRAINT flights_status_check CHECK (((status)::text = ANY (ARRAY[('On Time'::character varying)::text, ('Delayed'::character varying)::text, ('Departed'::character varying)::text, ('Arrived'::character varying)::text, ('Scheduled'::character varying)::text, ('Cancelled'::character varying)::text])))
	)
	partition by range (scheduled_departure);
	
Т.к. в маленькой базе полетные данные за июль, август и сентябрь 2017 года, то их можно разделить по месяцам. Делаю секционирование на 3 месяца, в т.ч. делаю партицию по умолчанию.
	
	CREATE TABLE flights_2017_07 partition of flights for values from ('2017-07-01') to ('2017-08-01');
	CREATE TABLE flights_2017_08 partition of flights for values from ('2017-08-01') to ('2017-09-01');
	CREATE TABLE flights_2017_09 partition of flights for values from ('2017-09-01') to ('2017-10-01');
	CREATE TABLE flights_default partition of flights default;
	
Далее зашел из-под пользователя postgres в psql, командная строка: 	
	
	artem@postgre-2022-10-new:~$ sudo -u postgres psql
	
Создал базу 'demo', командная строка: 
	
	postgres=# CREATE DATABASE demo;
	
Перед накатом измененного скрипта задал юзеру postgres пароль '123'. Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres psql
	postgres=# \password postgres 
		
Накатил измененный скрипт demo-small-20170815.sql. Командная строка:

	artem@postgre-2022-10-new:~$ sudo psql -h 127.0.0.1 -U postgres -d demo -a -f demo-small-20170815.sql
	
Опять зашел из-под пользователя postgres в psql, перешел в базу 'demo'. Командная строка:

	postgres=# \c demo
	
Посмотрел, что именно там есть (чтоб убедиться, что таблицы из скрипта накатились). Командная строка:

	demo=# \d
	
Вывод в консоли:

							List of relations
	  Schema  |         Name          |       Type        |  Owner
	----------+-----------------------+-------------------+----------
	 bookings | aircrafts             | view              | postgres
	 bookings | aircrafts_data        | table             | postgres
	 bookings | airports              | view              | postgres
	 bookings | airports_data         | table             | postgres
	 bookings | boarding_passes       | table             | postgres
	 bookings | bookings              | table             | postgres
	 bookings | flights               | partitioned table | postgres
	 bookings | flights_2017_07       | table             | postgres
	 bookings | flights_2017_08       | table             | postgres
	 bookings | flights_2017_09       | table             | postgres
	 bookings | flights_default       | table             | postgres
	 bookings | flights_flight_id_seq | sequence          | postgres
	 bookings | flights_v             | view              | postgres
	 bookings | routes                | view              | postgres
	 bookings | seats                 | table             | postgres
	 bookings | ticket_flights        | table             | postgres
	 bookings | tickets               | table             | postgres
	(17 rows)

Посмотрел, как записи распределены в секционированных таблицах. Командная строка:

	demo=# SELECT tableoid::regclass, count(*) FROM flights GROUP BY tableoid;

Вывод в консоли:

		tableoid     | count
	-----------------+-------
	 flights_2017_09 |  7595
	 flights_2017_07 |  8691
	 flights_2017_08 | 16835
	(3 rows)
	
Убедился, что в родительской таблице flights данных нет. Командная строка:

	demo=# SELECT * FROM ONLY flights;
	
Вывод в консоли:	

	 flight_id | flight_no | scheduled_departure | scheduled_arrival | departure_airport | arrival_airport | status | aircraft_code | actual_departure | actual_arrival
	-----------+-----------+---------------------+-------------------+-------------------+-----------------+--------+---------------+------------------+----------------
	(0 rows)

	(END)
	-----------+-----------+---------------------+-------------------+-------------------+-----------------+--------+---------------+------------------+----------------
	(0 rows)

На этом всё.

Отключился от ВМ.