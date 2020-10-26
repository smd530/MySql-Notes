# 事务隔离级别和MVCC

**ACID**

原子性：要么做 要么不做

隔离性：现实世界中的两次状态转换应该是互不影响

一致性：主键、唯一索引、外键、声明某个列为`NOT NULL`来拒绝`NULL`值的插入

持久性：数据库需改的数据保留到磁盘

保证`原子性`、`隔离性`、`一致性`和`持久性`的一个或多个数据库操作称之为一个`事务`（英文名是：`transaction`）。

```mysql
CREATE TABLE hero (
    number INT,
    name VARCHAR(100),
    country varchar(100),
    PRIMARY KEY (number)
) Engine=InnoDB CHARSET=utf8;

INSERT INTO hero VALUES(1, '刘备', '蜀');
```

## 事务隔离级别

**事务并发执行遇到的问题**

**脏写**：一个事务修改了另一个未提交事务修改过的数据

**脏读**：一个事务读到了另一个未提交事务修改过的数据

![](http://img.tomato530.com/dirtyRead.jpg)

如上图，`Session A`和`Session B`各开启了一个事务，`Session B`中的事务先将`number`列为`1`的记录的`name`列更新为`'关羽'`，然后`Session A`中的事务再去查询这条`number`为`1`的记录，如果读到列`name`的值为`'关羽'`，而`Session B`中的事务稍后进行了回滚，那么`Session A`中的事务相当于读到了一个不存在的数据，这种现象就称之为`脏读`。

**不可重复读（Non-Repeatable Read）**：

如果一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值，那就意味着发生了`不可重复读` 

![](http://img.tomato530.com/noRepeatRead.jpg)

如上图，我们在`Session B`中提交了几个隐式事务（注意是隐式事务，意味着语句结束事务就提交了），这些事务都修改了`number`列为`1`的记录的列`name`的值，每次事务提交之后，如果`Session A`中的事务都可以查看到最新的值，这种现象也被称之为`不可重复读`。

**幻读（Phantom）**

如果一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来，那就意味着发生了`幻读`。

![](http://img.tomato530.com/phantom.jpg)

如上图，`Session A`中的事务先根据条件`number > 0`这个条件查询表`hero`，得到了`name`列值为`'刘备'`的记录；之后`Session B`中提交了一个隐式事务，该事务向表`hero`中插入了一条新记录；之后`Session A`中的事务再根据相同的条件`number > 0`查询表`hero`，得到的结果集中包含`Session B`中的事务新插入的那条记录，这种现象也被称之为`幻读`。

**SQL标准中的四种隔离级别**

按问题严重性排序：脏写 > 脏读 > 不可重复读 > 幻读

标准中设立了4个`隔离级别`：

- `READ UNCOMMITTED`：读未提交。
- `READ COMMITTED`：读已提交。
- `REPEATABLE READ`：可重复读。
- `SERIALIZABLE`：可串行化。

`SQL标准`中规定，针对不同的隔离级别，并发事务可以发生不同严重程度的问题，具体情况如下：

|      隔离级别      |     脏读     |  不可重复读  |     幻读     |
| :----------------: | :----------: | :----------: | :----------: |
| `READ UNCOMMITTED` |   Possible   |   Possible   |   Possible   |
|  `READ COMMITTED`  | Not Possible |   Possible   |   Possible   |
| `REPEATABLE READ`  | Not Possible | Not Possible |   Possible   |
|   `SERIALIZABLE`   | Not Possible | Not Possible | Not Possible |

- `READ UNCOMMITTED`隔离级别下，可能发生`脏读`、`不可重复读`和`幻读`问题。
- `READ COMMITTED`隔离级别下，可能发生`不可重复读`和`幻读`问题，但是不可以发生`脏读`问题。
- `REPEATABLE READ`隔离级别下，可能发生`幻读`问题，但是不可以发生`脏读`和`不可重复读`的问题。
- `SERIALIZABLE`隔离级别下，各种问题都不可以发生。

**MySQL在REPEATABLE READ隔离级别下，是可以禁止幻读问题的发生的**

**HOW设置事务的隔离级别**

```mysql
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL level;
SET GLOBAL TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

其中的`level`可选值有4个：

```mysql
level: {
     REPEATABLE READ
   | READ COMMITTED
   | READ UNCOMMITTED
   | SERIALIZABLE
}
```

如果我们在服务器启动时想改变事务的默认隔离级别，可以修改启动参数`transaction-isolation`的值，比方说我们在启动服务器时指定了`--transaction-isolation=SERIALIZABLE`，那么事务的默认隔离级别就从原来的`REPEATABLE READ`变成了`SERIALIZABLE`

想要查看当前会话默认的隔离级别可以通过查看系统变量`transaction_isolation`的值来确定：

```mysql
mysql> SHOW VARIABLES LIKE 'transaction_isolation';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.02 sec)
```

## MVCC原理

**版本链** *****

`InnoDB`存储引擎的表来说，它的聚簇索引记录中都包含两个必要的隐藏列（`row_id`并不是必要的，我们创建的表中有主键或者非NULL的UNIQUE键时都不会包含`row_id`列）：

- `trx_id`：每次一个事务对某条聚簇索引记录进行改动时，都会把该事务的`事务id`赋值给`trx_id`隐藏列。
- `roll_pointer`：每次对某条聚簇索引记录进行改动时，都会把旧的版本写入到`undo日志`中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

hero表插入一条数据

![](http://img.tomato530.com/mvcc1.jpg)

假设之后两个`事务id`分别为`100`、`200`的事务对这条记录进行`UPDATE`操作，操作流程如下：

![](http://img.tomato530.com/mvcc2.jpg)

每次对记录进行改动，都会记录一条`undo日志`，每条`undo日志`也都有一个`roll_pointer`属性（`INSERT`操作对应的`undo日志`没有该属性，因为该记录并没有更早的版本），可以将这些`undo日志`都连起来，串成一个链表，所以现在的情况就像下图一样：

![](http://img.tomato530.com/mvcc4.jpg)

对该记录每次更新后，都会将旧值放到一条`undo日志`中，就算是该记录的一个旧版本，随着更新次数的增多，所有的版本都会被`roll_pointer`属性连接成一个链表，我们把这个链表称之为`版本链`，版本链的头节点就是当前记录最新的值。另外，每个版本中还包含生成该版本时对应的`事务id`。

**ReadView**

对于使用`READ UNCOMMITTED`隔离级别的事务来说，由于可以读到未提交事务修改过的记录，所以直接读取记录的最新版本就好了

对于使用`SERIALIZABLE`隔离级别的事务来说，设计`InnoDB`的大叔规定使用加锁的方式来访问记录（加锁是啥我们后续文章中说哈）

对于使用`READ COMMITTED`和`REPEATABLE READ`隔离级别的事务来说，都必须保证读到已经提交了的事务修改过的记录，也就是说假如另一个事务已经修改了记录但是尚未提交，是不能直接读取最新版本的记录的，核心问题就是：需要判断一下版本链中的哪个版本是当前事务可见的。为此，设计`InnoDB`的大叔提出了一个`ReadView`的概念，这个`ReadView`中主要包含4个比较重要的内容：

- `m_ids`：表示在生成`ReadView`时当前系统中活跃的读写事务的`事务id`列表。
- `min_trx_id`：表示在生成`ReadView`时当前系统中活跃的读写事务中最小的`事务id`，也就是`m_ids`中的最小值。
- `max_trx_id`：表示生成`ReadView`时系统中应该分配给下一个事务的`id`值。注意max_trx_id并不是m_ids中的最大值，事务id是递增分配的。比方说现在有id为1，2，3这三个事务，之后id为3的事务提交了。那么一个新的读事务在生成ReadView时，m_ids就包括1和2，min_trx_id的值就是1，max_trx_id的值就是4。

+ `creator_trx_id`：表示生成该`ReadView`的事务的`事务id`。只有在对表中的记录做改动时（执行INSERT、DELETE、UPDATE这些语句时）才会为事务分配事务id，否则在一个只读事务中的事务id值都默认为0。

有了这个`ReadView`，这样在访问某条记录时，只需要按照下边的步骤判断记录的某个版本是否可见：

- 如果被访问版本的`trx_id`属性值与`ReadView`中的`creator_trx_id`值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
- 如果被访问版本的`trx_id`属性值小于`ReadView`中的`min_trx_id`值，表明生成该版本的事务在当前事务生成`ReadView`前已经提交，所以该版本可以被当前事务访问。
- 如果被访问版本的`trx_id`属性值大于或等于`ReadView`中的`max_trx_id`值，表明生成该版本的事务在当前事务生成`ReadView`后才开启，所以该版本不可以被当前事务访问。
- 如果被访问版本的`trx_id`属性值在`ReadView`的`min_trx_id`和`max_trx_id`之间，那就需要判断一下`trx_id`属性值是不是在`m_ids`列表中，如果在，说明创建`ReadView`时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建`ReadView`时生成该版本的事务已经被提交，该版本可以被访问。

**在`MySQL`中，`READ COMMITTED`和`REPEATABLE READ`隔离级别的的一个非常大的区别就是它们生成ReadView的时机不同**

**事务执行过程中，只有在第一次真正修改记录时（比如使用INSERT、DELETE、UPDATE语句），才会被分配一个单独的事务id，这个事务id是递增的**

**READ COMMITTED —— 每次读取数据前都生成一个ReadView**