1. тестовая таблица
```
CREATE TABLE testDb (i serial, amount int);
INSERT INTO testDb(amount) VALUES (100),(500);
```

до настройки логирования блокировок: 
```
SHOW log_lock_waits;
 log_lock_waits
----------------
 off
(1 row)
```

2. выполним настройку
```
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = 200;
SELECT pg_reload_conf();
SHOW deadlock_timeout;
SHOW log_lock_waits;


ALTER SYSTEM
ALTER SYSTEM
 pg_reload_conf
----------------
 t
(1 row)

 deadlock_timeout
------------------
 200ms
(1 row)

 log_lock_waits
----------------
 on
(1 row)
```

3. добавление записей в журнал блокировок

включение лог коллектора: 
```
alter system set logging_collector = on;
```
 
 запустим в 2 сеансах 2 транзакции незакоммиченные
```
BEGIN;
UPDATE testDb SET amount = 3 WHERE i = 1;

BEGIN;
UPDATE testDb SET amount = 1 WHERE i = 1;
```

журнал логов после коммита операций: 
 ```
tail -n 10 /var/lib/postgresql/data/log/postgresql-2025-07-28_133223.log

2025-07-28 13:37:23.267 UTC [27] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.013 s, sync=0.006 s, total=0.086 s; sync files=2, longest=0.003 s, average=0.003 s; distance=0 kB, estimate=0 kB; lsn=0/7F972978, redo lsn=0/7F972940
2025-07-28 13:39:06.997 UTC [102] LOG:  process 102 still waiting for ShareLock on transaction 235083 after 200.608 ms
2025-07-28 13:39:06.997 UTC [102] DETAIL:  Process holding the lock: 41. Wait queue: 102.
2025-07-28 13:39:06.997 UTC [102] CONTEXT:  while updating tuple (0,7) in relation "testdb"
2025-07-28 13:39:06.997 UTC [102] STATEMENT:  UPDATE testDb SET amount = 22 WHERE i = 1;
2025-07-28 13:39:26.923 UTC [102] LOG:  process 102 acquired ShareLock on transaction 235083 after 20126.733 ms
2025-07-28 13:39:26.923 UTC [102] CONTEXT:  while updating tuple (0,7) in relation "testdb"
2025-07-28 13:39:26.923 UTC [102] STATEMENT:  UPDATE testDb SET amount = 22 WHERE i = 1;
2025-07-28 13:42:23.308 UTC [27] LOG:  checkpoint starting: time
2025-07-28 13:42:23.529 UTC [27] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.115 s, sync=0.047 s, total=0.222 s; sync files=2, longest=0.044 s, average=0.024 s; distance=0 kB, estimate=0 kB; lsn=0/7F972D20, redo lsn=0/7F972CE8

 ```

4. блокировка 3 транзакций
```
locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid |       mode       | granted | fastpath |           waitstart
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-----+------------------+---------+----------+-------------------------------
 relation      |        5 |    12073 |      |       |            |               |         |       |          | 6/14               | 243 | AccessShareLock  | t       | t        |
 virtualxid    |          |          |      |       | 6/14       |               |         |       |          | 6/14               | 243 | ExclusiveLock    | t       | t        |
 relation      |        5 |    24637 |      |       |            |               |         |       |          | 5/218              | 235 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 5/218      |               |         |       |          | 5/218              | 235 | ExclusiveLock    | t       | t        |
 relation      |        5 |    24637 |      |       |            |               |         |       |          | 4/109              | 102 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 4/109      |               |         |       |          | 4/109              | 102 | ExclusiveLock    | t       | t        |
 relation      |        5 |    24637 |      |       |            |               |         |       |          | 3/7                |  41 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 3/7        |               |         |       |          | 3/7                |  41 | ExclusiveLock    | t       | t        |
 transactionid |          |          |      |       |            |        235087 |         |       |          | 5/218              | 235 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |        235085 |         |       |          | 3/7                |  41 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |        235085 |         |       |          | 4/109              | 102 | ShareLock        | f       | f        | 2025-07-28 13:52:55.769093+00
 tuple         |        5 |    24637 |    0 |     9 |            |               |         |       |          | 4/109              | 102 | ExclusiveLock    | t       | f        |
 tuple         |        5 |    24637 |    0 |     9 |            |               |         |       |          | 5/218              | 235 | ExclusiveLock    | f       | f        | 2025-07-28 13:53:02.298431+00
 transactionid |          |          |      |       |            |        235086 |         |       |          | 4/109              | 102 | ExclusiveLock    | t       | f        |
(14 rows)

```

после комита  транзакций: 
```
  locktype  | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid |      mode       | granted | fastpath | waitstart
------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-----+-----------------+---------+----------+-----------
 relation   |        5 |    12073 |      |       |            |               |         |       |          | 6/15               | 243 | AccessShareLock | t       | t        |
 virtualxid |          |          |      |       | 6/15       |               |         |       |          | 6/15               | 243 | ExclusiveLock   | t       | t        |
(2 rows)
```

Выводы: для блокировки обновления одного и того же ресурса 
- RowExclusiveLock режим блокировки уровня таблицы, который предотвращает одновременное изменение данных в таблице другими транзакциями, но при этом позволяет параллельное чтение данных. 
- ExclusiveLock - блокирование данных на чтение и изменение
- ShareLock - режим блокировки таблицы, который позволяет нескольким транзакциям одновременно читать данные из таблицы, но при этом запрещает другим транзакциям изменять эту таблицу

5. взаимные блокировки
пример: взаимное обновление данных 2 полей

 ```
 ### 1 console
1) BEGIN;
UPDATE testDb SET amount = amount + 1 WHERE i = 1;

### 2 console
2) BEGIN;
UPDATE testDb SET amount = amount - 2 WHERE i = 2;

### 1 console
3) UPDATE testDb SET amount = amount + 1 WHERE i = 2;

### 2 console
4) UPDATE testDb SET amount = amount + 2 WHERE i = 1;

 ```

 после выполнение 4) команды, получили ошибку: 
 ```
ERROR:  deadlock detected
DETAIL:  Process 235 waits for ShareLock on transaction 235096; blocked by process 41.
Process 41 waits for ShareLock on transaction 235097; blocked by process 102.
Process 102 waits for ShareLock on transaction 235098; blocked by process 235.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,12) in relation "testdb"
 ```
  причина: 
- первая транзакция изменяет данные i = 1
- вторая транзакция изменяет данные i = 2
- третья транзакция изменяет данные i = 3
- после последующих апдейтов этих строк, изменение которых не закоммичено, возникает циклическое ожидание. первая транзакция, не получив доступ к ресурсу, инициирует проверку взаимоблокировки и обрывается сервером



 данные в журнале сообщений: 

```
2025-07-28 17:32:36.211 UTC [102] LOG:  process 102 still waiting for ShareLock on transaction 235098 after 200.475 ms
2025-07-28 17:32:36.211 UTC [102] DETAIL:  Process holding the lock: 235. Wait queue: 102.
2025-07-28 17:32:36.211 UTC [102] CONTEXT:  while updating tuple (0,16) in relation "testdb"
2025-07-28 17:32:36.211 UTC [102] STATEMENT:  UPDATE testDb SET amount = amount + 2 WHERE i = 3;
2025-07-28 17:32:57.022 UTC [235] LOG:  process 235 detected deadlock while waiting for ShareLock on transaction 235096 after 200.913 ms
2025-07-28 17:32:57.022 UTC [235] DETAIL:  Process holding the lock: 41. Wait queue: .
2025-07-28 17:32:57.022 UTC [235] CONTEXT:  while updating tuple (0,12) in relation "testdb"
2025-07-28 17:32:57.022 UTC [235] STATEMENT:  UPDATE testDb SET amount = amount +5 WHERE i = 1;
2025-07-28 17:32:57.023 UTC [235] ERROR:  deadlock detected
2025-07-28 17:32:57.023 UTC [235] DETAIL:  Process 235 waits for ShareLock on transaction 235096; blocked by process 41.
        Process 41 waits for ShareLock on transaction 235097; blocked by process 102.
        Process 102 waits for ShareLock on transaction 235098; blocked by process 235.
        Process 235: UPDATE testDb SET amount = amount +5 WHERE i = 1;
        Process 41: UPDATE testDb SET amount = amount + 1 WHERE i = 2;
        Process 102: UPDATE testDb SET amount = amount + 2 WHERE i = 3;
2025-07-28 17:32:57.023 UTC [235] HINT:  See server log for query details.
2025-07-28 17:32:57.023 UTC [235] CONTEXT:  while updating tuple (0,12) in relation "testdb"
2025-07-28 17:32:57.023 UTC [235] STATEMENT:  UPDATE testDb SET amount = amount +5 WHERE i = 1;
2025-07-28 17:32:57.023 UTC [102] LOG:  process 102 acquired ShareLock on transaction 235098 after 21012.363 ms
2025-07-28 17:32:57.023 UTC [102] CONTEXT:  while updating tuple (0,16) in relation "testdb"
2025-07-28 17:32:57.023 UTC [102] STATEMENT:  UPDATE testDb SET amount = amount + 2 WHERE i = 3;
```

как видно из логов, последовательно описан процесс ожидания выполнения транзакции с последующим событием дедлока - взаимной блокировкой транзакций доступа к одному ресурсу.


