<h1 align = "center">MySQL学习</h1>

## 1、基础篇

1. DDL

   

2. DML

3. DQL

4. DCL

## 2、进阶篇

### 2.1、触发器介绍

触发器是与表有关的数据库对象，指在insert/update/delete之前或之后，触发并执行触发器中定义的SQL语句的集合。该特性可协助应用在数据库端确保数据的完整性，日志记录，数据校验等操作。

OLD 和 NEW 引用触发器中发生变化的纪录内容。目前只支持行级触发，不支持语句级触发（只触发一次）。

### 2.2、语法

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

案例：

```
#创建日志标
create table user_logs(
  id int(11) not null auto_increment,
  operation varchar(20) not null comment '操作类型, insert/update/delete',
  operate_time datetime not null comment '操作时间',
  operate_id int(11) not null comment '操作的ID',
  operate_params varchar(500) comment '操作参数',
  primary key(`id`)
)engine=innodb default charset=utf8;
```

```
#插入数据触发器
create trigger tb_user_insert_trigger
    after insert on tb_user for each row 
begin 
    insert into user_logs(id, operation, operate_time, operate_id, operate_params) VALUES (null, 'insert', 
    now(),NEW.id, concat('插入数据的内容为：id=',NEW.id,'name=',NEW.name,...))
end;
```

```
#修改数据触发器
create trigger tb_user_updata_trigger
    after updata on tb_user for each row 
begin 
    insert into user_logs(id, operation, operate_time, operate_id, operate_params) VALUES (null, 'updata', 
    now(),NEW.id, concat('更新之前的数据：id=',old.id,'name=',old.name,...,'更新之后的数据：id=',NEW.id,'name=',NEW.name,...))
end;
```

```
#删除数据触发器
create trigger tb_user_delete_trigger
    after delete on tb_user for each row 
begin 
    ins into user_logs(id, operation, operate_time, operate_id, operate_params) VALUES (null, 'delete', 
    now(),'删除的数据：id=',old.id,'name=',old.name,...))
end;
```

