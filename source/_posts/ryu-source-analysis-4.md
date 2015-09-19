title: Ryu代码解析（四）
date: 2014/12/4 19:38:14 
tags: 
- Ryu
- Controller
categories: Ryu
---

## Ryu OFPHandler类 ##

	class OFPHandler(ryu.base.app_manager.RyuApp):

OFPHandler类也是RyuApp的子类，但它是一个特殊的app，负责处理OF协议的相关的事件，同时将事件发送给不同的app。

	def __init__(self, *args, **kwargs):
        super(OFPHandler, self).__init__(*args, **kwargs)
		//这里将OFPHandler类的名字定义为ofp_event，后面会用到
        self.name = 'ofp_event' 
<!--more-->
**start函数**：与其他app不同的是，OFPHandler重载了start函数，首先调用父类的start函数，父类的start函数主要是轮询本app中的事件，但是貌似OFPHandler app并通过这个线程处理事件？？然后启动另一个线程OpenFlowController线程，该线程就是ryu控制器的监听线程，负责接受交换机的连接。

	 def start(self):
		//调用基类RyuApp的start函数，启动轮询事件的线程
        super(OFPHandler, self).start() 
		//启动OpenFlowController线程
        return hub.spawn(OpenFlowController())

我们知道，普通的Ryu app通过set\_ev_cls装饰器来定义事件的处理函数，尤其是Packet\_In事件，但是对于控制器与交换机建立连接并且彼此交换配置信息时的处理函数是通过set\_ev\_handler来定义的。在OFPHandler中定义了这些处理函数。

**set\_ev\_handler**函数与set\_ev_cls的定义几乎一模一样，但是在为每个事件新建\_Caller对象时，ev_source为空。

	def set_ev_handler(ev_cls, dispatchers=None):
	    def _set_ev_cls_dec(handler):
	        if 'callers' not in dir(handler):
	            handler.callers = {}
	        for e in _listify(ev_cls):
	            handler.callers[e] = _Caller(_listify(dispatchers), None) #只有这一行不同
	        return handler
    return _set_ev_cls_dec

**hello_handler**函数：计算出switch和controller都支持的最大的OF版本，并且发送features\_request请求（来获取交换机的基本性能，详细结构如下图），同时将该交换机的datapath的state设置为（CONFIG_DISPATCHER）。datapath为交换机在控制器中的一个对象，负责记录交换机的属性以及与交换机通信（接受，发送消息），详见[Ryu代码解析（五）](http://geekwei.com/2014/12/09/ryu-source-analysis-5/)

	def hello_handler(self, ev):
        self.logger.debug('hello ev %s', ev)
        msg = ev.msg
        datapath = msg.datapath

        # check if received version is supported.
        # pre 1.0 is not supported
        elements = getattr(msg, 'elements', None)
        if elements:
           //这里省略了代码，这段代码根据Hello消息中的elements来确定OF的版本
        else:
           //这里省略了代码，这段代码根据Hello消息头中的version字段来确定OF版本

        if not usable_versions:
            error_desc = (
                'unsupported version 0x%x. '
                'If possible, set the switch to use one of the versions %s' % (
                    msg.version, sorted(datapath.supported_ofp_version)))
            self._hello_failed(datapath, error_desc)
            return
		
        datapath.set_version(max(usable_versions))

        # now send feature
        features_reqeust = datapath.ofproto_parser.OFPFeaturesRequest(datapath)
        datapath.send_msg(features_reqeust)

        # now move on to config state
        self.logger.debug('move onto config mode')
		//设置交换机 datapath的状态
        datapath.set_state(CONFIG_DISPATCHER)

![Features_Reauest](/img/ryu-source-analysis-4-features.jpg)

**switch\_features\_handler**函数：features回复消息的处理函数，但是没看到保存收到的features信息，只保存了datapath的id。然后发送switch config消息。

	 def switch_features_handler(self, ev):
        msg = ev.msg
        datapath = msg.datapath
        self.logger.debug('switch features ev %s', msg)

        datapath.id = msg.datapath_id

        # hacky workaround, will be removed. OF1.3 doesn't have
        # ports. An application should not depend on them. But there
        # might be such bad applications so keep this workaround for
        # while.
        if datapath.ofproto.OFP_VERSION < 0x04:
            datapath.ports = msg.ports
        else:
            datapath.ports = {}

        ofproto = datapath.ofproto
        ofproto_parser = datapath.ofproto_parser
        set_config = ofproto_parser.OFPSetConfig(
            datapath, ofproto.OFPC_FRAG_NORMAL,
            128  # TODO:XXX
        )
        datapath.send_msg(set_config)


		//version 1.0 = 0x01, 1.2 = 0x03, 1.3 = 0x04, 1.4 = 0x05
        if datapath.ofproto.OFP_VERSION < 0x04:
            self.logger.debug('move onto main mode')
            ev.msg.datapath.set_state(MAIN_DISPATCHER) #这代码写的？？不是有获取好的datapath么
        else:
            port_desc = datapath.ofproto_parser.OFPPortDescStatsRequest(
                datapath, 0) 
            datapath.send_msg(port_desc)

**multipart\_reply\_handler**:port信息处理函数，之所以叫multipart是因为肯能一次消息没传完，记录port的信息，如果完成就进入MAIN\_DISPATCHER状态

    def multipart_reply_handler(self, ev):
        msg = ev.msg
        datapath = msg.datapath
        for port in msg.body:
            datapath.ports[port.port_no] = port

        if msg.flags & datapath.ofproto.OFPMPF_REPLY_MORE:
            return
        self.logger.debug('move onto main mode')
        ev.msg.datapath.set_state(MAIN_DISPATCHER)

**echo\_request\_handler**函数：echo消息处理函数，向交换机发送echo消息

	def echo_request_handler(self, ev):
        msg = ev.msg
        datapath = msg.datapath
        echo_reply = datapath.ofproto_parser.OFPEchoReply(datapath)
        echo_reply.xid = msg.xid
        echo_reply.data = msg.data
        datapath.send_msg(echo_reply)

**error\_msg\_handler**：错误消息处理函数，只做了log

	def error_msg_handler(self, ev):
        msg = ev.msg
        self.logger.debug('error msg ev %s type 0x%x code 0x%x %s',
                          msg, msg.type, msg.code, utils.hex_array(msg.data))