# redis 操作

### 1.开启远程访问和设置密码

```sh
#打开配置文件
#在vim编辑模式下，输入行数+gg可以快捷跳行。例如跳到第138行，输入：138gg
vim redis.conf

#是否开启后台运行
daemonize no 修改为daemonize yes

#开启远程访问
1.将68行 `bind 127.0.0.1`注释 
2.将87行 `protected-mode` 改为 no 

#设置密码
将790行 `requirepass` 注释解开并设置密码


#首先查询到redis的pid后，kill掉,然后重启
#ps: process status 将某个进程显示出来
#-ef: -e 显示所有进程。-f 全格式
#grep:Global Regular Expression Print 表示全局正则表达式版本，它的使用权限是所有用户
[root@localhost bin]# ps -ef|grep redis
root      20940      1  0 12:12 ?        00:00:18 ./redis-server *:6379 
[root@localhost bin]# kill 20940
[root@localhost bin]# ./redis-server redis.conf 

#远程连接
./redis-cli -h [Host] -p [Port] -a [Pass]
```

