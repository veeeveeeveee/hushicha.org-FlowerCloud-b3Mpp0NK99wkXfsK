
# PostgreSQL中将对象oid转为对象名


使用pg的内部数据类型将对象oid转为对象名，可以简化一些系统视图的关联查询。


## 数据库类型转换对应类型的oid


可以用以下数据库类型转换对应类型的oid（以pg12为例）



```
postgres=# select typname from pg_type where typname ~ '^reg';
    typname
---------------
 regclass
 regconfig
 regdictionary
 regnamespace
 regoper
 regoperator
 regproc
 regprocedure
 regrole
 regtype
(10 rows)

```

## 对应关系




| 对象名称 | 类型 | 转换规则 |
| --- | --- | --- |
| pg\_class | regclass | pg\_class.oid::regclass |
| pg\_ts\_dict | regdictionary | pg\_ts\_dict.oid::regdictionary |
| pg\_namespace | regnamespace | pg\_namespace.oid::regnamespace |
| pg\_operator | regoperator | pg\_operator.oid::regoperator |
| pg\_proc | regproc | pg\_proc.oid::regproc |
| pg\_rolespg\_user | regrole | pg\_roles.oid::regrolepg\_user.usesysid::regrole |
| pg\_type | regtype | pg\_type.oid::regtype |
|  | 以下几个类型暂不确定用途，待研究： |  |
|  | regprocedure |  |
|  | regoper |  |
|  | regconfig |  |


## 创建测试数据



```
psql -U postgres
create user test password 'test';
create database testdb with owner=test;
\c testdb
CREATE SCHEMA AUTHORIZATION test;
psql -U test -d testdb
create table test_t1(id int);
create table test_t2(id int);
create table test_t3(id int);

```

基于如上测试数据，查询test模式下有哪些表，以及表的owner


传统表关联的方式使用以下SQL，关联pg\_class、pg\_namespace、pg\_roles/pg\_user



```
psql -U test -d testdb
-- 查询用户关联pg_user查询
SELECT
  t3.nspname AS SCHEMA,
  t1.relname AS tablename,
  t2.usename AS OWNER 
FROM
  pg_class t1
  JOIN pg_user t2 ON t1.relowner = t2.usesysid
  JOIN pg_namespace t3 ON t1.relnamespace = t3.OID 
WHERE
  t1.relkind = 'r' 
  AND t3.nspname = 'test';

 schema | tablename | owner
--------+-----------+-------
 test   | test_t1   | test
 test   | test_t2   | test
 test   | test_t3   | test
(3 rows)

-- 查询用户关联pg_roles查询
SELECT
  t3.nspname AS SCHEMA,
  t1.relname AS tablename,
  t2.rolname AS OWNER 
FROM
  pg_class t1
  JOIN pg_roles t2 ON t1.relowner = t2.oid
  JOIN pg_namespace t3 ON t1.relnamespace = t3.OID 
WHERE
  t1.relkind = 'r' 
  AND t3.nspname = 'test';

 schema | tablename | owner
--------+-----------+-------
 test   | test_t1   | test
 test   | test_t2   | test
 test   | test_t3   | test
(3 rows)

```

如上为了实现查询效果需要关联三张表，查询比较繁琐，如果使用对象转换就很简单了，如下：



```
psql -U test -d testdb
SELECT
  relnamespace :: REGNAMESPACE AS SCHEMA,
  relname AS tablename,
  relowner :: REGROLE AS OWNER 
FROM
  pg_class 
WHERE
  relnamespace :: REGNAMESPACE :: TEXT = 'test' 
  AND relkind = 'r';

 schema | tablename | owner
--------+-----------+-------
 test   | test_t1   | test
 test   | test_t2   | test
 test   | test_t3   | test
(3 rows)

```

# 将对象名转为oid类型


## 转换关系




| 对象类型 | 转换规则 |
| --- | --- |
| table | '表名'::regclass::oid |
| function/procedure | '函数名/存储过程名'::regproc::oid |
| schema | '模式名'::regnamespace::oid |
| user/role | '用户名/角色名'::regrole::oid |
| type | '类型名称'::regtype::oid |


## 测试示例


表转换



```
drop table if exists test_t;
create table test_t(id int);

postgres=# select oid from pg_class where relname = 'test_t';
  oid
-------
 16508
(1 row)

postgres=# select 'test_t'::regclass::oid;
  oid
-------
 16508
(1 row)

```

函数转换



```
CREATE OR REPLACE FUNCTION test_fun(
    arg1 INTEGER,
    arg2 INTEGER,
    arg3 TEXT
)
RETURNS INTEGER
AS $$
BEGIN
    RETURN arg1 + arg2;
END;
$$ LANGUAGE plpgsql;


postgres=# select oid,proname from pg_proc where proname = 'test_fun';
  oid  | proname
-------+----------
 16399 | test_fun
(1 row)

postgres=# select 'test_fun'::regproc::oid;
  oid
-------
 16399
(1 row)

```

模式转换



```
create schema test_schema;

postgres=# select oid,nspname from pg_namespace where nspname='test_schema';
  oid  |   nspname
-------+-------------
 16511 | test_schema
(1 row)

postgres=# select 'test_schema'::regnamespace::oid;
  oid
-------
 16511
(1 row)

```

用户/角色



```
create user test_user;

postgres=# select usesysid,usename from pg_user where usename='test_user';
 usesysid |  usename
----------+-----------
    16512 | test_user
(1 row)

postgres=# select 'test_user'::regrole::oid;
  oid
-------
 16512
(1 row)

```

类型



```
CREATE TYPE type_sex AS ENUM ('male', 'female');

postgres=# select oid,typname from pg_type where typname='type_sex';
  oid  | typname
-------+----------
 16514 | type_sex
(1 row)

postgres=# select 'type_sex'::regtype::oid;
  oid
-------
 16514
(1 row)

```

 本博客参考[蓝猫机场](https://fenfang.org)。转载请注明出处！
