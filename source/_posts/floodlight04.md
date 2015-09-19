title: Floodlight No.4
date: 2014/10/30 22:14:35 
tags: 
- Floodlight
- Controller
categories: Floodlight
---

## Controller (core/internal/Controller.java)(2) ##
![控制器运行过程](/img/floodlight03-controller.png)

本文主要介绍controller中的run函数，run函数的运行图如上图。
<!--more-->
    public void run() {
        if (log.isDebugEnabled()) {
            logListeners();
        } 
        try {            
           final ServerBootstrap bootstrap = createServerBootStrap(); //新建bootstrap server，用于与交换机之间通信

            bootstrap.setOption("reuseAddr", true);
            bootstrap.setOption("child.keepAlive", true);
            bootstrap.setOption("child.tcpNoDelay", true);
            bootstrap.setOption("child.sendBufferSize", Controller.SEND_BUFFER_SIZE);

            ChannelPipelineFactory pfact = 
                    new OpenflowPipelineFactory(this, null);  //新建pipeline，见下详解，这里是重点
            bootstrap.setPipelineFactory(pfact); //设置pipeline
            InetSocketAddress sa = new InetSocketAddress(openFlowPort);
            final ChannelGroup cg = new DefaultChannelGroup();
            cg.add(bootstrap.bind(sa)); //开始监听
            
            log.info("Listening for switch connections on {}", sa);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        // main loop
        while (true) {
            try {
                IUpdate update = updates.take();
                update.dispatch();
            } catch (InterruptedException e) {
                return;
            } catch (StorageException e) {
                log.error("Storage exception in controller " + 
                          "updates loop; terminating process", e);
                return;
            } catch (Exception e) {
                log.error("Exception in controller updates loop", e);
            }
        }
    }

Floodlight使用Netty serverBootstrap来处理与交换机之间的网络通信。在controller的run函数中，初始化serverBootstrap，并且添加Pipeline来负责处理接收到的交换机消息的处理顺序。

Netty中的pipeline处理顺序分为upstream和downstream，处理时首先按照添加顺序处理upstream的handler，然后按照添加顺序的逆序处理downstream的handler。以Floodlight中的pipeline为例：

    public ChannelPipeline getPipeline() throws Exception {
        OFChannelState state = new OFChannelState();
        
        ChannelPipeline pipeline = Channels.pipeline();
        pipeline.addLast("ofmessagedecoder", new OFMessageDecoder());
        pipeline.addLast("ofmessageencoder", new OFMessageEncoder());
        pipeline.addLast("idle", idleHandler);
        pipeline.addLast("timeout", readTimeoutHandler);
        pipeline.addLast("handshaketimeout",
                         new HandshakeTimeoutHandler(state, timer, 15));
        if (pipelineExecutor != null)
            pipeline.addLast("pipelineExecutor",
                             new ExecutionHandler(pipelineExecutor));
        pipeline.addLast("handler", controller.getChannelHandler(state));
        return pipeline;
    }
- ofmessagedecoder, upstream
- ofmessageencoder, donwtream
- idleStateHandler upstream
- readTimeoutHandler upstream
- handshakeTimeoutHandler upstream
- handler, 即of的handler，upstream

因此，当一个包到时的执行顺序为Ofmessagedecoder-> IdleStateHandler-> ReadTimeoutHandler-> HandshakeTimeoutHandler-> Handler-> Ofmessageencoder。在这其中，handler是controller主要的处理逻辑。

之后，主函数进入主循环。主循环主要处理一些更新，包括switchUpdate（包括add,remove,portchange），HARoleUpdate，HAControllerNodeIPUpdate。

这些消息为什么要异步处理？

OFChannelHandler类，该类实现一个channel handler负责处理和交换机的连接，并将接受的消息分发到合适的位置，每个OFChannelHandler对象，对应一个switch。

    public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
            log.info("New switch connection from {}",
                     e.getChannel().getRemoteAddress());
            
            sw = new OFSwitchImpl();
            sw.setChannel(e.getChannel());
            sw.setFloodlightProvider(Controller.this);
            sw.setThreadPoolService(threadPool);
            
            List<OFMessage> msglist = new ArrayList<OFMessage>(1);
            msglist.add(factory.getMessage(OFType.HELLO));
            e.getChannel().write(msglist);
        }
ChannelConnected是重写函数，在交换机建立和控制器的连接时被自动调用。函数首先新建一个交换机实例，设置其通信channel，floodlightprovider和线程池。最后，发送HELLO消息。

    public void channelDisconnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
            if (sw != null && state.hsState == HandshakeState.READY) {
                if (activeSwitches.containsKey(sw.getId())) {
                    // It's safe to call removeSwitch even though the map might
                    // not contain this particular switch but another with the 
                    // same DPID
                    removeSwitch(sw);
                }
                synchronized(roleChanger) {
                    connectedSwitches.remove(sw);
                }
                sw.setConnected(false);
            }
            log.info("Disconnected switch {}", sw);
        }
    protected void removeSwitch(IOFSwitch sw) {
                log.debug("removeSwitch: {}", sw);
        if (!this.activeSwitches.remove(sw.getId(), sw) || !sw.isConnected()) {
            log.debug("Not removing switch {}; already removed", sw);
            return;
        }
        sw.cancelAllStatisticsReplies();
            
        updateInactiveSwitchInfo(sw);
        SwitchUpdate update = new SwitchUpdate(sw, SwitchUpdateType.REMOVED);
        try {
            this.updates.put(update);
        } catch (InterruptedException e) {
            log.error("Failure adding update to queue", e);
        }
    }
channelDisconnected函数也是重写函数，在交换机和控制器断开连接时被自动调用。该函数从两个map中移除交换机：activeSwitches和connectedSwitches。RemoveSwitch函数将交换机从activeSwitches中移除。	在removeSwitch函数中首先调用activeSwitches.remove将其从活动交换机map中移除。然后调用cancelAllStatisticsReplies函数取消交换机所有的统计请求。updateInactiveSwitchInfo函数将非活跃状态的交换机同步到数据库。最后将交换机的移除消息放在updates中，由run函数的主循环处理更新消息的listener

    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e)
                throws Exception {
            if (e.getMessage() instanceof List) {
                @SuppressWarnings("unchecked")
                List<OFMessage> msglist = (List<OFMessage>)e.getMessage();

                for (OFMessage ofm : msglist) {
                    try {
                        processOFMessage(ofm);
                    }
                    catch (Exception ex) {
                        Channels.fireExceptionCaught(ctx.getChannel(), ex);
                    }
                }

                // Flush all flow-mods/packet-out generated from this "train"
                OFSwitchImpl.flush_all();
            }
        }
Receivemessage函数是重写函数，在交换机的channel收到消息时自动调用该函数。通过MessageEvent获取消息，消息是以List方式保存的，因此遍历消息列表，并调用processOFMessage函数处理每一个消息。最后调用flush_all将要发送消息全部发送，该函数是静态函数，会将所有交换机的消息发送。

    protected void processOFMessage(OFMessage m)
该函数处理OF消息，如果是简单消息（如HELLO或ECHO_REQUEST等）就直接处理，复杂消息会再调用handleMessage函数进行进一步处理。

    protected void handleMessage(IOFSwitch sw, OFMessage m,  FloodlightContext bContext)
在handleMessage函数中，如果消息的类型是PACKET_IN要做一个特殊处理（这里不懂），然后进入default，然后获取注册了该消息类型的所有listener，然后分别执行listener的receive函数处理该消息。具体如文章一开始的控制器运行图所示。