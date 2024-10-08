
https://zhuanlan.zhihu.com/p/603066874

除了指标之外，Explore 还允许你在以下数据源中调查你的日志。

- [Elasticsearch](https://link.zhihu.com/?target=https%3A//grafana.com/docs/grafana/latest/datasources/elasticsearch/)
- [InfluxDB](https://link.zhihu.com/?target=https%3A//grafana.com/docs/grafana/latest/datasources/influxdb/)
- [Loki](https://link.zhihu.com/?target=https%3A//grafana.com/docs/grafana/latest/datasources/loki/)

在基础设施监控和事件响应期间，你可以深入挖掘指标和日志，找到原因。Explore 还允许你通过并排查看指标和日志来进行关联。这创造了一个新的调试工作流程。

1. 接到一个警报。
2. 深入研究并检查指标。
3. 再次深入，搜索与指标和时间间隔有关的日志（将来还有分布式跟踪）。

# 1 日志可视化

日志查询的结果在图表中以直方图的形式显示，个别日志在下面的章节中会有解释。

如果数据源支持全范围的日志量直方图，则会自动显示所有输入的日志查询的日志分布图。目前 Elasticsearch 和 Loki 数据源支持此功能。

如果数据源不支持加载全范围的日志量直方图，日志模型会根据自动计算的时间间隔的日志行数来计算一个时间序列，然后将第一个日志行的时间戳锚定在结果的直方图的起点。时间序列的末端被锚定在时间选择器的**To**范围内。

## 1.1 日志级别

对于指定了级别标签的日志，我们使用标签的值来确定日志的级别并相应地更新颜色。如果日志没有指定级别标签，我们会尝试找出其内容是否与任何支持的表达式相匹配（更多信息见下文）。日志级别总是由第一个匹配项决定的。如果 Grafana 无法确定一个日志级别，它将以未知的日志级别进行可视化。

> **Tip:** 如果你使用 Loki 数据源，并且 "级别 "在你的日志线中，使用解析器（`JSON`、`logfmt`、`regex`...）将级别信息提取为一个级别标签，用来确定日志级别。这将使直方图在不同的柱状图中显示不同的日志级别。

**支持的日志级别以及日志级别缩写和表达式的映射：**

# 2 日志导航

日志行旁边的日志导航可以用来请求更多的日志。你可以通过点击导航底部的 Older logs 按钮来做到这一点。当你遇到行数限制，你想看到更多的日志时，这一点特别有用。每个从导航中运行的请求都会作为单独的页面显示在导航中。每一页都显示传入日志行的起止时间戳。你可以通过点击页面查看以前的结果。Explore 缓存了从日志导航中运行的最后五个请求，所以当你点击这些页面时，你不会重新运行相同的查询。

![](https://pic1.zhimg.com/80/v2-fb1e4fbea323b5c35db8b013075bb814_720w.webp)

# 3 可视化选项

你可以自定义日志的显示方式，并选择显示哪些列。

Time
显示或隐藏时间列。这是与日志行相关的时间戳，由数据源报告。

Unique labels
显示或隐藏仅包括非常见标签的独特标签栏。所有常见的标签都显示在上面。

换行
如果你想让显示器使用换行，将此设置为`true`; 设置为`False`，将导致水平滚动。

Prettify JSON
将此设置为`true`以漂亮地打印所有 JSON 日志。这个设置不影响 JSON 以外的任何格式的日志。

Deduping（去重）
日志数据可能是非常重复的，Explore 可以通过隐藏重复的日志行来帮助。你可以使用几种不同的重复数据删除算法。

- **精确** - 精确匹配是在整个行中进行的，除了日期字段。
- **数字** - 在剥离数字后的行上进行匹配，如持续时间、IP 地址等。
- **签名** - 最激进的剔除，这将剥离所有的字母和数字，并在剩余的空白处和标点符号上进行匹配。


Flip results order（翻转结果顺序）
你可以将收到的日志的顺序从默认的降序（最新的先）改为升序（最旧的先）。

# 4 Labels and detected fields（标签和检测字段）

每个日志行都有一个可扩展的区域，有它的标签和检测字段，以实现更强大的互动。对于所有的标签，我们增加了过滤（正向过滤）和过滤（反向过滤）选定标签的能力。每个字段或标签也有一个统计图标，以显示与所有显示的日志有关的特别统计数据。

## 4.1 Derived fields links（衍生字段链接）

通过使用衍生字段，你可以把日志消息的任何部分变成内部或外部链接。创建的链接在日志细节视图中的 Detected 字段旁边以按钮的形式显示。

![](https://pic2.zhimg.com/80/v2-bb85b1c3e3cb523d4db22e15ec5955b9_720w.webp)

## 4.2 Toggle detected fields（切换检测到的字段）

> **Note:** 在 Grafana 7.2 及更高版本中可用。

如果你的日志是以`json`或`logfmt`构造的，那么你可以显示或隐藏检测到的字段。展开一个日志行，然后点击眼睛图标来显示或隐藏字段。

![动图封面](https://pic4.zhimg.com/v2-6316ea886ba253476e9bc61e0318e2a3_b.jpg)

# 5 Loki 特有的功能

如前所述，其中一个日志集成是针对 Grafana Labs 的新的开源日志聚合系统--[Loki](https://link.zhihu.com/?target=https%3A//github.com/grafana/loki)。Loki 的设计非常具有成本效益，因为它不对日志的内容进行索引，而是为每个日志流提供一组标签。Loki 的日志查询方式与 Prometheus 中使用标签选择器的查询方式类似。它使用标签对日志流进行分组，可以使之与你的 Prometheus 标签相匹配。关于 Grafana Loki 的更多信息，请参考 [Grafana Loki](https://link.zhihu.com/?target=https%3A//github.com/grafana/loki) 或 Grafana Labs 的托管版本：[Grafana Cloud Logs](https://link.zhihu.com/?target=https%3A//grafana.com/loki)。

更多信息请参考 [Loki 的数据源文档](https://link.zhihu.com/?target=https%3A//grafana.com/docs/grafana/latest/datasources/loki/) 关于如何查询日志数据的信息。

## 5.1 从指标转换到日志

如果你从 Prometheus 查询切换到日志查询（你可以先做一个分割，让你的指标和日志并排），那么它将保留你查询中存在于日志中的标签，并使用这些标签来查询日志流。例如，下面的 Prometheus 查询。

`grafana_alerting_active_alerts{job="grafana"}`

切换到 Logs 数据源后，查询结果变为：

`{job="grafana"}`

这将返回所选时间范围内的一大块日志，可以进行 grepped/text 搜索。

## 5.2 实时滚动 (Live Tailing)

使用实时滚动功能来查看支持的数据源的实时日志。

点击 Explore 工具栏上的**实时**按钮，切换到实时滚动视图。

在实时滚动视图中，新的日志会从屏幕的底部出现，并且会有渐变的对比背景，因此你可以跟踪新的内容。点击**暂停**按钮或滚动日志视图来暂停实时跟踪，不间断地探索以前的日志。点击**恢复**按钮恢复实时跟踪，或点击**停止**按钮退出实时跟踪，回到标准的探索视图。

![动图封面](https://pic4.zhimg.com/v2-f96d115360c70bb7032b3e130c2ce963_b.jpg)