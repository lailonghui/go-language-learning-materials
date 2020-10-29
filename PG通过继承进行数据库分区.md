# PG通过继承进行数据库分区 

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

### 4.向不分区主表 partition_test_all 中插入一亿条数据

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
    generate_series(1,100000000,1);  
```

### 5.插入同样的测试数据到分区主表 partition_test

```sql
INSERT INTO partition_test SELECT * FROM partition_test_all;  
```

### 6.partition_test查询2020-10-1日所有数据。

```sql

 select  * from partition_test_all where date_key = date '2020-10-1'
 result: 2 secs 431 msec
 
```

### 7.partition_test_all查询2020-10-1日所有数据。

```sql
select  * from partition_test_all where date_key = date '2020-10-1'  
result: 5 secs 258 msec。
```

