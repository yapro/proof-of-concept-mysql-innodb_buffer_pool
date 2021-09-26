```shell
docker run -it --rm \
--name mysql-test \
-w /etc/mysql/conf.d \
-v $(pwd)/my.cnf:/etc/mysql/my.cnf \
-e MYSQL_ROOT_PASSWORD=my-secret-pw \
-p 3308:3306 \
mysql:5.7 --max_allowed_packet=5000M

docker exec -it mysql-test mysql -u root -pmy-secret-pw
```
Init информация:
```sql
Initializing buffer pool, total size = 128M, instances = 1, chunk size = 128M
Loading buffer pool(s) from /var/lib/mysql/ib_buffer_pool
Buffer pool(s) load completed at 210331 14:52:04
Shutting down plugin 'INNODB_BUFFER_POOL_STATS'
Shutting down plugin 'INNODB_BUFFER_PAGE_LRU'
Shutting down plugin 'INNODB_BUFFER_PAGE'
InnoDB: Dumping buffer pool(s) to /var/lib/mysql/ib_buffer_pool
InnoDB: Buffer pool(s) dump completed at 210331 14:52:07
```

Оценивать скорость будем запросом:
```sql
SELECT COUNT(*), MAX(n) FROM test_table;
```
*данный запрос будем выполнять до и после увеличения innodb_buffer_pool_size.

Создаю базу, таблицу и данные:
```sql
create database test_db;
create table test_table
(
id int auto_increment,
name nvarchar(255) not null,
constraint test_table_pk
primary key (id)
);

SET @name = REPEAT('a',255);
INSERT INTO test_table (name) SELECT SUBSTRING(CONCAT(n, @name), 1, 255) FROM generator_16m;
```

1. Генерируем 1 Гб таблицу с 2 полями (1 PK + 1 без индекса)
2. Смотрим дефолтные значения для параметров:
```sql
   SELECT @@innodb_buffer_pool_size/1024/1024/1024 =  
   SELECT @@innodb_buffer_pool_instances: 16
   SELECT @@innodb_buffer_pool_chunk_size: 128M
   X: innodb_buffer_pool_instances * innodb_buffer_pool_chunk_size = 2G
   тут важно что X кратно innodb_buffer_pool_size (нет остатка от деления innodb_buffer_pool_size % X)
```

3. Увеличиваем innodb_buffer_pool_size до 1 Гб и проверяем, что настройки применены:
```sql
SELECT TABLE_NAME AS `Table`, ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024) AS `Size (MB)`
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'test_db'
ORDER BY  (DATA_LENGTH + INDEX_LENGTH) DESC;

SHOW ENGINE INNODB STATUS;

SELECT * FROM INFORMATION_SCHEMA.INNODB_BUFFER_POOL_STATS\G;
```

```shell
=====================================
2021-03-26 09:20:42 0x7f9fb80d9700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 12 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 140 srv_active, 0 srv_shutdown, 928928 srv_idle
srv_master_thread log flush and writes: 929068
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 22476
OS WAIT ARRAY INFO: signal count 11969
RW-shared spins 0, rounds 38438, OS waits 18275
RW-excl spins 0, rounds 132025, OS waits 497
RW-sx spins 1164, rounds 21447, OS waits 269
Spin rounds per wait: 38438.00 RW-shared, 132025.00 RW-excl, 18.43 RW-sx
------------
TRANSACTIONS
------------
Trx id counter 154969
Purge done for trx's n:o < 154967 undo n:o < 0 state: running but idle
History list length 19
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421799271593824, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
830 OS file reads, 29669 OS file writes, 22291 OS fsyncs
4.17 reads/s, 16384 avg bytes/read, 2.33 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 11381 merges
merged operations:
insert 0, delete mark 0, delete 0
discarded operations:
insert 0, delete mark 0, delete 0
Hash table size 34673, node heap has 0 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 0 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 2 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
0.83 hash searches/s, 3.75 non-hash searches/s
---
LOG
---
Log sequence number 265587039
Log flushed up to   265587039
Pages flushed up to 265587039
Last checkpoint at  265587030
0 pending log flushes, 0 pending chkp writes
13677 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 625018
Buffer pool size   8191
Free buffers       5352
Database pages     2832
Old database pages 1047
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 752, not young 171
0.00 youngs/s, 0.00 non-youngs/s
Pages read 676, created 19381, written 8146
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 979 / 1000, young-making rate 0 / 1000 not 87 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 2832, unzip_LRU len: 0
I/O sum[71]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=1, Main thread ID=140323998537472, state: sleeping
Number of rows inserted 6359, updated 362, deleted 72, read 6684
40.75 inserts/s, 0.00 updates/s, 0.00 deletes/s, 70.41 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================
```

*если желаете попробовать на своих данных:
```shell
mysql -u root -p --force name_database < /path/to/file.sql
```

Дамп таблиц заливается такой командой (перед этим не забудьте зайти в директорию с файлами дампов):
```shell
for i in `ls −1 *.sql` ; do echo processing $i ; mysql sv < $i ; done
```

Кстати при заливке таблицы db_sv_test_relations_copy получил такую ошибку:
```sql
processing db_sv_test_relations_copy.sql
ERROR 1071 (42000) at line 25: Specified key was too long; max key length is 1000 bytes
```

Проблема возникает из-за того, что mysqldump делает недостаточно правильные CREATE TABLE, забывая про кодировки в 
которых находятся поля и сама таблица. Например, таблица в кодировке utf8_general_ci, а CREATE TABLE будет просто utf8.
