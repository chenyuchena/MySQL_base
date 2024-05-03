<h1 align = "center">MySQL学习</h1>

## 1 基础篇

1. DDL

2. DML

3. DQL

4. DCL

## 2 进阶篇

### 2.1 触发器

触发器是与表有关的数据库对象，指在insert/update/delete之前或之后，触发并执行触发器中定义的SQL语句的集合。该特性可协助应用在数据库端确保数据的完整性，日志记录，数据校验等操作。

OLD 和 NEW 引用触发器中发生变化的纪录内容。目前只支持行级触发，不支持语句级触发（只触发一次）。

#### 2.1.1 语法

- 创建

  ```
  creat triggen trigger_name
  
  before/after insert/update/delete
  
  on tbl_name for each row #行级触发器
  
  beign
  	trigger_stmt
  end
  ```

- 查看

  ```
  show triggers;
  ```

- 删除

  ```
  drop trigger [schema_name]trigger_name; #如果没有指定schema_name，默认当前数据库。
  ```

#### 2.1.2 案例

- 创建日志表

  ```
  create table user_logs(
    id int(11) not null auto_increment,
    operation varchar(20) not null comment '操作类型, insert/update/delete',
    operate_time datetime not null comment '操作时间',
    operate_id int(11) not null comment '操作的ID',
    operate_params varchar(500) comment '操作参数',
    primary key(`id`)
  )engine=innodb default charset=utf8;
  ```

- 插入数据触发器

  ```
  create trigger tb_user_insert_trigger
      after insert on tb_user for each row 
  begin 
      insert into user_logs(id, operation, operate_time, operate_id, operate_params) VALUES (null, 'insert', 
      now(),NEW.id, concat('插入数据的内容为：id=',NEW.id,'name=',NEW.name,...))
  end;
  ```

- 修改数据触发器

  ```
  create trigger tb_user_updata_trigger
      after updata on tb_user for each row 
  begin 
      insert into user_logs(id, operation, operate_time, operate_id, operate_params) VALUES (null, 'updata', 
      now(),NEW.id, concat('更新之前的数据：id=',old.id,'name=',old.name,...,'更新之后的数据：id=',NEW.id,'name=',NEW.name,...))
  end;
  ```

- 删除数据触发器

  ```
  create trigger tb_user_delete_trigger
      after delete on tb_user for each row 
  begin 
      insert into user_logs(id, operation, operate_time, operate_id, operate_params) VALUES (null, 'delete', 
      now(),'删除的数据：id=',old.id,'name=',old.name,...))
  end;
  ```

### 2.2 锁

计算机协调多个进程或线程并发访问某一资源的机制。

#### 2.2.1 全局锁

整个数据库中的表，加锁后，DDL、DML用不了。

```
flush tables with read lock ;

mysqldump -u'用户' -p'密码' database1 > database.sql #数据备份(不是MySQL中的命令语句)

unlock tables;
```

在InnoDB引擎中，可在备份时加上参数  –single-transaction 参数来完成不加锁的一致性数据备份。

```
mysqldump --single-transaction -u'用户' -p'密码' database1 > database.sql
```

#### 2.2.2 表级锁

每次锁住整张表。锁定粒度大，发生锁冲突的概率最高，并发度最低。

- 表锁

  ```
  lock tables 'table_name' read #表共享读锁,客户端都只能读，不能写
  
  lock tables 'table_name' write #表独占写锁,当前客端能读能写，其他不能
  
  unlock tables #解锁 
  ```

- 元数据锁（meta data lock, MDL）

  系统自动控制，无需显式使用。主要作用是维护表元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。避免DML与DDL冲突，保证读写的正确性。（开启事务后，对table进行select等操作时，事务未提交，不能进行DDL操作，处于阻塞状态）

- 意向锁

  为了避免DML在执行时，加的行锁与表锁的冲突。使得表锁不用检查每行数据是否加锁，使用意向锁来减少表锁的检查。

  - 意向共享锁（IS）：与表锁（read）兼容，与write互斥。

    ```
    select...lock in share mode
    ```

  - 意向排他锁（IX）：与表锁read和write都互斥。意向锁之间不互斥。

    ```
    insert、updata、delete、select...for updata
    ```

  ```
  #查看意向锁及行锁的加锁情况
  
  select object_schema,object_name,index_name,lock_type.lock_data from performance_schema.data_locks;
  ```

#### 2.2.3 行级锁

每次锁住对应的行数据。锁定粒度小，发生锁冲突的概率最低，并发度最高。应用在InnoDB存储引擎中。通过对索引上的索引项加锁来实现。

- 行锁（Record Lock）:锁定单个行纪录。（READ COMMITED）RC不可重复读、（REPEATABLE READ）RR可重复读隔离级别支持。

  - 共享锁（S）：本身兼容，其他互斥。
  - 排他锁（X）：都互斥。

  若不通过索引条件检索数据，此时行锁升级为表锁。

- 间隙锁（Gap Lock）：锁定间隙，防止其他事务在这个间隙进行insert，产生幻读。RR隔离级别支持。可共存。
  1. 索引上的等值查询（唯一索引），给不存在的纪录加锁时，优化为间隙锁。
  2. 索引上的等值查询（普通索引），向右遍历时最后一个值不满足查询需求时，Next-Key Lock退化为间隙锁。
  3. 索引上的范围查询（唯一索引），会访问到不满足条件的第一个值为止。
- 临建锁（Next-Key Lock）：行锁和间隙锁组合。RR隔离级别支持。

### 2.3 InnoDB引擎

#### 2.3.1 逻辑存储结构

表空间（bid文件），一个MySQL实例可以对应多个表空间，用于储存记录、索引等数据。

段，分为数据段（Leaf node segment）B+数叶子节点、索引段（Non-leaf segment）B+数非叶子节点、回滚段（Rollback segment），用来管理区，InnoDB是索引组织表。

区，表空间的单元结构，每个区大小为1M。一个区共64页。

页，InnoDB存储引擎磁盘管理的最小单元，每个页默认16KB。为了保证页的连续性，InnoDB存储引擎每次从磁盘申请4-5个区。

行，InnoDB存储引擎数据是按行存放的。

![1](E:\study\java学习\MySQL\MySQL_base\img\1.png)

#### 2.3.2 架构

1. 内存架构

   - Buffer Pool：缓存磁盘上经常操作的数据，加快数据处理速度。以Page为单位。

   - Change Pool：更改缓冲区（针对非唯一二级索引）。在执行DML语句时，若数据没在缓冲区，不会直接操作磁盘，而是将数据变更存在更改缓冲区，在未来数据被读取时，在将数据合并恢复到缓冲区中，最后将合并的数据刷新到磁盘。

   - Adaptive Hash Index：自适应hash索引，用于优化对Buffer Pool数据的查询。

   - Log Buffer：日志缓冲区，用来保存要写入到磁盘中的log日志数据，默认大小16M，里面的日志会定期刷新到磁盘。节省磁盘I/O。

     参数：

     ​	innodb_log_buffer_size：缓冲区大小

     ​	innodb_flush_log_at_trx_commit：日志刷新到磁盘时机（1： 日志在每次事务提交时写入并刷新到磁盘；0：每秒将日志写入并刷新到磁盘一   	次；2：日志在每次事务提交后写入，并每秒刷新到磁盘一次。 ）

2. 磁盘结构

   - System Tablespace：系统表空间，更改缓冲区的存储区域。

     参数：innodb_data_file_path

   - File-Per-Table Tablespaces：每个表的文件表空间包含单个InnoDB表的数据和索引，并存储在文件系统上的单个数据文件中。

     参数：innodb_file_per_table

   - General Tablespace：通用表空间，需要创建，创建时可指定该表空间。

     ```
     create tablespace xxx add datafile 'file_name' engine = engine_name;
     
     create table xxx engine = engine_name tablespace  xxx;
     ```

   - Undo Tablespace：撤销表空间，自动创建两个默认undo表空间，16M，储存undo log日志。

   - Temporary Tablespace：会话和全局临时表空间，存储用户创建的临时表等数据。

   - Doublewrite Buffer Files：双写缓冲区，数据从Buffer Pool刷新到磁盘前，先写入双写缓冲区文件中，便于系统异常时恢复数据。

   - Redo Log：重做日志用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲(redo log buffer)以及重做日志文件(redo log)，前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都会存到该日志中，用于在刷新脏页到磁盘时,发生错误时，进行数据恢复使用。

3. 后台线程

   -  Master Thread

     核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中，保持数据的一致性，还包括脏页的刷新、合并插入缓存、undo页的回收。

   -  IO Thread
     在InnoDB存储引擎中大量使用了AIO来处理IO请求,这样可以极大地提高数据库的性能，而IO Thread主要负责这些IO请求的回调。

     ![2](E:\study\java学习\MySQL\MySQL_base\img\2.png)

   - Purge Thread
     主要用于回收事务已经提交了的undo log，在事务提交之后，undo log可能不用了，就用它来回收。
   - Page Cleaner Thread
     协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻塞。

#### 2.3.3 事务原理

- 事务
  事务 是一组操作的集合，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败。

- 特性

  - 原子性(Atomicity)：事务是不可分割的最小操作单元，要么全部成功，要么全部失败。
  - 一致性(Consistency)：事务完成时，必须使所有的数据都保持一致状态。
  - 隔离性(Isolation)：数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环境下运行。

  - 持久性(Durabiity)：事务一旦提交或回滚，它对数据库中的数据的改变就是永久的。

原子性、一致性、持久性由InnoDB中的redo log 和undo log保证，隔离性由锁和MVCC（多版本并发控制）保证。

- redo log（持久性）

  重做日志，记录的是事务提交时数据页的物理修改，是用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲(redolog buffer)以及重做日志文件(redolog file)，前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都存到该日志文件中，用于在刷新脏页到磁盘，发生错误时，进行数据恢复使用。

- undo log（原子性）

  回滚日志，用于记录数据被修改前的信息，作用包含两个：提供回滚 和 MVCC(多版本并发控制)。undo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。
  Undo log销毁：undo log在事务执行时产生，事务提交时，并不会立即删除undolog，因为这些日志可能还用于MVCC。 
  Undo log存储：undo log采用段的方式进行管理和记录，存放在前面介绍的 rollback segment 回滚段中，内部包含1024个undo log segment。

- MVCC-基本概念

  - 当前读
    读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。对于我们日常的操作，如:select ... lock in share mode(共享锁)，select ... for update、update、insert、delete(排他锁)都是一种当前读。
  - 快照读
    简单的select(不加锁)就是快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据，不加锁，是非阻塞读。
    Read committed：每次select，都生成一个快照读。
    Repeatable Read：开启事务后第一个select语句才是快照读的地方。
    Serializable：快照读会退化为当前读。
  - MVCC
    全称 Multi-Version Concurrency Control，多版本并发控制。指维护一个数据的多个版本，使得读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读功能。MVCC的具体实现，还需要依赖于数据库记录中的三个隐式字段、undoloq日志、readView。

- MVCC-实现原理

  - 记录中的隐藏字段

    ![3](E:\study\java学习\MySQL\MySQL_base\img\3.png)

  - undo log 版本链

    ![4](E:\study\java学习\MySQL\MySQL_base\img\4.png)

  - readview

    ![5](E:\study\java学习\MySQL\MySQL_base\img\5.png)

    不同的隔离级别，生成Readview的时机不同：
    READ COMMITTED：在事务中每一次执行快照读时生成Readview。
    REPEATABLE READ：仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView。

    ![6](E:\study\java学习\MySQL\MySQL_base\img\6.png)![7](E:\study\java学习\MySQL\MySQL_base\img\7.png)

### 2.4 MySQL管理

#### 2.4.1 系统数据库

Mysql数据库安装完成后，自带了一下四个数据库，具体作用如下:

![8](E:\study\java学习\MySQL\MySQL_base\img\8.png)

#### 2.4.2 常用工具

- mysql

  该mysql不是指mysql服务，而是指mysql的客户端工具。

  语法：

  ```
  mysql [options] [database]
  ```

  选项：

  ```
  -u,--user=name			#指定用户名
  -p,--password[=name]	#指定密码
  -h,--host=name,A		#指定服务器IP或域名
  -P,--port=port			#指定连接端口
  -e,--execute=name		#执行SQL语句并退出
  ```

  -e选项可以在Mysql客户端执行SQL语句，而不用连接到MySQL数据库再执行，对于一些批处理脚本，这种方式尤其方便。

  ```
  mysgl -uroot -p123456 db01 -e "select * from stu";
  ```

- mysqladmin

  mysqladmin 是一个执行管理操作的客户端程序。可以用它来检查服务器的配置和当前状态、创建并删除数据库等。

  示例：

  ```
  mysqladmin -uroot -p123456 drop 'test01';
  mysqladmin -uroot -p123456 version;
  ```

- mysqlbinlog

  由于服务器生成的二进制日志文件以二进制格式保存，所以如果想要检查这些文本的文本格式，就会使用到mysqlbinlog 日志管理工具。

  语法：

  ```
  mysqlbinlog [options] log-files1 log-files2 ...
  ```

  选项：

  ```
  -d,--database=name							#指定数据库名称，只列出指定的数据库相关操作。
  -o,--offset=#								#忽略掉日志中的前n行命令。
  -r.--result-file=name						#将输出的文本格式日志输出到指定文件。
  -s,--short-form								#显示简单格式，省略掉一些信息。
  -start-datatime=datel --stop-datetime=date2	#指定日期间隔内的所有日志。
  -start-position=p0s1 --stop-position=p0s2	#指定位置间隔内的所有日志。
  ```

- mysqlshow

  mysqlshow 客户端对象查找工具，用来很快地查找存在哪些数据库、数据库中的表、表中的列或者索引。

  语法：

  ```
  mysqlshow [options][db_name [table_name [col_name]]
  ```

  选项：

  ```
  --count		#显示数据库及表的统计信息（数据库，表均可以不指定)
  -i			#显示指定数据库或者指定表的状态信息
  ```

  示例：

  ```
  mysqlshow -uroot -p2143 --count				#查询每个数据库的表的数量及表中记录的数量
  mysqlshow -uroot -p2143 test --count		#查询test库中每个表中的字段数，及行数
  mysqlshow -uroot -p2143 test book --count	#查询test库中book表的详细情况
  ```

- mysqldump

  mysqldump客户端工具用来备份数据库或在不同数据库之间进行数据迁移。备份内容包含创建表，及插入表的SQL语句。

  语法：

  ```
  rmysqldurnp [options] db_name [tables]
  rmysqldump[options] --database/-B db1 [db2 db3...]
  mysqldump [options] --all-databases/-A
  ```

  连接选项：

  ```
  -u,--user=name			#指定用户名
  -p,--password[=name]	#指定密码
  -h, --host=name			#指定服务器ip或域名
  -P:--port=#				#指定连接端口
  ```

  输出选项：

  ```
  --add-drop-database		#在每个数据库创建语句前加上 drop database 语句
  --add-drop-table		#在每个表创建语句前加上 drop table 语句,默认开启;不开启(--skip-add-drop-table)
  -n,--no-create-db		#不包含数据库的创建语句
  -t,--no-create-info		#不包含数据表的创建语句
  -d --no-data			#不包含数据
  -T--tab=name			#自动生成两个文件:一个.sql文件，创建表结构的语句;一个.txt文件，数据文件
  ```

- mysqlimport/source

  mysqlimport是客户端数据导入工具，用来导入mysqldump加-T参数后导出的文本文件。

  语法：

  ```
  mysqlimport [options] db. name textfile1 etxtile2...
  ```

  示例：

  ```
  mysqlimport -uroot -p2143 test /tmp/city.txt
  ```

  如果需要导入sql文件，可以使用mysql中的source指令：

  ```
  source /root/xxx.sql
  ```

## 3 运维篇

### 3.1 日志





