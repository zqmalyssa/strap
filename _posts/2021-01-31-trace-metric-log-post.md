---
layout: post
title: 监控三大领域：Tracing、Logging和Metrics
tags: [monitor]
author-id: zqmalyssa
---

微服务监控三个领域:Tracing、Logging和Metrics，它们相辅相成，合力支撑多维度、多形态的监控体系。

#### 概况

1 Tracing：它在单次请求的范围内，处理信息。 任何的数据、元数据信息都被绑定到系统中的单个事务上。例如：一次调用远程服务的RPC执行过程；一次实际的SQL查询语句；一次HTTP请求的业务性ID；（Request scoped）

在Tracing方面，已经从最开始的Google Dapper论文逐渐演进形成了OpenTracing规范，并有着众多的开源实现，基于Dpaaer的开源实现（Zipkin，Pinpoint，EagleEye（淘宝），Hydra（京东）），CNCF为了解决不同的分布式tracing系统的API不兼容问题，形成了OpenTracing规范，开源实现有（CNCF Jaeger，Open Zipkin，SkyWalking，Elastic APM），业务监控，基于业务信息数据埋点，对入口渠道、用户、交易金额等进行监控分析，与metric整合，对交易量、交易内部调用次数，RPC调用次数、数据库调用次数等对应用和代码质量进行分析检测

2 Logging：它描述一些离散的（不连续的）事件。 例如：应用通过一个滚动的文件输出debug或error信息，并通过日志收集系统，存储到Elasticsearch中； 审批明细信息通过Kafka，存储到数据库（BigTable）中；又或者，特定请求的元数据信息，从服务请求中剥离出来，发送给一个异常收集服务，如NewRelic；（Events）

大的方向上ELK依然牢牢占据着日志采集与展示的龙头（Es，Logstash，Kibana）

3 Metrics：特点是可累加的：他们具有原子性，每个都是一个逻辑计量单元，或者一个时间段内的柱状图。例如：队列的当前深度可以被定义为一个计量单元，在写入或读取时被更新统计； 输入HTTP请求的数量可以被定义为一个计数器，用于简单累加； 请求的执行时间可以被定义为一个柱状图，在指定时间片上更新和统计汇总。（Aggregatable）

CNCF主推的Prometheus以及配套的Grafana已经逐渐盖过了Zabbix的风头，成为云原生时代炙手可热的监控利器

#### tracing

随着微服务架构的流行，一次请求往往需要涉及到多个服务，因此服务性能监控和排查就变得更复杂，因此，就需要一些可以帮助理解系统行为、用于分析性能问题的工具，以便发生故障的时候，能够快速定位和解决问题，这就是APM系统，全称是（Application Performance Monitor，当然也有叫 Application Performance Management tools）

AMP最早是谷歌公开的论文提到的 Google Dapper。Dapper是Google生产环境下的分布式跟踪系统，自从Dapper发展成为一流的监控系统之后，给google的开发者和运维团队帮了大忙，所以谷歌公开论文分享了Dapper

Dapper的设计目标

a、性能消耗低

APM组件服务的影响应该做到足够小。服务调用埋点本身会带来性能损耗，这就需要调用跟踪的低损耗，实际中还会通过配置采样率的方式，选择一部分请求去分析请求路径。在一些高度优化过的服务，即使一点点损耗也会很容易察觉到，而且有可能迫使在线服务的部署团队不得不将跟踪系统关停。

b、应用透明，也就是代码的侵入性小

即也作为业务组件，应当尽可能少入侵或者无入侵其他业务系统，对于使用方透明，减少开发人员的负担。

对于应用的程序员来说，是不需要知道有跟踪系统这回事的。如果一个跟踪系统想生效，就必须需要依赖应用的开发者主动配合，那么这个跟踪系统也太脆弱了，往往由于跟踪系统在应用中植入代码的bug或疏忽导致应用出问题，这样才是无法满足对跟踪系统“无所不在的部署”这个需求。

c、可扩展性

一个优秀的调用跟踪系统必须支持分布式部署，具备良好的可扩展性。能够支持的组件越多当然越好。或者提供便捷的插件开发API，对于一些没有监控到的组件，应用开发者也可以自行扩展。

d、数据的分析

数据的分析要快 ，分析的维度尽可能多。跟踪系统能提供足够快的信息反馈，就可以对生产环境下的异常状况做出快速反应。分析的全面，能够避免二次开发。


Tracing有个OpenTracing规范，[参考](https://opentracing.io/docs/overview/spans/)

这其中有个Span的概念：有Tags，Logs，SpanContext，Operation Name等等，Span是dapper的基本工作单元，一次链路调用（可以是RPC，DB等没有特定的限制）创建一个span，通过一个64位ID标识它；同时附加（Annotation）作为payload负载信息，用于记录性能等数据，如果有两个子调用，那么子调用会生成新的spanId，而它们的父id是整个开始的span id（root span）

上图说明了span在一次大的跟踪过程中是什么样的。Dapper记录了span名称，以及每个span的ID和父ID，以重建在一次追踪过程中不同span之间的关系。如果一个span没有父ID被称为root span。所有span都挂在一个特定的跟踪上，也共用一个跟踪id。

span的数据结构

```html

type Span struct {
    TraceID    int64 // 用于标示一次完整的请求id
    Name       string
    ID         int64 // 当前这次调用span_id
    ParentID   int64 // 上层服务的调用span_id  最上层服务parent_id为null
    Annotation []Annotation // 用于标记的时间戳
    Debug      bool
}
```

TraceID，类似于 树结构的Span集合，表示一次完整的跟踪，从请求到服务器开始，服务器返回response结束，跟踪每次rpc调用的耗时，存在唯一标识trace_id。比如：你运行的分布式大数据存储一次Trace就由你的一次请求组成

Span中还有annotation

```html

(1) cs：Client Start，表示客户端发起请求
(2) sr：Server Receive，表示服务端收到请求
(3) ss：Server Send，表示服务端完成处理，并将结果发送给客户端
(4) cr：Client Received，表示客户端获取到服务端返回信息

type Annotation struct {
    Timestamp int64
    Value     string
    Host      Endpoint
    Duration  int32
}

```

整个过程大致是这样的：

a请求到来生成一个全局TraceID，通过TraceID可以串联起整个调用链，一个TraceID代表一次请求。
b除了TraceID外，还需要SpanID用于记录调用父子关系。每个服务会记录下parent id和span id，通过他们可以组织一次完整调用链的父子关系。
c一个没有parent id的span成为root span，可以看成调用链入口。
d所有这些ID可用全局唯一的64位整数表示；
e整个调用过程中每个请求都要透传TraceID和SpanID。
f每个服务将该次请求附带的TraceID和附带的SpanID作为parent id记录下，并且将自己生成的SpanID也记录下。
g要查看某次完整的调用则 只要根据TraceID查出所有调用记录，然后通过parent id和span id组织起整个调用父子关系。

调用链需要做哪些工作呢：

a调用链数据生成，对整个调用过程的所有应用进行埋点并输出日志。
b调用链数据采集，对各个应用中的日志数据进行采集。
c调用链数据存储及查询，对采集到的数据进行存储，由于日志数据量一般都很大，不仅要能对其存储，还需要能提供快速查询。
d指标运算、存储及查询，对采集到的日志数据进行各种指标运算，将运算结果保存起来。
e告警功能，提供各种阀值警告功能。

整体部署架构（案例）

a.通过AGENT生成调用链日志。
b.通过logstash采集日志到kafka。
c.kafka负责提供数据给下游消费。
d.storm计算汇聚指标结果并落到es。
e.storm抽取trace数据并落到es，这是为了提供比较复杂的查询。比如通过时间维度查询调用链，可以很快查询出所有符合的traceID，根据这些traceID再去 Hbase 查数据就快了。
f.logstash将kafka原始数据拉取到hbase中。hbase的rowkey为traceID，根据traceID查询是很快的。

AGENT无侵入部署??

a.服务内AGENT，这种方式是通过 Java 的agent机制，对服务内部的方法调用层次信息进行数据收集，如方法调用耗时、入参、出参等信息。

b.跨服务AGENT，这种情况需要对主流RPC框架以插件形式提供无缝支持。并通过提供标准数据规范以适应自定义RPC框架：
（1）Dubbo支持；

（2）Rest支持；

（3）自定义RPC支持；


```html

Tags:
- db.instance:"customers"
- db.statement:"SELECT * FROM mytable WHERE foo='bar'"
- peer.address:"mysql://127.0.0.1:3306/customers"

Logs:
- message:"Can't connect to mysql server on '127.0.0.1'(10061)"

SpanContext:
- trace_id:"abc123"
- span_id:"xyz789"
- Baggage Items:
  - special_id:"vsid1738"


```

OpenTracing的规范定义每一个Span都包含了以下内容：

操作名（Operation Name），标志该操作是什么
标签 （Tag），标签是一个名值对，用户可以加入任何对追踪有意义的信息
日志（Logs），日志也定义为名值对。用于捕获调试信息，或者相关Span的相关信息
跨度上下文呢 （SpanContext），SpanContext负责子微服务系统边界传递数据。它主要包含两部分：
和实现无关的状态信息，例如Trace ID，Span ID
行李项 （Baggage Item）。如果把微服务调用比做从一个城市到另一个城市的飞行, 那么SpanContext就可以看成是飞机运载的内容。Trace ID和Span ID就像是航班号，而行李项就像是运送的行李。每次服务调用，用户都可以决定发送不同的行李。

要实现分布式追踪，如何传递SpanContext是关键。OpenTracing定义了两个方法Inject和Extract用于SpanContext的注入和提取。

```html

Inject 伪代码

span_context = ...
outbound_request = ...

# We'll use the (builtin) HTTP_HEADERS carrier format. We
# start by using an empty map as the carrier prior to the
# call to `tracer.inject`.
carrier = {}
tracer.inject(span_context, opentracing.Format.HTTP_HEADERS, carrier)

# `carrier` now contains (opaque) key:value pairs which we pass
# along over whatever wire protocol we already use.
for key, value in carrier:
    outbound_request.headers[key] = escape(value)

```

这里的注入的过程就是把context的所有信息写入到一个叫Carrier的字典中，然后把字典中的所有名值对写入 HTTP Header。


```html

Extract 伪代码

inbound_request = ...

# We'll again use the (builtin) HTTP_HEADERS carrier format. Per the
# HTTP_HEADERS documentation, we can use a map that has extraneous data
# in it and let the OpenTracing implementation look for the subset
# of key:value pairs it needs.
#
# As such, we directly use the key:value `inbound_request.headers`
# map as the carrier.
carrier = inbound_request.headers
span_context = tracer.extract(opentracing.Format.HTTP_HEADERS, carrier)
# Continue the trace given span_context. E.g.,
span = tracer.start_span("...", child_of=span_context)

# (If `carrier` held trace data, `span` will now be ready to use.)


```

抽取过程是注入的逆过程，从carrier，也就是HTTP Headers，构建SpanContext。

整个过程类似客户端和服务器传递数据的序列化和反序列化的过程。这里的Carrier字典支持Key为string类型，value为string或者Binary格式（Bytes）。

其实是对Header做些东西，Python的例子

```html

import requests
import sys
import time
from lib.tracing import init_tracer
from opentracing.ext import tags
from opentracing.propagation import Format


def say_hello(hello_to):
    with tracer.start_active_span('say-hello') as scope:
        scope.span.set_tag('hello-to', hello_to)
        hello_str = format_string(hello_to)
        print_hello(hello_str)

def format_string(hello_to):
    with tracer.start_active_span('format') as scope:
        hello_str = http_get(8081, 'format', 'helloTo', hello_to)
        scope.span.log_kv({'event': 'string-format', 'value': hello_str})
        return hello_str

def print_hello(hello_str):
    with tracer.start_active_span('println') as scope:
        http_get(8082, 'publish', 'helloStr', hello_str)
        scope.span.log_kv({'event': 'println'})

def http_get(port, path, param, value):
    url = 'http://localhost:%s/%s' % (port, path)

    span = tracer.active_span
    span.set_tag(tags.HTTP_METHOD, 'GET')
    span.set_tag(tags.HTTP_URL, url)
    span.set_tag(tags.SPAN_KIND, tags.SPAN_KIND_RPC_CLIENT)
    headers = {}
    tracer.inject(span, Format.HTTP_HEADERS, headers)

    r = requests.get(url, params={param: value}, headers=headers)
    assert r.status_code == 200
    return r.text


# main
assert len(sys.argv) == 2

tracer = init_tracer('hello-world')

hello_to = sys.argv[1]
say_hello(hello_to)

# yield to IOLoop to flush the spans
time.sleep(2)
tracer.close()

````

客户端完成了以下的工作：

初始化Tracer，trace的名字是‘hello-world’
创建一个客户端操作say_hello，该操作关联一个Span，取名‘say-hello’，并调用span.set_tag加入标签
在操作say_hello中调用第一个HTTP 服务A，format_string， 该操作关联另一个Span取名‘format’，并调用span.log_kv加入日志
之后调用另一个HTTP 服务B，print_hello， 该操作关联另一个Span取名‘println’，并调用span.log_kv加入日志
对于每一个HTTP请求，在Span中都加入标签，标志http method，http url和span kind。并调用tracer.inject把SpanContext注入到http header 中。

```html

服务A 代码

from flask import Flask
from flask import request
from lib.tracing import init_tracer
from opentracing.ext import tags
from opentracing.propagation import Format

app = Flask(__name__)
tracer = init_tracer('formatter')

@app.route("/format")
def format():
    span_ctx = tracer.extract(Format.HTTP_HEADERS, request.headers)
    span_tags = {tags.SPAN_KIND: tags.SPAN_KIND_RPC_SERVER}
    with tracer.start_active_span('format', child_of=span_ctx, tags=span_tags):
        hello_to = request.args.get('helloTo')
        return 'Hello, %s!' % hello_to

if __name__ == "__main__":
    app.run(port=8081)

```

服务A响应format请求，调用tracer.extract从http headers中提取信息，构建spanContext。

```html

服务B 代码

from flask import Flask
from flask import request
from lib.tracing import init_tracer
from opentracing.ext import tags
from opentracing.propagation import Format

app = Flask(__name__)
tracer = init_tracer('publisher')

@app.route("/publish")
def publish():
    span_ctx = tracer.extract(Format.HTTP_HEADERS, request.headers)
    span_tags = {tags.SPAN_KIND: tags.SPAN_KIND_RPC_SERVER}
    with tracer.start_active_span('publish', child_of=span_ctx, tags=span_tags):
        hello_str = request.args.get('helloStr')
        print(hello_str)
        return 'published'

if __name__ == "__main__":
    app.run(port=8082)


```

之后在支持分布式追踪的软件UI上（下图是Jaeger UI），就可以看到类似下图的追踪信息。我们可以看到服务hello-word和三个操作say-hello/format/println的详细追踪信息

Zipkin 和 Jaeger是目前用的比较多的开源分布式tracing系统

Jaeger基于分布式的架构设计，主要包含以下几个组件：

Jaeger Client，负责在客户端收集跟踪信息。
Jaeger Agent，负责和客户端通信，把收集到的追踪信息上报个收集器 Jaeger Collector
Jaeger Colletor把收集到的数据存入数据库或者其它存储器
Jaeger Query 负责对追踪数据进行查询
Jaeger UI负责用户交互
这个架构很像ELK，Collector之前类似Logstash负责采集数据，Query类似Elastic负责搜索，而UI类似Kibana负责用户界面和交互。这样的分布式架构使得Jaeger的扩展性更好，可以根据需要，构建不同的部署。


Zipkin，Pinpoint，SkyWalking，CAT的对比分析，[这篇很有用](https://www.jianshu.com/p/0fbbf99a236e)


其中Pinpoint 和 SkyWalking用到了 java探针，字节码增强，CAT是代码埋点，有侵入性

Pinpoint 提供有 Java Agent 探针，通过字节码注入的方式实现调用拦截和数据收集，可以做到真正的代码无侵入，只需要在启动服务器的时候添加一些参数，就可以完成探针的部署，Pinpoint 的后端存储基于 HBase

Zipkin 的 Java 接口实现 Brave，只提供了基本的操作 API，如果需要与框架或者项目集成的话，就需要手动添加配置文件或增加代码，Zipkin 基于 Cassandra

两者都是将服务调用拆分成若干有级联关系的 Span，通过 SpanId 和 ParentSpanId 来进行调用关系的级联；最后再将整个调用链流经的所有的 Span 汇聚成一个 Trace，报告给服务端的 collector 进行收集和存储

即便在这一点上，Pinpoint 所采用的概念也不完全与那篇论文一致。比如他采用 TransactionId 来取代 TraceId，而真正的 TraceId 是一个结构，里面包含了 TransactionId, SpanId 和 ParentSpanId。而且 Pinpoint 在 Span 下面又增加了一个 SpanEvent 结构，用来记录一个 Span 内部的调用细节（比如具体的方法调用等等），因此 Pinpoint 默认会比 Zipkin 记录更多的跟踪数据。但是理论上并没有限定 Span 的粒度大小，所以一个服务调用可以是一个 Span，那么每个服务中的方法调用也可以是个 Span，这样的话，其实 Brave 也可以跟踪到方法调用级别，只是具体实现并没有这样做而已

`字节码注入 vs API 调用`

重点输出！！

Pinpoint 实现了基于字节码注入的 Java Agent 探针，而 Zipkin 的 Brave 框架仅仅提供了应用层面的 API，但是细想问题远不那么简单。字节码注入是一种简单粗暴的解决方案，理论上来说无论任何方法调用，都可以通过注入代码的方式实现拦截，也就是说没有实现不了的，只有不会实现的。但 Brave 则不同，其提供的应用层面的 API 还需要框架底层驱动的支持，才能实现拦截。比如，MySQL 的 JDBC 驱动，就提供有注入 interceptor 的方法，因此只需要实现 StatementInterceptor 接口，并在 Connection String 中进行配置，就可以很简单的实现相关拦截；而与此相对的，低版本的 MongoDB 的驱动或者是 Spring Data MongoDB 的实现就没有如此接口，想要实现拦截查询语句的功能，就比较困难。
因此在这一点上，Brave 是硬伤，无论使用字节码注入多么困难，但至少也是可以实现的，但是 Brave 却有无从下手的可能，而且是否可以注入，能够多大程度上注入，更多的取决于框架的 API 而不是自身的能力。

`难度及成本`

经过简单阅读 Pinpoint 和 Brave 插件的代码，可以发现两者的实现难度有天壤之别。在都没有任何开发文档支撑的前提下，Brave 比 Pinpoint 更容易上手。Brave 的代码量很少，核心功能都集中在 brave-core 这个模块下，一个中等水平的开发人员，可以在一天之内读懂其内容，并且能对 API 的结构有非常清晰的认识。

Pinpoint 的代码封装也是非常好的，尤其是针对字节码注入的上层 API 的封装非常出色，但是这依然要求阅读人员对字节码注入多少有一些了解，虽然其用于注入代码的核心 API 并不多，但要想了解透彻，恐怕还得深入 Agent 的相关代码，比如很难一目了然的理解 addInterceptor 和 addScopedInterceptor 的区别，而这两个方法就是位于 Agent 的有关类型中。

因为 Brave 的注入需要依赖底层框架提供相关接口，因此并不需要对框架有一个全面的了解，只需要知道能在什么地方注入，能够在注入的时候取得什么数据就可以了。就像上面的例子，我们根本不需要知道 MySQL 的 JDBC Driver 是如何实现的也可以做到拦截 SQL 的能力。但是 Pinpoint 就不然，因为 Pinpoint 几乎可以在任何地方注入任何代码，这需要开发人员对所需注入的库的代码实现有非常深入的了解，通过查看其 MySQL 和 Http Client 插件的实现就可以洞察这一点，当然这也从另外一个层面说明 Pinpoint 的能力确实可以非常强大，而且其默认实现的很多插件已经做到了非常细粒度的拦截。

针对底层框架没有公开 API 的时候，其实 Brave 也并不完全无计可施，我们可以采取 AOP 的方式，一样能够将相关拦截注入到指定的代码中，而且显然 AOP 的应用要比字节码注入简单很多。

以上这些直接关系到实现一个监控的成本，在 Pinpoint 的官方技术文档中，给出了一个参考数据。如果对一个系统集成的话，那么用于开发 Pinpoint 插件的成本是 100，将此插件集成入系统的成本是 0；但对于 Brave，插件开发的成本只有 20，而集成成本是 10。从这一点上可以看出官方给出的成本参考数据是 5:1。但是官方又强调了，如果有 10 个系统需要集成的话，那么总成本就是 10 * 10 + 20 = 120，就超出了 Pinpoint 的开发成本 100，而且需要集成的服务越多，这个差距就越大。

`Tracing和Monitor区别`

Monitor可分为系统监控和应用监控。系统监控比如CPU，内存，网络，磁盘等等整体的系统负载的数据，细化可具体到各进程的相关数据。这一类信息是直接可以从系统中得到的。应用监控需要应用提供支持，暴露了相应的数据。比如应用内部请求的QPS，请求处理的延时，请求处理的error数，消息队列的队列长度，崩溃情况，进程垃圾回收信息等等。Monitor主要目标是发现异常，及时报警。

Tracing的基础和核心都是调用链。相关的metric大多都是围绕调用链分析得到的。Tracing主要目标是系统分析。提前找到问题比出现问题后再去解决更好。

Tracing和应用级的Monitor技术栈上有很多共同点。都有数据的采集，分析，存储和展式。只是具体收集的数据维度不同，分析过程不一样。
