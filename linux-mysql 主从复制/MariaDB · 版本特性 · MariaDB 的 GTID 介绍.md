# MariaDB · 版本特性 · MariaDB 的 GTID 介绍

## 简介

简单来说，MariaDB（MySQL）的复制机制是这样的：

在Master端所有数据库的变更（包括DML和DDL）都会以 Binlog Event 的方式写入Binlog中。Slave会连上Master然后读取 Binlog Event，再重放这些操作到自身的数据中。一个实例可以既是Master同时又是Slave，做成双向复制。也可以一级一级串联，做成级联复制，Binlog Event 中包含的 server_id 可以识别产生 Event 的实例，避免重复执行。

Slave会保存最后一次收到和应用的Binlog的位置，因此Slave重连Master时可以从中断的位置继续开始复制。也可以在暂停Slave后，将其整体拷贝到新的位置，然后作为一个新的Slave继续复制。

全局事务ID（Global transaction ID，GTID）为每个 Event Group （就是一系列 Event 组成的一个原子单元，要么一起提交要么都无法提交）引入了一个标识，因此 GTID 是标识“事务”的最佳方式（尽管 Event 里面还包含一些非事务的DML语句和DDL，它们可以作为一个单独的 Event Group ）。每当一个 Event Group 从Master复制到Slave时，它的 GTID 也通过 GTID Event 被传到Slave。因为每个 GTID 在整个复制拓扑结构中都是一个唯一标志，所以这使得在不同的实例之间识别相同的 Binlog Events 非常简单，然而在有 GTID 之前，想做到这点是很困难的。MariaDB 从 10.0.2 开始提供 GTID 支持，但是 MariaDB 的 GTID 与 MySQL 的 GTID 在实现原理上并不相同，因为 MariaDB 支持像多源复制啊、多主复制等官方暂时还没考虑的复制模型。下面我们来看看 MariaDB 的 GTID 的实现。

## 优势

使用 GTID 有两个主要的优势：

1. 在级联复制、一主多从等复杂的复制场景下，可以更简单地将一个Slave的复制修改到另一个Master上，而不用人工去寻找复制的起始位点。从5.0一路走来的同学应该很能理解这种痛苦。

   这是因为Slave会保存最后一个执行的 Event Group 的 GTID，因此可以通过这个 GTID 很容易地在新Master上找到相应的复制起点。而在使用 Binlog File 和 Binlog Pos 的时代，这是很难办到的。

2. Slave的状态是 Crash-Safe 的。

   Slave的执行状态（最后一个执行的 GTID）被记录在 `mysql.gtid_slave_pos` 系统表中。如果这张表使用的是事务引擎（例如InnoDB，默认就是），那么 **修改用户表的数据** 和 **修改Slave状态的系统表** 这两个操作在就可以放在一个事务中完成，这就保证了Slave状态是 Crash-Safe 的，如果Slave崩溃了，那么 Crash Recovery 就可以在重启的时候把用户数据表和Slave状态系统表恢复到一个一致的位点。而在非 GTID 复制的旧版本中，这也是做不到的，Slave状态只是简单的存放在 relay-log.info 文件中 （MySQL是可以把 Binlog File 和 Binlog Pos 也存在`slave_relay_log_info` 和 `slave_master_info` 系统表中），而且需要靠不断的 `fsync()` 调用才能同步到磁盘上，一旦宕机很可能导致Slave状态跟实际不一致（但是也只有事务引擎的DML能保证一致，非事务引擎和DDL本身就不是Crash-Safe的）。

基于这两个优势，通常我们都建议使用GTID复制。并且传统的基于Binlog文件位置的复制方式，和 GTID 的复制方式，在 MariaDB 中是可以相互之间平滑的切换的。

## 实现方式

每个GTID，都包含三个数字部分，分别用’-‘号隔开，例如：

0-1-10

第一个数字’0’是Domain ID，这是一个32位的无符号整型。

第二个数字’1’是Server ID，这跟传统的主备复制中 Server ID 的含义是一样的，也是一个32位无符号整型。因此在一个复制拓扑中每个实例的Server ID必须是唯一的。

第三个数字是序列号（Sequence Number）。这是一个64位的无符号整型。每个新产生的 Event Group 记录到Binlog时都会新生成一个单调递增的序列号。

这个规则使得 (server_id, sequence_number) 总是唯一的，因此GTID也是全局唯一的。

使用64位数字可以提供充足序列号支持庞大的 Event Group 数量，在可预见的时间内，应该来说是没有溢出风险的。但是一个可见的风险是，人为地设置一个很高的 `gtid_seq_no` 值，导致 GTID 的起始序列号就很高，是可能导致序列号逼近64位数值的上限的。

## Domain ID

当 Events 从Master复制到Slave时，Events 总是按照从Master读取的顺序记录在Slave的Binlog中。因此，如果同一时刻只有一个Master接收变更操作（不包括复制带来的变更操作），那么 Binlog 中 Events 的顺序在所有参与复制关系的实例上应该都是一样的。

这种一致的 Binlog 顺序，可以被Slave用来追踪当前复制的位置。只要Slave记住最后一个从Master复制过来的 Event Group 的 GTID，重连到Master时（不管是原来的Master还是新的Master），就可以发送这个 GTID 给Master，然后Master就可以开始继续发送这个 GTID 之后的 Event.

然而，如果用户同时在多个实例上做了更新，那么一般来说各个实例上的Binlog顺序是不可能一样的。当使用多源复制、或者实例之间构成环状拓扑结构时，这种情况是可能出现的；或者人为手动对一个Slave做了更新操作，也可能发生这种情况。如果 Binlog 顺序在新老Master之间不一样，那么仅仅使用一个独立的GTID（不包含 Domain ID）并不足以记录当前的状态。

而 Domain ID 的任务, 就是为了解决这种情况。

通常，Binlog 并非是一个单一有序的数据流（Strem），相反，它是由许多不同的数据流组成的，每个数据流都由一个自己的 Domain ID 来识别。对每个数据流，GTID 总是以相同的 Binlog 顺序存储在每个实例中。但是，不同的数据流可以以不同的方式在不同的实例中交错。

Slave通过记录每个复制数据流（Replication Stream）中最后一次应用的 GTID 位置来跟踪复制的位点。当连上一个新的Master时，Slave可以为每个 Domain ID 从不同的 Binlog 位点开始复制。

后面有一节我们专门阐述了如何设置 Domain ID，以及如何在多源复制、多主复制的场景下利用 Domain ID.

只有一个Master的简单场景是不用考虑 Domain ID 的，任意时刻只有一个Master会有应用去更新，所以只需要一个单独的 Replication Stream 即可，Domain ID 可以直接忽略，在所有实例上用默认值0就行了。

## GTID的应用

从 MariaDB 10.0.2 开始，GTID 是默认自动打开的。每个 Event Group 写到 Binlog 时会先收到一个GTID_EVENT，用MariaDB的 mysqlbinlog 工具或者 SHOW BINLOG EVENTS 命令可以看到这个Event。

Slave自动记录了最后一次应用的 Event Group 的 GTID，可以通过 `gtid_slave_pos` 变量来查看：

```
SELECT @@GLOBAL.gtid_slave_pos
0-1-1
```

当Slave连接到Master时，可以选择是否使用 GTID 方式，或者使用原来的文件位置的方式来判断起始的复制点位。如果使用 GTID 方式复制，那么在 CHANGE MASTER 的时候使用 `master_use_gtid` 选项来设置：

```
CHANGE MASTER TO master_use_gtid = { slave_pos | current_pos | no }
```

`CHANGE MASTER TO master_use_gtid=slave_pos` 将把Slave配置为使用 GTID 方式。当Slave连接到Master时，Master将从最后一个GTID开始给Slave复制 Binlog，可以通过 `@@gtid_slave_pos`这个变量来查看目前最后一个GTID是什么。由于GTID在所有参与复制的实例之间都是相同的，因此Slave可以被指向不同的Master，Master可以自动决定正确的复制起始位置。

但是，假设我们设置了两个实例A和B，并且让A是Master，B是他的Slave。运行一段时间后，我们关闭A，然后让B成为新的Master，然后一段时间后我们再把A加回来作为B的Slave。

由于A从来没有成为Slave，它没记录任何之前复制的GTID，所以 `@@gtid_slave_pos` 是空的。如果要让A自动被加为Slave，可以使用 `master_use_gtid=current_pos` 这个方法。这样做在连接的时候，Slave会把 `@@gtid_current_pos` 存的GTID发给Master，而不是 `@@gtid_slave_pos` ，这样就把A做Master时产生的最后一个GTID发送给了B，然后从这个位置开始复制。

使用 `master_use_gtid=current_pos` 可能是最简单的方式，因为这样不需要考虑之前这个实例是作为Master还是Slave。但是，必须注意的是，如果这样做的话，就不要在Slave上做任何非复制带来的修改操作，否则就乱了。如果出现了这种情况，复制可能无法继续，因为这些事务在Master运行时并没有在Master上出现过，当切换Master和Slave身份时，就出现了未知的GTID。为了避免这种情况，可以在Slave上设置 `@@sql_log_bin=0` 。

如果这不是Slave期望的运行方式，Slave上就是可能有一些数据变更，那么就应该使用 `master_use_gtid=slave_pos` 方式。这样Slave总是使用最后一次复制的GTID发送给Master来获取之后的Event Group。这可以避免上面的方式在一些不可控因素修改了Slave本地的数据却没有在Binlog之中有所记录的问题。

当GTID严格模式开启时（ `@@GLOBAL.gtid_strict_mode=1` ），通常最好是用 `current_pos` 。在严格模式下，不允许有额外的事务。

如果Slave没有开启Binlog，那么 `current_pos` 和 `slave_pos` 是一回事。

即使当Slave被配置为旧的复制方式时（ `CHANGE MASTER TO master_log_file=..., master_log_pos=...` ），MariaDB依然会跟踪当前的GTID位置并保存在 `@@GLOBAL.gtid_slave_pos` 。这意味着一个用非GTID模式复制的Slave可以很容易地修改为GTID方式：

```
CHANGE MASTER TO master_use_gtid = slave_pos
```

Slave会保存 `master_use_gtid=slave_pos|master_pos` 的信息以为后来进行连接，直到复制方式被修改为指定 `master_log_file/pos=...` 或者 `master_use_gtid=no` 。当前的复制方式可以通过SHOW SLAVE STATUS的Using_Gtid列来判断:

```
SHOW SLAVE STATUS\G
...
Using_Gtid: Slave_pos
```

Slave内部用 `mysql.gtid_slave_pos` 表来存储GTID位置（所以重启后 `@@GLOBAL.gtid_slave_pos` 的值会重新被填充）。升级MariaDB的版本到10.0之后，必须使用 `mysql_upgrade` 来保证这些表和列被创建了。

为了保证Crash-Safe，这张表必须使用事务引擎，例如InnoDB。当MariaDB第一次安装或者升级到10.0.2+时，这张表会用默认的存储引擎创建 - 默认就是InnoDB。当然如果把默认引擎改成MyISAM的话，这张表就会被创建成MyISAM。如果需要修改引擎，可以直接用ALTER TABLE的方式：

```
ALTER TABLE mysql.gtid_slave_pos ENGINE = InnoDB
```

`mysql.gtid_slave_pos` 表不应该被Slave线程之外的方式修改。尤其是不要尝试直接修改表的数据来改变Slave的GTID位置，如果要修改应该用这种方式：

```
SET GLOBAL gtid_slave_pos = '0-1-1'
```

### 配置一个新的Slave使用GTID

设置一个新的使用GTID的Slave跟设置一个非GTID方式复制的Slave差别很大，基本步骤是:

1. 配置一个新的实例并且载入初始数据。
2. 从相应的Master Binlog位置开始Slave的复制。

#### 从一个空实例开始

出于测试的目的，最简单的方式就是新建一个新的空实例，然后从Master复制所有的Binlog（在实际生产环境中这几乎是不可能的，因为最早的Binlog文件应该早就被清除了）。

正常方式安装的Slave实例，默认情况下GTID位点都是空的，因此可以从Master的第一个Binlog文件开始复制。但是如果Slave之前被用作其他目的，那么初始位置需要手动设置为空：

```
SET GLOBAL gtid_slave_pos = "";
```

下一步就是用CHANGE MASTER来指向Master，指定 `master_host` 什么的。只是不再用 `master_log_file` 和 `master_log_pos` ，而是用 `master_use_gtid=current_pos` （或者slave_pos）:

```
CHANGE MASTER TO master_host="127.0.0.1", master_port=3310, master_user="root", master_use_gtid=current_pos;
START SLAVE;
```

#### 从备份集设置

一般来说创建Slave的方式都是通过备份集来恢复出一个新的实例，然后找到Master上复制的起始点创建复制关系。

因而找到正确的复制起始位置是非常重要的，否则Slave可能因为数据与Master不一致而导致复制中断。

一般来说备份都是用 XtraBackup 或者 mysqldump。这两种方式都可以在非阻塞的情况下获得备份时正确的Binlog位点（所有表都要是事务引擎），当然，如果备份时不会有写入，那么 SHOW MASTER STATUS 也能提供正确的位点。

一旦获取了备份时正确的Binlog位点（文件名和偏移量），那么就可以用 `BINLOG_GTID_POS()`函数来计算GTID:

```
SELECT BINLOG_GTID_POS("master-bin.000001", 600);
```

从MariaDB 10.0.13版本开始，mysqldump会自动完成这个工作，并且把GTID的写在导出文件中，只要设置 –master-data 或 –dump-slave 的同时设置 –gtid 即可。

这样的话新的SLAVE就可以通过设置 `@@gtid_slave_pos` 的值来设定复制的起始位置，用 CHANGE MASTER 把这个值传给主库，然后开始复制：

```
SET GLOBAL gtid_slave_pos = "0-1-2";
CHANGE MASTER TO master_host="127.0.0.1", master_port=3310, master_user="root", master_use_gtid=slave_pos;
START SLAVE;
```

用Master备份搭建一个Slave时这种方式尤其有用。不过一定要记得确保Master和Slave的server_id要设置成不一样的。

如果备份是从现有的Slave实例创建的，那么GTID的位置已经存在 `mysql.gtid_slave_pos` 表中了，并且跟其他事务引擎表都是在一个一致状态。这种情况下，就没必要去找GTID的位置然后设置变量之类的，因为已经从 `mysql.gtid_slave_pos` 载入了正确的值。然而从Master做备份就没这个福利了，因为正确的GTID位点信息在Binlog中，而不是在 `mysql.gtid_slave_pos` 。

#### 将非GTID复制的SLAVE切换为GTID方式

如果已经有一个SLAVE运行在Binlog文件名和偏移量的复制模式下，那么可以直接修改为GTID模式。对于升级来说这是很有用的方式。

当一个SLAVE通过Binlog位点的方式跟Master连接，并且Master支持GTID，那么SLAVE会自动把GTID位置的信息也获取过来，并且在复制过程中会不断更新。因此，当一个SLAVE已经连上了一个支持GTID的Master，那么并不需要额外的动作就可以把复制切换为使用GTID：

```
STOP SLAVE;
CHANGE MASTER TO master_host="127.0.0.1", master_port=3310, master_user="root", master_use_gtid=current_pos;
START SLAVE;
```

（后面更新的版本可能会增加一种方式，如果第一次连接的时候使用的是原来的Binlog文件位点的方式，那么只要主库是支持GTID的，后面再连接的时候就自动切换为GTID方式）

### 更换Slave的Master

一旦复制运行在GTID模式下（ `master_use_gtid=current_pos|slave_pos` ），Slave就可以很容易地用CHANGE MASTER更换到新的Master：

```
STOP SLAVE;
CHANGE MASTER TO master_host='127.0.0.1', master_port=3312;
START SLAVE;
```

Slave已经记录了最后执行的旧Master的GTID，而且由于GTID在整个复制拓扑中都是全局唯一的，因此Slave只需要根据GTID在新Master的Binlog里找到合适的位置，继续复制就行了。

Binlog是一组有序的Events数据流（或者多个数据流，每个复制域（Replication Domain）都是一个数据流，参照接下来那一节），数据流内的Event在每个Slave上总是按照同一顺序被应用。MariaDB 的 GTID 凭借这个顺序，使得它足以记住每个数据流中的这个点。由于Event顺序在每个实例上是相同的，切换到另一个实例的Binlog中相同的GTID点将会得到一样的结果。

比较通俗易懂的讲，就是MariaDB的GTID复制是全异步的，并且是非常灵活的。甚至Binlog顺序的一致性被破坏了，切换Master后，GTID依然可以尝试从当前GTID位点继续复制。

Binlog顺序在不同的实例之间产生不一致，最常见的情况是用户或者DBA直接更新了Slave实例（并且把日志写到Binlog了）。这会导致Slave上的有些Event，在Master和其他Slave上都没有。虽然可以通过设置 `sql_log_bin=0` 来避免这些变更写到Binlog。

这通常是避免实例之间Binlog不一样的最好的办法。话虽然这么说，但MariaDB的复制就是为最大的灵活性而设计的，而且有时这种不一致是有合理需求的。这种情况下，只需要理解GTID位点在每个Binlog数据流（每个Replication Domain就有数据流）中是一个单独的点。

当一个复制拓扑中两个Master在同一时间都是活跃的时候，也可能出现不一致。比如使用环形多主复制的时候。但是只要保证旧Master上的变更在完全复制到所有Slave之前不允许切换到新的Master。通常情况下，要切换Master，首先原Master上的写应该先停止，然后等待所有变更复制到了新的Master，然后写开始发送到新的Master。有意使用多个Master写的情况也是支持的，下一节会描述这种情况。

GTID严格模式将用于强制保证所有实例之间的Binlog相同。当这个选项开启，如果遇到任何不一致情况都会导致复制停止并且报错。

### 使用多源复制和其他多主情况的设置

MariaDB的GTID支持同时有多个活跃的Master。通常这种情况发生在多源复制或者环形多主情况下。

在这样的设置下，每个活跃的Master都必须用友自己独特的Replication Domain ID， `gtid_domain_id` 。然后Binlog实际上就会由多个独立的数据流组成，每个活跃的Master都有一个。在每个Replication Domain中，每个实例的Binlog顺序总是一样的。但是两个不同的数据流在不同的实例的Binlog里可能是交错的。

这样的话，一个GTID位点不单单是一个单独的GTID了，它将是每个Domain ID下最后一个执行的Event Group的GTID，实际上这表示了每个Binlog数据流达到的位置。当Slave连上Master的时候，它可以在不同的Binlog位置上继续复制一个数据流。由于一个数据流内的数据顺序在不同实例上是一样的，这使得Slave可以在任何一个新的Master上找到正确的位点继续复制。

Domain ID是由DBA按照应用的需要分配的， `@@GLOBAL.gtid_domain_id` 的默认值是0。对于大部分的复制场景这都是个合适的值，只要复制拓扑中同时只有一个活跃的Master。MariaDB永远不会自己写一个新的domain_id到Binlog中。

当使用多源复制的时候，一个Slave同时会连上多个Master，每个Master都应该配置一个不同的Domain ID。

同样的，在环形多主复制拓扑结构中，所有的环上所有的Master都会被应用并发写入（可以通过一些机制来避免冲突，例如区分ID段），每个实例也需要配置不同的Domain ID。（在环形多主情况下，如果应用程序保证同一时刻只会更新一个Master，那么一个Domain ID就够了）

正常情况下，一个Slave实例不应该直接接受任何更新（这导致了跟Master的Binlog不一致）。因此在Slave上 `gtid_domain_id` 被设为什么值并不重要，尽管它可能是有意义的，例如把它设置为跟Master一样（如果不使用多主），来使其更容易把Slave改成新的Master。当然，如果Slave自身是一个活跃的Master，例如在环形多主拓扑中，那么Domain ID还是应该设置的，因为这时候实例的角色是一个活跃的Master。

需要注意的是，Domain ID和Server ID是不同的东西。虽然为每个实例设置不同的Domain ID也是可以的，但这通常不是所希望的情况。这会导致当前的GTID位置（ `@@global.gtid_slave_pos`）更难理解，并且失去了所有实例拥有一个一致的Binlog数据流的好处。只建议在每个会被应用同时更新的活跃的Master实例上配置Domain ID。

没有正确的配置Domain ID本身并不是一个错误（比如根本没配）。例如，一个5.5版本的环形多主复制环境，升级到10.0。这个环可以继续像之前一样继续工作下去，即使所有的实例上Domain ID还是默认值0。甚至有可能还是使用GTID在实例之间复制。然而，当切换Slave的Master时必须小心翼翼，如果Binlog顺序在新旧Master不一样，那么用一个单独的GTID位置（没有Domain ID）从新的Master继续复制，数据就可能出错。