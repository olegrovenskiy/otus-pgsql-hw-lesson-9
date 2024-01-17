# otus-pgsql-hw-lesson-9

## 1. Настройте выполнение контрольной точки раз в 30 секунд.

        postgres=# alter system set checkpoint_timeout = '30s';
        ALTER SYSTEM
        postgres=#

С последующим рестартом кластера


## 2. 10 минут c помощью утилиты pgbench подавайте нагрузку.
 
 pgbench -c8 -P 30 -T 600 -U postgres postgres с выводом каждые 30 секунд

                bash-4.2$ pgbench -c8 -P 30 -T 600 -U postgres postgres
                Password:
                pgbench (15.5)
                starting vacuum...end.
                progress: 30.0 s, 1673.3 tps, lat 4.770 ms stddev 2.141, 0 failed
                progress: 60.0 s, 1631.1 tps, lat 4.899 ms stddev 2.234, 0 failed
                progress: 90.0 s, 1601.8 tps, lat 4.988 ms stddev 2.318, 0 failed
                progress: 120.0 s, 1573.0 tps, lat 5.079 ms stddev 2.354, 0 failed
                progress: 150.0 s, 1589.2 tps, lat 5.027 ms stddev 2.264, 0 failed
                progress: 180.0 s, 1555.4 tps, lat 5.137 ms stddev 2.373, 0 failed
                progress: 210.0 s, 1603.2 tps, lat 4.982 ms stddev 2.172, 0 failed
                progress: 240.0 s, 1575.2 tps, lat 5.072 ms stddev 2.291, 0 failed
                progress: 270.0 s, 1588.7 tps, lat 5.028 ms stddev 2.214, 0 failed
                progress: 300.0 s, 1556.6 tps, lat 5.132 ms stddev 2.327, 0 failed
                progress: 330.0 s, 1563.8 tps, lat 5.109 ms stddev 2.299, 0 failed
                progress: 360.0 s, 1551.0 tps, lat 5.151 ms stddev 2.416, 0 failed
                progress: 390.0 s, 1556.7 tps, lat 5.132 ms stddev 2.315, 0 failed
                progress: 420.0 s, 1594.0 tps, lat 5.012 ms stddev 2.355, 0 failed
                progress: 450.0 s, 1634.1 tps, lat 4.889 ms stddev 2.133, 0 failed
                progress: 480.0 s, 1605.3 tps, lat 4.977 ms stddev 2.310, 0 failed
                progress: 510.0 s, 1615.7 tps, lat 4.945 ms stddev 2.192, 0 failed
                progress: 540.0 s, 1611.9 tps, lat 4.956 ms stddev 2.256, 0 failed
                progress: 570.0 s, 1620.3 tps, lat 4.931 ms stddev 2.156, 0 failed
                progress: 600.0 s, 1605.3 tps, lat 4.977 ms stddev 2.306, 0 failed
                transaction type: <builtin: TPC-B (sort of)>
                scaling factor: 1
                query mode: simple
                number of clients: 8
                number of threads: 1
                maximum number of tries: 1
                duration: 600 s
                number of transactions actually processed: 957181
                number of failed transactions: 0 (0.000%)
                latency average = 5.008 ms
                latency stddev = 2.274 ms
                initial connection time = 23.514 ms
                tps = 1595.332627 (without initial connection time)
                bash-4.2$




## 3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.


                postgres=# SELECT * FROM pg_ls_waldir();
                           name           |   size   |      modification
                --------------------------+----------+------------------------
                 0000000100000002000000FD | 16777216 | 2023-12-31 01:19:10-05
                .....................
                .....................
                 0000000100000002000000FB | 16777216 | 2023-12-31 01:18:41-05
                 0000000100000002000000FC | 16777216 | 2023-12-31 01:18:50-05
                (64 rows)
                
                postgres=# 

было сгенерено 64 файла по 16М, то есть по 1 файлу на одну контрольную точку


## 4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

Статистику можно посмотреть следующим образом:

                postgres=# SELECT * FROM pg_stat_bgwriter \gx
                -[ RECORD 1 ]---------+------------------------------
                checkpoints_timed     | 5461
                checkpoints_req       | 27
                checkpoint_write_time | 3682075
                checkpoint_sync_time  | 2876
                buffers_checkpoint    | 111647
                buffers_clean         | 200974
                maxwritten_clean      | 1184
                buffers_backend       | 2319236
                buffers_backend_fsync | 0
                buffers_alloc         | 1683520
                stats_reset           | 2023-12-29 05:49:51.267617-05
                
                postgres=#


        checkpoints_timed     | 5461 - эти контрольные точки выполнялись по расписанию
        
        checkpoints_req       | 27 - checkpoints_req — выполнение по требованию, не по распиманию (в том числе по достижению max_wal_size).


## 5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

                pgbench -c8 -P 6 -T 60 -U postgres postgres
                
                postgres=# ALTER SYSTEM SET synchronous_commit = on;
                ALTER SYSTEM
                postgres=#
                
                tps = 1636.367446 (without initial connection time)

тестируем при асинхронном режиме 

                postgres=# ALTER SYSTEM SET synchronous_commit = off;
                ALTER SYSTEM
                postgres=#
                
                tps = 2476.425078 (without initial connection time)

При синхронной фиксации получали примерно 1600 транзакций в секунду (tps), при асинхронной — 2500. 

Асинхронный режим более производительный, т.к. Этот алгоритм нацелен на то, чтобы по возможности не синхронизировать одну и ту же страницу несколько раз, что важно при большом потоке изменений. Асинхронная запись эффективнее синхронной — фиксация изменений не ждет записи. Но надежность уменьшается.
Подробно расписано на https://habr.com/ru/companies/postgrespro/articles/461523/

## 6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблиц

### 6.1 Создайте новый кластер с включенной контрольной суммой страниц.

Проверяем включены ли проверки чексум

                pg_controldata /var/lib/pgsql/15/data
                
                
                bash-4.2$ psql
                Password for user postgres:
                psql (15.5)
                Type "help" for help.
                
                postgres=# show data_checksums;
                 data_checksums
                ----------------
                 off
                (1 row)
                
                postgres=#

для включения проверки чексум останавливаем кластер

                 systemctl stop postgresql-15

включаем чек суму

                bash-4.2$ pg_checksums -e /var/lib/pgsql/15/data
                Checksum operation completed
                Files scanned:   967
                Blocks scanned:  195668
                Files written:  801
                Blocks written: 195668
                pg_checksums: syncing data directory
                pg_checksums: updating control file
                Checksums enabled in cluster
                bash-4.2$

запускаем кластер
                 systemctl start postgresql-15

можем проверить

                postgres=# show data_checksums;
                 data_checksums
                ----------------
                 on
                (1 row)

### 6.2 Создайте таблицу. Вставьте несколько значений.

 
                create table test_text(t text);
                
                insert into test_text (t) values ('test1');
                insert into test_text (t) values ('test2');
                insert into test_text (t) values ('test3');
                
                postgres=# select * from test_text;
                   t
                -------
                 test1
                 test2
                 test3
                (3 rows)
                
                postgres=#

### 6.3 Измените пару байт в таблице.

Ищем где находится файл с таблицей

                postgres=# SELECT pg_relation_filepath('test_text');
                 pg_relation_filepath
                ----------------------
                 base/5/16622
                (1 row)

Файл присутствует в директории

                bash-4.2$ ls -l | grep 16622
                -rw-------. 1 postgres postgres       8192 Jan 17 02:23 16622
                bash-4.2$

остонавливаем кластер

добавляем в конец файла 16622 два символа 99

запускаеи кластер
Кластер запустился
Делаем выборку из таблицы

                postgres=# select * from test_text;
                   t
                -------
                 test1
                 test2
                 test3
                (3 rows)

ничего не изменилось, возможно изменения после спец символов не влияет на работу

Делаю то же самое, но в файле меняю данные test3 на tes99

запускаеи кластер
Кластер запустился
Делаем выборку из таблицы

                postgres=# select * from test_text;
                WARNING:  page verification failed, calculated checksum 44525 but expected 31978
                ERROR:  invalid page in block 0 of relation base/5/16622
                postgres=#

### 6.4 Что и почему произошло? как проигнорировать ошибку и продолжить работу?

то есть таблица повреждена и чек сумма это видит.

Отключаю проверку чек сум, запускаю кластер и делаю выборку из таблицы

                postgres=# select * from test_text;
                   t
                -------
                 tes99
                 test2
                 test3
                (3 rows)
                
                postgres=#

теперь могу работать с таблицей









