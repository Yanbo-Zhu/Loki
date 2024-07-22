
![](image/v2-8241c492086474a8e105fb8cd5cc0c68_720w.webp)

# 1 选择器_LogStreamSelector

对于查询表达式的标签部分，将其包装在花括号中`{}`，然后使用键值对的语法来选择标签，多个标签表达式用逗号分隔，比如：

```
{app="mysql",name="mysql-backup"}
```

目前支持以下标签匹配运算符：

- `=`等于
- `!=`不相等
- `=~`正则表达式匹配
- `!~`不匹配正则表达式

比如：
```
{name=~"mysql.+"}
{name!~"mysql.+"}
```

```
{app=”mysql”,name=~”mysql-backup.+”}

=: exactly equal.
!=: not equal.
=~: regex matches.
!~: regex does not match.

```

适用于`Prometheus`标签选择器规则同样也适用于`Loki`日志流选择器。




# 2 过滤器_FilterExpression

编写日志流选择器后，您可以通过编写搜索表达式来进一步过滤结果。搜索表达式可以只是文本或正则表达式。

对于查询表达式的标签部分，将其包装在花括号中 {}，然后使用键值对的语法来选择标签，多个标签表达式用逗号分隔，比如：
```
{instance=~"kafka-[23]",name="kafka"} != "kafka.server:type=ReplicaManager"
```

> |=：日志行包含字符串  
> !=：日志行不包含字符串  
> |~：日志行匹配正则表达式  
> !~：日志行与正则表达式不匹配

```
1 # 精确匹配：|="2020-11-16 "
2 {app_kubernetes_io_instance="admin-service-test2-container-provider"}|="2020-11-16 "

1 # 模糊匹配：|~"2020-11-16 "
2 {app_kubernetes_io_instance="admin-service-test2-container-provider"}|~"2020-11-16 "

1 # 排除过滤：!=/!~ "数据中心"
2 {app_kubernetes_io_instance="admin-service-master-container-provider"}!="数据中心"
3 {app_kubernetes_io_instance="admin-service-master-container-provider"}!~"数据中心"

1 # 正则匹配： |~ "()"
2 {app_kubernetes_io_instance="admin-service-master-container-provider"}!~"(admin|web)"
3 {app_kubernetes_io_instance="admin-service-master-container-provider"}|~"ERROR|error"

{job="mysql"} |= "error"
{name="kafka"} |~ "tsdb-ops.*io:2003"
{instance=~"kafka-[23]",name="kafka"} != kafka.server:type=ReplicaManager

过滤器运算符可以被链接，并将顺序过滤表达式-结果日志行将满足每个过滤器。例如：
{job="mysql"} |= "error" != "timeout"
```


举例，我需要查询包含关键字packages

```
{job="varlogs"}  |= "packages"
```

效果如下：

[![](https://raw.githubusercontent.com/wsgzao/storage-public/master/img/20201103160550.png)](https://raw.githubusercontent.com/wsgzao/storage-public/master/img/20201103160550.png)

# 3 Metric Queries

这个其实就跟prometheus中的很想像了.
rate：计算每秒的条目数

```
rate({job="mysql"} |= "error" != "timeout" [5m])
```

- `rate`: calculates the number of entries per second
- `count_over_time`: counts the entries for each log stream within the given range.
- `bytes_rate`: calculates the number of bytes per second for each stream.
- `bytes_over_time`: counts the amount of bytes used by each log stream for a given range.


```
4 # 12h小时内出现错误的速率
5 rate({app_kubernetes_io_instance=~".*master-container.*"} |~ "ERROR|error" [12h])

rate(({job="mysql"} |= "error" != "timeout")[10s])
This query gets the per-second rate of all non-timeout errors within the last ten seconds for the MySQL job.
```



# 4 范围查询_count_over_time


count_over_time：计算给定范围内每个日志流的条目

```
# 三十分钟日志行记录
count_over_time({app_kubernetes_io_instance="admin-service-master-container-web"}[30m]) 


count_over_time({job="mysql"}[5m])
This query counts all the log lines within the last five minutes for the MySQL job.
```





# 5 Aggrestion运算

与vPromQL 一样，LogQL 支持内置聚合运算符的一个子集，可用于聚合单个向量的元素，从而产生具有更少元素但具有集合值的新向量：

```
sum：计算标签上的总和
min：选择最少的标签
max：选择标签上方的最大值
avg：计算标签上的平均值
stddev：计算标签上的总体标准差
stdvar：计算标签上的总体标准方差
count：计算向量中元素的数量
bottomk：通过样本值选择最小的k个元素
topk：通过样本值选择最大的k个元素



sum: Calculate sum over labels
min: Select minimum over labels
max: Select maximum over labels
avg: Calculate the average over labels
stddev: Calculate the population standard deviation over labels
stdvar: Calculate the population standard variance over labels
count: Count number of elements in the vector
bottomk: Select smallest k elements by sample value
topk: Select largest k elements by sample value


```


```
# 统计1个小时日志量最大的前10个服务 
topk(10,sum(rate({app_kubernetes_io_instance=~".*master-container.*"}[60m])) by(container))
 
# 统计最近6小时内错误日志计数
sum(count_over_time({app_kubernetes_io_instance=~".*master-container.*"}|~"ERROR"[6h])) by (container)

sum(count_over_time({job="mysql"}[5m])) by (level)
Get the count of logs during the last five minutes, grouping by level.

avg(rate(({job="nginx"} |= "GET")[10s])) by (region)
```



