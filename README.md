# go-language-learning-materials
learning golang,postgres and more



```json
github使用文档:
https://docs.github.com/cn/free-pro-team@latest/github

PostgreSQL阿里课程： 
https://edu.aliyun.com/lesson_52_1755spm=5176.10731542.0.0.148a3b1840OD2B#_1755

go语言实战: pdf

go语言圣经:
https://docs.hacknode.org/gopl-zh/

Go语言高级编程:
https://books.studygolang.com/advanced-go-programming-book/

Learn Go with tests：
https://studygolang.gitbook.io/learn-go-with-tests/

gqlgen:
https://gqlgen.com/

postgis:空间数据库
https://postgis.net/

gin全局统一错误处理：https://zhuanlan.zhihu.com/p/76967528
输入表单校验:https://github.com/go-playground/validator
go使用日志：https://github.com/uber-go/zap

postgres查看数据库各个表占用的磁盘空间:
SELECT
    table_schema || '.' || table_name AS table_full_name,
    pg_size_pretty(pg_total_relation_size('"' || table_schema || '"."' || table_name || '"')) AS size
FROM information_schema.tables
ORDER BY
    pg_total_relation_size('"' || table_schema || '"."' || table_name || '"') DESC

```

