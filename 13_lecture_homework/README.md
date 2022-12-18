# postgre_2022_10

**13-я лекция**

ВМ с ОС Ubuntu 22.04 LTS - уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с ключом ssh, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@51.250.108.165

Установил с нуля PostgreSQL 14.

Проверяю, что кластер PG запущен, командная строка:

	artem@postgre-2022-10-new:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
	
Midnight Commander уже установлен - чтоб удобнее было ходить по каталогам.

---	
	
Зашел в папку, где лежат конфигурационные файлы: /etc/postgresql/14/main
В файле postgresql.conf в параметре include_dir указал папку, где будут храниться файлы с расширением *.conf c параметрами, с которыми я буду эксперементировать.
Так удобнее и быстрее менять параметры и при этом не "испортить" конфиг postgresql.conf.

	include_dir='conf.d'
	
Создал с помощью Midnight Commander папку conf.d  в /etc/postgresql/14/main

Затем сгенерировал рекомендуемые параметры на сайте https://pgtune.leopard.in.ua
В соответствии с характеристиками ВМ: 2 vCPU, RAM 4 ГБ, HDD, установлен PG 14.

В папке conf.d c помощью редактора Nano создал файл pgtune.conf и вставил в него параметры.

	max_connections = 100
	shared_buffers = 1GB
	effective_cache_size = 3GB
	maintenance_work_mem = 256MB
	checkpoint_completion_target = 0.9
	wal_buffers = 16MB
	default_statistics_target = 100
	random_page_cost = 4
	effective_io_concurrency = 2
	work_mem = 2621kB
	min_wal_size = 1GB
	max_wal_size = 4GB

Рестартовал кластер PG для применения параметров. Командная строка:

	artem@postgre-2022-10-new:~$ sudo pg_ctlcluster 14 main restart
	
Проверил, что параметры применились (на примере shared_buffers). Командная строка:

	postgres=# show shared_buffers;
	
Вывод в консоль:

	 shared_buffers
	----------------
	 1GB
	 
Все ок, т.к. из коробки shared_buffers = 128MB.
Подготовил базу postgres к выполнению pgbench. Командная строка:	

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -i postgres
	
Вывод в консоль (последняя строка):

	done in 0.50 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.27 s, vacuum 0.09 s, primary keys 0.12 s).

В течение 30 сек подавал нагрузку на базу postgres в синхронном режиме (включен по умолчанию).
Задал в нагрузке 50 коннектов, в 2 потоках, инфа о прогрессе выводится каждые 10 сек. Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 30 postgres
	
Вывод в консоль (строки с прогрессом):
	
	progress: 10.0 s, 629.0 tps, lat 78.406 ms stddev 72.431
	progress: 20.0 s, 767.2 tps, lat 65.174 ms stddev 51.206
	progress: 30.0 s, 733.8 tps, lat 68.121 ms stddev 56.310
	
Максимум 767.2 tps. Среднее - 710. В дальнейшем всё буду сравнивать с этими значениями.

Затем сделал более нагруженный расширенный тест, параметр -M.
Задал в нагрузке 50 коннектов, но каждая транзакция будет в новом коннекте (параметр -C), в 2 потоках, инфа о прогрессе выводится каждые 10 сек. Командная строка:

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -c 50 -C -j 2 -P 10 -T 30 -M extended postgres	
	
Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 200.8 tps, lat 233.215 ms stddev 296.717
	progress: 20.0 s, 156.6 tps, lat 321.636 ms stddev 380.229
	progress: 30.0 s, 199.7 tps, lat 248.445 ms stddev 292.197

Вижу, как просели tps, максимум уже 200.8 tps. Среднее 185,7.

Теперь буду пробовать менять параметры, чтобы добиться максимальной производительности.
Буду задавать обычную нагрузку в 50 коннектов, в 2 потоках, инфа о прогрессе выодится каждые 10 сек.
 
1) Увеличу shared_buffers (кеш данных в памяти). Подниму сразу до 40% от RAM 4 ГБ на машине. Значения выше 40% уже редко влияют на производительность.

	shared_buffers = 1.6GB
	
Почему-то tps стали хуже. Проверил через команду "show shared_buffers" изменение параметра. Он не был изменен. Понял, что забыл перестартовать PG.
Однако рестартовать не смог. Выдалась ошибка.

	Job for postgresql@14-main.service failed because the service did not take the steps required by its unit configuration.
	See "systemctl status postgresql@14-main.service" and "journalctl -xeu postgresql@14-main.service" for details.

Команда pg_lsclusters показала, что кластер в дауне.

	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

Любые дальнейшие команды restart и start сопровождались той же ошибкой.
Удалил кластер, создал новый, стартовал. Та же самая ошибка.
Остановил ВМ, запустил снова. Кластер PG был так же в дауне, стартовать не смог.

Выполнил команду из сообщения об ошибке:
	
	artem@postgre-2022-10-new:~$ systemctl status postgresql@14-main.service
	
Увидел содержание ошибки:

	syntax error in file "/etc/postgresql/14/main/conf.d/pgtune.conf" line 2, near token "GB"
	
Вернул для проверки в настройках shared_buffers = 1GB, и кластер PG стартовал.
Делаю вывод, что мое значение 1.6GB было некорректно написанным, поэтому задам его в мегабайтах, это примерно 1638MB, и проверю параметр после рестарта.

	 shared_buffers
	----------------
	 1638MB

Задаю нагрузку с помощью pgbench.

	artem@postgre-2022-10-new:~$ sudo -u postgres pgbench -c 50 -j 2 -P 10 -T 30 postgres
	
Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 386.9 tps, lat 126.410 ms stddev 122.259
	progress: 20.0 s, 754.3 tps, lat 66.906 ms stddev 51.663
	progress: 30.0 s, 695.4 tps, lat 71.379 ms stddev 61.470

Максимум 754.3 tps. Среднее - 612,23. По сравнению с первым нагрузочным тестом лучше не стало.
Уменьшу shared_buffers, но чтоб было более 1GB. Задам как 1228MB. Результаты теста.

	progress: 10.0 s, 590.2 tps, lat 83.315 ms stddev 69.646
	progress: 20.0 s, 473.3 tps, lat 106.288 ms stddev 105.562
	progress: 30.0 s, 800.7 tps, lat 62.393 ms stddev 46.834
	
Максимум 800.7 tps. Среднее - 621,4. Уже лучше, но среднее не дотягивает до первого результата.

2) Увеличу effective_cache_size (подсказка для планировщика, сколько оперативки у него в запасе). Обычное значение 75% RAM. У меня это 3GB.
Поставлю 3584MB, т.к. других программ в системе кроме PG практически нет. И верну shared_buffers в 1GB.

Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 681.6 tps, lat 72.619 ms stddev 58.243
	progress: 20.0 s, 468.1 tps, lat 105.918 ms stddev 84.220
	progress: 30.0 s, 361.4 tps, lat 138.793 ms stddev 101.803

Максимум 681.6 tps. Среднее - 503,7. До результатов первого нагрузочного теста по-прежнему далеко.

Раз результаты с обычным тестом при "разгоне" параметров не улучшаются, то попробую выполнить более нагруженный расширенный тест с увеличенным effective_cache_size.

Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 204.1 tps, lat 231.567 ms stddev 304.015
	progress: 20.0 s, 211.7 tps, lat 233.128 ms stddev 280.707
	progress: 30.0 s, 249.2 tps, lat 200.429 ms stddev 232.558
	
По сравнению с прежним "тяжелым" тестом (выполненным единожды) есть прирост. Максимум 249.2 tps, а было 200.8. Среднее - 221,67, а было 185,7.

Буду теперь давать нагрузку "тяжелым" тестом.
Попробую опять увеличить shared_buffers до 1228MB.

Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 206.2 tps, lat 231.936 ms stddev 265.471
	progress: 20.0 s, 252.8 tps, lat 194.841 ms stddev 245.328
	progress: 30.0 s, 236.9 tps, lat 207.599 ms stddev 233.571
	
Есть небольшой прирост. Максимум 252.8 tps. Среднее - 231,97. 

3) Увеличу work_mem (чтобы больше операций при сортировках и построении hash таблиц выполнялось в памяти, а обращение к жесткому диску уменьшилось) до 4MB.

Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 242.9 tps, lat 197.924 ms stddev 193.144
	progress: 20.0 s, 193.3 tps, lat 253.226 ms stddev 305.660
	progress: 30.0 s, 222.0 tps, lat 220.751 ms stddev 263.441
	
Стало хуже. Максимум 222.0 tps. Среднее - 219,4.
Увеличивал до 8MB и более, всё еще хуже становилось.

Верну work_mem обратно - 2621kB.

4) Увеличу maintenance_work_mem (максимальное количество ОП для операций типа VACUUM, CREATE INDEX, CREATE FOREIGN KEY) до 512MB.

Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 225.3 tps, lat 209.814 ms stddev 238.712
	progress: 20.0 s, 252.3 tps, lat 195.092 ms stddev 249.093
	progress: 30.0 s, 230.6 tps, lat 215.049 ms stddev 277.671

Есть крохотный прирост (в сравнении с увеличением shared_buffers до 1228MB). Максимум чуть недотянул - 252.3 tps. А среднее чуть выше - 236,1. 

Увеличил maintenance_work_mem до 1 GB.

Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 226.7 tps, lat 210.927 ms stddev 235.572
	progress: 20.0 s, 227.2 tps, lat 215.229 ms stddev 258.108
	progress: 30.0 s, 236.6 tps, lat 207.624 ms stddev 225.766

Стало похуже. Максимум 236.6. Среднее - 230,2.

Вернул maintenance_work_mem к значению - 512MB.

5) Увеличил wal_buffers (объём разделяемой памяти, используемый для буферизации данных WAL) до 32MB.

Вывод в консоль (строки с прогрессом):

	progress: 10.0 s, 234.3 tps, lat 203.679 ms stddev 240.908
	progress: 20.0 s, 230.5 tps, lat 212.626 ms stddev 274.682
	progress: 30.0 s, 230.1 tps, lat 212.517 ms stddev 238.383
	
До теста, где я увеличивал maintenance_work_mem до 512MB не дотянул. Максимум 230.5 tps. Среднее - 231,6	

Вернул wal_buffers к значению - 16MB.

Закончил тестирование нагрузки.

Отключился от ВМ.