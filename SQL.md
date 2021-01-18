# SQL

## 读写分离

​	主从同步：

​		

## 数据库拆分

###### 	垂直拆分

​		是指按功能模块拆分。比如分为订单库、商品库、用户库...这种方式多个数据库之间的表结构不同。如系统可以分为，订单系统，商品管理系统，用户管理系统业务系统比较明的，垂直拆分能很好的起到分散数据库压力的作用。

缺点：

1. 无法在数据库内做表关联
2. 单表大数据依然存在性能瓶颈
3. 程序端比较复杂，需要知道具体使用哪个库，对于开发人员工作量增加
4. 事务比较复杂

###### 水平拆分（推荐使用）

​	可以按照单表的业务逻辑进行表的拆分，比如一张日志表，可以按照记录的实体类型进行拆分(或者创建时间)，在数据不是非常大的情况下，对于开发人员还是很友好的

​	优点：可部分迁移

​	缺点：数据分布不均，如：卡片的历史记录有1000W，而版本的历史记录只有100W

​	

###### PARTITION BY RANGE

​	MYSQL 5.5以上版本 PARTITION BY RANGE支持按范围分区的，对于开发人员的开发并无太大影响，直接select from employees;就好了，并不需要在程序中过多处理

> ```sql
> CREATE TABLE employees (
>     id INT NOT NULL,
>     fname VARCHAR(30),
>     lname VARCHAR(30),
>     hired DATE NOT NULL DEFAULT '1970-01-01',
>     separated DATE NOT NULL DEFAULT '9999-12-31',
>     job_code INT NOT NULL,
>     store_id INT NOT NULL
> )
> PARTITION BY RANGE (store_id) (
>     PARTITION p0 VALUES LESS THAN (6),
>     PARTITION p1 VALUES LESS THAN (11),
>     PARTITION p2 VALUES LESS THAN (16),
>     PARTITION p3 VALUES LESS THAN (21)
> );
> 
> alter table employees add index ix_store_id(store_id) ;
> alter table employees add index ix_job_code(job_code) ;
> 
> insert into employees(id,job_code,store_id) values(1,1001,1),(2,1002,2),(3,1003,3),(4,1004,4);
> ```
>
> 

###### MySQL 子分区

创建表

```
CREATE TABLE tb_sub (id INT, purchased DATE)
    PARTITION BY RANGE( YEAR(purchased) )
    SUBPARTITION BY HASH( TO_DAYS(purchased) )
    SUBPARTITIONS 2 (
        PARTITION p0 VALUES LESS THAN (1990),
        PARTITION p1 VALUES LESS THAN (2000),
        PARTITION p2 VALUES LESS THAN MAXVALUE
    );
```

## Sharding-JDBC

###### 简介

​	轻量级Java框架，在Java的JDBC层提供的额外服务。 它使用客户端直连数据库，以jar包形式提供服务，无需额外部署和依赖，可理解为增强版的JDBC驱动，完全兼容JDBC和各种ORM框架。在单库单表的情况下，会有稍微的性能问题，但是在多库多表的情况下，会有极大的性能提升。但是数据库之间的主从同步是需要数据库取支持的。

- ​	适用于任何基于Java的ORM框架，如：JPA, Hibernate, Mybatis, Spring JDBC Template或直接使用JDBC。

- ​    基于任何第三方的数据库连接池，如：DBCP, C3P0, BoneCP, Druid, HikariCP等。
- ​    支持任意实现JDBC规范的数据库。目前支持MySQL，Oracle，SQLServer和PostgreSQL。

