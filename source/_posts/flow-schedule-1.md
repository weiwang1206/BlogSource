title: 数据中心网络Flow Schedule
date: 2013-12-01 20:14:10
tags: 
- Network
- DataCenter Network
- Flow Schedule
categories: Research
---


当今各大互联网公司的访问量呈爆炸式增长，巨大的访问量对其DataCenter造成巨大的压力，尤其是数据中心内部网络。数据中心现有的业务很多也是Deadline-aware的，比如Web search，social network，recommendation system等，每多1ms的延迟，就会造成很大的损失，这不是危言损听。而且这些业务也都有SLA（Service Level Agreement）的要求，大概数字在300ms左右[D2TCP]。因此，保证DC内部网络的软实时性很重要。

<!--more-->

**Datacenter Network 背景及Traffic特征**

（比较零碎，等形成完整的思路再整合）


- 内部拓扑基本上就是Spine-Leaf，一个机架上的所有主机连接在一个ToR（Top of Rack）上，这些交换机连接在每个core switch上，这样做的好处是不让上层的switch成为bottleneck，并且提供了multipath

![spine_leaf](/img/spine_leaf.png)




- 现在的数据中心99%的traffic跑着TCP连接[DCTCP]


- traffic中绝大多数是deadline aware short flow；也有少数的elephant long flow，但没有严格的deadline，占据着较大的带宽；两种类型的flow混合着争夺着现有的带宽资源。


- 带宽情况，edge switch出口1G最多10G，core switch 10G。


- switch的buffer很小，大概在百K左右，上M的少，不过Cisco有16M的，但是总体来说不大。尤其是数据中心里都是普通commodity switch。而且这个仅有的buffer也是各端口共享的。buffer用来干嘛？queue！


- traffic pattern主要是partition-aggregation，比如web search和hadoop。这就带来了incast问题。


**传统TCP为什么不能胜任**

这个主要还是没有很好的解决incast的问题，引起这个的原因是如果一个switch同时接受了很多flow，超过buffer之后肯定是要丢包的，丢包会引起TCP重传，但是重传是由RTO控制的，RTO长了延时过长，RTO短了既浪费带宽又导致队列长度过长也增加了延时。

而且基本上对网络资源的利用也是fair sharing，在switch里基本就是FCFS。

**Related Work**

在学术界，近几年对这个问题比较关注，都在flow schedule方面发文章，从sigcom09到sigcomm13都有相关的文章。

不过大约分为两类

一类是不关注路径，single path，但是关注congestion contrll，rate controll，及交换机根据优先级进行处理

一类是类似Load Balance的，专注multipath。

**第一类代表文章**

----------

DCTCP （2010） --> D3 （2011） --> D2TCP & PDQ & Detail （2012） --> pFabric & OVN （2013）


----------

分别对每篇文章的方法，优势和劣势做下简单总结（等慢慢丰富）：

**DCTCP**

方法：使用优化的ECN，就是使用连续几个EC bit来计算网络拥塞程度，来调节TCP的window size从而控制拥塞，switch保持较低的值。

优势：充分分析了DCN traffic，较早提出这个方法

劣势：fair shareing，不关注deadline，无法保证SLA

----------

**D3**

方法：使用贪心的方法，根据deadline和remaining flow size来确定一个rate，然后由switch为每个flow预留带宽。

优势：提出使用deadline aware的方法保证按时交付和FCT

缺陷：使用贪心的方法，导致先来的flow可能阻碍稍微后来一点的更重要的flow的延迟。先来先服务。

----------


**PDQ**

方法：提出抢占式，后来的高优先级的可以是之前的flow先pause，就是针对D3的问题解决的。

优势：抢占式很好，但是还有有些小问题，比如flow平滑过度，utilization等问题

劣势：没看出来，在pFabric里说不好实现之类的

----------

**第二类代表文章**

**Hedera**

Hedera关注的是数据中心内部不同机架之间的流量调度问题，和DCTCP，D3等关注的问题不太一样。Hedera认为inter-rack存在网络瓶颈。而且解决思路也不同，举个例子来说，第一类方法是红绿灯，在拥堵的路口指挥那辆车先走并且控制来这里的车辆的速率，Hedera是线路规划局，直接指挥车辆走别的路。

方法：充分利用数据中心equal cost的特性，预估流的带宽需求，设计优化的flow schedule算法，将流调度到带宽压力较小的switch上。这种需要全局信息来做调度，可以充分利用SDN的思想，做全局优化。

优势：比ECMP静态的路径要有优势，可以充分利用多条路径，避开繁忙的链路

缺陷：流在过程中会改变路径，调度算法太简单，没有抢占，没有优先级，在共享内存的交换机中，简单的流调度可能不好，因为elephant background flow可能占据了大量的内存，这样的flow应该为deadline aware的流做出让步。Flow的带宽预测有些weight，其实根据deadline和flow size就可以确定一个流的重要性了。Schedule的运行频率也是一个影响performance的因素，频率太小，效果不明显，频率高造成浪费资源。

基于Hedera这里可以写一个Flow schedule算法

**Per-packet Load-balanced low latency routing for clos-based data center Networks**

这篇文章的核心思想是为Clos based network（比如Fat tree，VL2）提供packet level的load balanced算法，这个算法DRB在server端选择一个core switch作为其中转的destination（IP in IP，在core switch 解包发到真正的destination），并且根据拓扑的特性，确定唯一的路径，使用静态路由

缺点：re-odering，switch解包，需要server知道拓扑状态尤其是所有的core switch（其实这一步感觉在网络中做更好，host端做好rate control就好了），静态路由扩展性较差

（未完待续）