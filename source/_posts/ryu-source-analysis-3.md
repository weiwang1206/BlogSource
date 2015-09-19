
title: Ryu代码解析（三）
date: 2014/12/2 21:35:13 
tags: 
- Ryu
- Controller
categories: Ryu
---

## RyuApp基类 ##

**\_CONTEXTS**

该变量是RyuApp基类中定义的上下文字典，app子类来填充这个变量。该变量用来说明子类想要使用的上下文模块。但是这个上下文模块的初始化是由AppManager来做的，而且相同的上下文模块对象在不同的app子类之间是共享的。例如：

	 _CONTEXTS = {
            'network': network.Network
        }

        def __init__(self, *args, *kwargs):
			// 这里的kwargs就是传进来的共享的上下文，然后获取其中的‘network’来初始化每个子类中的关心的上下文变量
            self.network = kwargs['network'] 
<!--more-->
**\_EVENTS**

A list of event classes which this RyuApp subclass would generate.This should be specified if and only if event classes are defined in a different python module from the RyuApp subclass is.

暂时不太懂怎么用，请懂的朋友留言指点。

**init函数**：初始化成员变量

	def __init__(self, *_args, **_kwargs):
        super(RyuApp, self).__init__()
        self.name = self.__class__.__name__
        self.event_handlers = {}        # ev_cls -> handlers:list 事件处理函数字典key是事件，value是处理函数
        self.observers = {}     # ev_cls -> observer-name -> states:set key是消息类，value是关注该消息的app
        self.threads = [] #线程句柄
        self.events = hub.Queue(128)
        if hasattr(self.__class__, 'LOGGER_NAME'):
            self.logger = logging.getLogger(self.__class__.LOGGER_NAME)
        else:
            self.logger = logging.getLogger(self.name)
        self.CONF = cfg.CONF

        # prevent accidental creation of instances of this class outside RyuApp
        class _EventThreadStop(event.EventBase):
            pass
        self._event_stop = _EventThreadStop()
        self.is_active = True #激活

**start函数**：启动该app，新建一个线程，线程执行\_event\_loop函数

 	def start(self):
        """
        Hook that is called after startup initialization is done.
        """
        self.threads.append(hub.spawn(self._event_loop))

**\_event\_loop函数**： app主线程执行的函数，从消息队列event中取出事件，并根据事件类型调用不同的处理函数

	def _event_loop(self):
        while self.is_active or not self.events.empty():
            ev, state = self.events.get()
            if ev == self._event_stop:
                continue
            handlers = self.get_handlers(ev, state) #从event_handlers成员变量中获得key是ev类型的处理函数，而且处理函数的_dispatcher要符合当前的state才会返回该handler。
            for handler in handlers:
                handler(ev)

**stop函数**：停止app主线程

	def stop(self):
        self.is_active = False
        self._send_event(self._event_stop, None)
        hub.joinall(self.threads)

**register\_handler**:注册处理函数,保存在event\_handlers变量中，key是事件类，value是一个list，保存关注该事件的处理函数，关注同一个事件的处理函数可能有多个所以保存在list中。

	def register_handler(self, ev_cls, handler):
        assert callable(handler)
        self.event_handlers.setdefault(ev_cls, [])
        self.event_handlers[ev_cls].append(handler)

**unregister\_handler**: 注销处理函数，具体操作为从event\_handlers中删除该handler

	def unregister_handler(self, ev_cls, handler):
        assert callable(handler)
        self.event_handlers[ev_cls].remove(handler)
        if not self.event_handlers[ev_cls]:
            del self.event_handlers[ev_cls]
**register\_observer**函数：该函数的主要作用是注册某个事件的观察者app（observer）。观察者的主要用途是，当本app收到一个事件时，会将该事件发送到注册的关心该事件的观察者的event队列中。主要是ofp\_handler调用该函数来注册各种app，从而将其接受到的事件发送给各app

	 def register_observer(self, ev_cls, name, states=None):
        states = states or set()
        ev_cls_observers = self.observers.setdefault(ev_cls, {})
        ev_cls_observers.setdefault(name, set()).update(states)

**unregister\_observer**函数：将app从ev\_cls事件的观察者set中去除。

	def unregister_observer(self, ev_cls, name):
        observers = self.observers.get(ev_cls, {})
        observers.pop(name)

**observe\_event**函数：向提供ev\_cls事件的app注册本app为ev_cls的观察者，这个函数跟上面的不同的是让其他的app注册自己为一个观察者。

	 def observe_event(self, ev_cls, states=None):
        brick = _lookup_service_brick_by_ev_cls(ev_cls)
        if brick is not None:
            brick.register_observer(ev_cls, self.name, states)

**observe\_event**函数：从提供ev\_cls事件的app的观察者列表中删除本app。

	def unobserve_event(self, ev_cls):
        brick = _lookup_service_brick_by_ev_cls(ev_cls)
        if brick is not None:
            brick.unregister_observer(ev_cls, self.name)

get_handlers函数: 该函数主要功能是获取 交换机处于state状态时的ev事件的处理函数。在当某一个app编写一个handler的时候使用se\_ev\_cls装饰器时，就指定了该处理函数处理的事件以及其关心的dispatcher（也就是这里的state），因此当调用get\_handlers函数时，state和处理函数定义的dispatcher不符合时，就不包括该处理函数了。如果state是空，则返回ev的所有处理函数。

	def get_handlers(self, ev, state=None):
        """Returns a list of handlers for the specific event.

        :param ev: The event to handle.
        :param state: The current state. ("dispatcher")
                      If None is given, returns all handlers for the event.
                      Otherwise, returns only handlers that are interested
                      in the specified state.
                      The default is None.
        """
        ev_cls = ev.__class__
		#先不管state，先获取事件的处理函数
        handlers = self.event_handlers.get(ev_cls, [])
        if state is None:
            return handlers
		# 该函数判断处理函数 h 是否关心处于state状态的时间，如果关心则返回true
        def test(h): 
            if not hasattr(h, 'callers') or ev_cls not in h.callers:
                # dynamically registered handlers does not have
                # h.callers element for the event.
                return True
            states = h.callers[ev_cls].dispatchers
            if not states:
                # empty states means all states
                return True
            return state in states

		\#使用filter函数，过滤符合state条件的处理函数，并返回
        return filter(test, handlers)

**get\_getobservers函数：**获取observers中保存的所有的观察者。

	 def get_observers(self, ev, state):
        observers = []
        for k, v in self.observers.get(ev.__class__, {}).iteritems():
            if not state or not v or state in v:
                observers.append(k)

        return observers

**send\_request**函数：发送一个同步的请求。seq是一个EventRequestBase的实例，sync属性设置为真，表示为同步请求，回复结果保存在reply\_q队列中，通过send\_event函数发送给req.dst，然后阻塞自己更待回复。


	def send_request(self, req):
        """
        Make a synchronous request.
        Set req.sync to True, send it to a Ryu application specified by
        req.dst, and block until receiving a reply.
        Returns the received reply.
        The argument should be an instance of EventRequestBase.
        """

        assert isinstance(req, EventRequestBase)
        req.sync = True
        req.reply_q = hub.Queue()
        self.send_event(req.dst, req)
        # going to sleep for the reply
        return req.reply_q.get()

**send\_event**函数: 发送事件ev给名叫name的app实例。

	def send_event(self, name, ev, state=None):
        """
        Send the specified event to the RyuApp instance specified by name.
        """

        if name in SERVICE_BRICKS:
            if isinstance(ev, EventRequestBase):
                ev.src = self.name #发送的事件的源设置为自己
            LOG.debug("EVENT %s->%s %s" %
                      (self.name, name, ev.__class__.__name__))
            SERVICE_BRICKS[name]._send_event(ev, state) #获取名字为name的app实例，调用其_send_event函数将事件加入其事件队列，我觉得函数名应该叫_add_event更加贴切
        else:
            LOG.debug("EVENT LOST %s->%s %s" %
                      (self.name, name, ev.__class__.__name__))

	def _send_event(self, ev, state):
        self.events.put((ev, state)) #将事件加入事件队列

send\_event\_to\_observers函数:将事件发送给observers中的app实例。

	def send_event_to_observers(self, ev, state=None):
        """
        Send the specified event to all observers of this RyuApp.
        """

        for observer in self.get_observers(ev, state):
            self.send_event(observer, ev, state)

reply\_to\_request函数：这个是send_request函数的应答函数，将rep回复发送给req端。

	def reply_to_request(self, req, rep):
        """
        Send a reply for a synchronous request sent by send_request.
        The first argument should be an instance of EventRequestBase.
        The second argument should be an instance of EventReplyBase.
        """

        assert isinstance(req, EventRequestBase)
        assert isinstance(rep, EventReplyBase)
        rep.dst = req.src
        if req.sync:
            req.reply_q.put(rep)
        else:
            self.send_event(rep.dst, rep)

close函数：关闭app函数，什么都没做。

	 def close(self):
        """
        teardown method.
        The method name, close, is chosen for python context manager
        """
        pass