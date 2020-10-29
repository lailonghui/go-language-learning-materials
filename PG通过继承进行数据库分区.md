# PG数据库分区实现方式一 

##### 原理: 通过INHERITS(继承)的方式，在父表上挂载触发器，在找不到子表时动态创建表，目前根据时间分区，一天分一张表。

### 1.创建分区主表 partition_test

```sql
CREATE TABLE partition_test  
(  
    date_key date,  
    hour_key smallint,  
    client_key integer,  
    item_key integer,  
    account integer,  
    expense numeric  
)  

```

### 2.创建触发函数和触发器

```sql
--创建触发器函数
CREATE OR REPLACE FUNCTION partition_test_trigger()  
RETURNS TRIGGER AS $$  
DECLARE date_text TEXT;  
DECLARE insert_statement TEXT;  
BEGIN  
    SELECT to_char(NEW.date_key, 'YYYY_MM_DD') INTO date_text;  
    insert_statement := 'INSERT INTO partition_test_'  
        || date_text  
        ||' VALUES ($1.*)';  
    EXECUTE insert_statement USING NEW;  
    RETURN NULL;  
    EXCEPTION  
    WHEN UNDEFINED_TABLE  
    THEN  
        EXECUTE  
            'CREATE TABLE IF NOT EXISTS partition_test_'  
            || date_text  
            || '(CHECK (date_key = '''  
            || date_text  
            || ''')) INHERITS (partition_test)';  
        RAISE NOTICE 'CREATE NON-EXISTANT TABLE partition_test_%', date_text;  
        EXECUTE  
            'CREATE INDEX partition_test_date_key_'  
            || date_text  
            || ' ON partition_test_'  
            || date_text  
            || '(date_key)';  
        EXECUTE insert_statement USING NEW;  
    RETURN NULL;  
END;  
$$  
LANGUAGE plpgsql;  

--挂载分区Trigger  
CREATE TRIGGER insert_partition_test_trigger  
BEFORE INSERT ON partition_test  
FOR EACH ROW EXECUTE PROCEDURE partition_test_trigger(); 
```

### 3.创建不分区主表 partition_test_all

```sql
CREATE TABLE partition_test_all  
(  
    date_key date,  
    hour_key smallint,  
    client_key integer,  
    item_key integer,  
    account integer,  
    expense numeric  
); 
```

### 4.向不分区主表 partition_test_all 中插入2亿条数据

```sql
INSERT INTO partition_test_all  
select  
    (select  
        array_agg(i::date)  
     from generate_series('2020-10-01'::date, '2020-10-31'::date, '1 day'::interval) as t(i)  
    )[floor(random()*31)+1] as date_key,  
    floor(random()*24) as hour_key,  
    floor(random()*1000000)+1 as client_key,  
    floor(random()*100000)+1 as item_key,  
    floor(random()*20)+1 as account,  
    floor(random()*10000)+1 as expense  
from  
    generate_series(1,200000000,1);  
```

### 5.插入同样的测试数据到分区主表 partition_test

```sql
INSERT INTO partition_test SELECT * FROM partition_test_all;  
```

### 6.partition_test查询2020-10-1日**不同client_key的平均消费额** 。

```sql
explain analyze
select
    avg(expense)  
from  
    (select  
        client_key,  
        sum(expense) as expense  
    from  
        partition_test
    where  
        date_key = date '2020-10-1'  
    group by 1  
    )foo;
 Execution Time: 4919.409 ms
 
```

### 7.partition_test_all查询2020-10-1日**不同client_key的平均消费额** 。

```sql
explain analyze
select
    avg(expense)  
from  
    (select  
        client_key,  
        sum(expense) as expense  
    from  
        partition_test_all
    where  
        date_key = date '2020-10-1'  
    group by 1  
    )foo; 
Execution Time: 44579.715 ms
```

