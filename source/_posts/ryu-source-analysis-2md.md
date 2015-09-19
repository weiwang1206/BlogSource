title: Ryu代码解析（二）
date: 2014/12/1 0:20:33 
tags: 
- Ryu
- Controller
categories: Ryu
---


## Ryu事件处理函数的挂接方式分析 ##

Ryu支持用户自定义事件处理函数，当该事件发生时，用户定义的处理函数会被自动的调用。那么这个机制具体是如何实现的呢？本篇博客就针对该问题，做一个简单的梳理，才疏学浅，欢迎指正！

首先来看RYU文档中的那个第一个APP程序：

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

<!--more-->
这段代码中有两个关键的地方，第一是L2Switch类继承了app\_manager.RyuAPP类（这是自定义APP要求的）。第二个使用set_ev_cls装饰器装饰了我们自定义的事件处理函数packet\_in\_handler.

首先，我们来研究一下set\_ev\_cls装饰器到底做了些什么事情？ set\_ev\_cls函数定义如下：

	def set_ev_cls(ev_cls, dispatchers=None):
	    def _set_ev_cls_dec(handler):
	        if 'callers' not in dir(handler):
	            handler.callers = {}
	        for e in _listify(ev_cls):
	            handler.callers[e] = _Caller(_listify(dispatchers), e.__module__)
	        return handler
	    return _set_ev_cls_dec

首先这个装饰器是带参数的装饰器，在使用@set\_ev\_cls修饰packet\_in\_hander函数是，实际的过程是：

**packet\_in\_handler=set\_ev\_cls(ofp\_event.EventOFPPacketIn, MAIN\_DISPATCHER)(packet\_in\_handler)**

其实就是将packet\_in\_handler作为参数传入了\_set\_ev\_cls\_dec函数，做了一些事情之后，又把packet\_in\_handler返回回来。那么\_set\_ev\_cls\_dec函数做了什么呢，其实很简单。

	 if 'callers' not in dir(handler): 
	            handler.callers = {}
	        for e in _listify(ev_cls):
	            handler.callers[e] = _Caller(_listify(dispatchers), e.__module__)

首先判断handler函数中有没有callers属性，函数还可以有属性？是的，举个例子

	def foo():
		print "I am a function"

	foo.attr = 1
	print foo.attr

结果会输出1.

因为一个handler可能会被多个装饰器装饰，所以先判断有没有该属性，没有的话新建一个字典。ev\_cls就是传进来的ofp\_event.EventOFPPacketIn类，OPF事件类是在不同的OF协议中实现的（如在OF1.4文件ofproto\_1\_4\_parser中）。这里他把事件类转化为list，这里不懂为什么要转为为list，难道还有多个子事件？不过怎么样，这里将在caller属性中添加一个元素，key是事件类型，value是一个\_Caller对象，该对象主要就是记下dispatchers，和这个事件的模块名。

	class _Caller(object):
	    """Describe how to handle an event class.
	    """
	
	    def __init__(self, dispatchers, ev_source):
	        """Initialize _Caller.
	
	        :param dispatchers: A list of states or a state, in which this
	                            is in effect.
	                            None and [] mean all states.
	        :param ev_source: The module which generates the event.
	                          ev_cls.__module__ for set_ev_cls.
	                          None for set_ev_handler.
	        """
	        self.dispatchers = dispatchers
	        self.ev_source = ev_source

到这里set\_ev\_cls装饰器就分析完了，回想一下它到底做了什么？其实很简单，就是给handler函数添加了一个属性callers，然后在callers中保存了该handler关心的事件类型和dispatchers。

其实到这里，并没有和系统运行过程中的消息路由关联起来，我们接着往下看。

还记得在[代码分析(一)](http://geekwei.com/2014/11/27/ryu-source-analysis-1/#more)中讲的main函数么，该函数在加载了app之后，调用了instantiate\_apps()函数,该函数对每个app进行初始化。

	def instantiate_apps(self, *args, **kwargs):
        for app_name, cls in self.applications_cls.items():
            self._instantiate(app_name, cls, *args, **kwargs)

        self._update_bricks()
        self.report_bricks()

        threads = []
        for app in self.applications.values():
            t = app.start()
            if t is not None:
                threads.append(t)
        return threads

在该函数中，首先调用\_instantiate实例化app，并且调用regeister\_app()函数，

	def register_app(app):
	    assert isinstance(app, RyuApp)
	    assert app.name not in SERVICE_BRICKS
	    SERVICE_BRICKS[app.name] = app
	    register_instance(app)

该函数有两行关键代码，第一将app加入 SERVICE_BRICKS中， SERVICE_BRICKS是一个全局变量，记录着所有的app实例，之后的代码中会用到它，暂且记住。之后调用register\_instance(app)函数（该函数在handler.py中定义）
	
	def register_instance(i):
	    for _k, m in inspect.getmembers(i, inspect.ismethod):
	        # LOG.debug('instance %s k %s m %s', i, _k, m)
	        if _has_caller(m):
	            for ev_cls, c in m.callers.iteritems():
	                i.register_handler(ev_cls, m)

	def register_handler(self, ev_cls, handler):
        assert callable(handler)
        self.event_handlers.setdefault(ev_cls, [])
        self.event_handlers[ev_cls].append(handler)

该函数首先使用inspect遍历该app中的所有方法，然后通过\_has\_caller方法判断该方法是否有“caller”属性，上文中介绍了使用装饰器函数set\_ev\_cls时为每一个handler函数添加了caller属性，因此这里就是遍历所有的handler函数，将callers中保存的事件及函数（ev\_cls,m）保存在app类的event\_handlers属性中。

接下来调用\_update\_bricks函数, 这个函数非常重要

	 def _update_bricks(self):
        for i in SERVICE_BRICKS.values():
            for _k, m in inspect.getmembers(i, inspect.ismethod):
                if not hasattr(m, 'callers'):
                    continue
                for ev_cls, c in m.callers.iteritems():
                    if not c.ev_source:
                        continue

                    brick = _lookup_service_brick_by_mod_name(c.ev_source)
                    if brick:
                        brick.register_observer(ev_cls, i.name,
                                                c.dispatchers)

                    # allow RyuApp and Event class are in different module
                    for brick in SERVICE_BRICKS.itervalues():
                        if ev_cls in brick._EVENTS:
                            brick.register_observer(ev_cls, i.name,
                                                    c.dispatchers)

这里有一个技巧，其实c.ev_source如果是的module那么就是ofp\_event,而OFPHandler类的名字正好就是'ofp\_event'（在下文中会看到其实就是OpenFlowController）,所以这里的brick就是OFPHandler，然后将将每种ev\_cls的类型和app名字注册到该类中，其实本质上就是OFPHandler作为了一个消息源。这个函数非常重要，将每个app的handler与消息源OFPHandler建立了联系

还是没将其与系统消息路由挂起来啊！接着往下看

在main函数中ryu会将所有app都启动，启动后就进入\_event\_loop循环
	def _event_loop(self):
        while self.is_active or not self.events.empty():
            ev, state = self.events.get()
            if ev == self._event_stop:
                continue
            handlers = self.get_handlers(ev, state)
            for handler in handlers:
                handler(ev)。

其中一个特殊的app是opf\_handler app，其重写了start函数，其实调用start后就是启动OpenFlowController类，OpenFlowController启动后，ryu开始监听来自交换机的新连接，当有一个新连接时，就调用datapath\_connection\_factory函数创建一个新的datapath对象，对象中保存着对端交换机的通信socket和address。如果交换机发来新的消息，该datapath对象在\_recv\_loop方法中就会获取该消息，

	def _recv_loop(self):
        buf = bytearray()
        required_len = ofproto_common.OFP_HEADER_SIZE

        count = 0
        while self.is_active:
            ret = self.socket.recv(required_len)
            if len(ret) == 0:
                self.is_active = False
                break
            buf += ret
            while len(buf) >= required_len:
                (version, msg_type, msg_len, xid) = ofproto_parser.header(buf)
                required_len = msg_len
                if len(buf) < required_len:
                    break

                msg = ofproto_parser.msg(self,
                                         version, msg_type, msg_len, xid, buf)
                # LOG.debug('queue msg %s cls %s', msg, msg.__class__)
                if msg:
                    ev = ofp_event.ofp_msg_to_ev(msg)
    **********************************我们关注这里 开始************************************
                    self.ofp_brick.send_event_to_observers(ev, self.state)
    **********************************我们关注这里 结束************************************
                    dispatchers = lambda x: x.callers[ev.__class__].dispatchers
                    handlers = [handler for handler in
                                self.ofp_brick.get_handlers(ev) if
                                self.state in dispatchers(handler)]
                    for handler in handlers:
                        handler(ev)
    
                buf = buf[required_len:]
                required_len = ofproto_common.OFP_HEADER_SIZE

                # We need to schedule other greenlets. Otherwise, ryu
                # can't accept new switches or handle the existing
                # switches. The limit is arbitrary. We need the better
                # approach in the future.
                count += 1
                if count > 2048:
                    count = 0
                    hub.sleep(0)


首先通过lambda表达式定义了一个函数dispatchers，该函数获取handler的callers属性中ev事件的dispatchers。

接着使用生成器，首先opf\_brick定义如下：

	 self.ofp_brick = ryu.base.app_manager.lookup_service_brick('ofp_event')

还记得我们上边提到的SERVICE\_BRICK么，lookup\_service\_brick('ofp_event')函数

	def lookup_service_brick(name):
    	return SERVICE_BRICKS.get(name)

ofp\_brick就是ofp\_hander对象，然后调用send\_event\_to\_observers函数，将事件发送到关心该事件和该状态（由dispachers判断）的类中。

	def send_event_to_observers(self, ev, state=None):
        """
        Send the specified event to all observers of this RyuApp.
        """

        for observer in self.get_observers(ev, state):
            self.send_event(observer, ev, state)

这里就用到了上文中提到的通过\_update\_brick函数填充的ovservers，然后通过send\_event函数发送到每个ovserver的app的event队列中，然后每个app在其\_event\_loop中就可以获取该事件了！

