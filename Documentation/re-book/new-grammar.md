## TenDB Cluster新增语法

TSpider新增了一些语法，下面将介绍如何使用

### kill thread

实现三种kill threads的方式:  
1). kill thread_id safe(kill 9 safe)  
与直接kill thread_id不同的是，只有当该thread id是sleep状态且非事务中，才会被kill,  
否则报错： ER_THREAD_IN_USING_OR_TRANSACTION , "thread %lu is in using or in transaction, can't be killed safe"  
2). kill threads all  
用来安全的kill当前threads。 sleep状态且非事务中的threads会被kill掉，其它thread则等待当前query执行完成或者事务结束才被kill  
3). kill threads all force  
用来kill当前所有运行的thread  

增加一个会话级状态: 'Max_thread_id_on_kill'， 表示在执行kill threads all (force)时，当前最大的thread id  
show status like 'Max_thread_id_on_kill';


示例SQL：
```
MariaDB [tendb_test]>  kill threads all force;
Query OK, 1 row affected (0.00 sec)

MariaDB [tendb_test]> show status like 'Max_thread_id_on_kill';
+-----------------------+--------+
| Variable_name         | Value  |
+-----------------------+--------+
| Max_thread_id_on_kill | 171010 |
+-----------------------+--------+
1 row in set (0.00 sec)
```

### 新增语法 flush  table with write lock
<a id="write_lock"></a>
对于分布式场景下的TSpider集群主备切换, `flush tables with read lock`已经不能保证数据的一致性了，为此新增语法`flush tables with write lock`,来获取事务级的锁，保证分布式场景下的主备安全切换。

>1.执行 flush  table with write lock 后，会申请事务级锁，阻塞新的事务开启。  
2.在已有事务commit后，加锁成功,此时修改mysql.servers路由信息，并执行flush privileges使路由生效。  
3.由unlock tables释放事务锁。解锁后，新的请求可以进入TSpider，此时TSpider侧的新连接会使用新路由规则来请求后端存储TenDB，切换完成。

<font color="#dd0000">注意:</font>   
>该特性仅在spider_internal_xa=ON时生效。


### 跨分片扫描信息查询
引入percona的QUERY_RESPONSE_TIME来统计请求时间在某个区间的query个数及总时间开销，另外TSpider额外统计了跨分区操作的响应时间。   
跨分区时间视图依赖增加的P_COUNT，P_TOTAL两个字段，其中P_COUNT表示跨分区请求的个数，P_TOTAL表示跨分区请求花费的时间。   
> 跨分区操作是指TSpider接受到的请求，需要分发到多于1个存储shard上请求才能返回结果      

采用如下方式可以查看响应时间视图：
```
MariaDB [mysql]> show QUERY_RESPONSE_TIME;
+----------------+-------+-----------------------+---------+-----------------------+
| TIME           | COUNT | TOTAL                 | P_COUNT | P_TOTAL               |
+----------------+-------+-----------------------+---------+-----------------------+
|       0.000015 | 70865 |       0.126465        |       0 |       0.000000        |
|       0.000030 |  7352 |       0.202521        |       0 |       0.000000        |
|       0.000061 | 86036 |       4.340680        |       0 |       0.000000        |
|       0.000122 | 45729 |       3.788752        |       0 |       0.000000        |
|       0.000244 |   342 |       0.047792        |       0 |       0.000000        |
|       0.000488 |    42 |       0.016067        |       6 |       0.002419        |
|       0.000976 |    71 |       0.042179        |       0 |       0.000000        |
|       0.001953 |     5 |       0.008613        |       0 |       0.000000        |
|       0.003906 |    19 |       0.051909        |       0 |       0.000000        |
|       0.007812 |     1 |       0.006720        |       0 |       0.000000        |
|       0.015625 |     6 |       0.075120        |       1 |       0.014913        |
|       0.031250 |     3 |       0.074285        |       1 |       0.022601        |
|       0.062500 |     6 |       0.244472        |       0 |       0.000000        |
.............
| TOO LONG       |     0 | Since 200316 14:57:02 |       0 | Since 200316 14:57:02 |
+----------------+-------+-----------------------+---------+-----------------------+
```
也可以通过INFORMATION_SCHEMA里面的表QUERY_RESPONSE_TIME查看:
```
MariaDB [mysql]> SELECT * FROM information_schema.QUERY_RESPONSE_TIME;
+----------------+-------+-----------------------+---------+-----------------------+
| TIME           | COUNT | TOTAL                 | P_COUNT | P_TOTAL               |
+----------------+-------+-----------------------+---------+-----------------------+
|       0.000015 | 70865 |       0.126465        |       0 |       0.000000        |
|       0.000030 |  7352 |       0.202521        |       0 |       0.000000        |
|       0.000061 | 86036 |       4.340680        |       0 |       0.000000        |
|       0.000122 | 45729 |       3.788752        |       0 |       0.000000        |
|       0.000244 |   342 |       0.047792        |       0 |       0.000000        |
|       0.000488 |    42 |       0.016067        |       6 |       0.002419        |
|       0.000976 |    71 |       0.042179        |       0 |       0.000000        |
|       0.001953 |     5 |       0.008613        |       0 |       0.000000        |
|       0.003906 |    19 |       0.051909        |       0 |       0.000000        |
|       0.007812 |     1 |       0.006720        |       0 |       0.000000        |
|       0.015625 |     6 |       0.075120        |       1 |       0.014913        |
|       0.031250 |     3 |       0.074285        |       1 |       0.022601        |
|       0.062500 |     6 |       0.244472        |       0 |       0.000000        |
.............
| TOO LONG       |     0 | Since 200316 14:57:02 |       0 | Since 200316 14:57:02 |
+----------------+-------+-----------------------+---------+-----------------------+
```
这里可以看到，响应时间介于0.000244秒和0.000488秒之间的query有42个，总响应时间为0.016067秒；跨分区SQL有6个，总响应时间为0.002419秒。