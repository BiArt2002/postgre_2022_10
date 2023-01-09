# postgre_2022_10

**19-я лекция**
(2-й вариант)

ВМ с ОС Ubuntu 22.04 LTS - уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@51.250.16.252

PostgreSQL 14 уже установлен.

Проверяю, что кластер PG запущен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
	
Midnight Commander уже установлен - чтоб удобнее было ходить по каталогам.

---	

Задание 19-й лекции я делал позже, чем 20-й. Поэтому на ВМ уже есть демо-база flights https://postgrespro.com/education/demodb , которую я использовал в 20-м домашнем задании.
Эта база содержит полетные данные за июль, август и сентябрь 2017 года.

В моем кластере PG сейчас имеется база 'demo', на которой и развернута демо-база flights.
Я буду делать 2-й вариант домашнего задания (различные варианты соединения таблиц). В качестве примера использую базу flights, т.к. там достаточно много данных.
Структуру таблиц, для которых выполнялись соединения, можно посмотреть на рисунке. 

![flights_database_tables_schema](https://user-images.githubusercontent.com/79360703/211261769-a7f9d4c9-5da9-4ff9-b787-0d929f01101c.gif)

Зашел из-под пользователя postgres в psql, перешел в базу 'demo'. Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres psql
	postgres=# \c demo
	
Посмотрел, какие таблицы имееются в базе 'demo'. Командная строка:

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

Примечание: с 20-го занятия в базе 'demo' осталась секционированная таблица flights.

---
1) прямое соединение двух или более таблиц.

Для удобства написания запросов я изменил параметр search_path, добавив в него схему bookings. Командная строка:

	demo=# SET search_path = bookings, public;
	
Посмотрю, какие рейсы может принимать московский аэропорт Домодедово (код DME) для самолетов из питерского аэропорта Пулково (код LED). Ограничу количество рейсов десятью, а то их много.

	demo=# select f.flight_no from flights f
			join airports a on a.airport_code = f.arrival_airport
			where f.arrival_airport = 'DME' and f.departure_airport = 'LED' limit 10;
	
Вывод в консоли:

	 flight_no
	-----------
	 PG0406
	 PG0409
	 PG0408
	 PG0407
	 PG0406
	 PG0407
	 PG0408
	 PG0409
	 PG0406
	 PG0409
	(10 rows)

Теперь выведу эти же рейсы с указанием модели самолета и его и максимальной дальности полета (соединение трех таблиц).

	demo=# select f.flight_no, ac.model, ac.range from flights f
			join airports ap on ap.airport_code = f.arrival_airport
			join aircrafts ac on ac.aircraft_code = f.aircraft_code			
			where f.arrival_airport = 'DME' and f.departure_airport = 'LED' limit 10;
	
Вывод в консоли:
	
	 flight_no |      model       | range
	-----------+------------------+-------
	 PG0406    | Аэробус A321-200 |  5600
	 PG0409    | Аэробус A321-200 |  5600
	 PG0408    | Аэробус A321-200 |  5600
	 PG0407    | Аэробус A321-200 |  5600
	 PG0406    | Аэробус A321-200 |  5600
	 PG0407    | Аэробус A321-200 |  5600
	 PG0408    | Аэробус A321-200 |  5600
	 PG0409    | Аэробус A321-200 |  5600
	 PG0406    | Аэробус A321-200 |  5600
	 PG0409    | Аэробус A321-200 |  5600
	(10 rows)

---			
2) Левостороннее (или правостороннее) соединение двух или более таблиц	

Есть ситуации, когда билет на самолет продан, а посадочный талон не был получен (человек опоздал). Можно найти такие билеты. Сначала выведу десять записей о билетах и посадочных талонах на рейсах в Домодедово из Пулково. Использую left join, вывожу все билеты, а посадочных талонов может и не быть.

	demo=# select t.ticket_no, t.passenger_name, bp.boarding_no, bp.seat_no from tickets t
			left join boarding_passes bp on bp.ticket_no = t.ticket_no
			join ticket_flights tf on tf.ticket_no = t.ticket_no 
			join flights f on f.flight_id = tf.flight_id
			where f.arrival_airport = 'DME' and f.departure_airport = 'LED' limit 10;
			
Вывод в консоли:
			
	   ticket_no   |  passenger_name   | boarding_no | seat_no
	---------------+-------------------+-------------+---------
	 0005432002062 | PETR KUDRYASHOV   |          43 | 22D
	 0005432002062 | PETR KUDRYASHOV   |          21 | 6C
	 0005432002063 | LARISA ANDREEVA   |          24 | 12C
	 0005432002063 | LARISA ANDREEVA   |          57 | 15A
	 0005432002064 | ANASTASIYA TITOVA |          48 | 23D
	 0005432002064 | ANASTASIYA TITOVA |          36 | 9F
	 0005432816941 | NATALYA FILIPPOVA |          79 | 31E
	 0005432816941 | NATALYA FILIPPOVA |          21 | 11B
	 0005432816941 | NATALYA FILIPPOVA |          40 | 19D
	 0005432816941 | NATALYA FILIPPOVA |           5 | 2A
	(10 rows)
			
В случайных записях не оказалось таких, когда не был получен посадочный талон. Добавлю условие с boarding_no is null.			
			
	demo=# select t.ticket_no, t.passenger_name, bp.boarding_no, bp.seat_no from tickets t
			left join boarding_passes bp on bp.ticket_no = t.ticket_no
			join ticket_flights tf on tf.ticket_no = t.ticket_no 
			join flights f on f.flight_id = tf.flight_id
			where f.arrival_airport = 'DME' and f.departure_airport = 'LED' and bp.boarding_no is null limit 10;			
	
Вывод в консоли:
	
	   ticket_no   |   passenger_name    | boarding_no | seat_no
	---------------+---------------------+-------------+---------
	 0005433102936 | OKSANA VLASOVA      |             |
	 0005432392044 | MIKHAIL GORBUNOV    |             |
	 0005433102886 | NADEZHDA MOROZOVA   |             |
	 0005432045642 | ELENA LAZAREVA      |             |
	 0005433102919 | VALENTINA KOROLEVA  |             |
	 0005432392041 | NADEZHDA FILATOVA   |             |
	 0005432020650 | GALINA FROLOVA      |             |
	 0005433102904 | SVETLANA KONDRATEVA |             |
	 0005433102920 | EVDOKIYA ANDREEVA   |             |
	 0005432020653 | VALERIY OSIPOV      |             |
	(10 rows)

Все эти люди купили билеты, но никуда не полетели.

Правостороннее соединение не буду рассматривать, по сути это же самое левостороннее, когда левая и правая таблицы поменяны местами в join.

---			
3) Кросс соединение двух или более таблиц 







Отключился от ВМ.
