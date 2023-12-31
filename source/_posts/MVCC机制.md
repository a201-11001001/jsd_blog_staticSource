---
title: MVCC机制
date: 2023-11-17 10:06:58
tags: 
    - MySql
description: innodb引擎对MVCC机制的实现 ## 页面描述
keywords: mvcc  ## 关键字
comments: true  ## 评论模块
cover: /top_img/MVCC机制top_img.jpeg ## 文章缩略图
# top_img:  ## 顶部图片
aside: false  ## 显示侧边栏（默认 true）
---

#### 多版本并发控制 (Multi-Version Concurrency Control)
MVCC 是一种并发控制机制，用于在多个并发事务同时读写数据库时保持数据的一致性和隔离性。它是通过在每个数据行上维护多个版本的数据来实现的。当一个事务要对数据库中的数据进行修改时，MVCC 会为该事务创建一个数据快照，而不是直接修改实际的数据行。

> MVCC 机制只有在 PR、PC下兼容，其他隔离界别下没有

---

##### InnoDB 对 MVCC 的实现
MVCC 的实现依赖于：隐藏字段、Read View、undo log。在内部实现中，InnoDB 通过数据行的 DB_TRX_ID 和 Read View 来判断数据的可见性，如不可见，则通过数据行的 DB_ROLL_PTR 找到 undo log 中的历史版本。每个事务读到的数据版本可能是不一样的，在同一个事务中，用户只能看到该事务创建 Read View 之前已经提交的修改和该事务本身做的修改

##### Innodb隐藏字段
1. DB_ROW_ID
行标识（隐藏单调自增 ID），大小为 6 字节，如果表没有主键，InnoDB 会自动生成一个隐藏主键，因此会出现这个列。如果没有设置主键且该表没有唯一非空索引时，InnoDB 会使用该 id 来生成聚簇索引

2. DB_TRX_ID（最近一次提交事务的ID）：修改表数据时，都会提交事务，每个事务都有一个唯一的ID，这个字段就记录了最近一次提交事务的ID。另外，每条记录的头信息（record header）里都有一个专门的 bit（deleted_flag）来表示当前记录是否已经被删除。

3. DB_ROLL_PTR（上个版本的地址）：修改表数据时，旧版本的数据都会被记录到Undo Log日志中，每个版本的数据都有一个版本地址，这个字段记录的就是上个版本的地址。（回滚指针，指向该行的 undo log 。如果该行未被更新，则为空）

##### read view
> 在 read committed 和 read repeatable 隔离级别下，MVCC 采用read view 来实现的，它们的区别在于创建 read view 时机不同：
> Read View 主要是用来做可见性判断，里面保存了 “当前对本事务不可见的其他活跃事务”

##### 主要有以下字段：

* m_low_limit_id：目前出现过的最大的事务 ID+1，即下一个将被分配的事务 ID。大于等于这个 ID 的数据版本均不可见
* m_up_limit_id：活跃事务列表 m_ids 中最小的事务 ID，如果 m_ids 为空，则 m_up_limit_id 为 m_low_limit_id。小于这个 ID 的数据版本均可见
* m_ids：Read View 创建时其他未提交的活跃事务 ID 列表。创建 Read View时，将当前未提交事务 ID 记录下来，后续即使它们修改了记录行的值，对于当前事务也是不可见的。m_ids 不包括当前事务自己和已提交的事务（正在内存中）
* m_creator_trx_id：创建该 Read View 的事务 ID

##### 事务可见性示意图
![事务可见性示意图](read-view-visibility.png)


##### undo-log
> undo log 主要有两个作用：

当事务回滚时用于将数据恢复到修改前的样子
另一个作用是 MVCC ，当读取记录时，若该记录被其他事务占用或当前版本对该事务不可见，则可以通过 undo log 读取之前的版本数据，以此实现非锁定读

##### 在 InnoDB 存储引擎中 undo log 分为两种：insert undo log 和 update undo log：

1. insert undo log：指在 insert 操作中产生的 undo log。因为 insert 操作的记录只对事务本身可见，对其他事务不可见，故该 undo log 可以在事务提交后直接删除。不需要进行 purge 操作
insert 时的数据初始状态：
![mvcc_1](mvcc_1.png)

2. update undo log：update 或 delete 操作中产生的 undo log。该 undo log可能需要提供 MVCC 机制，因此不能在事务提交时就进行删除。提交时放入 undo log 链表，等待 purge线程 进行最后的删除

##### 数据第一次被修改时：
![mvcc_2](MVCC_2.png)

##### 数据第二次被修改时：
![mvcc_3](MVCC_3.png)

不同事务或者相同事务的对同一记录行的修改，会使该记录行的 undo log 成为一条链表，链首就是最新的记录，链尾就是最早的旧记录。

##### 数据可见性算法
在 InnoDB 存储引擎中，创建一个新事务后，执行每个 select 语句前，都会创建一个快照（Read View），**快照中保存了当前数据库系统中正处于活跃（没有 commit）的事务的 ID 号**。其实简单的说保存的是系统中当前不应该被本事务看到的其他事务 ID 列表（即 m_ids）。当用户在这个事务中要读取某个记录行的时候，InnoDB 会将该记录行的 DB_TRX_ID 与 Read View 中的一些变量及当前事务 ID 进行比较，判断是否满足可见性条件

具体的比较算法如下(图源)：
![mvcc_4](MVCC_4.png)

1. 如果记录 DB_TRX_ID < m_up_limit_id，那么表明最新修改该行的事务（DB_TRX_ID）在当前事务创建快照之前就提交了，所以该记录行的值对当前事务是可见的

2. 如果 DB_TRX_ID >= m_low_limit_id，那么表明最新修改该行的事务（DB_TRX_ID）在当前事务创建快照之后才修改该行，所以该记录行的值对当前事务不可见。跳到步骤 5

3. m_ids 为空，则表明在当前事务创建快照之前，修改该行的事务就已经提交了，所以该记录行的值对当前事务是可见的

4. 如果 m_up_limit_id <= DB_TRX_ID < m_low_limit_id，表明最新修改该行的事务（DB_TRX_ID）在当前事务创建快照的时候可能处于“活动状态”或者“已提交状态”；所以就要对活跃事务列表 m_ids 进行查找（源码中是用的二分查找，因为是有序的）

5. 如果在活跃事务列表 m_ids 中能找到 DB_TRX_ID，表明：① 在当前事务创建快照前，该记录行的值被事务 ID 为 DB_TRX_ID 的事务修改了，但没有提交；或者 ② 在当前事务创建快照后，该记录行的值被事务 ID 为 DB_TRX_ID 的事务修改了。这些情况下，这个记录行的值对当前事务都是不可见的。跳到步骤 5

6. 在活跃事务列表中找不到，则表明“id 为 trx_id 的事务”在修改“该记录行的值”后，在“当前事务”创建快照前就已经提交了，所以记录行对当前事务可见

7. 在该记录行的 DB_ROLL_PTR 指针所指向的 undo log 取出快照记录，用快照记录的 DB_TRX_ID 跳到步骤 1 重新开始判断，直到找到满足的快照版本或返回空

##### RC 和 RR 隔离级别下 MVCC 的差异
在事务隔离级别 RC 和 RR （InnoDB 存储引擎的默认事务隔离级别）下，InnoDB 存储引擎使用 MVCC（非锁定一致性读），但它们生成 Read View 的时机却不同

* 在 RC 隔离级别下的 每次select 查询前都生成一个Read View (m_ids 列表)
* 在 RR 隔离级别下只在事务开始后 第一次select 数据前生成一个Read View（m_ids 列表）

##### MVCC 解决不可重复读问题
虽然 RC 和 RR 都通过 MVCC 来读取快照数据，但由于 生成 Read View 时机不同，从而在 RR 级别下实现可重复读

举个例子：
![mvcc_5](MVCC_5.png)

##### 在 RC 下 ReadView 生成情况
1. 假设时间线来到 T4 ，那么此时数据行 id = 1 的版本链为：
![mvcc_6](MVCC_6.png)

由于 RC 级别下每次查询都会生成Read View ，并且事务 101、102 并未提交，此时 103 事务生成的 Read View 中活跃的事务 m_ids 为：[101,102] ，m_low_limit_id为：104，m_up_limit_id为：101，m_creator_trx_id 为：103

* 此时最新记录的 DB_TRX_ID 为 101，m_up_limit_id <= 101 < m_low_limit_id，所以要在 m_ids 列表中查找，发现 DB_TRX_ID 存在列表中，那么这个记录不可见
* 根据 DB_ROLL_PTR 找到 undo log 中的上一版本记录，上一条记录的 DB_TRX_ID 还是 101，不可见
* 继续找上一条 DB_TRX_ID为 1，满足 1 < m_up_limit_id，可见，所以事务 103 查询到数据为 name = 菜花

2. 时间线来到 T6 ，数据的版本链为：
![mvcc_7](MVCC_7.png)

因为在 RC 级别下，重新生成 Read View，这时事务 101 已经提交，102 并未提交，所以此时 Read View 中活跃的事务 m_ids：[102] ，m_low_limit_id为：104，m_up_limit_id为：102，m_creator_trx_id为：103

* 此时最新记录的 DB_TRX_ID 为 102，m_up_limit_id <= 102 < m_low_limit_id，所以要在 m_ids 列表中查找，发现 DB_TRX_ID 存在列表中，那么这个记录不可见

* 根据 DB_ROLL_PTR 找到 undo log 中的上一版本记录，上一条记录的 DB_TRX_ID 为 101，满足 101 < m_up_limit_id，记录可见，所以在 T6 时间点查询到数据为 name = 李四，与时间 T4 查询到的结果不一致，不可重复读！

3. 时间线来到 T9 ，数据的版本链为：
![mvcc_8](MVCC_8.png)

重新生成 Read View， 这时事务 101 和 102 都已经提交，所以 m_ids 为空，则 m_up_limit_id = m_low_limit_id = 104，最新版本事务 ID 为 102，满足 102 < m_low_limit_id，可见，查询结果为 name = 赵六

> 总结： 在 RC 隔离级别下，事务在每次查询开始时都会生成并设置新的 Read View，所以导致不可重复读

##### 在 RR 下 ReadView 生成情况
在可重复读级别下，只会在事务开始后第一次读取数据时生成一个 Read View（m_ids 列表）

1. 在 T4 情况下的版本链为：
![mvcc_9](MVCC_9.png)

在当前执行 select 语句时生成一个 Read View，此时 m_ids：[101,102] ，m_low_limit_id为：104，m_up_limit_id为：101，m_creator_trx_id 为：103

此时和 RC 级别下一样：

* 最新记录的 DB_TRX_ID 为 101，m_up_limit_id <= 101 < m_low_limit_id，所以要在 m_ids 列表中查找，发现 DB_TRX_ID 存在列表中，那么这个记录不可见
* 根据 DB_ROLL_PTR 找到 undo log 中的上一版本记录，上一条记录的 DB_TRX_ID 还是 101，不可见
* 继续找上一条 DB_TRX_ID为 1，满足 1 < m_up_limit_id，可见，所以事务 103 查询到数据为 name = 菜花

2. 时间点 T6 情况下：
![mvcc_10](MVCC_10.png)

在 RR 级别下只会生成一次Read View，所以此时依然沿用 m_ids：[101,102] ，m_low_limit_id为：104，m_up_limit_id为：101，m_creator_trx_id 为：103

* 最新记录的 DB_TRX_ID 为 102，m_up_limit_id <= 102 < m_low_limit_id，所以要在 m_ids 列表中查找，发现 DB_TRX_ID 存在列表中，那么这个记录不可见

* 根据 DB_ROLL_PTR 找到 undo log 中的上一版本记录，上一条记录的 DB_TRX_ID 为 101，不可见

* 继续根据 DB_ROLL_PTR 找到 undo log 中的上一版本记录，上一条记录的 DB_TRX_ID 还是 101，不可见

* 继续找上一条 DB_TRX_ID为 1，满足 1 < m_up_limit_id，可见，所以事务 103 查询到数据为 name = 菜花

3. 时间点 T9 情况下：
![mvcc_11](MVCC_11.png)
<!-- {% asset_img MVCC_11.png mvcc_11 %} -->
<!-- {% asset_img MVCC_11.png [title] %} -->

此时情况跟 T6 完全一样，由于已经生成了 Read View，此时依然沿用 m_ids：[101,102] ，所以查询结果依然是 name = 菜花

#### MVCC➕Next-key-Lock 防止幻读
---
InnoDB存储引擎在 RR 级别下通过 MVCC和 Next-key Lock 来解决幻读问题
1、执行普通 select，此时会以 MVCC 快照读的方式读取数据

在快照读的情况下，RR 隔离级别只会在事务开启后的第一次查询生成 Read View ，并使用至事务提交。所以在生成 Read View 之后其它事务所做的更新、插入记录版本对当前事务并不可见，实现了可重复读和防止快照读下的 “幻读”

2、执行 select...for update/lock in share mode、insert、update、delete 等当前读

在当前读下，读取的都是最新的数据，如果其它事务有插入新的记录，并且刚好在当前事务查询范围内，就会产生幻读！InnoDB 使用 Next-key Lock 来防止这种情况。当执行当前读时，会锁定读取到的记录的同时，锁定它们的间隙，防止其它事务在查询范围内插入数据。只要我不让你插入，就不会发生幻读