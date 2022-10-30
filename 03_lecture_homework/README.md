# postgre_2022_10

**3-я лекция**

ВМ с ОС Ubuntu 22.04 LTS у меня уже есть в облаке Яндекса.

Мой логин на ВМ - artem.
Публичный IPv4  - меняется каждый раз после остановки и запуска ВМ.

Законнектился к ВМ с помощью PuTTY. 

Уставил Docker Engine на ВМ, командная строка:

	artem@postgre-2022-10:~$ curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER
	
Создал docker-сеть, командная строка:

	artem@postgre-2022-10:~$ sudo docker network create pg-net
	
Вывод в консоли:	
	
	64d999e6bea1303b1c918182b9966496d0f611f978828dba770e0bffd509c693

Каталог /var/lib/postgres у меня уже был, проверяю, командная строка:

	artem@postgre-2022-10:~$ pg_lsclusters
	
Вывод в консоли:
	
	Ver Cluster Port Status Owner    Data directory              Log file
	14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
	
Подключаю docker-сеть, к контейнеру сервера PG, с монтированием в него /var/lib/postgres. Командная строка:

	artem@postgre-2022-10:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

Вывод в консоли:

	docker: Error response from daemon: driver failed programming external connectivity on endpoint pg-server (ac19d0d5657c64222dfcb342c068f82754439d79683a6682b91316516aa02a7d): Error starting userland proxy: listen tcp4 0.0.0.0:5432: bind: address already in use.
	
Понимаю, что надо остановить и удалить кластер PG 14  main. Командная строка:

	artem@postgre-2022-10:~$ sudo pg_ctlcluster 14 main stop
	artem@postgre-2022-10:~$ sudo pg_dropcluster 14 main
	
Опять подключаю docker-сеть, к контейнеру сервера PG.

Вывод в консоли:

	docker: Error response from daemon: Conflict. The container name "/pg-server" is already in use by container "18d452e00f559fe0e078bbab0188bb5487c96ef18e4ceb909646a1d007702a60". You have to remove (or rename) that container to be able to reuse that name.
	
Понимаю, что придется удалить контейнер pg-server. Командная строка:

	artem@postgre-2022-10:~$ sudo docker rm pg-server
	
Снова подключаю docker-сеть, к контейнеру сервера PG. Командная строка:

	artem@postgre-2022-10:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

Вывод в консоли:

	04abde3706d4bb6aaf89816dfe3739478aa755ebd3e7f209f2e3b6ee1fe1ddaf

Теперь запускаю отдельный контейнер с клиентом PG в общей сети с БД. Командная строка: 

	artem@postgre-2022-10:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres
	
Вывод в консоли:
	
	Password for user postgres:
	
Указываю пароль - postgres
Вывод в консоли:
	
	postgres=#

Посмотрел список БД. Командная строка:
	
	postgres-# \l
	
Их три - postgres, template0, template1.

Создаю БД iso, командная строка: 

	postgres=# CREATE DATABASE iso;
	postgres=# \c iso
	
Вывод в консоли:

	You are now connected to database "iso" as user "postgres".
	iso=#
	
Создаю таблицу и в ней 2 записи. Командная строка:  

	iso=# CREATE TABLE testA (num serial, amount int);
	iso=# INSERT INTO testA(amount) VALUES (3);
	iso=# INSERT INTO testA(amount) VALUES (4);
	iso=# SELECT * FROM testA;

Вывод в консоли:

	num | amount
	-----+--------
	1 |      3
	2 |      4
	(2 rows)
	
Запускаю на ноутбуке PowerShell. Подключаюсь к ВМ в облаке Яндекса со своим логином artem (не получается), затем с ubuntu (не получается). Командная строка:

	PS C:\Users\aibel> ssh artem@51.250.27.11
	PS C:\Users\aibel> ssh ubuntu@51.250.27.11
	
Вывод в консоли: 

	PS C:\Users\aibel> artem@51.250.27.11: Permission denied (publickey).
	PS C:\Users\aibel> ubuntu@51.250.27.11: Permission denied (publickey).
	
Ключ ssh был сгенерирован PuTTY. Решил сделать вторую ВМ, для нее сгенерить ssh так, как описано в Яндексе https://cloud.yandex.ru/docs/compute/operations/vm-connect/ssh
PowerShell. Командная строка:

	PS C:\Users\aibel> ssh-keygen -t ed25519
	
Запрашивается имя файла для ключа. Ввожу: id_artem
В каталоге C:\Users\aibel генерится два ключа - приватный id_artem и публичный id_artem.pub
Открываю id_artem.pub в блокноте, копирую текст в поле SSH-ключ для новой ВМ.

Подключаюсь из PowerShell к ВМ в облаке Яндекса с новым ключом, у которого явно задан путь (ЭТО ВАЖНО). Командная строка:
	
	PS C:\Users\aibel> ssh -i C:\Users\aibel\id_artem artem@51.250.27.11

Вывод в консоли: 

	Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-52-generic x86_64)
	artem@postgre-2022-10-new:~$
	
Всё получилось. Кстати, через PuTTY с новым ключом не подключается.
Теперь надо на новой ВМ опять установить репозиторий PostgreSQL. Установить Docker Engine, развернуть контейнер с сервером и клиентом postgres, создать тестовую таблицу testA и т.д.
Всё сделал, строки в тестовой таблице завёл. Отключился от ВМ.

Подключился из PowerShell к ВМ. Для проверки подключился к кластеру Postgre в Docker, используя адрес localhost. Командная строка:

	artem@postgre-2022-10-new:~$ psql -h localhost -U postgres -d postgres
	
Указывал запрошенный пароль - postgres
Вывод в консоли:
	
	postgres=#

Отключился от ВМ.
Подключаюсь к контейнеру Docker с сервером Postgre с ноутбука из PowerShell. Командная строка: 

	PS C:\Users\aibel> psql -p 5432 -U postgres -h 51.250.27.11 -d postgres -W

Вывод в консоли:		

	psql : Имя "psql" не распознано как имя командлета, функции, файла сценария или выполняемой программы. Проверьте правильность написания имени, а также наличи
е и правильность пути, после чего повторите попытку.

Не ясно, как подключиться к контейнеру Docker с сервером Postgre извне, не заходя на ВМ в облаке.
Опять зашёл на ВМ.
Подключился к кластеру Postgre в Docker, используя публичный IPv4 ВМ. Командная строка:

	artem@postgre-2022-10-new:~$ psql -p 5432 -U postgres -h 51.250.27.11 -d postgres -W
	
Всё ок. Вышел из кластера Postgre в Docker.
Остановил контейнер сервера PG. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo docker stop pg-server
	
Удалил контейнер сервера PG. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo docker rm pg-server
	
Опять создаю контейнер сервера PG. Командная строка:

	artem@postgre-2022-10-new:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
	
Опять запускаю отдельный контейнер с клиентом PG в общей сети с БД. Командная строка: 

	artem@postgre-2022-10-new:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres

Посмотрел список БД. Командная строка:
	
	postgres-# \l
	
Их четыре - iso, postgres, template0, template1. БД iso, созданная мной до удаления контейнера сервера PG, - в наличии.

Перехожу в БД iso, командная строка: 

	postgres=# \c iso
	
Вывод в консоли:

	You are now connected to database "iso" as user "postgres".
	iso=#
	
Делаю селект из своей тестовой таблицы testA, командная строка: 

	iso=# SELECT * FROM testA;

Вывод в консоли:

	num | amount
	-----+--------
	1 |      3
	2 |      4
	(2 rows)
	
Все данные остались на месте. Отключаюсь.