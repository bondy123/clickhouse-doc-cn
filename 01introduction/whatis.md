什么是ClickHouse
===================
ClickHouse是一个列式的OLAP数据库管理系统
在通常的面向行的数据库中，数据是按照这样的顺序存储：

<pre> 
5123456789123456789     1       Eurobasket - Greece - Bosnia and Herzegovina - example.com      1       2011-09-01 01:03:02     6274717   1294101174      11409   612345678912345678      0       33      6       http://www.example.com/basketball/team/123/match/456789.html http://www.example.com/basketball/team/123/match/987654.html       0       1366    768     32      10      3183      0       0       13      0\0     1       1       0       0                       2011142 -1      0               0       01321     613     660     2011-09-01 08:01:17     0       0       0       0       utf-8   1466    0       0       0       5678901234567890123               277789954       0       0       0       0       0
5234985259563631958     0       Consulting, Tax assessment, Accounting, Law       1       2011-09-01 01:03:02     6320881   2111222333      213     6458937489576391093     0       3       2       http://www.example.ru/         0       800     600       16      10      2       153.1   0       0       10      63      1       1       0       0                       2111678 000       0       588     368     240     2011-09-01 01:03:17     4       0       60310   0       windows-1251    1466    0       000               778899001       0       0       0       0       0
...
</pre>

换句话说，一行的所有数据是连续存储的。MySQL，Postgres,SQL Server等都是典型的行式数据库。
在列式数据库中，数据是这样存储的：
<pre class="text-example" style="white-space: pre; ">
<b>WatchID:</b>    5385521489354350662     5385521490329509958     5385521489953706054     5385521490476781638     5385521490583269446     5385521490218868806     5385521491437850694   5385521491090174022      5385521490792669254     5385521490420695110     5385521491532181574     5385521491559694406     5385521491459625030     5385521492275175494   5385521492781318214      5385521492710027334     5385521492955615302     5385521493708759110     5385521494506434630     5385521493104611398
<b>JavaEnable:</b> 1       0       1       0       0       0       1       0       1       1       1       1       1       1       0       1       0       0       1       1
<b>Title:</b>      Yandex  Announcements - Investor Relations - Yandex     Yandex — Contact us — Moscow    Yandex — Mission        Ru      Yandex — History — History of Yandex    Yandex Financial Releases - Investor Relations - Yandex Yandex — Locations      Yandex Board of Directors - Corporate Governance - Yandex       Yandex — Technologies
<b>GoodEvent:</b>  1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1
<b>EventTime:</b>  2016-05-18 05:19:20     2016-05-18 08:10:20     2016-05-18 07:38:00     2016-05-18 01:13:08     2016-05-18 00:04:06     2016-05-18 04:21:30     2016-05-18 00:34:16     2016-05-18 07:35:49     2016-05-18 11:41:59     2016-05-18 01:13:32
...
</pre>

这些例子仅仅是用来展示数据是怎样排布的。
不同列的数据时分开存储的，而同一列的数据存储在一起。典型的列式数据库有： Vertica, Paraccel (Actian Matrix) (Amazon Redshift), Sybase IQ, Exasol, Infobright, InfiniDB, MonetDB (VectorWise) (Actian Vector), LucidDB, SAP HANA, Google Dremel, Google PowerDrill, Druid, kdb+ 等。
不同的存储方式会适应不同的场景。数据获取的场景是指请求如何发出，频率如何，比例如何。每种请求有多少数据被读出，包括按行，按列，按字节数；读取和更新数据之间的关系如何，活跃的数据有多大以及本地性如何；是否使用了事务，以及是否独立；对于数据持久性的要求如何，每种请求的延迟和吞吐量的要求如何等等。

整个系统的负载越高，针对场景定制化系统就越重要，定制化程度也越高。没有任何系统是能够很好的使用完全不同的各种场景的。如果一个系统适应很宽泛的场景需求，在负载很高的情况下，要么对每种场景的处理都很差，要么只在其中一种场景下工作的很好。

在OLAP(Online Analytical Processing)的场景下，具备以下特征：

- 绝大多数请求是读请求。
- 数据是批量更新的（每次1000行以上），而不是单条更新，或者从来不更新。
- 数据不会被修改。
- 读的时候，一次会读很多行，不过仅包括少量的列。
- 表很宽，也就是说有大量的列
- 请求频率相对不高（每台机器每秒小于几百个请求）
- 对一般的请求，50ms左右的延迟是允许的。
- 单列的数据很小，通常是数字或短字符串（比如60字节的URL信息）
- 处理单个请求的时候需要很大的吞吐量（每台机器每秒上十亿行）
- 没有事务要求
- 对数据一致性要求低
- 对每个请求来讲有一个表特别大，其他涉及的表都很小。
- 查询的结果比原始数据小得多，也就是说数据被过滤或聚合了。结果的数据量能够放入单台机器的内存中。

很容易看出来OLAP的场景和其他的流行的场景有很大不同（比如OLTP或KV读取的场景），因此完全没有必要去尝试使用OLTP数据库或KV数据库来处理分析类的请求。例如不要使用MongoDB或Elliptics做数据分析，相比于使用OLAP数据库性能会很差。

列式数据库会更适应OLAP的场景（至少1000倍的性能优势），有以下两个原因：

1. 从IO来讲：
    - 对于分析类的请求，每一行只有少量的几列需要读。对于列式数据库，你可以只读取你需要的数据，比如100列中你只需要5列，你就节省了20倍的IO。
    - 因为数据是按包读取的，更容易压缩，列式数据库能得到更好的压缩效果。
    - 因为IO数据量减少，系统的缓存能够缓存更多的数据。
  
  例如，“统计每个广告平台的记录的数量”这个请求需要读取“广告平台ID“这个列，未压缩时占用了1个字节。如果大部分的流量不是来自广告平台，你可以期待获得十倍的压缩。当使用一个快速的压缩算法时，每秒压缩几个G的数据是可能的。也就是说单台机器可以每秒处理数十亿行数据。而这个速度也实际达到了。
2. 从CPU来讲：（这部分没看懂）
  因为执行一条请求需要处理大量的行，将整个操作向量而不是单独的行进行分发会更合理，或者实现一个请求引擎来避免分发的开销。如果不这样，对于任何half-decent disk subsystem，请求解释器将会占满CPU。Since executing a query requires processing a large number of rows, it helps to dispatch all operations for entire vectors instead of for separate rows, or to implement the query engine so that there is almost no dispatching cost. If you don't do this, with any half-decent disk subsystem, the query interpreter inevitably stalls the CPU.
  
  因此将数据按列存储，并在可能的时候按列处理是很重要的。
  
  有两种方法可以做到这一点：
  
  1. 一个vector engine. 所有的操作为向量操作而写的，而不是单独的数据。这意味着你不需要经常调用操作，分发的开销就可以忽略不计。操作的代码包括了优化过的内部循环。A vector engine. All operations are written for vectors, instead of for separate values. This means you don't need to call operations very often, and dispatching costs are negligible. Operation code contains an optimized internal cycle.
  2. 代码生成。为请求而生成的代码包括了简介的操作。Code generation. The code generated for the query has all the indirect calls in it.

  在不同的数据库中并没有实现这些方法，因为对于简单的请求没有太大意义。然而也有例外，MemSQL使用了代码生成来降低处理SQL请求时的延迟。（相反，对于分析类的数据库更需要优化吞吐量而不是延迟）
  
  注意一点，为了提高CPU的效率，查询语句必须是声明式的（SQL或MDX），或至少是向量式的，请求中应该只包含隐式循环，方便优化。

<a name="features" ></a>
# ClickHouse独有的特性

1. 真正面向列的数据库管理系统
2. 数据压缩
3. 磁盘存储数据
4. 多核上的并行处理
5. 多机的分布式处理
6. SQL支持
7. 向量引擎
8. 实时数据更新
9. 索引
10. 适应在线请求
11. 非精确计算的支持
12. 支持嵌套的数据结构，支持数组作为数据类型
13. 支持查询复杂度限制，包括限额
14. 数据多副本和数据完整性支持

下面是详细的信息：

1. 真正面向列的数据库管理系统
  在真正面向列的数据库系统中，内容中没有任何”垃圾“数据。例如，固定长度的数据必须是支持的，避免存储一个长度信息。十亿个Uint8类型的数据必须实际占用大概1G未压缩的存储空间，不然会大大影响CPU的使用。保证数据未压缩情况下紧密的存储是很重要的，因为解压缩的效率也很大程度上决定于压缩前数据的大小。
  
  有一些系统能够把列单独的存储，但是因为他们为其他场景而优化，因此无法高效的处理分析请求，这样就没有多少意义。比如HBase,BigTable,Cassandra,HyperTable等，在这些系统里，你只能达到几十万的吞吐量，而不是几亿。

  同时要注意，，ClickHouse是一个数据库管理系统，而不是简单的一个数据库，它能够实时的在线创建表和创建数据库，加载数据，执行查询，而不需要重新配置或重启server。

2. 数据压缩
  一些面向列的数据库存储系统（比如InfiniDB CE和MonetDB）并不压缩数据。但是数据压缩的确能够提高性能。
  
3. 磁盘存储

  许多面向列的数据库比如（SAP HANA和Google PowerDrill）只能在内存中工作，但是即使使用几千台机器，内存的总量相比于我么要处理的数据量也太小了。
  
4. 多核并行处理

  大型请求自然的进行并行处理。
  
5. 多机分布式处理

  前面列举的列式的数据库几乎没有分布式处理的支持。
  
  在ClickHouse中，数据会被分为不同的分片，每一个分片都是一组副本用来防止失败。数据在所有分片中并行处理，并且对用户透明。
  
6. SQL支持

  如果你对标准的SQL不熟悉，我们无法讨论SQL支持的问题
  
  NULLs不支持。所有的函数都有不同的名字。然而有一个基于SQL的声明式查询语言，在大部分情况下和SQL没有区别。
  
  JOINs是支持的，子查询在FROM，IN，JOIN中支持，标量子查询支持。
  
  Correllated子查询不支持

7. 向量引擎

  数据不仅仅是按列存储的，还是按向量来处理的，向量代表一部分列，这能够帮助我们获得更高的CPU表现。

8. 实时数据更新

  ClickHouse支持主键表。为了支持主键上的区域查询，数据使用merge tree增量存储。因此数据可以连续的添加到表中，而不会锁住查询。

9. 索引

  拥有主键索引有助于帮助特殊的客户端（比如Metrica Counter)快速读取一段特定时间的数据，延迟在几十ms之内。

10. 适应在线请求

  这个特性让我们可以把它作为web接口的后端。低延迟意味着请求能够在页面加载过程中在线的处理。

11. 非精确计算的支持

    1. 系统包含了各种聚合函数用来估算数据量，中位数，分位数等。
    2. 允许在一部分抽样数据上执行查询来得到估算结果，在这种情况下从磁盘加载的数据会更少。
    3. 允许随机的在有限的数据中执行聚合函数，而不是全部的数据。当数据以某种分布存储时，者能够提供一个合理的计算结果，而使用更少的资源。

12. 支持嵌套的数据结构，支持数组作为数据类型
13. 支持查询复杂度限制，包括限额
14. 数据多副本和数据完整性支持

  使用异步的多主复制。当数据写入到任何一个可用的副本时，会自动复制到剩余的副本中，系统保证在不同的副本中保持相同的数据。数据在发生错误时会自动恢复，或在复杂的情况下手动恢复。更多的信息请参考“数据副本”章节
  
# ClickHouse可能是劣势的特性

1. 没有事务支持
2. 对聚合任务，查询的结果必须能够放入一台机器的内存中。当然原始数据可能会非常大。
3. 缺乏完美的UPDATE/DELETE实现

# Yandex.Metrica 任务

我们需要基于点击和会话信息获取定制化的报告，并按照用户定制化的分段。生成报告的数据是实时更新的，并且请求必须实时获得结果。我们必须有能力为任何时间段生成报表数据。复杂的聚合运算很重要，比如独立的访客数据统计等。

2014年4月的数据，Yandex.Metrica大约每天收录120亿条数据。所有的数据必须被存储下来。单条请求需要在几秒钟内扫描数亿行数据，或在几百毫秒内扫描几百万行数据。

## 是否聚合数据

现在有一个很流行的观点，就是为了高效的计算统计数据，你必须把数据聚合，来减少数据的总量。然而这是一个非常受限的解决方案，因为以下原因：

- 你必须使用一个预定义好的报告形式，而用户无法定制报告
- 当聚合相当大数据量的key的时候，结果集并没有太小，聚合是无效的
- 对于大量的报告形式来讲，有太多的聚合类型（组合爆炸）
- 当使用可能性特别多的字段（比如URL）来进行聚合时，结果并没有减少很多（小于两倍），因此聚合后的数据量可能是增大而不是减小。
- 用户不会查看我们计算出的所有报表，因此大部分的计算是无用的
- 在进行各种聚合的时候，数据的逻辑完整性难以保证

如果我们不把数据进行聚合，而是针对原始数据进行计算，有可能会减少计算的总量。当然，预先聚合数据能够把相当大的一部分工作量放在离线完成，相对更加平稳，而在线计算需要计算的越快越好，因为用户正在等待结果。

Yandex.Metrica有一个针对聚合数据的特殊系统叫做Metrage，用来生成大部分的报表工作。从2009年开始，Yandex.Metrica也用了一个特殊的OLAP数据库来处理非聚合数据，叫做OLAPServer。它在非聚合数据上工作的很好，但是有很多的限制，以至于很多需要的报表无法生成。包括缺少数据类型的支持（仅支持数字），无法实时增量的添加数据（只能每天覆盖写入）。OLAPServer不是一个数据库管理系统，而仅仅是一个特异化的数据库。为了解决OLAPServer的限制，以及解决在非聚合数据上生成报表的需求，我们开发了ClickHouse。

# 在Yandex.Metrica和其他Yandex服务中的应用

ClickHouse在Yandex.Metrica中有多种用途，其中最主要的任务时使用非聚合的数据实时生成报表信息。它使用了374台服务器，存储了超过8万亿行数据。压缩后数据的总量，不考虑冗余和重复，大概有800TB，非压缩数据（使用tsv格式)的数据大概有7PB。

ClickHouse也被用来：

- 存储WebVisor数据
- 处理中间数据
- 生成分析后的总体报表
- 运行用来调试Metrica引擎的查询
- 分析API和用户接口产生的日志

ClickHouse在Yandex中有至少十几个其他的应用，在search verticals, Market, Direct, business analytics, mobile development, AdFox, personal service等等。

# 可能的类似系统

目前没有ClickHouse类似的系统，目前（2016年3月），没有其他任何一个开源免费的系统拥有上面列出的所有特性。然而这些特性对于Yandex.Metrica是十分重要的。

# 可能有点愚蠢的问题

1. 为什么没有使用类似MapReduce的系统？

MapReduce类型的系统是分布式的计算系统，reduce阶段使用分布式的排序完成的。不考虑这个因素，MapReduce和其他的一些系统比如YAMR，Hadoop，YT等很类似。

这些系统并不适合做在线查询，因为延迟太大，没有办法作为web服务的后端。这些系统也不适合在线更新数据。

如果操作的结果和所有中间结果能够放入单台服务器的内存中，对Reduce操作来讲，分布式排序并不是一个合适的方案，而这个前提往往是成立的。在这种情况下，最适合的reduce操作的方式是使用哈希表。一个MapReduce操作常见的优化就是使用内存中的哈希表来组合操作。这一步通常是由用户手动完成。分布式的排序操作是简单的MapReduce任务耗费过长时间的最主要原因。

类似的MapReduce系统能够在集群中运行任何代码。但对于OLAP的场景，声明式的查询语言会比代码更合适，因为能够更快的实施。比如Hadoop有Hive和Pig。还有其他系统比如 Cloudera Impala, Shark (depricated) and Spark SQL for Spark, Presto, Apache Drill.

但是，这些任务在这类系统中的表现相比于专门优化过的系统要差很多，而且高延迟决定了无法在web服务中使用。

YT允许存储把列分组单独存储，但是YT不是一个真正面向列的存储系统，没有固定长度的数据类型，也没有向量引擎。在YT中，任务是使用流式的任意代码执行的，因此也无法针对性的进行更好的优化。在2014-2016年，YT在开发动态排序表”dynamic table sorting"，加入了Merge Tree, 强类型，SQL类似的查询语言的支持。动态排序表并不适应OLAP类的任务，因为数据是按行存储的。查询语句还在孵化阶段，目前无法真正全力开发。YT的开发者正在考虑把动态排序表用于OLTP和KV存储的场景中。

# 性能表现

根据内部测试的结果，ClickHouse在所有可比较的系统中，使用可比较的操作场景，表现出最佳的性能表现。包括对于长查询最高的吞吐量，对于短查询最低的延迟，测试数据[点这里](https://clickhouse.yandex/benchmark.html)

### 单个大型查询的吞吐率

吞吐率使用每秒查询的行数或MB数来衡量。如果数据放在page cache中，不太复杂的查询可以让数据以单台服务器每秒2-10GB的速度进行处理，特别简单的情况可能可以达到30GB。如果数据没有在page cache中，速度取决于磁盘和数据压缩率。比如磁盘读取速度400MB/s，数据压缩率为3，就能够处理每秒1.2GB。要得到按行计算的吞吐率，就把这个除以使用的所有列的字节数之和。比如共10字节的情况下，速度就能到1到2亿行每秒。这个处理速度在分布式处理中几乎线性增长，只要聚合和排序后的结果中行数不要太大。（因为结果在单机内存中？）

### 处理短查询的延迟

如果查询使用了主键，并且没有选择太多行来处理（比如数十万行），也没有使用太多列，我们可以预期延迟在50ms左右，前提是数据在page cache中。否则，延迟主要由寻道次数决定。如果使用了rotating drives，系统也没有过载，延迟大概为寻道时间（10ms) * 列数 * 数据分片数。

### 处理大量短查询的吞吐率

在同样的场景下，ClickHouse单机可以处理数百个短查询，，由于这个量级的查询不是典型的分析类数据库的场景，我们通常建议不超过每秒100个请求。

### 数据插入的性能

我们建议单次插入数据的包不小于1000行，或每秒不超过一个请求。我们使用MergeTree来插入数据，插入速度在50-200MB每秒。如果插入的行在1K左右大小，大概能够做到5w-20w行每秒。如果单行很小，性能会更好（在 Yandex Banner System 中大概50w行每秒，在Graphite数据中超过100w行每秒），要提高性能，可以并行使用多个INSERT语句，性能能够线性提升。

