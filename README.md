CLICKHOUSE
Проверить настройки CH

Задачи:
Создание тестовой базы
1. Создать тестовую db_test базу

Импорт CVS
1. Создать тестовую таблицу table_csv с полями id, k_id, name, date
2. Импортировать данные используя csv файл

Импорт данных HTTP
1. Создать тестовую таблицу table_oracle с полями id, k_id, name, date
2. Создать скрипт на PYTHON для чтения из таблицы Oracle и импорта в CH

Создание Disterbuted table
1. Настроить реплики
2. Создать основную таблицу table_dist id, k_id, name, date
3. Создать копии table_dist таблицы на всех нодах кластера
4. Импортировать данные одним из способов выше

Создание витрин для проверки реализуемости и сравнения производительности
Структуры витрин:
  - Витрина аналогичная существующей
  - Витрина объединенная, столбцы + столбцы (обьединенная view)
  - Витрина транспонированная, id к столбцам

1. Создать базу для каждой из витрин
2. Создание основных таблиц и реплик таблиц
3. Импорт данных, создание перекладчиков-агентов для импорта данных 
4. Подключение инструментов визуализации

ClickHouse: MergeTree
ClickHouse задумывался для аналитики, под большую пропускную способность записи и чтения. Чтобы получить максимальную пропускную способность записи можно просто писать в конец файла, без поддержания всяких индексов и произвольных seek’ов, но как без них обеспечить скорость чтения? Ведь поиск по произвольным данным без индекса быстрым быть не может. Получается, нужно либо упорядочивать в момент вставки, либо фуллсканить в момент чтения. Оба варианта гробят пропускную способность. ClickHouse решил упорядочивать посередине: в фоне.


Так и появился компромисс в виде MergeTree: пишем на диск небольшие куски отсортированных данных, которые ClickHouse сливает в фоне для поддержания упорядоченности.

Cоздадим таблицу с небольшим числом колонок. В качестве первичного ключа выберем user_id. 
create table clicks
(
    date      DateTime,
    user_id   Int64,
    banner_id String
) engine = MergeTree() order by user_id settings index_granularity = 2;

ClickHouse создал папку /var/lib/clickhouse/data/default/clicks.

Вставим 10 строчек:
+-------------------+-------+---------+
|date               |user_id|banner_id|
+-------------------+-------+---------+
|2021-01-01 15:43:58|157    |lqe(|    |
|2021-01-01 15:45:38|289    |freed    |
|2021-01-01 15:47:18|421    |08&N0    |
|2021-01-01 15:48:58|553    |n5UD$    |
|2021-01-01 15:50:38|685    |1?!Up    |
|2021-01-01 15:52:18|817    |caHy6    |
|2021-01-01 15:53:58|949    |maXZD    |
|2021-01-01 15:55:38|1081   |Fx:BO    |
|2021-01-01 15:57:18|1213   |\v8j+    |
|2021-01-01 15:58:58|1345   |szEG)    |
+-------------------+-------+---------+

На диске появился один кусок данных (data part) all_1_1_0. Посмотрим, из чего он состоит:
all — название партиции. Поскольку выражения партиционирования в CREATE TABLE мы не задали, вся таблица будет в одной партиции.
1_1 — это срез блоков, который хранится в парте.
0 — это уровень в дереве слияний. Нулевой уровень у первых «протопартов», если два парта слить, то их уровень увеличится на 1.

Появился первичный индекс primary.idx и по два файла на каждую колонку: mrk2 и bin.
columns.txt — информация о колонках.
count.txt — число строк в куске.

Первичный индекс primary.idx, он же разрежённый

В primary.idx лежат засечки: отсортированные значения выражения первичного ключа, заданного в CREATE TABLE, для каждой index_granularity строки: для строки 0, index_granularity, index_granularity*2 и т.д. Размер гранулы index_granularity — это степень разреженности индекса primary.idx. Для каждого запроса ClickHouse читает с диска целое количество гранул. Если задать большой размер гранулы, будет прочитано много лишних строк, если маленький — увеличится размер первичного индекса, который хранится в оперативке для быстродействия.

Последняя засечка (здесь 1345) нужна, чтобы знать, на чём заканчивается таблица. 
od -i просто отображает байты как целые положительные четырёхбайтные числа.

od -i all_1_1_0/primary.idx
0000000               157               0             421               0
0000020               685               0             949               0
0000040              1213               0            1345               0
0000060

Файлы данных .bin

Файлы bin содержат значения колонок, отсортированные по выражению первичного ключа, в нашем случае — user_id. Данные хранятся в сжатом виде, единица сжатия — блок. N-ое значение принадлежит к N-ой строке.


banner_id.bin:
cat  all_1_1_0/banner_id.bin | clickhouse-compressor -d | od -a
0000000  enq   l   q   e   (   | enq   f   r   e   e   d enq   0   8   &
0000020    N   0 enq   n   5   U   D   $ enq   1   ?   !   U   p enq   c
0000040    a   H   y   6 enq   m   a   X   Z   D enq   F   x   :   B   O
0000060  enq   \   v   8   j   + enq   s   z   E   G   )

user_id.bin:
cat  all_1_1_0/user_id.bin | clickhouse-compressor -d | od -i
0000000               157               0             289               0
0000020               421               0             553               0
0000040               685               0             817               0
0000060               949               0            1081               0
0000100              1213               0            1345               0

Ну и date.bin, в виде epoch-time:
cat  all_1_1_0/date.bin | clickhouse-compressor -d | od -i
0000000        1609515838      1609515938      1609516038      1609516138
0000020        1609516238      1609516338      1609516438      1609516538
0000040        1609516638      1609516738
0000050
#от пятница,  1 января 2021 г. 15:43:58 (UTC) 
#до пятница,  1 января 2021 г. 15:58:58 (UTC)

Стоп, а почему данные отсортированы? Ведь мы вставляли строки в произвольном порядке? Дело в том, что ClickHouse сортирует вставляемые строки в оперативке с использованием красно-чёрного дерева, и время от времени сбрасывает его на диск в виде immutable дата-парта.
Есть DELETE и UPDATE, но эти команды не для рутинного использования, они работают в фоне, и не меняют старые парты, а создают новые.

Благодаря тому, что столбцы хранятся в своих файлах, можно читать только те столбцы, которые указаны в SELECT, также эффективнее сжатие за счет однотипности данных. По этой причине ClickHouse и называется колончатой СУБД.

Файлы засечек .mrk2

Для каждой засечки из primary.idx mrk2 знает, где именно в bin-файле начинаются значения соответствующей колонки.

Засечка в mrk2 состоит из 3 положительных 8-битных чисел, всего 24 байта: смещение блока в bin-файле, смещение в разжатом блоке и количество строк в грануле. Третье число пока не важно.

Читаем третью засечку:

#читаем 24 байта как числа long начиная с 48 байт смещения
od -l -j 48 -N 24 all_1_1_0/user_id.mrk2
0000060                                 0                              32
0000100                                 2
0000110

Пройдём по этому указателю, пропустив 32 байта в нулевом блоке:

cat  all_1_1_0/user_id.bin | clickhouse-compressor -d |od -j 32 -i -N 4
0000040               685
0000044

Видим значение первичного ключа из четвёртой строки. Это и есть начало третьей засечки в primary.idx!

То есть mrk2-файлы нужны просто для чтения bin-файлов, в них лежит переход от номера строки к байтовому смещению диска. Можно представить это как клей между реляционной абстракцией и физическим хранилищем.

Поиск по MergeTree

Рассмотрим, как выполняется запрос с условием на первичный ключ. ClickHouse с помощью бинарного поиска по primary.idx вычисляет, с какой строки нужно читать данные. То есть по первичному индексу вычисляются области строк таблицы, которые могут удовлетворять запросу.


Видно, что 982 лежит между гранулами 949 и 1213, поэтому можно прочесть только одну гранулу:


--включаем логи в clickhouse-client
set send_logs_level = 'trace';

SELECT *
FROM clicks
WHERE user_id = 982

Selected 1 parts by date, 1 parts by key, 1 marks by primary key, 1 marks to read from 1 ranges
Reading approx. 2 rows with 1 streams

А вот если сделать поиск по колонке вне первичного ключа, придётся делать фуллскан:


SELECT *
FROM clicks
WHERE banner_id = 'genbykj[';

Selected 1 parts by date, 1 parts by key, 5 marks by primary key, 5 marks to read from 1 ranges
Reading approx. 10 rows with 1 streams

Это делает ClickHouse СУБД плохо справляющейся с точечными запросами, ведь приходится читать много лишних строк. Если размер гранулы по умолчанию 8192, а в результате только одна строка, то эффективность чтения 1/8192 = 0.0001.


Партиции

Теперь разберемся, что такое партиции. Добавим выражение партиционирования в CREATE TABLE, обрезая дату до дня:
create table clicks
(
    date      DateTime,
    user_id   Int64,
    banner_id String
) engine = MergeTree() order by user_id partition by toYYYYMMDD(date);
+-------------------+-------+---------+
|date               |user_id|banner_id|
+-------------------+-------+---------+
|2021-01-16 13:34:29|157    ||^/g~    |
|2021-01-16 18:51:09|289    |/y;ny    |
|2021-01-17 00:07:49|421    |@7bbc    |
|2021-01-17 05:24:29|553    |.[e/{    |
|2021-01-17 10:41:09|685    |0Wj)m    |
|2021-01-17 15:57:49|817    |W6@Oo    |
|2021-01-17 21:14:29|949    |tvQZ&    |
|2021-01-18 02:31:09|1081   |ZPeCE    |
|2021-01-18 07:47:49|1213   |H|$PI    |
|2021-01-18 13:04:29|1345   |a'0^J    |
+-------------------+-------+---------+

После вставки 10 строк, на диске появились три парта вместо одного: на 16, 17, 18 января 2021 года:


clicks
├── 20210116_1_1_0
├── 20210117_2_2_0
├── 20210118_3_3_0

Внутри парты такие же, как и без партиционирования, но добавился файл, хранящий партицию, к которой относится парт:


cat clicks/20210118_3_3_0/partition.dat | od -i
0000000    20210118
0000004

И minmax индекс по дате, в котором хранятся минимальное и максимальное значение даты в парте:


od -i 20210116_1_1_0/minmax_date.idx
0000000  1610804069  1610823069
0000010
date --utc -d @1610804069
Sat Jan 16 13:34:29 UTC 2021
date --utc -d @1610823069
Sat Jan 16 18:51:09 UTC 2021

Теперь посмотрим, как партиции помогают в поиске:
--запрос по выражению, участвующего в партиционировании
SELECT *
FROM clicks
WHERE (date >= toUnixTimestamp('2021-01-17 00:00:00', 'UTC')) AND (date < toUnixTimestamp('2021-01-17 16:00:00', 'UTC'))
--зная границы дат каждого парта, легко узнать, какие парты читать не нужно
MinMax index condition: (column 0 in [1610841600, +inf)), (column 0 in (-inf, 1610899199]), and
Not using primary index on part 20210117_2_2_0
Selected 1 parts by date, 1 parts by key, 1 marks by primary key, 1 marks to read from 1 ranges
Reading approx. 8192 rows with 1 streams
┌────────────────date─┬─user_id─┬─banner_id─┐
│ 2021-01-17 03:07:49 │     421 │ @7bbc     │
│ 2021-01-17 08:24:29 │     553 │ .[e/{     │
│ 2021-01-17 13:41:09 │     685 │ 0Wj)m     │
│ 2021-01-17 18:57:49 │     817 │ W6@Oo     │
└─────────────────────┴─────────┴───────────┘
Read 4 rows, 104.00 B in 0.002051518 sec., 1949 rows/sec., 49.51 KiB/sec.

--запрос без участия партиций
SELECT *
FROM clicks
WHERE banner_id = 'genbykj['
--приходится читать все парты, в три параллельных потока
Not using primary index on part 20210117_2_2_0
Not using primary index on part 20210116_1_1_0
Not using primary index on part 20210118_3_3_0
Selected 3 parts by date, 3 parts by key, 3 marks by primary key, 3 marks to read from 3 ranges
Reading approx. 24576 rows with 3 streams
Read 10 rows, 140.00 B in 0.001798808 sec., 5559 rows/sec., 76.01 KiB/sec.

Дата-парты также удобно смотреть через системную таблицу:

SELECT name, rows, min_time, max_time
FROM system.parts
WHERE table = 'clicks'

┌─name───────────┬─rows─┬────────────min_time─┬────────────max_time─┐
│ 20210116_1_1_0 │    2 │ 2021-01-16 16:34:29 │ 2021-01-16 21:51:09 │
│ 20210117_2_2_0 │    4 │ 2021-01-17 03:07:49 │ 2021-01-17 18:57:49 │
│ 20210118_3_3_0 │    4 │ 2021-01-18 00:14:29 │ 2021-01-18 16:04:29 │
└────────────────┴──────┴─────────────────────┴─────────────────────┘

Итого мы увидели, что каждый дата-парт принадлежит к одной партиции, и при поиске ClickHouse старается читать только нужные партиции. Ещё партиции можно отдельно удалять, отключать и производить над ними другие операции.


Слияние дата-партов

Для того, чтобы количество партов не разрасталось, ClickHouse производит фоновое слияние кусков. При слиянии также срабатывает логика в Replacing, Summing, Collapsing и других вариациях движка MergeTree. При слиянии двух отсортированных партов появляется один отсортированный.


--остановим процесс слияния и вставим строк
system stop merges clicks;
insert into clicks(date, user_id, banner_id) 
select now() , number * 132 + 157, randomPrintableASCII(5)
from system.numbers limit 50;
Ok.
--появился 1 парт
+--------------+------+
|name          |active|
+--------------+------+
|20210116_1_1_0|1     |
+--------------+------+

--вставим ещё строк
insert into clicks(date, user_id, banner_id)  
select now() , number * 132 + 157, randomPrintableASCII(5) 
from system.numbers limit 50;

--уже 2 парта
+--------------+------+
|name          |active|
+--------------+------+
|20210116_1_1_0|1     |
|20210116_2_2_0|1     |
+--------------+------+

--возобновим процесс слияния
system start merges clicks;
-- и попросим ClickHouse запустить слияние
optimize table clicks final;

--через некоторое время видим, что два парта слились в один 20210116_1_2_1, у которого увеличился уровень. 
--Неактивные парты будут удалены со временем.
+--------------+------+----+
|name          |active|rows|
+--------------+------+----+
|20210116_1_1_0|0     |50  |
|20210116_1_2_1|1     |100 |
|20210116_2_2_0|0     |50  |
+--------------+------+----+


Часть про реплецирование и шардированние

ClickHouse специально проектировался для работы в кластерах, расположенных в разных дата-центрах. Масштабируется СУБД линейно до сотен узлов. Так, например, Яндекс.Метрика на момент написания статьи — это кластер из более чем 400 узлов.


ClickHouse предоставляет шардирование и репликацию "из коробки", они могут гибко настраиваться отдельно для каждой таблицы. Для обеспечения реплицирования требуется Apache ZooKeeper (рекомендуется использовать версию 3.4.5+). Для более высокой надежности мы используем ZK-кластер (ансамбль) из 5 узлов. Следует выбирать нечетное число ZK-узлов (например, 3 или 5), чтобы обеспечить кворум. Также отметим, что ZK не используется в операциях SELECT, а применяется, например, в ALTER-запросах для изменений столбцов, сохраняя инструкции для каждой из реплик.


Шардирование

Шардирование в ClickHouse позволяет записывать и хранить порции данных в кластере распределенно и обрабатывать (читать) данные параллельно на всех узлах кластера, увеличивая throughput и уменьшая latency. Например, в запросах с GROUP BY ClickHouse выполнит агрегирование на удаленных узлах и передаст узлу-инициатору запроса промежуточные состояния агрегатных функций, где они будут доагрегированы.


Для шардирования используется специальный движок Distributed, который не хранит данные, а делегирует SELECT-запросы на шардированные таблицы (таблицы, содержащие порции данных) с последующей обработкой полученных данных. Запись данных в шарды может выполняться в двух режимах: 1) через Distributed-таблицу и необязательный ключ шардирования или 2) непосредственно в шардированные таблицы, из которых далее данные будут читаться через Distributed-таблицу. Рассмотрим эти режимы более подробно.


В первом режиме данные записываются в Distributed-таблицу по ключу шардирования. В простейшем случае ключом шардирования может быть случайное число, т. е. результат вызова функции rand(). Однако в качестве ключа шардирования рекомендуется брать значение хеш-функции от поля в таблице, которое позволит, с одной стороны, локализовать небольшие наборы данных на одном шарде, а с другой — обеспечит достаточно ровное распределение таких наборов по разным шардам в кластере. Например, идентификатор сессии (sess_id) пользователя позволит локализовать показы страниц одному пользователю на одном шарде, при этом сессии разных пользователей будут распределены равномерно по всем шардам в кластере (при условии, что значения поля sess_id будут иметь хорошее распределение). Ключ шардирования может быть также нечисловым или составным. В этом случае можно использовать встроенную хеширующую функцию cityHash64. В рассматриваемом режиме данные, записываемые на один из узлов кластера, по ключу шардирования будут перенаправляться на нужные шарды автоматически, увеличивая, однако, при этом трафик.


Более сложный способ заключается в том, чтобы вне ClickHouse вычислять нужный шард и выполнять запись напрямую в шардированную таблицу. Сложность здесь обусловлена тем, что нужно знать набор доступных узлов-шардов. Однако в этом случае запись становится более эффективной, и механизм шардирования (определения нужного шарда) может быть более гибким.


Репликация

ClickHouse поддерживает репликацию данных, обеспечивая целостность данных на репликах. Для репликации данных используются специальные движки MergeTree-семейства:


ReplicatedMergeTree
ReplicatedCollapsingMergeTree
ReplicatedAggregatingMergeTree
ReplicatedSummingMergeTree

Репликация часто применяется вместе с шардированием. Например, кластер из 6 узлов может содержать 3 шарда по 2 реплики. Следует отметить, что репликация не зависит от механизмов шардирования и работает на уровне отдельных таблиц.


Запись данных может выполняться в любую из таблиц-реплик, ClickHouse выполняет автоматическую синхронизацию данных между всеми репликами.


Примеры конфигурации ClickHouse-кластера

В качестве примеров будем рассматривать различные конфигурации для четырех узлов: ch63.smi2, ch64.smi2, ch65.smi2, ch66.smi2. Настройки содержатся в конфигурационном файле /etc/clickhouse-server/config.xml.


Один шард и четыре реплики

<remote_servers>
    <!-- One shard, four replicas -->
    <repikator>
       <shard>
           <!-- replica 01_01 -->
           <replica>
               <host>ch63.smi2</host>
           </replica>

           <!-- replica 01_02 -->
           <replica>
               <host>ch64.smi2</host>
           </replica>

           <!-- replica 01_03 -->
           <replica>
               <host>ch65.smi2</host>
           </replica>

           <!-- replica 01_04 -->
           <replica>
               <host>ch66.smi2</host>
           </replica>
       </shard>
    </repikator>
</remote_servers>

Пример SQL-запроса создания таблицы для указанной конфигурации:
CREATE DATABASE IF NOT EXISTS dbrepikator
;

CREATE TABLE IF NOT EXISTS dbrepikator.anysumming_repl_sharded (
    event_date Date DEFAULT toDate(event_time),
    event_time DateTime DEFAULT now(),
    body_id Int32,
    views Int32
) ENGINE = ReplicatedSummingMergeTree('/clickhouse/tables/{repikator_replica}/dbrepikator/anysumming_repl_sharded', '{replica}', event_date, (event_date, event_time, body_id), 8192)
;

CREATE TABLE IF NOT EXISTS  dbrepikator.anysumming_repl AS dbrepikator.anysumming_repl_sharded
      ENGINE = Distributed( repikator, dbrepikator, anysumming_repl_sharded , rand() )
      
Преимущество данной конфигурации:


Наиболее надежный способ хранения данных.

Недостатки:


Для большинства задач будет храниться избыточное количество копий данных.
Поскольку в данной конфигурации только 1 шард, SELECT-запрос не может выполняться параллельно на разных узлах.
Требуются дополнительные ресурсы на многократное реплицирование данных между всеми узлами.

Четыре шарда по одной реплике
<remote_servers>
    <!-- Four shards, one replica -->
    <sharovara>
       <!-- shard 01 -->
       <shard>
           <!-- replica 01_01 -->
           <replica>
               <host>ch63.smi2</host>
           </replica>
       </shard>

       <!-- shard 02 -->
       <shard>
           <!-- replica 02_01 -->
           <replica>
               <host>ch64.smi2</host>
           </replica>
       </shard>

       <!-- shard 03 -->
       <shard>
           <!-- replica 03_01 -->
           <replica>
               <host>ch65.smi2</host>
           </replica>
       </shard>

       <!-- shard 04 -->
       <shard>
           <!-- replica 04_01 -->
           <replica>
               <host>ch66.smi2</host>
           </replica>
       </shard>
    </sharovara>
</remote_servers>

CREATE DATABASE IF NOT EXISTS testshara 
;
CREATE TABLE IF NOT EXISTS testshara.anysumming_sharded (
    event_date Date DEFAULT toDate(event_time),
    event_time DateTime DEFAULT now(),
    body_id Int32,
    views Int32
) ENGINE = ReplicatedSummingMergeTree('/clickhouse/tables/{sharovara_replica}/sharovara/anysumming_sharded_sharded', '{replica}', event_date, (event_date, event_time, body_id), 8192)
;
CREATE TABLE IF NOT EXISTS  testshara.anysumming AS testshara.anysumming_sharded
      ENGINE = Distributed( sharovara, testshara, anysumming_sharded , rand() )
      
Преимущество данной конфигурации:


Поскольку в данной конфигурации 4 шарда, SELECT-запрос может выполняться параллельно сразу на всех узлах кластера.

Недостаток:


Наименее надежный способ хранения данных (потеря узла приводит к потере порции данных).

Два шарда по две реплики

<remote_servers>
    <!-- Two shards, two replica -->
    <pulse>
        <!-- shard 01 -->
       <shard>
           <!-- replica 01_01 -->
           <replica>
               <host>ch63.smi2</host>
           </replica>

           <!-- replica 01_02 -->
           <replica>
               <host>ch64.smi2</host>
           </replica>
       </shard>

       <!-- shard 02 -->
       <shard>
           <!-- replica 02_01 -->
           <replica>
               <host>ch65.smi2</host>
           </replica>

           <!-- replica 02_02 -->
           <replica>
               <host>ch66.smi2</host>
           </replica>
       </shard>
    </pulse>
</remote_servers>

CREATE DATABASE IF NOT EXISTS dbpulse 
;

CREATE TABLE IF NOT EXISTS dbpulse.normal_summing_sharded (
    event_date Date DEFAULT toDate(event_time),
    event_time DateTime DEFAULT now(),
    body_id Int32,
    views Int32
) ENGINE = ReplicatedSummingMergeTree('/clickhouse/tables/{pulse_replica}/pulse/normal_summing_sharded', '{replica}', event_date, (event_date, event_time, body_id), 8192)
;

CREATE TABLE IF NOT EXISTS  dbpulse.normal_summing AS dbpulse.normal_summing_sharded
      ENGINE = Distributed( pulse, dbpulse, normal_summing_sharded , rand() )
      
Данная конфигурация воплощает лучшие качества из первого и второго примеров:


Поскольку в данной конфигурации 2 шарда, SELECT-запрос может выполняться параллельно на каждом из шардов в кластере.
Относительно надежный способ хранения данных (потеря одного узла кластера не приводит к потере порции данных).


ВОПРОС КОДИРОВКИ

Что делать, если у меня проблема с кодировками при использовании Oracle через ODBC? 
Если вы используете Oracle через драйвер ODBC в качестве источника внешних словарей, необходимо задать правильное значение для переменной окружения NLS_LANG в /etc/default/clickhouse. Подробнее читайте в Oracle NLS_LANG FAQ.
