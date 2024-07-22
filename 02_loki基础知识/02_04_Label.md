
标签是键值对，可以定义为任何内容！我们喜欢将它们称为元数据，用于描述日志流。如果您熟悉Prometheus，您可能已经见过一些标签，比如`job`和`instance`，在接下来的示例中我将使用这些标签。

我们在Grafana Loki中提供的抓取配置也定义了这些标签。如果您正在使用Prometheus，Loki和Prometheus之间具有一致的标签是Loki的一项超能力，使得将应用程序指标与日志数据进行关联变得非常容易。

# 1 Loki如何使用标签

Loki中的标签扮演着非常重要的角色：它们定义了一个日志流。更具体地说，每个标签键和值的组合定义了一个日志流。如果仅有一个标签值发生变化，就会创建一个新的日志流。

如果您熟悉Prometheus，那里使用的术语是系列（series）；然而，Prometheus还有一个额外的维度：指标名称（metric name）。Loki 简化了这一点，它没有指标名称，只有标签，而我们决定使用日志流（stream）而不是系列（series）来表示。


> 重点的单词: Stream


# 2 格式

Loki 对标签名称的命名有与Prometheus相同的限制：

它可以包含ASCII字母和数字，以及下划线和冒号。它必须符合正则表达式`[a-zA-Z_:][a-zA-Z0-9_:]*`。

注意：冒号保留给用户定义的记录规则。导出器exporters或直接仪表化的过程不应使用冒号。


# 3 Loki 标签示例

以下一系列示例将说明 Loki 中标签的基本用法和概念。

我们以一个例子为例：

```
scrape_configs:

 - job_name: system

   pipeline_stages:

   static_configs:

   - targets:

      - localhost

     labels:

      job: syslog

      __path__: /var/log/syslog
```

这个配置将追踪一个文件并分配一个标签：job=syslog。您可以按照以下方式进行查询：
```
{job="syslog"}
```

这将在 Loki 中创建一个日志流。


------------


现在让我们稍微扩展一下这个例子：

```
scrape_configs:

 - job_name: system

   pipeline_stages:

   static_configs:

   - targets:

      - localhost

     labels:

      job: syslog

      __path__: /var/log/syslog

 - job_name: apache

   pipeline_stages:

   static_configs:

   - targets:

      - localhost

     labels:

      job: apache

      __path__: /var/log/apache.log
```


现在我们将追踪两个文件。每个文件只有一个标签和一个值，因此 Loki 现在将存储两个日志流。

我们可以以几种方式查询这些日志流：

```
{job="apache"} - 显示标签 job 为 apache 的日志
{job="syslog"} - 显示标签 job 为 syslog 的日志
{job=~"apache|syslog"} - 显示标签 job 为 apache 或 syslog 的日志
```


----


在最后一个示例中，我们使用了正则表达式标签匹配器来记录使用具有两个值的job标签的日志流。现在考虑如何使用额外的标签：
```
scrape_configs:

 - job_name: system

   pipeline_stages:

   static_configs:

   - targets:

      - localhost

     labels:

      job: syslog

      env: dev

      __path__: /var/log/syslog

 - job_name: apache

   pipeline_stages:

   static_configs:

   - targets:

      - localhost

     labels:

      job: apache

      env: dev

      __path__: /var/log/apache.log

```

```
{env="dev"} - 将返回所有具有 env=dev 的日志，在这种情况下，包括两个日志流。
```


希望现在您开始看到标签的威力了。通过使用单个标签，您可以查询多个日志流。通过组合多个不同的标签，您可以创建非常灵活的日志查询。
标签是 Loki 日志数据的索引。它们用于查找独立存储为块的压缩日志内容。每个标签和值的唯一组合定义了一个日志流，而该日志流的日志被批量处理、压缩并以块形式存储。
为了让 Loki 高效且具有成本效益，我们必须负责地使用标签。接下来的部分将更详细地探讨这个问题。


# 4 Cardinality（基数）

前面的两个示例使用静态定义的具有单个值的标签；然而，还有一些方法可以动态定义标签。让我们来看一个使用Apache日志和一个可以用来解析这样一个日志行的庞大正则表达式的示例：

```
11.11.11.11 - frank [25/Jan/2000:14:00:01 -0500] "GET /1986.js HTTP/1.1" 200 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"
```


```
- job_name: system

   pipeline_stages:

      - regex:

        expression: "^(?P<ip>\\S+) (?P<identd>\\S+) (?P<user>\\S+) \\[(?P<timestamp>[\\w:/]+\\s[+\\-]\\d{4})\\] \"(?P<action>\\S+)\\s?(?P<path>\\S+)?\\s?(?P<protocol>\\S+)?\" (?P<status_code>\\d{3}|-) (?P<size>\\d+|-)\\s?\"?(?P<referer>[^\"]*)\"?\\s?\"?(?P<useragent>[^\"]*)?\"?$"

    - labels:

        action:

        status_code:

   static_configs:

   - targets:

      - localhost

     labels:

      job: apache

      env: dev

      __path__: /var/log/apache.log
```


这个正则表达式匹配日志行的每个组件，并将每个组件的值提取到一个捕获组中。在管道代码中，这些数据被放置在一个临时数据结构中，在处理该日志行时可以多次使用。关于这方面的更多详细信息可以在Promtail管道文档中找到。

根据该正则表达式，我们将使用两个捕获组来根据日志行的内容动态设置两个标签：
- action（例如，action=“GET”、action=“POST”）
- status_code（例如，status_code=“200”、status_code=“400”）




现在让我们逐行解释一些示例行：
```
11.11.11.11 - frank [25/Jan/2000:14:00:01 -0500] "GET /1986.js HTTP/1.1" 200 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"

11.11.11.12 - frank [25/Jan/2000:14:00:02 -0500] "POST /1986.js HTTP/1.1" 200 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"

11.11.11.13 - frank [25/Jan/2000:14:00:03 -0500] "GET /1986.js HTTP/1.1" 400 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"

11.11.11.14 - frank [25/Jan/2000:14:00:04 -0500] "POST /1986.js HTTP/1.1" 400 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"
```


在Loki中，将创建以下日志流：
```
{job="apache",env="dev",action="GET",status_code="200"} 11.11.11.11 - frank [25/Jan/2000:14:00:01 -0500] "GET /1986.js HTTP/1.1" 200 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"

{job="apache",env="dev",action="POST",status_code="200"} 11.11.11.12 - frank [25/Jan/2000:14:00:02 -0500] "POST /1986.js HTTP/1.1" 200 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"

{job="apache",env="dev",action="GET",status_code="400"} 11.11.11.13 - frank [25/Jan/2000:14:00:03 -0500] "GET /1986.js HTTP/1.1" 400 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"

{job="apache",env="dev",action="POST",status_code="400"} 11.11.11.14 - frank [25/Jan/2000:14:00:04 -0500] "POST /1986.js HTTP/1.1" 400 932 "-" "Mozilla/5.0 (Windows; U; Windows NT 5.1; de; rv:1.9.1.7) Gecko/20091221 Firefox/3.5.7 GTB6"
```
这四个日志行将变成四个单独的日志流，并开始填充四个单独的块。


**进一步的解析**
与这些标签/值组合匹配的任何其他日志行将添加到现有的日志流中。如果出现另一个唯一的标签组合（例如，status_code=“500”），将创建另一个新的日志流。

现在想象一下，如果您为 IP 设置了一个标签。不仅每个用户的请求都会成为一个唯一的日志流，而且来自同一用户但具有不同操作或状态代码的每个请求都将拥有自己的日志流。

做一些快速的计算，如果有大约四个常见的操作（GET、PUT、POST、DELETE）和大约四个常见的状态代码（尽管可能有多于四个！），那么将会有 16 个日志流和 16 个单独的块。现在如果乘以每个用户（如果我们使用 IP 标签），您很快就会拥有成千上万个日志流。

这就是高基数。这可能会使 Loki 崩溃。

当我们谈论基数时，我们指的是标签和值的组合以及它们创建的日志流数量。高基数是指使用具有大范围可能值的标签，例如 IP，或者即使使用具有小且有限值集的许多标签，例如使用 status_code 和 action。

高基数会导致 Loki 构建一个巨大的索引（read：），并将成千上万个微小的块刷新到对象存储中（read：缓慢）。在当前配置下，Loki的性能非常差，也是成本效益最低且最不好玩的运行和使用方式。



# 5 使用并行化的最佳Loki性能


现在你可能在问:如果使用大量标签或带有大量值的标签不好，我该如何查询我的日志？ 如果所有数据都没有索引，查询不会很慢吗？

当我们看到使用Loki的人习惯于其他索引重的解决方案时，他们似乎觉得有义务定义许多标签，以便有效地查询他们的日志。毕竟，许多其他日志解决方案都与索引有关，这是常见的思维方式。

使用Loki时，您可能需要忘记您所知道的，看看如何通过并行化以不同的方式解决问题。Loki的超能力是将查询分解成小块，并并行调度，以便您可以在短时间内查询大量日志数据。

This kind of brute force approach might not sound ideal, but let me explain why it is.

Large indexes are complicated and expensive. 通常，日志数据的全文索引的大小与日志数据本身相同或更大. 要查询您的日志数据，您需要加载此索引，为了性能，它可能应该在内存中。This is difficult to scale, and as 你摄取了更多的日志，你的索引很快就会变大.

Now let’s talk about Loki, where the index is typically an order of magnitude smaller than your ingested log volume. So if you are doing a good job of keeping your streams and stream churn to a minimum, the index grows very slowly compared to the ingested logs.现在我们来谈谈Loki，那里的索引通常比你摄入的日志量小一个数量级。因此，如果您在将streams 和 stream churn 保持在最低限度方面做得很好，那么与摄入的日志相比，该索引的增长速度非常缓慢。

Loki将有效地使您的静态成本尽可能低（索引大小和内存要求以及静态日志存储），并使查询性能成为您在运行时可以通过水平缩放来控制的。

看看这是如何运作的, 让我们回顾一下我们为特定IP地址查询访问日志数据的示例。我们不想使用标签来存储IP地址。相反，我们使用[filter expression](https://link.zhihu.com/?target=https%3A//grafana.com/docs/loki/latest/logql/log_queries/%23line-filter-expression)来查询它

```text
{job="apache"} |= "11.11.11.11"
```

Behind the scenes, Loki will break up that query into smaller pieces (shards), and open up each chunk for the streams matched by the labels and start looking for this IP address.

Loki会将该查询分解成更小的片段（碎片），并为与标签匹配的流打开每个块，并开始查找此IP地址。

这些碎片的大小和并行化量是可配置的，并基于您提供的资源。If you want to, you can configure the shard interval down to 5m, deploy 20 queriers, and process gigabytes of logs in seconds. Or you can go crazy and provision 200 queriers and process terabytes of logs!

This trade-off of smaller index and parallel brute force querying vs. a larger/faster full-text index is what allows Loki to save on costs versus other systems. The cost and complexity of operating a large index is high and is typically fixed – you pay for it 24 hours a day if you are querying it or not.

The benefits of this design mean you can make the decision about how much query power you want to have, and you can change that on demand. Query performance becomes a function of how much money you want to spend on it. Meanwhile, the data is heavily compressed and stored in low-cost object stores like S3 and GCS. This drives the fixed operating costs to a minimum while still allowing for incredibly fast query capability.




