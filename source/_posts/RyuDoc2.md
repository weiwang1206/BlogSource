title: Ryu文档-2-1-第一个Ryu应用
date: 2014/11/8 20:04:42 
tags: 
- Ryu
- Controller
categories: Ryu
---

## 2.1 第一个Ryu应用 ##
如果你想用自己的方式来管理网络设备，比如路由器、交换机等，你可以自己来编写一个Ryu的应用程序来实现你的想法。你的应用程序可以告诉Ryu来如何管理这些设备，然后Ryu通过OpenFlow协议来管理设备。编写Ryu应用程序非常简单，直接通过Python脚本就可以实现了。接下来我们就看看如果写一个简单的Ryu应用。
<!--more-->
## Dumb Switch ##
我们首先来实现一个简单的交换机，交换机将所有的收到的包在各个端口洪泛发送。代码如下

    from ryu.base import app_manager
    from ryu.controller import ofp_event
    from ryu.controller.handler import MAIN_DISPATCHER
    from ryu.controller.handler import set_ev_cls
    class L2Switch(app_manager.RyuApp):
        def __init__(self, *args, **kwargs):
			super(L2Switch, self).__init__(*args, **kwargs)
		@set_ev_cls(ofp_event.EventOFPPacketIn, MAIN_DISPATCHER)
		def packet_in_handler(self, ev):
			msg = ev.msg
			dp = msg.datapath
			ofp = dp.ofproto
			ofp_parser = dp.ofproto_parser

			actions = [ofp_parser.OFPActionOutput(ofp.OFPP_FLOOD)]
			out = ofp_parser.OFPPacketOut(
			datapath=dp, buffer_id=msg.buffer_id, in_port=msg.in_port,
			actions=actions)
			dp.send_msg(out)

接下来，我们分析一下这段代码。初始化函数，没有做实际的事情，不多说。我们来看一下packet\_in\_handler函数。每当Ryu收到OpenFlow协议中的packet\_in消息时，这个函数就会被调用。原因一定是这个函数被注册为回调函数了。在这里的关键就是set\_ev\_cls装饰器。其实这个函数就是将packet\_in\_handler函数注册到packet\_in消息上，每当有packet\_in消息就调用该函数，使用装饰器使得代码显得比较简洁优美。我们来看一下这个装饰器的参数:

- 第一个参数是指定触发函数被调用的事件，这里即Packet\_in事件。
- 第二个参数是指定交换机的状态。比如，当交换机处于与控制器协商（negotiation）阶段时,可能你想忽略此时的packet\_in消息，那我们就可以使用MAIN\_DISPATCHER作为参数来表明当协商完成后该函数才被调用。

接下来我们来分析packet\_in\_handler的函数体：
ev.msg是packet\_in的消息对象，存储着消息的数据部分
msg.dp代表发送该消息的交换机（datapath）
dp.ofproto是交换机和控制器协商好的OF协议
dp.ofproto\_parser是OF协议的解释器，用于解析和封装符合OF协议的数据。
OFPActionOutput函数用来生成一个action，该action指定包输出的转发端口。在我们这个应用中，交换机将包发送没每一个端口，因此使用OFPP_FLOOD常量作为参数。
OFPPacketOut函数用来生成packet\_out包
send\_msg函数将包发送给相关的交换机

到这儿，我们就完成了我们第一个Ryu应用程序。我们可以将其保存为l2.py文件，然后执行如下命令就启动我们的应用了。

	% ryu-manager l2.py
	loading app l2.py
	instantiating app l2.py

