title: Ryu代码解析（五）
date: 2014/12/9 21:17:36 
tags: 
- Ryu
- Controller
categories: Ryu
---

## OpenFlowController类 ##
该类位于ryu\controller\controller.py

OpenFlowController类非常简单，它其实是作为OFPHandler类的一个线程存在的，而不是一个独立app。该类也没有OF协议相关的处理函数，那这个类究竟做什么呢？其实他的工作很简单，就是在著名的6633端口监听来自交换机的连接，并且建立与交换机之间的连接，并且在控制层面创建于其对应的dp类，连接建立后其余的工作就交给OFPHander类处理交换机发来的包了（从Hello包开始）。我们来详细看一下这个类的组成。
<!--more-->
**\_\_call\_\_函数**：这个函数是python的内置函数，该函数在类对象初始化时被自动的调用，从[上一篇博客中](http://geekwei.com/2014/12/04/ryu-source-analysis-4/)我们知道，OFPHandle在start函数中新建了OpenFlowController对象，此时就调用了__call__函数，该函数中调用server_loop函数。


	 def __call__(self):
        # LOG.debug('call')
        self.server_loop()

**server_loop函数**：该函数新建一个监听server，然后开始监听，就这么简单。只不过ryu使用了一个叫做[Gevent](http://www.gevent.org/)的轻量级高并发框架。博主不太了解该框架。我们只关注其功能，可以看到在函数中第一步就是新建一个StreamServer，在其参数中我们只关注datapath\_connection\_factory这个参数，其实他是一个函数句柄，当一个server成功接收一个来自交换机的连接时就调用该函数，后面会讲该函数。新建server之后，调用serve_forever，从函数名可知就是启动服务器开始服务。

	def server_loop(self):
        if CONF.ctl_privkey is not None and CONF.ctl_cert is not None:
            if CONF.ca_certs is not None:
                server = StreamServer((CONF.ofp_listen_host,
                                       CONF.ofp_ssl_listen_port),
                                      datapath_connection_factory,
                                      keyfile=CONF.ctl_privkey,
                                      certfile=CONF.ctl_cert,
                                      cert_reqs=ssl.CERT_REQUIRED,
                                      ca_certs=CONF.ca_certs,
                                      ssl_version=ssl.PROTOCOL_TLSv1)
            else:
                server = StreamServer((CONF.ofp_listen_host,
                                       CONF.ofp_ssl_listen_port),
                                      datapath_connection_factory,
                                      keyfile=CONF.ctl_privkey,
                                      certfile=CONF.ctl_cert,
                                      ssl_version=ssl.PROTOCOL_TLSv1)
        else:
            server = StreamServer((CONF.ofp_listen_host,
                                   CONF.ofp_tcp_listen_port),
                                  datapath_connection_factory)

        # LOG.debug('loop')
        server.serve_forever()

OpenFlowController类就以上两个主要的函数，功能就是启动监听。

接下来我们看一下上面提到的当成功建立与一台交换机的连接之后调用的函数。

**datapath\_connection\_factory函数**： 该函数的参数是socket和address，是由streamserver建立与交换机连接之后传入的交换机的socket和address。然后就是一个麻烦的python特有语法 with ... as ...，是try...final的简易写法。在这里的流程是就是新建一个datapath(socket,address)，赋值给datapath，然后如果创建成功就执性try中的语句，如果创建都没成功就关闭它。执行try中的语句时，如果try中的语句出错就执行except中的语句，最后再执行contextlib.closing函数关闭新建的datapath。我们假设创建datapath是成功的，那么就执行datapath的serve函数(后面讲该函数)，有关datapath类的详细信息后面会介绍。可见当OpenFlowController的server每当成功连接一个交换机后，就会创建一个于其对应的datapath，以后与交换机的通信就通过该datapath。

	def datapath_connection_factory(socket, address):
	    LOG.debug('connected socket:%s address:%s', socket, address)
	    with contextlib.closing(Datapath(socket, address)) as datapath:
	        try:
	            datapath.serve()
	        except:
	            # Something went wrong.
	            # Especially malicious switch can send malformed packet,
	            # the parser raise exception.
	            # Can we do anything more graceful?
	            if datapath.id is None:
	                dpid_str = "%s" % datapath.id
	            else:
	                dpid_str = dpid_to_str(datapath.id)
	            LOG.error("Error in the datapath %s from %s", dpid_str, address)
	            raise

## datapath类 ##
datapath类对象是与连接到控制器的交换机一一对应的，保存交换机的信息，并且接收和发送信息给交换机。我们来看一下这个类。

**\_\_init\_\_函数**：初始化函数

	def __init__(self, socket, address):
        super(Datapath, self).__init__()

        self.socket = socket //与交换机通信的套接字
        self.socket.setsockopt(IPPROTO_TCP, TCP_NODELAY, 1)
        self.address = address //IP地址
        self.is_active = True //交换机状态：设置为活跃

        # The limit is arbitrary. We need to limit queue size to
        # prevent it from eating memory up
        self.send_q = hub.Queue(16) //发送队列，最大16个包

        self.xid = random.randint(0, self.ofproto.MAX_XID) //xid相当于消息的序列号，每发送一个消息曾 1
        self.id = None  # datapath_id is unknown yet //id在之后的negotiation阶段获得
        self.ports = None //现在还没有端口
        self.flow_format = ofproto_v1_0.NXFF_OPENFLOW10
        self.ofp_brick = ryu.base.app_manager.lookup_service_brick('ofp_event') //获取opf处理类，就是那个ofp_handler类
        self.set_state(handler.HANDSHAKE_DISPATCHER) //将状态设置为握手阶段

**close函数**：关闭datapath，将状态设置为.DEAD_DISPATCHE

	def close(self):
        self.set_state(handler.DEAD_DISPATCHER)

**set\_state函数**：设置交换机状态，生成状态改变事件，并通过send\_event\_to\_observers函数（[之前有讲该函数](http://geekwei.com/2014/12/02/ryu-source-analysis-3/)）发送给关心该事件的类。

	 def set_state(self, state):
        self.state = state
        ev = ofp_event.EventOFPStateChange(self) //生成状态改变事件
        ev.state = state //将当前状态保存在事件中
        self.ofp_brick.send_event_to_observers(ev, state) //发送给关心该事件的类

**\_recv\_loop函数**：这是datapath中的主线程，负责接收来自交换机的消息。详细请看代码注释。

	def _recv_loop(self):
        buf = bytearray()
        required_len = ofproto_common.OFP_HEADER_SIZE

        count = 0
        while self.is_active:
            ret = self.socket.recv(required_len) //阻塞的接收来自交换机的包
            if len(ret) == 0: //如果交换机断开了连接，recv函数会返回0
                self.is_active = False //将交换机状态设置为false，那么主线程也就结束了
                break
            buf += ret
            while len(buf) >= required_len: 
			//while函数开始这里很有技巧，功能就是为了收到一个完整的包含包头和包体的消息
			//当第一次收到消息时，required\_len=OFP\_HEADER\_SIZE,因此在上面的recv函数处设置的
			//缓冲区大小也是包头的大小，进入while循环后，执行下面这行函数，获取包头消息，这里我们关注
			//的是msg\_len字段，表明该消息的总大小，然后接着把required\_len设置为msg_len，如果我们
			//当前的buf中的信息长度不够msg\_len，那么退出循环，继续接受剩余的信息。
                (version, msg_type, msg_len, xid) = ofproto_parser.header(buf)
                required_len = msg_len
                if len(buf) < required_len:
                    break
			//一旦函数到达这里，表明我们接收到了一个完整的消息，消息保存在buf中
                msg = ofproto_parser.msg(self,
                                         version, msg_type, msg_len, xid, buf) //获取消息体
                # LOG.debug('queue msg %s cls %s', msg, msg.__class__)
                if msg: //消息体不为空
                    ev = ofp_event.ofp_msg_to_ev(msg) //获取消息中的事件
					//下面这行函数很重要，通过ofp\_brick（该对象其实就是ofp\_handler对象）将事件发送
					//关心该事件的app类，这里也体现出ryu文档中说的ofp_handler是将消息进行路由的功能
                    self.ofp_brick.send_event_to_observers(ev, self.state)

					//定义一个函数dispatchers，该函数的作用是获取类x中对于事件ev的关心时间阶段
                    dispatchers = lambda x: x.callers[ev.__class__].dispatchers
					//这里才是ofp\_handler app获取处理函数的地方，从ofp\_brick(opf\_handler)中获取
					//事件ev的处理函数，并且datapath的当前的状态符合该处理函数定义的dispatchers
                    handlers = [handler for handler in
                                self.ofp_brick.get_handlers(ev) if
                                self.state in dispatchers(handler)]
                    for handler in handlers:
                        handler(ev) //调用每个处理函数处理事件

                buf = buf[required_len:] //从buf中去掉已经处理过的当前消息
                required_len = ofproto_common.OFP_HEADER_SIZE //表明要开始接收新的包了

                # We need to schedule other greenlets. Otherwise, ryu
                # can't accept new switches or handle the existing
                # switches. The limit is arbitrary. We need the better
                # approach in the future.
				//当一个datapath处理了2048个消息后，主动sleep，让出cpu。还要主动休息一会儿？有待提高
                count += 1
                if count > 2048:
                    count = 0
                    hub.sleep(0)

**\_send_loop函数**：这是datapath另外一个线程，发送线程，将send\_q中的消息全部发出去。如果发送失败，首先保存当前消息队列中的消息，然后将send\_q设置为空，避免有新的消息加入，然后清空消息队列。

	def _send_loop(self):
        try:
            while self.is_active:
                buf = self.send_q.get()
                self.socket.sendall(buf)
        finally:
            q = self.send_q
            # first, clear self.send_q to prevent new references.
            self.send_q = None
            # there might be threads currently blocking in send_q.put().
            # unblock them by draining the queue.
            try:
                while q.get(block=False):
                    pass
            except hub.QueueEmpty:
                pass

**send函数**：发送消息，将消息放在send\_q中

	def send(self, buf):
        if self.send_q:
            self.send_q.put(buf)


**set\_xid函数**：给msg设置xid

	def set_xid(self, msg):
        self.xid += 1
        self.xid &= self.ofproto.MAX_XID
        msg.set_xid(self.xid)
        return self.xid

**send\_msg函数**：发送消息

	 def send_msg(self, msg):
        assert isinstance(msg, self.ofproto_parser.MsgBase)
        if msg.xid is None:
            self.set_xid(msg) //设置xid
        msg.serialize() //将消息序列化
        # LOG.debug('send_msg %s', msg)
        self.send(msg.buf) //发送消息

**serve函数**：这是新建datapath后，首先执行的主函数，负责启动datapath的主线程（\_recv_loop）和副线程(\_send_loop)

	def serve(self):
        send_thr = hub.spawn(self._send_loop) //启动副线程

        # send hello message immediately
        hello = self.ofproto_parser.OFPHello(self)
        self.send_msg(hello) //建立连接后，控制器直接向交换机发送hello消息

        try:
            self._recv_loop() //执行主线程
        finally:
            hub.kill(send_thr)
            hub.joinall([send_thr])

datapath还定义了一些方便发送常用消息的函数，如下，就不具体介绍了。

	def send_packet_out(self, buffer_id=0xffffffff, in_port=None,
                        actions=None, data=None):
       
    def send_flow_mod(self, rule, cookie, command, idle_timeout, hard_timeout,
                      priority=None, buffer_id=0xffffffff,
                      out_port=None, flags=0, actions=None):
        
    def send_flow_del(self, rule, cookie, out_port=None):
        self.send_flow_mod(rule=rule, cookie=cookie,
                           command=self.ofproto.OFPFC_DELETE,
                           idle_timeout=0, hard_timeout=0, priority=0,
                           out_port=out_port)

    def send_delete_all_flows(self):
        
    def send_barrier(self):
       
    def send_nxt_set_flow_format(self, flow_format):
        