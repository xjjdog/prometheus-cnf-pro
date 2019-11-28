# 昔日教人类用火的prometheus，如今在努力报警

>原创：小姐姐味道（微信公众号ID：xjjdog），欢迎分享，转载请保留出处。

像我这么热爱野外生活的人，初冬时节，还找了个隐蔽的地方去野炊。现在的社会，为了找找到这么一个静谧的存在，我可谓煞费苦心。

初冬的夜，连虫鸣声都没有，星空高而深远。蜷缩在篝火旁边，我想起了普罗米修斯。在希腊神话中，他教会人类学会使用火，彻底告别了茹毛饮血的年代。

早在2012年，还有一部叫做《普罗米修斯》的电影上演，它是《异行》的前传，其壮丽宏大的场景让人印象深刻。

在这无尽的时空和未知的领域面前，我一个小小程序员，真是连屁都不如。比我强大的比我用功，比如立下flag学python的`潘总`，他应该能勉强算得上个屁。

普罗米修斯的英文是`prometheus`，从这拗口的名字就可以看出，它是个舶来品。prometheus是google内部监控报警系统的开源版本，现在非常流行。

监控系统是老生常谈的问题了，xjjdog在以前也专门总结过。[《这么多监控组件，总有一款适合你》](https://mp.weixin.qq.com/s/rdD54-zyapjlubM5K1jUeQ)，这篇长文，从多个维度上介绍了监控体系，而prometheus面向的就是`metric`类型的数据监控。

今天，我们要着重介绍的，就是prometheus。来势汹汹，大有一统天下的架势。本篇文章主要是为了引起你的兴趣，并提供了很多实践过的配置文件。​说了这么多废话，就让我们正式开始吧。

## 1、介绍

你在使用一些类似于grafana的展示组件时，能够发现底层的数据存储，能够使用Prometheus，而这两个东西明显没有血缘关系。在享受grafana至高无上的美丽时，应该认识到一个现实：现在的监控系统，都拆分成了非常具体的细小模块，让你可以根据经验和习惯进行精细化选择。

Prometheus生态系统由多个组件组成，其中一些是可选的。多数Prometheus组件由Go语言编写，使这些组件很容易编译和部署。网上很多文章翻译的晦涩难懂，xjjdog在这里便使用人话描述。

>注：敲黑板，带★是重点，要考 。不学没法用

★1）`Prometheus Server`：主要负责数据`采集`和`存储`，提供PromQL`查询语言`的支持。注意，它同时是一个存储！

2）客户端SDK：支持非常多语言的类库，越多越好。

3）`Push Gateway`：Prometheus获取监控数据的主要方式是`拉取`模式，但总有一些瞬时发生的监控项，这种信息无法pull。所以这个组件是为了支持一些存活时间很短的事件，把这些信息进行缓冲。

4）`PromDash`：使用Rails开发可视化的Dashboard，用于可视化指标数据

★5）`Exporter`：数据采集组件，也就是一些agent。负责从目标处搜集数据，并将其转化为Prometheus支持的格式

★6）`Alertmanager`：报警管理器，用于发送到正确的逻辑分组。常见的接收方式有：电子邮件，pagerduty，OpsGenie, webhook 等

7）`prometheus_cli`：命令行工具

8）其他辅助性工具：多种导出工具，可以支持Prometheus存储数据转化为HAProxy、StatsD、Graphite等工具所需要的数据存储格式
![](media/15738700687914/15740579247067.jpg)

长篇大论的扯了一通，还是逃脱不了收集、处理、展示三大部件。看了以上的介绍，你应该能够看懂这张官方图。看那些大框框，不要关注细节。

我们来抽取一下比较特殊的要点。  
1）它获取监控数值的方式是拉模式
2）它有一个存储数据的时序数据库，查询语言灵活，但不是SQL
3）有SDK、Agent、中间网关三种数据汇总方式
4）可以使用grafana代替它自带的丑八怪界面
5）能够细粒度配置报警，统一管理

# 2、安装和配置

了解了上面的组件，安装配置就顺利的多。先把Prometheus下载下来，然后解压。

```bash
cd /opt
wget -c https://github.com/prometheus/prometheus/releases/download/v2.14.0/prometheus-2.14.0.linux-amd64.tar.gz
tar zxvf prometheus-2.14.0.linux-amd64.tar.gz
```

## 2.1、配置server
在安装目下编辑配置文件`prometheus.yml`，我们解释比较重要的部分。很多时候，我们需要监控非常多的组件，比如系统状态、canal、kafka、jvm等等，如果将所有的配置文件都放在这里，势必会又臭又长，所以一般采用子配置文件的方式。注意，配置文件分了两部分，上面的是触发规则，下面的是主机名称，注意区别。

各个被监控的组件，需要自行部署Exporter，然后把http接口暴露出来。
```
alerting:
  alertmanagers:
  - static_configs:
    - targets: ["10.81.28.227:9093"]

# 分文件配置，我们以sys系统参数为例说明
rule_files:
  - "sys_monit.yml"
  - "kafka_monit.yml"
# 抓取配置
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
      labels:
          group: "监控服务端"

  - job_name: 'system'
    #通过文件去动态发现配置
    file_sd_configs:  
    - refresh_interval: 1m 
      files:
       - sys.yml #配置文件路径

  - job_name: 'kafka'
    file_sd_configs:
    - refresh_interval: 1m
      files:
       - kafka.yml
```
再来看一下其中一个子配置文件`sys.yml`，它定义了一批要监控的机器。也就是到什么地方去找数据。
```json
[
   {
    "labels": {
      "job": "shop001.web.pub.pro.ali.dc",
      "instance": "system"
    },
    "targets": [
      "10.174.88.9:9100"
    ]
   }
]
```
然后我们启动Prometheus：
```bash
nohup ./prometheus &
```

## 2.2、报警配置

alertmanager需要单独下载，这种方式真是脑回路惊奇。
```bash
https://github.com/prometheus/alertmanager/releases
```

报警的配置文件分为两部分，其中一部分，放在上面的`prometheus.yml`文件中rule_files模块，用来指定匹配规则；另外一部分，放在alertmanager.yml中，用来指定报警的去向。

比如，我想要把报警发送到臭名昭著的dingding，就可以这么写。
```
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 10m
  receiver: 'pro'
  routes:
  - receiver: 'canal'
    match:
      alertname: 'canal 延迟'


receivers:
- name: 'sys'
  webhook_configs:
  - url: 'http://10.81.28.227:8060/dingtalk/pro/send'

- name: 'canal'
  webhook_configs:
  - url: 'http://10.81.28.227:8060/dingtalk/canal/send'
```

接下来看一下规则部分，比如sys_monit.yml，你应该看到这个过程是怎么运转起来的了。
```
roups:
    - name: netstat_tcp_time_wait
      rules:
      - alert: 主机连接数 time_wait
        expr: netstat_tcp_time_wait > 60000
        for: 5m
        labels:
          status: warning
        annotations:
          summary: "{{$labels.job}}"
          description: "{{ $labels.instance }} time_wait > 60k(当前值：{{ $value}})"
```
如果你安装了dashboard的话，应该能看到这些信息了。但是，每次更改阈值，都需要重启server，这一部分，做的不是很好。另外，yml深层的嵌套信息，在配置繁多的时候，显得比较杂乱。从上面的配置文件就可以看出，这些配置文件的解析，使用的是简单的占位符的方式。在设计报警之前，你需要知道每一个监控项的名称，以及意义。
![](media/15738700687914/15740651435967.jpg)

# 3、其他集成

telegraf是比较好用的数据收集agent。通常，它是通过push的方式汇报监控数据，但是通过加入几行配置，可以把push变成pull（暴露一个专用接口）。主要的配置文件如下：
```
[[outputs.prometheus_client]]
    listen = "0.0.0.0:9100"
```

以下是一个grafana主机监控的效果图。由于它的颜值比prometheus自带的高，所以一般都使用grafana。
![](media/15738700687914/15740676912566.jpg)
但是自带的UI也不是一无是处，比如下面查看整个系统机器状态的页面。
![](media/15738700687914/15740679754513.jpg)
用于调试查询语句的界面等。
![](media/15738700687914/15740680316090.jpg)
另外，prometheus可以很容易的接入以下的系统：

1) springboot。 参见
https://yq.aliyun.com/articles/272542

2） canal接入。在canal.properties 中添加
```
canal.metrics.pull.port=11112
```
>注意：canal版本必须canal 1.1.x系列以上

# End

一个监控系统，主要的难点是生态。prometheus最近几年发展迅速，非常多的组件都对其进行了集成，这对我们来说是比较大的福音。但是，它的配置文件还是有点复杂，尤其是在管理大型机器群的时候，需要频繁的更改配置文件，并不能够做到自动发现。

所以，通常使用prometheus的时候，会配合很多内部系统，写很多的shell脚本，自动更改这些配置进行重启，尽量做到自动化。

由于涉及的配置文件，实在是太多，且配置有相对的难度。我将这些信息，放在了仓库里。有需要的，自行参考。

```bash
https://github.com/xjjdog/prometheus-cnf-pro
```

清单：
1、prometheus配置，规则配置 (11个)
2、alertmanager配置 (1个)
3、telegraf (2个)
4、grafana(5个）

>作者简介：**小姐姐味道**  (xjjdog)，一个不允许程序员走弯路的公众号。聚焦基础架构和Linux。十年架构，日百亿流量，与你探讨高并发世界，给你不一样的味道。我的个人微信xjjdog0，欢迎添加好友，​进一步交流。​
