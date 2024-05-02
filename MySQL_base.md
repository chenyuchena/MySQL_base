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
  
  on tbl_name for each row --行级触发器
  
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
  drop trigger [schema_name]trigger_name; --如果没有指定schema_name，默认当前数据库。
  ```

#### 2.1.2 案例

- 创建日志标

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

#### 2.2.1 分类

- 全局锁：整个数据库中的表，加锁后，DDL、DML用不了。

  ```
  flush tables with read lock ;
  
  mysqldump -u'用户' -p'密码' database1 > database.sql (不是MySQL中的命令语句)
  
  unlock tables;
  ```

  在InnoDB引擎中，可在备份时加上参数  –single-transaction 参数来完成不加锁的一致性数据备份。

  ```
  mysqldump --single-transaction -u'用户' -p'密码' database1 > database.sql
  ```

- 表级锁：每次锁住整张表。锁定粒度大，发生锁冲突的概率最高，并发度最低。

  - 表锁

    ```
    表共享读锁 lock tables 'table_name' read #客户端都只能读，不能写
    
    表独占写锁 lock tables 'table_name' write #当前客端能读能写，其他不能
    
    解锁 unlock tables
    ```

  - 元数据锁（meta data lock, MDL）

    系统自动控制，无需显式使用。主要作用是维护表元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。避免DML与DDL冲突，保证读写的正确性。（开启事务后，对table进行select等操作时，事务未提交，不能进行DDL操作，处于阻塞状态）

  - 意向锁

    为了避免DML在执行时，加的行锁与表锁的冲突。使得表锁不用检查每行数据是否加锁，使用意向锁来减少表锁的检查。

    - 意向共享锁（IS）：

      ```
      select...lock in share mode
      ```

    - 意向排他锁（IX）：

      ```
      insert、updata、delete、select...for updata
      ```

- 行级锁

