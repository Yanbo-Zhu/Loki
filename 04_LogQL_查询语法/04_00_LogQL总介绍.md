
> Loki 使用一种称为 LogQL 的语法来进行日志检索，语法类似 PromQL

LogQL: Log Query Language

Everything starts with a log stream pipeline that collects logs from various sources and stores them into a logstream storage solution like Loki.


Loki comes with its own PromQL-inspired language for queries called _LogQL_. LogQL can be considered a distributed `grep` that aggregates log sources. LogQL uses labels and operators for filtering.

There are two types of LogQL queries:
- _Log queries_ return the contents of log lines.
- _Metric queries_ extend log queries and calculate sample values based on the content of logs from a log query.

受PromQL的启发，Loki也有自己的LogQL查询语句。根据官方的说法，它就像一个`分布式的grep日志聚合查看器`。和PromeQL一样，LogQL也是使用标签和运算符进行过滤，它主要分为两个部分：

- log stream selector （日志流选择器）
- filter expression （过滤器表达式）

[![](https://raw.githubusercontent.com/wsgzao/storage-public/master/img/20201103003623.png)](https://raw.githubusercontent.com/wsgzao/storage-public/master/img/20201103003623.png)

我们用这两部分就可以在Loki中组合出我们想要的功能，通常情况下我们可以拿来做如下功能

- 根据日志流选择器查看日志内容
- 通过过滤规则在日志流中计算相关的度量指标


LogQL is composed of a {Stream selector} and a log pipeline where each step of your pipeline is separated by a vertical bar - |.

![](image/Pasted%20image%2020240827170619.png)

