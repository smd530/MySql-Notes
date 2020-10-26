# Redo日志

`InnoDB`存储引擎是以页为单位来管理存储空间的，我们进行的增删改查操作其实本质上都是在访问页面（包括读页面、写页面、创建新页面等操作）

真正访问页面之前，需要把在磁盘上的页缓存到内存中的`Buffer Pool`之后才可以访问

如何保证持久性：

在事务提交完成之前把该事务所修改的所有页面都刷新到磁盘 但是这种做法有问题：

+ 刷新一个完整的数据页太浪费了
+ 随机IO刷起来比较慢

***

**我们只是想让已经提交了的事务对数据库中数据所做的修改永久生效，即使后来系统崩溃，在重启后也能把这种修改恢复出来 所以我们其实没有必要在每次事务提交时就把该事务在内存中修改过的全部页面刷新到磁盘，只需要把修改了哪些东西记录一下就好**

这样我们在事务提交时，把修改的内容刷新到磁盘中，即使之后系统崩溃了，重启之后只要按照上述内容所记录的步骤重新更新一下数据页，那么该事务对数据库中所做的修改又可以被恢复出来，也就意味着满足`持久性`的要求。因为在系统崩溃重启时需要按照上述内容所记录的步骤重新更新数据页，所以上述内容也被称之为`重做日志`，英文名为`redo log`，我们也可以土洋结合，称之为`redo日志`。与在事务提交时将所有修改过的内存中的页面刷新到磁盘中相比，只将该事务执行过程中产生的`redo`日志刷新到磁盘的好处如下：

+ `redo`日志占用的空间非常小 存储表空间ID、页号、偏移量以及需要更新的值所需的存储空间是很小的，关于`redo`日志的格式我们稍后会详细唠叨，现在只要知道一条`redo`日志占用的空间不是很大就好了。

+ `redo`日志是顺序写入磁盘的

  在执行事务的过程中，每执行一条语句，就可能产生若干条`redo`日志，这些日志是按照产生的顺序写入磁盘的，也就是使用顺序IO。

## redo日志格式

**`redo`日志本质上只是记录了一下事务对数据库做了哪些修改**

![](http://img.tomato530.com/Redo1.jpg)各个部分的详细释义如下：

- `type`：该条`redo`日志的类型。

  在`MySQL 5.7.21`这个版本中，设计`InnoDB`的大叔一共为`redo`日志设计了53种不同的类型

- `space ID`：表空间ID。

- `page number`：页号。

- `data`：该条`redo`日志的具体内容。

## 简单的redo log类型

`MLOG_WRITE_STRING`类型的`redo`日志表示写入一串数据，但是因为不能确定写入的具体数据占用多少字节，所以需要在日志结构中添加一个`len`字段：

![](http://img.tomato530.com/Redo2.jpg)

## 复杂的redo log类型

有时候执行一条语句会修改非常多的页面，包括系统数据页面和用户数据页面（用户数据指的就是聚簇索引和二级索引对应的`B+`树）

+ 表中包含多少个索引，一条`INSERT`语句就可能更新多少棵`B+`树。
+ 针对某一棵`B+`树来说，既可能更新叶子节点页面，也可能更新内节点页面，也可能创建新的页面（在该记录插入的叶子节点的剩余空间比较少，不足以存放该记录时，会进行页面的分裂，在内节点页面中添加`目录项记录`）。

**在语句执行过程中，`INSERT`语句对所有页面的修改都得保存到`redo`日志中去**

![](http://img.tomato530.com/Redo3.jpg)

把一条记录插入到一个页面时需要更改的地方非常多。这时我们如果使用上边介绍的简单的物理`redo`日志来记录这些修改时，可以有两种解决方案：

+ 方案一：在每个修改的地方都记录一条`redo`日志。

  也就是如上图所示，有多少个加粗的块，就写多少条物理`redo`日志。这样子记录`redo`日志的缺点是显而易见的，因为被修改的地方是在太多了，可能记录的`redo`日志占用的空间都比整个页面占用的空间都多了～

+ 方案二：将整个页面的`第一个被修改的字节`到`最后一个修改的字节`之间所有的数据当成是一条物理`redo`日志中的具体数据

复杂的redo日志类型：

- `MLOG_REC_INSERT`（对应的十进制数字为`9`）：表示插入一条使用非紧凑行格式的记录时的`redo`日志类型。
- `MLOG_COMP_REC_INSERT`（对应的十进制数字为`38`）：表示插入一条使用紧凑行格式的记录时的`redo`日志类型。

等.....

这些类型的`redo`日志既包含`物理`层面的意思，也包含`逻辑`层面的意思，具体指：

- 物理层面看，这些日志都指明了对哪个表空间的哪个页进行了修改。
- 逻辑层面看，在系统崩溃重启时，并不能直接根据这些日志里的记载，将页面内的某个偏移量处恢复成某个数据，而是需要调用一些事先准备好的函数，执行完这些函数后才可以将页面恢复成系统崩溃前的样子。

MLOG_COMP_REC_INSERT

![](http://img.tomato530.com/Redo4.jpg)

**总结：redo日志会把事务在执行过程中对数据库所做的所有修改都记录下来，在之后系统崩溃重启后可以把事务所做的任何修改都恢复出来。**

## Mini-Transaction

**以组的形式写入redo日志**

在执行语句的过程中产生的`redo`日志被设计`InnoDB`的大叔人为的划分成了若干个不可分割的组，比如：

- 更新`Max Row ID`属性时产生的`redo`日志是不可分割的。
- 向聚簇索引对应`B+`树的页面中插入一条记录时产生的`redo`日志是不可分割的。
- 向某个二级索引对应`B+`树的页面中插入一条记录时产生的`redo`日志是不可分割的。
- 还有其他的一些对页面的访问操作时产生的`redo`日志是不可分割的。

设计`InnoDB`的大叔们认为**向某个索引对应的`B+`树中插入一条记录的这个过程必须是原子的**，不能说插了一半之后就停止了

**设计`MySQL`的大叔把对底层页面中的一次原子访问的过程称之为一个`Mini-Transaction`，简称`mtr`**

向某个索引对应的`B+`树中插入一条记录的过程也算是一个`Mini-Transaction`

一个所谓的`mtr`可以包含一组`redo`日志，在进行崩溃恢复时这一组`redo`日志作为一个不可分割的整体。

一个事务可以包含若干条语句，每一条语句其实是由若干个`mtr`组成，每一个`mtr`又可以包含若干条`redo`日志，画个图表示它们的关系就是这样：

![](http://img.tomato530.com/Redo6.jpg)

## Redo日志的写入过程

**redo log block**

设计`InnoDB`的大叔为了更好的进行系统崩溃恢复，他们把通过`mtr`生成的`redo`日志都放在了大小为`512字节`的`页`中 这里把用来存储`redo`日志的页称为`block`

![](http://img.tomato530.com/Redo7.jpg)

真正的`redo`日志都是存储到占用`496`字节大小的`log block body`中，图中的`log block header`和`log block trailer`存储的是一些管理信息。我们来看看这些所谓的`管理信息`都是啥：

![](http://img.tomato530.com/Redo8.jpg)

`LOG_BLOCK_CHECKPOINT_NO`：表示所谓的`checkpoint`的序号

### Redo日志缓冲区

写入`redo`日志时也不能直接直接写到磁盘上，实际上在服务器启动时就向操作系统申请了一大片称之为`redo log buffer`的连续内存空间，翻译成中文就是`redo日志缓冲区`，我们也可以简称为`log buffer`。这片内存空间被划分成若干个连续的`redo log block`

![](http://img.tomato530.com/Redo10.jpg)

innodb_log_buffer_size = 16m

### Redo日志写入log buffer

顺序写入 一个block一个block写入

每个`mtr`都会产生一组`redo`日志，用示意图来描述一下这些`mtr`产生的日志情况：

![](http://img.tomato530.com/Redo11.jpg)

![](http://img.tomato530.com/Redo12.jpg)

## redo日志刷盘时机

`mtr`运行过程中产生的一组`redo`日志在`mtr`结束时会被复制到`log buffer`中，可是这些日志总在内存里呆着也不是个办法，在一些情况下它们会被刷新到磁盘里，比如：

- `log buffer`空间不足时

   log buffer`的大小是有限的（通过系统变量`innodb_log_buffer_size`指定），如果不停的往这个有限大小的`log buffer`里塞入日志，很快它就会被填满。设计`InnoDB`的大叔认为如果当前写入`log buffer`的`redo`日志量已经占满了`log buffer`总容量的大约一半左右，就需要把这些日志刷新到磁盘上。

- 事务提交时

  我们前边说过之所以使用`redo`日志主要是因为它占用的空间少，还是顺序写，在事务提交时可以不把修改过的`Buffer Pool`页面刷新到磁盘，但是为了保证持久性，必须要把修改这些页面对应的`redo`日志刷新到磁盘。

+ 后台线程不停的刷刷刷

  后台有一个线程，大约每秒都会刷新一次`log buffer`中的`redo`日志到磁盘。

+ 正常关闭服务器时

+ 做所谓的`checkpoint`时

## redo日志文件组

- `innodb_log_group_home_dir`

  该参数指定了`redo`日志文件所在的目录，默认值就是当前的数据目录。

- `innodb_log_file_size`

  该参数指定了每个`redo`日志文件的大小，在`MySQL 5.7.21`这个版本中的默认值为`48MB`，

- `innodb_log_files_in_group`

  该参数指定`redo`日志文件的个数，默认值为2，最大值为100。

磁盘上的`redo`日志文件不只一个，而是以一个`日志文件组`的形式出现

![](http://img.tomato530.com/Redo13.jpg)

总共的`redo`日志文件大小其实就是：`innodb_log_file_size × innodb_log_files_in_group`。

## redo日志文件格式

将log buffer中的redo日志刷新到磁盘的本质就是把block的镜像写入日志文件中

![](http://img.tomato530.com/Redo14.jpg)

![](http://img.tomato530.com/Redo15.jpg)

![](http://img.tomato530.com/Redo16.jpg)

**`checkpoint1`：记录关于`checkpoint`的一些属性**，看一下它的结构：

![](http://img.tomato530.com/Redo18.jpg)

各个属性的具体释义如下：

|            属性名             | 长度（单位：字节） | 描述                                                         |
| :---------------------------: | :----------------: | :----------------------------------------------------------- |
|      `LOG_CHECKPOINT_NO`      |        `8`         | 服务器做`checkpoint`的编号，每做一次`checkpoint`，该值就加1。 |
|     `LOG_CHECKPOINT_LSN`      |        `8`         | 服务器做`checkpoint`结束时对应的`LSN`值，系统崩溃恢复时将从该值开始。 |
|    `LOG_CHECKPOINT_OFFSET`    |        `8`         | 上个属性中的`LSN`值在`redo`日志文件组中的偏移量              |
| `LOG_CHECKPOINT_LOG_BUF_SIZE` |        `8`         | 服务器在做`checkpoint`操作时对应的`log buffer`的大小         |
|     `LOG_BLOCK_CHECKSUM`      |        `4`         | 本block的校验值，所有block都有，我们不关心                   |

## Log Sequence Number

日志序列号：LSN

初始值：8704

`lsn`增长的量就是该`mtr`生成的`redo`日志占用的字节数

**每一组由mtr生成的redo日志都有一个唯一的LSN值与其对应，LSN值越小，说明redo日志产生的越早**

## flushed_to_disk_lsn

`redo`日志是首先写到`log buffer`中，之后才会被刷新到磁盘上的`redo`日志文件。所以设计`InnoDB`的大叔提出了一个称之为`buf_next_to_write`的全局变量，标记当前`log buffer`中已经有哪些日志被刷新到磁盘中了。画个图表示就是这样：

![](http://img.tomato530.com/Redo19.jpg)

`lsn`是表示当前系统中写入的`redo`日志量，这包括了写到`log buffer`而没有刷新到磁盘的日志

刷新到磁盘中的`redo`日志量的全局变量，称之为`flushed_to_disk_lsn`

**当有新的`redo`日志写入到`log buffer`时，首先`lsn`的值会增长，但`flushed_to_disk_lsn`不变，随后随着不断有`log buffer`中的日志被刷新到磁盘上，`flushed_to_disk_lsn`的值也跟着增长。如果两者的值相同时，说明log buffer中的所有redo日志都已经刷新到磁盘中了。**

## lsn和redo日志文件偏移量的对应关系

`lsn`的值是代表系统写入的`redo`日志量的一个总和，一个`mtr`中产生多少日志，`lsn`的值就增加多少（当然有时候要加上`log block header`和`log block trailer`的大小），这样`mtr`产生的日志写到磁盘中时，很容易计算某一个`lsn`值在`redo`日志文件组中的偏移量，如图：

![](http://img.tomato530.com/Redo20.jpg)

## flush链表中的lsn

我们知道一个`mtr`代表一次对底层页面的原子访问，在访问过程中可能会产生一组不可分割的`redo`日志，在`mtr`结束时，会把这一组`redo`日志写入到`log buffer`中。除此之外，在`mtr`结束时还有一件非常重要的事情要做，就是把在mtr执行过程中可能修改过的页面加入到Buffer Pool的flush链表

**flush链表中的脏页按照修改发生的时间顺序进行排序**

## checkpoint 重点重点

`redo`日志文件组容量是有限的，我们不得不选择循环使用`redo`日志文件组中的文件，但是这会造成最后写的`redo`日志与最开始写的`redo`日志`追尾`，这时应该想到：redo日志只是为了系统崩溃后恢复脏页用的，如果对应的脏页已经刷新到了磁盘，也就是说即使现在系统崩溃，那么在重启后也用不着使用redo日志恢复该页面了，所以该redo日志也就没有存在的必要了，那么它占用的磁盘空间就可以被后续的redo日志所重用。也就是说：判断某些redo日志占用的磁盘空间是否可以覆盖的依据就是它对应的脏页是否已经刷新到磁盘里

![](http://img.tomato530.com/Redo21.jpg)

**全局变量`checkpoint_lsn`来代表当前系统中可以被覆盖的`redo`日志总量是多少，这个变量初始值也是`8704`。**

比方说现在`页a`被刷新到了磁盘，`mtr_1`生成的`redo`日志就可以被覆盖了，所以我们可以进行一个增加`checkpoint_lsn`的操作，我们把这个过程称之为做一次`checkpoint`。做一次`checkpoint`其实可以分为两个步骤：

+ 步骤一：计算一下当前系统中可以被覆盖的`redo`日志对应的`lsn`值最大是多少。

  `redo`日志可以被覆盖，意味着它对应的脏页被刷到了磁盘，只要我们计算出当前系统中被最早修改的脏页对应的`oldest_modification`值，那凡是在系统lsn值小于该节点的oldest_modification值时产生的redo日志都是可以被覆盖掉的，我们就把该脏页的`oldest_modification`赋值给`checkpoint_lsn`。比方说当前系统中`页a`已经被刷新到磁盘，那么`flush链表`的尾节点就是`页c`，该节点就是当前系统中最早修改的脏页了，它的`oldest_modification`值为8916，我们就把8916赋值给`checkpoint_lsn`（也就是说在redo日志对应的lsn值小于8916时就可以被覆盖掉）。

+ 步骤二：将`checkpoint_lsn`和对应的`redo`日志文件组偏移量以及此次`checkpint`的编号写到日志文件的管理信息（就是`checkpoint1`或者`checkpoint2`）中。

  设计`InnoDB`的大叔维护了一个目前系统做了多少次`checkpoint`的变量`checkpoint_no`，每做一次`checkpoint`，该变量的值就加1。我们前边说过计算一个`lsn`值对应的`redo`日志文件组偏移量是很容易的，所以可以计算得到该`checkpoint_lsn`在`redo`日志文件组中对应的偏移量`checkpoint_offset`，然后把这三个值都写到`redo`日志文件组的管理信息中。

  我们说过，每一个`redo`日志文件都有`2048`个字节的管理信息，但是上述关于checkpoint的信息只会被写到日志文件组的第一个日志文件的管理信息中。不过我们是存储到`checkpoint1`中还是`checkpoint2`中呢？设计`InnoDB`的大叔规定，当`checkpoint_no`的值是偶数时，就写到`checkpoint1`中，是奇数时，就写到`checkpoint2`中。

记录完`checkpoint`的信息之后，`redo`日志文件组中各个`lsn`值的关系就像这样：

![](http://img.tomato530.com/Redo22.jpg)

## 查看系统中各种lsn值

我们可以使用`SHOW ENGINE INNODB STATUS`命令查看当前`InnoDB`存储引擎中的各种`LSN`值的情况，比如：

```mysql
mysql> SHOW ENGINE INNODB STATUS\G

(...省略前边的许多状态)
LOG
---
Log sequence number 124476971
Log flushed up to   124099769
Pages flushed up to 124052503
Last checkpoint at  124052494
0 pending log flushes, 0 pending chkp writes
24 log i/o's done, 2.00 log i/o's/second
----------------------
(...省略后边的许多状态)
```

其中：

- `Log sequence number`：代表系统中的`lsn`值，也就是当前系统已经写入的`redo`日志量，包括写入`log buffer`中的日志。
- `Log flushed up to`：代表`flushed_to_disk_lsn`的值，也就是当前系统已经写入磁盘的`redo`日志量。
- `Pages flushed up to`：代表`flush链表`中被最早修改的那个页面对应的`oldest_modification`属性值。
- `Last checkpoint at`：当前系统的`checkpoint_lsn`值。

## 崩溃恢复

### 确定恢复的起点

`checkpoint_lsn`之前的`redo`日志都可以被覆盖，也就是说这些`redo`日志对应的脏页都已经被刷新到磁盘中了，既然它们已经被刷盘，我们就没必要恢复它们了。对于`checkpoint_lsn`之后的`redo`日志，它们对应的脏页可能没被刷盘，也可能被刷盘了，我们不能确定，所以需要从`checkpoint_lsn`开始读取`redo`日志来恢复页面。

`redo`日志文件组的第一个文件的管理信息中有两个block都存储了`checkpoint_lsn`的信息，我们当然是要选取最近发生的那次checkpoint的信息。衡量`checkpoint`发生时间早晚的信息就是所谓的`checkpoint_no`，我们只要把`checkpoint1`和`checkpoint2`这两个block中的`checkpoint_no`值读出来比一下大小，哪个的`checkpoint_no`值更大，说明哪个block存储的就是最近的一次`checkpoint`信息。这样我们就能拿到最近发生的`checkpoint`对应的`checkpoint_lsn`值以及它在`redo`日志文件组中的偏移量`checkpoint_offset`。

### 确定恢复的终点

普通block的`log block header`部分有一个称之为`LOG_BLOCK_HDR_DATA_LEN`的属性，该属性值记录了当前block里使用了多少字节的空间。对于被填满的block来说，该值永远为`512`。如果该属性的值不为`512`，那么就是它了，它就是此次崩溃恢复中需要扫描的最后一个block。

### 怎么恢复

![](http://img.tomato530.com/Redo25.jpg)

`redo 0`在`checkpoint_lsn`后边，恢复时可以不管它。我们现在可以按照`redo`日志的顺序依次扫描`checkpoint_lsn`之后的各条redo日志，按照日志中记载的内容将对应的页面恢复出来。这样没什么问题，不过设计`InnoDB`的大叔还是想了一些办法加快这个恢复的过程：

+ 使用哈希表
+ 跳过已经刷新到磁盘的页面

***

**先写redo log buffer 然后写redo 再写 buffer pool 再写磁盘**