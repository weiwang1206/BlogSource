title: Ryu文档-2-3-Ryu组件
date: 2014/11/26 22:21:38 
tags: 
- Ryu
- Controller
categories: Ryu
---

# 2.3 Ryu应用的API #
## 2.3.1 Ryu应用的编程模型 ##

ryu应用是单线程的实体，应用之间通过消息彼此通信。

ryu应用之间互相发送异步的消息。ryu的OpenFlow控制器不属于ryu应用，但也是产生消息的源头。虽然ryu的消息可以包含任意的python objects，但是并不推荐在ryu应用之间传递较大的实体。
<!--more-->
每个ryu应用有一个消息的接收队列，是一个FIFO队列。每个ryu应用都有一个事件处理线程，该线程不断的从接收队列冲取出消息，并根据消息类型调用相应的消息处理函数。消息处理函数似乎在实现处理线程的上下文环境中被调用的，因此必须做好阻塞。

ryu有很多类型的消息类型来实现ryu应用之间的异步消息。

contexts是ryu应用公用的一个objects，因此不要用于新的代码中。
（意思contexts这个变量是内部变量？不要随便用？）

## 2.3.2 创建一个Ryu应用##
一个ryu应用是作为一个python模块的，并且继承了ryu.base.app\_manager.RyuApp类。如果两个相同的ryu应用被定义在一个模块中，那么app\_manager将选择第一个出现的。ryu应用是单例的，只允许有一个给定的ryu应用。

## 2.3.3 如何获取ryu消息##
ryu应用通过注册的方式与消息挂钩，通过ryu.controller.handler.set\_ev\_cls 装饰器来注册某一个时间的处理函数。该装饰器第一个参数定义注册的消息类型，第二个参数可以指定一个接收的时间阶段（如下图），来指定在什么时候才为这个处理函数生成消息。

![phase](/img/ryudoc4phase.jpg)

## 2.3.4 如何生成ryu消息##
ryu应用可以使用send\_event 或者 send\_event\_to\_observers 函数发送消息

## 2.3.5 事件类##
事件类描述了ryu系统中的事件。ryu中所有的事件类都已“Event”作为前缀。消息事件可以由ryu的核心组件生成，也可以有ryu应用生成。

ryu.controller.ofp\_event 模块定义了openflow消息的事件类。该类消息被规范的命名为ryu.controller.ofp\_event.EventOFPxxxx。例如EventOFPPacketIn是OF的packet-in消息。OpenFlow控制器自通的解析Openflow消息并将消息时间发送给关心该消息类型的应用。

## 2.3.6 ryu.controller.controller.Datapath##

该类定义了与控制器相连的交换机的信息，类中定义的属性如下。

![datapath](/img/ryudoc4dp.jpg)


## 2.3.7 ryu.controller.event.EventBase##
这个类是所有事件类的基类。ryu应用可以继承这个类来自定义事件类型

## 2.3.8 ryu.controller.event.EventRequestBase##
请求事件的基类

## 2.3.9 ryu.controller.event.EventReplyBase##
应答事件基类

## 2.3.10 ryu.controller.ofp_event.EventOFPStateChange##
状态变更事件

## 2.3.11 ryu.controller.dpset.EventDP##
当交换机连接到控制器或者断开与控制器的连接时产生该事件，ryu.controller.ofp_event.EventOFPStateChange 类也可以获取该事件。从该事件中可以获取关联该事件的交换机dp。

## 2.3.12 ryu.controller.dpset.EventPortAdd（EventPortDelete、EventPortModify）##
当一个交换机添加（删除，修改）新端口时产生该事件，事件包含的信息包括dp和port

## 2.3.13 ryu.controller.network.EventNetworkPort##
这个时间跟一个网络相关（可能是一个虚拟网络？），当网络中添加或者删除一个端口时，产生该事件。该事件可以获取的信息包括，network_id, dpid, port_no, 以及add_del(是添加还是删除)

## 2.3.14 ryu.controller.network.EventNetworkDel##
当一个网络被删除时产生该事件，可以获取的信息是network_id

## 2.3.15 ryu.controller.network.EventMacAddress##
当一个端口的MAC地址添加时，产生该事件，可以获取的信息包括network_id dpid port_no mac_address add_del

## 2.3.16 ryu.controller.tunnels.EventTunnelKeyAdd(tunnels.EventTunnelKeyDel)##
当一个隧道的key被注册或者更新(删除)时，产生该事件。可以获取的信息包括 network_id tunnel_key

## 2.3.17 ryu.controller.tunnels.EventTunnelPort##
当一个隧道的端口被添加或删除时产生该事件，信息包括dpid port_no remote_dpid add_del