---
title: SQL基本语句
date: 2020-10-05
tags: SQL, database
---


### 建表

```sql
CREATE TABLE `tbl_file`(
    `id` INT(11) NOT NULL AUTO_INCREMENT,
    PRIMARY KEY(`id`),
    UNIQUE KEY `idx_user_file` (`user_name`, `file_sha1`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

- `PRIMARY KEY(<name>)`
- `UNIQUE KEY <name> (<name1...>)`表示name1...中所有字段是唯一的，命名为name
    * `INSERT IGNORE`时，要插入与unique相同的自动将会跳过
- `KEY <name> (<name>)`
- `SHOW CREATE TALBE <table_name>`
    * 显示键表过程


### 插入

`INSERT [INGORE] INTO <表名> <字段名...> VALUES <字段对应值>`

- `IGNORE`表示如果要插入的字段已存在，则忽略(不执行插入)


### 删除

- 软删除`DELETE`
- 硬删除`DROP`


### 更新

#### UPDATE语句

`UPDATE <table_name> SET <column>=<value>... WHERE <column>=<value>`

 **注意WHERE限定范围** ，否则所有记录都将被更新


#### REPALCE函数

- 插入替换
    * 似乎如果碰到unique key就会替换
    * `REPLACE INTO <表名> (<字段...>) VALUES <新字段,...>`
- replace()函数
    * `REPALCE(<object>, <search>, <replace>)`
    * 将内容`<object>`中`<search>`到的替换为`<repalce>`
    * 实例
        + `SELECT REPLACE('http://','http','https')`
        + `UPDATE <table_name> SET name=replace(name,'a','b')`
            + 将表中name字段中所有a替换成b



### 查询

`SELECT <字段...> FROM [数据库].<表> [WHERE statement] [LIMIT statement]`


### 修改删除字段

`ALTER TABLE <表名> <操作...>`，如：

- 新增, `ALTER TABLE <表名> ADD COlUMN <字段名> [定义...]`
- 删除字段，`ALTER TABLE <表名> DROP <字段名>`，如：
    ```
    ALTER TABLE tbl_user_file DROP INDEX `idx_user_file`;
    ```
- 修改字段，`ALTER TABLE <表名> CHANGE <旧字段名> <新字段名> <新数据类型>;`
    ```sql
    ALTER TABLE tb_emp1 CHANGE col1 col3 CHAR(30);
    ```
- 修改字段类型，`ALTER TABLE <表名> MODIFY <字段名> <数据类型>`



