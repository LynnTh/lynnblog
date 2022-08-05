---
title: "Prometheus"
date: 2022-08-05T23:58:37+08:00
draft: false
tags:
  - prometheus
  - grafana
categories:
  - monitor
---

# Prometheus&Grafana介绍

## Prometheus介绍

### 基础知识

#### 概述

[Prometheus](https://prometheus.io/) 是一个监控系统和时序数据库，特别适合监控云原生环境，它具有多维数据模型和强大的查询语言，并在一个生态系统中集成了检测、指标收集、服务发现和报警等功能。

#### 什么是Prometheus

[Prometheus](https://prometheus.io/) 是一个基于指标监控和报警的工具栈。 Prometheus 起源于 SoundCloud ，因为微服务迅速发展，导致实例数量以几何倍数递增，不得不考虑设计一个符合以下几个功能的监控系统：

- 多维数据模型，可以按照实例，服务，端点和方法之类的维度随意对数据进行切片和切块。
- 操作简单，可以随时随地部署监控服务，甚至在本地工作站上，而无需设置分布式存储后端或重新配置环境。
- 可扩展的数据收集和分散的架构，以便于可以可靠的监控服务的许多实例，独立团队可以部署独立的监控服务。
- 一种查询语言，可以利用数据模型进行有效的报警和图形展示。

但是，当时的情况是，以上的功能都分散在各个系统之中，直到 2012 年 SoundCloud 启动了一个孵化项目把这些所有功能集合到一起，也就是 Prometheus。Prometheus 是用 Go 语言编写，从一开始就是开源的，到 2016 年 Prometheus 成为继 Kubernetes 之后，成为 CNCF 的第二个成员。

到现在为止 Prometheus 提供的工具或与其他生态系统组件集成可以提供完整的监控管道：

- 检测（跟踪和暴露指标）
- 指标收集
- 指标存储
- 查询指标，用于报警、仪表板等

Prometheus 具有足够的通用性，可以监控各个级别的实例：你自己的应用程序、第三方服务、主机或网络设备等等。此外 Prometheus 特别适用于监控动态云环境和 Kubernetes 云原生环境。

整体来说与其他监控解决方案相比，Prometheus 提供了许多重要功能：

- 多维数据模型，允许对指标进行跟踪
- 强大的查询语言（PromQL）
- 时间序列处理和报警的整合
- 与服务发现集成来确定应监控的内容
- 操作简单
- 执行高效

### 系统架构

下图演示了 Prometheus 系统的整体架构

![20210406170610](.\img\20210406170610.png)

一个团队通常会运行一个或多个 Prometheus 服务器，这些服务器构成了 Prometheus 监控的核心。Prometheus 服务器可以配置为使用服务发现机制（如 DNS、Consul、Kubernetes 等）来自动发现一组指标源（所谓的`目标`），然后 Prometheus 通过 HTTP 定期从这些目标中以[文本格式](https://prometheus.io/docs/instrumenting/exposition_formats/)`抓取`指标，并将收集到的数据存储在本地时序数据库中。

抓取的目标可以是一个直接暴露 Prometheus 指标的应用程序，也是可以是一个将现有系统（比如 MySQL）的 metrics 指标转换为 Prometheus 指标标准格式的中间应用（也就是我们说的 `exporter`），然后 Prometheus 服务器通过其内置的 Web UI、其他仪表盘工具（比如 [Grafana](https://grafana.com/)）或者直接使用其 HTTP API 来提供收集到的数据进行查询。

另外我们还可以配置 Prometheus 根据收集的指标数据生成报警，但是 Prometheus 不会直接把报警通知发送给我们，而是将原始报警转发到 `Alertmanager` 服务，Alertmanager 是作为单独的服务运行的，可以从多个 Prometheus 服务器上接收报警，并可以对这些报警进行分组、汇总和路由，最后可以通过 Email、Slack、PagerDuty、企业微信、Webhook 或其他通知服务来发送通知。

### 数据模型

Prometheus 采集的监控数据都是以指标（metric）的形式存储在内置的 TSDB 数据库中，这些数据都是**时间序列**：一个带时间戳的数据，这些数据具有一个标识符和一组样本值。除了存储的时间序列，Prometheus 还可以根据查询请求产生临时的、衍生的时间序列作为返回结果。

#### 时间序列

Prometheus 会将所有采集到的样本数据以时间序列的形式保存在内存数据库中，并定时刷新到硬盘上，时间序列是按照时间戳和值的序列方式存放的，我们可以称之为**向量**（vector），每一条时间序列都由一个**指标名称**和一组**标签**（键值对）来唯一标识。

- 指标名称反映了被监控样本的含义（如 `http_request_total` 表示的是对应服务器处理的 HTTP 请求总数）。
- 标签可以用来区分不同的维度（比如 `method="GET"` 与 `method="POST"` 就可以用来区分这两种不同的 HTTP 请求指标数据）。

如下所示，可以将时间序列理解为一个以时间为 Y 轴的数字矩阵：

![image-20220719095605868](.\img\image-20220719095605868.png)

#### 样本

时间序列中的每一个点就称为一个样本（sample），样本由以下 3 个部分组成：

- **指标**：指标名称和描述当前样本特征的标签集
- **时间戳**：精确到毫秒的时间戳数
- **样本值**：一个 64 位浮点数

#### 指标

想要暴露 Prometheus 指标服务只需要暴露一个 HTTP 端点，并提供 Prometheus 基于文本格式的指标数据即可。这种指标格式是非常友好的，基本上的格式看起来类似于下面的这段代码：

```
# HELP http_requests_total The total number of processed HTTP requests.
# TYPE http_requests_total counter
http_requests_total{status="200"} 8556
http_requests_total{status="404"} 20
http_requests_total{status="500"} 68
```

其中 `#` 开头的行是注释信息，用来描述下面提供的指标含义，其他未注释行代表一个样本（带有指标名、标签和样本值），使其非常容易从系统和服务中暴露指标出来。

事实上所有的指标也都是通过如下所示的格式来标识的：

```
<metric name>{<label name>=<label value>, ...}
```

例如，指标名称是 `http_request_total`，标签集为 `method="POST", endpoint="/messages"`，那么我们可以用下面的方式来标识这个指标：

```
http_request_total{method="POST", endpoint="/messages"}
```

而事实上 Prometheus 的底层实现中指标名称实际上是以 `__name__=<metric name>` 的形式保存在数据库中的，所以上面的指标也等同与下面的指标：

```
{__name__="http_request_total", method="POST", endpoint="/messages"}
```

所以也可以认为一个指标就是一个标签集，只是这个标签集里面一定包含一个 `__name__` 的标签来定义这个指标的名称。

### 查询语言

为了能够使用收集的监控数据，Prometheus 实现了自己的查询语言 PromQL。PromQL 是一种功能性的语言（类似于 SQL 语句），可以对时间序列数据进行灵活高效的计算。和类似 SQL 的语言相比，PromQL 仅用于数据读取，而不是插入、更新或者删除数据（这些操作在查询引擎之外实现的）

比如下面的语句表示的是过滤所有状态码位为 `500` 的 HTTP 请求总数：

```
http_requests_total{status="500"}
```

下面的查询语句表示每个序列在 5 分钟的时间窗口内的平均每秒增长率：

```
rate(http_requests_total{status="500"}[5m])
```

我们还可以计算 `status="500"` 错误与按 HTTP 路径分组的请求总数的比率：

```
sum by(path) (rate(http_requests_total{status="500"}[5m]))
/
sum by(path) (rate(http_requests_total[5m]))
```

## Grafana介绍

### 基础知识

[Grafana](https://grafana.com/) 是一个监控仪表系统，它是由 Grafana Labs 公司开源的的一个系统监测工具，它可以大大帮助我们简化监控的复杂度，我们只需要提供需要监控的数据，它就可以帮助生成各种可视化仪表，同时它还有报警功能，可以在系统出现问题时发出通知。

Grafana 支持许多不同的数据源，每个数据源都有一个特定的查询编辑器，每个数据源的查询语言和能力都是不同的，我们可以把来自多个数据源的数据组合到一个仪表板，但每一个面板被绑定到一个特定的数据源。

### 部署grafana

使用rpm包安装：

```
yum install grafana-enterprise-8.5.6-1.x86_64.rpm

# 设置开机自启动grafana
systemctl enable grafana-server
# 启动grafana
systemctl start grafana-server
# 查看grafana进程状态
systemctl status grafana-server
```

有时需要重启grafana，重启命令：

```
systemctl restart grafana-server
```

正常启动完成后 Grafana 会监听在 3000 端口上，所以我们可以在浏览器中打开 Grafana 的 WebUI。

grafana涉及到的目录有几个，最重要的是`/var/lib/grafana/`

![image-20220718101620008](.\img\image-20220718101620008.png)

其中`grafana.db`是grafana依赖的数据库，*非常重要*。`plugins`是插件目录。

### 安装插件

插件下载地址：https://grafana.com/grafana/plugins/。一般来说，插件都可以在这里下载到。

下载插件后，将其解压，并将解压的文件夹复制到`/var/lib/grafana/plugins`文件夹中，并重启grafana。

### 面板

面板（Panel）是 Grafana 中基本可视化构建块，每个面板都有一个特定于面板中选择数据源的查询编辑器，每个面板都有各种各样的样式和格式选项，面板可以在仪表板上拖放和重新排列，它们也可以调整大小，所以要在 Grafana 上创建可视化的图表，面板是我们必须要掌握的知识点。

Panel 是 Grafana 中最基本的可视化单元，每一种类型的面板都提供了相应的查询编辑器(Query Editor)，让用户可以从不同的数据源（如 Prometheus）中查询出相应的监控数据，并且以可视化的方式展现，Grafana 中所有的面板均以插件的形式进行使用。Grafana 提供了各种可视化来支持不同的用例，目前内置支持的面板包括：Time series（时间序列）是默认的也是主要的图形可视化面板、State timeline（状态时间表）状态随时间变化 、Status history（状态历史记录）、Bar chart（条形图）、Histogram（直方图）、Heatmap（热力图）、Pie chart（饼状图）、Stat（统计数据）、Gauge、Bar gauge、Table（表格）、Logs（日志）、Node Graph（节点图）、Dashboard list（仪表板列表）、Alert list（报警列表）、Text panel（文本面板，支持 markdown 和 html）、News Panel（新闻面板，可以显示 RSS 摘要）等，除此之外，我们还可以通过官网的[面板插件页面](https://grafana.com/grafana/plugins/?type=panel)获取安装其他面板进行使用。

#### 图表

对于基于时间的折线图、面积图和条形图，我们建议使用默认的 Time series 时间序列进行可视化。

![20211104153720](.\img\20211104153720.png)

而对于分类数据，则使用 Bar Chart 条形图可视化。

![20211104153803](.\img\20211104153803.png)

#### 数据统计

用于数据统计相关的可视化可以使用 `Stat` 面板进行可视化，我们可以使用阈值或色标来控制背景或值颜色。

![20211104153940](.\img\20211104153940.png)

#### 仪表盘

如果想显示与最小值和最大值相关的值，则有两个选择，首先是如下所示的标准径 Gauge 面板。

![20211104154049](.\img\20211104154049.png)

其次 Grafana 还具有水平或垂直 Bar gauge（条形仪表面板），具有三种不同的不同显示模式。

![20211104154147](.\img\20211104154147.png)

#### 表格

要在表格布局中显示数据，需要使用 Table 表格面板进行可视化。

![20211104154332](.\img\20211104154332.png)

#### 饼状图

Grafana 现在支持 Pie Chart 饼状图面板可视化。

![20211104154624](.\img\20211104154624.png)

### 配置数据源

![image-20220718102351947](.\img\image-20220718102351947.png)

然后根据数据源类型选择，不同数据源类型的配置方法有所不同。

### 配置查询语句

配置grafana报表需要数据源查询语句，包括：

- prometheus：PromQL，参考网址：https://doc.cncf.vip/prometheus-handbook/parti-prometheus-ji-chu/promql。Prometheus常用来监控PaaS组件。

- ES：使用Lucene查询语句，参考网址：https://lucene.apache.org/core/3_0_3/queryparsersyntax.html。ES常用来查询日志等文本数据。
- mysql：SQl查询语句
- zabbix：IaaS监控工具，可以选择监控项。

## prometheus优化架构

![prometheus架构](.\img\prometheus架构.png)
