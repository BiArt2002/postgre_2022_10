# postgre_2022_10

**8-я лекция**

ВМ с ОС Ubuntu 22.04 LTS у меня уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@158.160.12.33

PostgreSQL 14 уже был установлен.

Проверяю, что кластер PG запущен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory    Log file
	14  main    5432 online postgres /mnt/vdb1/14/main /var/log/postgresql/postgresql-14-main.log	

Удалил все свои тестовые базы.	
Запустил из-под sudo Midnight Commander. Командная строка:

	artem@postgre-2022-10-new:~$ sudo mc

Зашел в папку, где лежат конфигурационные файлы: /etc/postgresql/14/main
Установил параметры кластера в файле postgresql.conf в соответствии с заданием.	
	
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB	

Чтобы все параметры точно применились, решил перезапустить PG.
Остановил PG. Командная строка: 	
	
	artem@postgre-2022-10-new:~$ sudo systemctl stop postgresql
	
Запустил PG. Командная строка: 	

	artem@postgre-2022-10-new:~$ sudo systemctl start postgresql
	
Проверяю, что кластер PG запущен. Показывает, что нет. Начались танцы с бубном. 
Затем решил войти в psql - все ок, и запросы sql вполняются. А кластер PG якобы в дауне.
В итоге остановил ВМ. Зашел снова. Показывает, что кластер PG запущен. 
В следующий раз буду делать рестарт PG, а не остановку и запуск.

Выполнил команду:

	artem@postgre-2022-10-new:~$ pgbench -i postgres
	
Вывод в консоль:

	pgbench: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  role "artem" does not exist
		
Значит, надо войти под пользователем postgres. 
Тут 2 варианта:
1) зайти под пользователем postgres, команда: "sudo su postgres" и оттуда выполнить pgbench ...
2) сразу выполнить: sudo -u postgres pgbench ...

Выполнил команду так:

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -i postgres
	
Вывод в консоль:
	
	done in 1.28 s (drop tables 0.13 s, create tables 0.31 s, client-side generate 0.29 s, vacuum 0.32 s, primary keys 0.23 s).
 
Далее запустил бенчмарк. Командная строка:   

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -c8 -P 60 -T 600 -U postgres postgres
	
Вывод в консоль:
	
	progress: 60.0 s, 445.3 tps, lat 17.957 ms stddev 18.781
	progress: 120.0 s, 583.1 tps, lat 13.718 ms stddev 12.385
	progress: 180.0 s, 567.5 tps, lat 14.099 ms stddev 12.384
	progress: 240.0 s, 430.9 tps, lat 18.564 ms stddev 21.235
	progress: 300.0 s, 476.2 tps, lat 16.802 ms stddev 14.400
	progress: 360.0 s, 585.0 tps, lat 13.676 ms stddev 11.864
	progress: 420.0 s, 546.4 tps, lat 14.642 ms stddev 14.053
	progress: 480.0 s, 419.1 tps, lat 19.090 ms stddev 19.736
	progress: 540.0 s, 616.1 tps, lat 12.983 ms stddev 10.810
	progress: 600.0 s, 514.2 tps, lat 15.562 ms stddev 14.308
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 8
	number of threads: 1
	duration: 600 s
	number of transactions actually processed: 311015
	latency average = 15.433 ms
	latency stddev = 15.084 ms
	initial connection time = 16.108 ms
	tps = 518.350760 (without initial connection time)

Видно, что tps не очень равномерный.
Можно поиграть с настройками autovacuum и добиться более равномерного значения tps.
Но я разработчик, а не DBA и не DevOps. Заниматься подобными настройками я потом не буду. Пропущу это.  
 
Отключился от ВМ.