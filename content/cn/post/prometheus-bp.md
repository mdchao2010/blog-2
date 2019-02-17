---
title: "Prometheus 不完全避坑指南"
date: 2019-02-16T16:00:05+08:00
lastmod: 2019-02-16T16:00:05+08:00
draft: false
keywords: ["prometheus","observability","monitoring","best practive"]
description: ""
tags: ["prometheus","observability","monitoring"]
categories: ["observablitiy"]
---

[Prometheus](https://github.com/prometheus/prometheus) 是一个开源监控系统，它本身已经成为了云原生中指标监控的事实标准，几乎所有 k8s 的核心组件以及其它云原生系统都以 Prometheus 的指标格式输出自己的运行时监控信息。我在工作中也比较深入地使用过 Prometheus，最大的感受就是它非常容易维护，突出一个简单省心成本低。当然，这当中也免不了踩过一些坑，下面就总结一下。

> 假如你没有用过 Prometheus，建议先看一遍 [官方文档](https://prometheus.io/docs/introduction/overview/)

## 接受准确性与可靠性的权衡

Prometheus 作为一个基于指标(Metric)的监控系统，在设计上就放弃了一部分数据准确性：

- 比如在两次采样的间隔中，内存用量有一个瞬时小尖峰，那么这次小尖峰我们是观察不到的；
- 再比如 QPS、RT、P95、P99 这些值都只能估算，无法和日志系统一样做到 100% 准确，下面也会讲一个相关的坑；

放弃一点准确性得到的是更高的可靠性，这里的可靠性体现为架构简单、数据简单、运维简单。假如你维护过 ELK 或其它日志架构的话，就会发现相比于指标，日志系统想要稳定地跑下去需要付出几十倍的机器成本与人力成本。既然是权衡，那就没有好或不好，只有适合不适合，**我推荐在应用 Prometheus 之初就要先考虑清楚这个问题，并且将这个权衡明确地告诉使用方。**

## 首先做好自监控

不知道你有没有考虑过一个问题，其它系统都用 Prometheus 监控起来了，报警规则也设置好了，那 Prometheus 本身由谁来监控？

答案是"另一个监控系统"，而这个监控系统可以是另一个 Prometheus。按照官方的 quickstart 或 helm 部署的 Prometheus 单实例自己监控自己的，我们当然不能指望一个系统挂掉之后自己发现自己挂了。因此我强烈建议**在上生产环境之前，一定要确保至少有两个独立的 Prometheus 实例互相做交叉监控。**交叉监控的配置也很简单，每台 Prometheus 都拉取其余所有 Prometheus 的指标即可。

还有一个点是警报系统(Alertmanager)，我们再考虑一下警报系统挂掉的情况：这时候 Prometheus 可以监控到警报系统挂了，但是因为警报挂掉了，所以警报自然就发不出来，这也是应用 Prometheus 之前必须搞定的问题。这个问题可以通过给警报系统做 HA 来应对。除此之外还有一个经典的兜底措施叫做 ["Dead man's switch"](https://en.wikipedia.org/wiki/Dead_man%27s_switch): 定义一条永远会触发的告警，不断通知，假如哪天这条通知停了，那么说明报警链路出问题了。

## 不要使用 NFS 做存储

如题，Prometheus 维护者也[在 issue 中表示过不支持 NFS](https://github.com/prometheus/prometheus/issues/3534)。这点我们有血泪教训（我们曾经有一台 Prometheus 存储文件发生损坏丢失了历史数据）。

## 尽早干掉维度(Cardinality)过高的指标

根据我们的经验，Prometheus 里有 50% 以上的存储空间和 80% 以上的计算资源(CPU、内存)都是被那么两三个维度超高的指标用掉的。而且这类维度超高的指标由于数据量很大，稍微查得野一点就会 OOM 搞死 Prometheus 实例。

首先要明确这类指标是对 Prometheus 的滥用，类似需求完全应该放到日志流或数仓里去算。但是指标的接入方关注的往往是业务上够不够方便，假如足够方便的话什么都可以往 label 里塞。这就需要我们防患于未然，一个有效的办法是**用警报规则找出维度过高的坏指标，然后在 Scrape 配置里 Drop 掉导致维度过高的 label。**

警报规则的例子：

```yaml
# 统计每个指标的时间序列数，超出 10000 的报警
count by (__name__)({__name__=~".+"}) > 10000
```

"坏指标"报警出来之后，就可以用 `metric_relabel_config` 的 `drop` 操作删掉有问题的 label（比如 userId、email 这些一看就是问题户），这里的配置方式可以查阅文档

对了，这条的关键词是**尽早**，最好就是部署完就搞上这条规则，否则等哪天 Prometheus 容量满了再去找业务方说要删 label，那业务方可能就要忍不住扇你了......

## Rate 类函数 + Recording Rule 的坑

可能你已经知道了 PromQL 里要先 `rate()` 再 `sum()`，不能 `sum()` 完再 `rate()`（不知道也没事，马上讲）。但当 `rate()` 已经同类型的函数如 `increase()` 和 recording rule 碰到一起时，可能就会不小心掉到坑里去。

当时，我们已经有了一个维度很高的指标（只能继续维护了，因为没有**尽早干掉**），为了让大家查询得更快一点，我们设计了一个 Recording Rule，用 `sum()` 来去掉维度过高的 `bad_label`，得到一个新指标。那么只要不涉及到 `bad_label`，大家就可以用新指标进行查询，Recording Rule 如下：

```yaml
sum(old_metric) without (bad_label)
```

用了一段时候后，大家发现 `new_metric` 做 `rate()` 得到的 QPS 趋势图里经常有奇怪的尖峰，但 `old_metric` 就不会出现。这时我们恍然大悟：绕了个弯踩进了 `rate()` 的坑里。

这背后与 `rate()` 的实现方式有关，`rate()` 在设计上假定对应的指标是一个 Counter，也就是只有 incr(增加) 和  reset(归0) 两种行为。而做了 `sum()` 或其他聚合之后，得到的就不再是一个 Counter 了，举个例子，比如 `sum()` 的计算对象中有一个归0了，那整体的和会**下降**，而不是归零，这会影响 `rate()` 中判断 reset(归0) 的逻辑，从而导致错误的结果。写 PromQL 时这个坑容易避免，但碰到 Recording Rule 就不那么容易了，因为不去看配置的话大家也想不到 `new_metric` 是怎么来的。

要完全规避这个坑，可以遵守一个原则：**Recording Rule 一步到位，直接算出需要的值，避免算出一个中间结果再拿去做聚合。**

## 警报和历史趋势图未必 Match

最近半年常常被问两个问题：

* 我的历史趋势图看上去超过水位线了，警报**为什么没报**？
* 我的历史趋势图看上去挺正常的，警报**为什么报了**？

这其中有一个原因是：趋势图上每个采样点的采样时间和警报规则每次的计算时间不是严格一致的。当时间区间拉得比较大的时候，采样点非常稀疏，不如警报计算的间隔来得密集，这个现象尤为明显，比如时序图采样了 0秒，60秒，120秒三个点。而警报在15秒，30秒，45秒连续计算出了异常，那在图上就看不出来。另外，经过越多的聚合以及函数操作，不同时间点的数据差异会来得越明显，有时确实容易混淆。

这个其实不是问题，碰到时将趋势图的采样间隔拉到最小，仔细比对一下，就能验证警报的准确性。而对于聚合很复杂的警报，可以先写一条 Recording Rule, 再针对 Recording Rule 产生的新指标来建警报。这种范式也能帮助我们更高效地去建分级警报（超过不同阈值对应不同的紧急程度）

## Alertmanager 的 group_interval 会影响 resolved 通知

Alertmanager 里有一个叫 group_interval 的配置，用于控制同一个 group 内的警报最快多久通知一次。这里有一个问题是 firing(激活) 和 resolved(已消除) 的警报通知是共享同一个 group 的。也就是说，假设我们的 group_interval 是默认的 5 分钟，那么一条警报激活十几秒后立马就消除了，它的消除通知会在报警通知的 5 分钟之后才到，因为在发完报警通知之后，这个 Group 需要等待 5 分钟的 group_interval 才能进行下一次通知。

这个设计让"警报消除就立马发送消除通知"变得几乎不可能，因为假如把 group_interval 变得很小的话，警报通知就会过于频繁，而调大的话，就会拖累到消除通知。

这个问题修改一点源码即可解决，不过无伤大雅，不修也完全没问题。

## 最后一条：不要忘记因何而来

最后一条撒点鸡汤：**监控的核心目标还是护航业务稳定，保障业务的快速迭代，永远不要忘记因何而来**

曾经有一端时间，我们追求"监控的覆盖率"，所有系统所有层面，一定要有指标，而且具体信息 label 分得越细越好，最后搞出几千个监控项，不仅搞得眼花缭乱还让 Prometheus 变慢了；

还有一段时间，我们追求"警报的覆盖率"，事无巨细必有要有警报，人人有责全体收警报（有些警报会发送给几十个人）。最后当然你也能预想到了，告警风暴让大家都对警报疲劳了；

这些事情乍看起来都是在努力工作，但其实一开始的方向就错了，监控的目标绝对不是为了达到 xxx 个指标，xxx 条警报规则，这些东西有什么意义？依我看，负责监控的开发就算不是 SRE 也要有 SRE 的心态和视野，不要为监控系统的功能或覆盖面负责（这样很可让导致开发在监控里堆砌功能和内容，变得越来越臃肿越来越不可靠），而要为整个业务的稳定性负责，同时站在稳定性的投入产出比角度去考虑每件事情的性质和意义，不要忘记我们因何而来。

假如你有建议或想法，欢迎在评论区或通过邮件与我讨论，你也可以在 [Aylei Wu@Twitter](https://twitter.com/AyleiWu) 或 [Yelei Wu@Linkedin](https://www.linkedin.com/in/yelei-wu-0850a5141/) 上找到我。