# MySQL 存储引擎对比

关系数据库表是用于存储和组织信息的数据结构，可以将表理解为由行和列组成的表格，类似于 Excel 的电子表格的形式。有的表简单，有的表复杂，有的表根 本不用来存储任何长期的数据，有的表读取时非常快，但是插入数据时去很差；而我们在实际开发过程中，就可能需要各种各样的表，不同的表，就意味着存储不同 类型的数据，数据的处理上也会存在着差异，那么。对于 MySQL 来说，它提供了很多种类型的存储引擎，我们可以根据对数据处理的需求，选择不同的存储引 擎，从而最大限度的利用 MySQL 强大的功能。这篇博文将总结和分析各个引擎的特点，以及适用场合，并不会纠结于更深层次的东西。

在 MySQL 客户端中，使用以下命令可以查看 MySQL 支持的引擎。

```
show engines;
```

## 特性

| 引擎/特性      | 事务                                                          | 读写阻塞                                                                                | 锁定                                                                               | 索引                                                                             | 缓存                                                                                                     |
| -------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **MyISAM**     | MyISAM 存储引擎不支持事务，所以对事务有要求的业务场景不能使用 | 不仅会在写入的时候阻塞读取，MyISAM 还会在读取的时候阻塞写入，但读本身并不会阻塞另外的读 | 其锁定机制是表级索引，这虽然可以让锁定的实现成本很小但是也同时大大降低了其并发性能 | 存放真实数据的地址                                                               | MyISAM 可以通过 key_buffer 缓存以大大提高访问性能减少磁盘 IO，但是这个缓存区只会缓存索引，而不会缓存数据 |
| **InnoDB**     | 具有较好的事务支持：支持 4 个事务隔离级别，支持多版本读       | 读写阻塞与事务隔离级别相关                                                              | 其锁定机制是行级锁定，通过索引实现，全表扫描仍然会是表锁，注意间隙锁的影响         | 存储的是真实的数据                                                               | 具有非常高效的缓存特性：能缓存索引，也能缓存数据                                                         |
| **NDBCluster** | 和 Innodb 一样，支持事务                                      |                                                                                         |                                                                                    | 新版本索引以及被索引的数据必须存放在内存中，老版本所有数据和索引必须存在与内存中 |                                                                                                          |

## 适用场景

| 引擎/场景      | 事务支持                         | 读/写比                                                                       | 数据一致性         | 并发                                                           | 内存                                                                                |
| -------------- | -------------------------------- | ----------------------------------------------------------------------------- | ------------------ | -------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| **MyISAM**     | 不需要事务支持的                 | 以读为主，数据修改相对较小(阻塞问题)                                          | 数据一致性要求不高 | 并发相对较低(锁定机制问题)                                     | 内存要求不大                                                                        |
| **InnoDB**     | 需要事务支持(具有较好的事务特性) | 数据更新较为频繁的场景                                                        | 数据一致性要求较高 | 行级锁定对高并发有很好的适应能力，但需要确保查询是通过索引完成 | 硬件设备内存较大，可以利用 InnoDB 较好的缓存能力来提高内存利用率，尽可能减少磁盘 IO |
| **NDBCluster** |                                  | 查询简单，过滤条件较为固定，每次请求数据量较少，又不希望自己进行水平 Sharding |                    | 具有非常高的并发需求，但是单个请求的响应并不是非常的 critical  |                                                                                     |

# MyISAM

MyISAM 表是独立于操作系统的，这说明可以轻松地将其从 Windows 服务器移植到 Linux 服务器；每当我们建立一个 MyISAM 引擎的表时，就会在本地磁盘上建立三个文件，文件名就是表名。例如，我建立了一个 MyISAM 引擎的 tb_Demo 表，那么就会生成以下三个文件：

1.tb_demo.frm，存储表定义；
2.tb_demo.MYD，存储数据；
3.tb_demo.MYI，存储索引。

MyISAM 表无法处理事务，这就意味着有事务处理需求的表，不能使用 MyISAM 存储引擎。MyISAM 存储引擎特别适合在以下几种情况下使用：1.选择密集型的表。MyISAM 存储引擎在筛选大量数据时非常迅速，这是它最突出的优点。2.插入密集型的表。MyISAM 的并发插入特性允许同时选择和插入数据。例如：MyISAM 存储引擎很适合管理邮件或 Web 服务器日志数据。

最佳实践
尽量索引(缓存机制)
调整读写优先级，根据实际需求确保重要操作更优先
启用延迟插入改善大批量写入性能
尽量顺序操作让 insert 数据都写入到尾部，减少阻塞
分解大的操作，降低单个操作的阻塞时间
降低并发数，某些高并发场景通过应用来进行排队机制
对于相对静态的数据，充分利用 Query Cache 可以极大的提高访问效率
MyISAM 的 Count 只有在全表扫描的时候特别高效，带有其他条件的 count 都需要进行实际的数据访问

# InnoDB

InnoDB 是一个健壮的事务型存储引擎，这种存储引擎已经被很多互联网公司使用，为用户操作非常大的数据存储提供了一个强大的解决方案。我的电脑 上安装的 MySQL 5.6.13 版，InnoDB 就是作为默认的存储引擎。InnoDB 还引入了行级锁定和外键约束，在以下场合下，使用 InnoDB 是最理想的选择：1.更新密集的表。InnoDB 存储引擎特别适合处理多重并发的更新请求。2.事务。InnoDB 存储引擎是支持事务的标准 MySQL 存储引擎。3.自动灾难恢复。与其它存储引擎不同，InnoDB 表能够自动从灾难中恢复。4.外键约束。MySQL 支持外键的存储引擎只有 InnoDB。5.支持自动增加列 AUTO_INCREMENT 属性。
一般来说，如果需要事务支持，并且有较高的并发读取频率，InnoDB 是不错的选择。

最佳实践
主键尽可能小，避免给 Secondary index 带来过大的空间负担
避免全表扫描，因为会使用表锁
尽可能缓存所有的索引和数据，提高响应速度
在大批量小插入的时候，尽量自己控制事务而不要使用 autocommit 自动提交
合理设置 innodb_flush_log_at_trx_commit 参数值，不要过度追求安全性
避免主键更新，因为这会带来大量的数据移动

InnoDB stores data in a file(s) called a tablespace. In newer versions of MySQL, you can store the data and indexes in separate files. MyISAM stores the data and indexes in two files (.MYD, .MYI). InnoDB is extremely critical about its tablespace and log files (They normally should be backed up together). You can't just backup your data by copying files like you would with MyISAM. In some situations you can only restore a table to the server from which you backed it up!
InnoDB are built on clustered indexes and uses MVCC to achieve high concurrency. This provides very fast primary key lookups. MyISAM doesn't support transactions, FK contraints, or row-level locks. MyISAM uses shared and exclusive locks on theentire table. However, concurrent reads & inserts for new rows are allowed.
InnoDB is crash safe (Assuming your flushes are truly durable on disk and not on some volatile cache). MyISAM is no where close to being crash safe. If you care about your data, use InnoDB. It might be OK to use MyISAM for read-only workloads.
InnoDB will support full-text indexes in MySQL 5.6. MyISAM currently supports this feature. Normally these type of things should be offloaded to something like aSphinx server.
InnoDB repairs are reasonably fast. MyISAM is slow and you might not get all your data back. Eek.
InnoDB requires a lot of memory (buffer pool). The data and indexes are cached in memory. Changes are written to the log buffer (physical memory) and are flushed every second to the log files (method depends on innodb_flush_log_at_trx_commit value). Having the data in memory is a huge performance boost. MyISAM only caches indexes (key_buffer_size) so that's where you would allocate most of your memory if you're only using MyISAM.

To me, the most important difference is that MyISAM does not write to disk synchronously. It writes to the filesystem cache, and your operating system decides when to finally write the cache to disk. Any time you have a server reboot, you can lose some MyISAM data that you thought you had written to your database.

Thus MyISAM is not Durable, and MyISAM tables frequently get corrupted. I wouldn't use MyISAM for any data that is important or not easily reproducible.

# MEMORY

使用 MySQL Memory 存储引擎的出发点是速度。为得到最快的响应时间，采用的逻辑存储介质是系统内存。虽然在内存中存储表数据确实会提供很高的性能，但当 mysqld 守护进程崩溃时，所有的 Memory 数据都会丢失。获得速度的同时也带来了一些缺陷。它要求存储在 Memory 数据表里的数据使用的是长度不 变的格式，这意味着不能使用 BLOB 和 TEXT 这样的长度可变的数据类型，VARCHAR 是一种长度可变的类型，但因为它在 MySQL 内部当做长度固定不 变的 CHAR 类型，所以可以使用。
一般在以下几种情况下使用 Memory 存储引擎：1.目标数据较小，而且被非常频繁地访问。在内存中存放数据，所以会造成内存的使用，可以通过参数 max_heap_table_size 控制 Memory 表的大小，设置此参数，就可以限制 Memory 表的最大大小。2.如果数据是临时的，而且要求必须立即可用，那么就可以存放在内存表中。3.存储在 Memory 表中的数据如果突然丢失，不会对应用服务产生实质的负面影响。
Memory 同时支持散列索引和 B 树索引。B 树索引的优于散列索引的是，可以使用部分查询和通配查询，也可以使用<、>和>=等 操作符方便数据挖掘。散列索引进行“相等比较”非常快，但是对“范围比较”的速度就慢多了，因此散列索引值适合使用在=和<>的操作符中，不 适合在<或>操作符中，也同样不适合用在 order by 子句中。
可以在表创建时利用 USING 子句指定要使用的版本。例如：

```sql
create table users
(
    id smallint unsigned not null auto_increment,
    username varchar(15) not null,
    pwd varchar(15) not null,
    index using hash (username),
    primary key (id)
)engine=memory;
```

上述代码创建了一个表，在 username 字段上使用了 HASH 散列索引。下面的代码就创建一个表，使用 BTREE 索引。

```sql
create table users
(
    id smallint unsigned not null auto_increment,
    username varchar(15) not null,
    pwd varchar(15) not null,
    index using btree (username),
    primary key (id)
)engine=memory;
```

# MERGE

MERGE 存储引擎是一组 MyISAM 表的组合，这些 MyISAM 表结构必须完全相同，尽管其使用不如其它引擎突出，但是在某些情况下非常有用。说 白了，Merge 表就是几个相同 MyISAM 表的聚合器；Merge 表中并没有数据，对 Merge 类型的表可以进行查询、更新、删除操作，这些操作实际上 是对内部的 MyISAM 表进行操作。Merge 存储引擎的使用场景。
对于服务器日志这种信息，一般常用的存储策略是将数据分成很多表，每个名称与特定的时间端相关。例如：可以用 12 个相同的表来存储服务器日志数据，每个表用对应各个月份的名字来命名。当有必要基于所有 12 个日志表的数据来生成报表，这意味着需要编写并更新多表查询，以反映这些表中的信息。与其编写这 些可能出现错误的查询，不如将这些表合并起来使用一条查询，之后再删除 Merge 表，而不影响原来的数据，删除 Merge 表只是删除 Merge 表的定义，对内部的表没有任何影响。

# ARCHIVE

Archive 是归档的意思，在归档之后很多的高级功能就不再支持了，仅仅支持最基本的插入和查询两种功能。在 MySQL 5.5 版以前，Archive 是不支持索引，但是在 MySQL 5.5 以后的版本中就开始支持索引了。Archive 拥有很好的压缩机制，它使用 zlib 压缩库，在记录被请求时会实时压缩，所以它经常被用来当做仓库使 用。

# NDBCluster | 分布式存储引擎

尽可能让查询简单，避免数据的跨节点传输，尽可能满足 SQL 节点的计算性能，大一点的集群 SQL 节点会明显多余 Data 节点，在各节点之间尽可能使用万兆网络环境互联，以减少数据在网络层传输过程中的延时。
