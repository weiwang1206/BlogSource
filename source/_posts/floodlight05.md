title: Floodlight No.5
date: 2014/10/30 22:25:17 
tags: 
- Floodlight
- Controller
categories: Floodlight
---


## LinkDiscoverManager模块 ##

    protected Command handlePacketIn(long sw, OFPacketIn pi,
                                     FloodlightContext cntx) {
        Ethernet eth = 
                IFloodlightProviderService.bcStore.get(cntx, 
                                                       IFloodlightProviderService.CONTEXT_PI_PAYLOAD);

        if(eth.getEtherType() == Ethernet.TYPE_BSN) {
            BSN bsn = (BSN) eth.getPayload();
            if (bsn == null) return Command.STOP;
            if (bsn.getPayload() == null) return Command.STOP;
            // It could be a packet other than BSN LLDP, therefore
            // continue with the regular processing.
            if (bsn.getPayload() instanceof LLDP == false)
                return Command.CONTINUE;
            return handleLldp((LLDP) bsn.getPayload(), sw, pi, false, cntx);
        } else if (eth.getEtherType() == Ethernet.TYPE_LLDP)  {
            return handleLldp((LLDP) eth.getPayload(), sw, pi, true, cntx);
        } else if (eth.getEtherType() < 1500) {
            long destMac = eth.getDestinationMAC().toLong();
            if ((destMac & LINK_LOCAL_MASK) == LINK_LOCAL_VALUE){
                if (log.isTraceEnabled()) {
                    log.trace("Ignoring packet addressed to 802.1D/Q " +
                            "reserved address.");
                }
                return Command.STOP;
            }
        }

        // If packet-in is from a quarantine port, stop processing.
        NodePortTuple npt = new NodePortTuple(sw, pi.getInPort());
        if (quarantineQueue.contains(npt)) return Command.STOP;

        return Command.CONTINUE;
    }
<!--more-->

在收到PacketIn消息是，调用handlePacketIn函数，处理LLDP包。首先检查以太网包的类型，一共有两种类型。一种直接是LLDP类型，一种是BSN类型，BSN类型是bigswitch公司自定义的一种类型（BDDP包，广播包），BSN类型包的负载也可能是LLDP包，因此在函数中做判断。如果发现是LLDP包，则调用handleLldp函数进行处理。LinkDiscoverManager模块在startup函数中，启动3个线程。其实，每个交换机不是自己主动发送LLDP包的，是controller定时发送action，使所有交换机发送LLDP包，然后交换机再将受到的LLDP包以PacketIn的方式返回给controller来处理，从而维护全网的link。

第一个线程是：discoveryTask，标准LLDP协议定时发现线程

    discoveryTask = new SingletonTask(ses, new Runnable() {
            @Override
            public void run() {
                try {
                    discoverLinks();
                }
     ……

    if (role == null || role == Role.MASTER) {
            log.trace("Setup: Rescheduling discovery task. role = {}", role);
            discoveryTask.reschedule(DISCOVERY_TASK_INTERVAL, TimeUnit.SECONDS);
        } else {
                log.trace("Setup: Not scheduling LLDP as role = {}.", role);
        }
    protected void discoverLinks() {

        // timeout known links.
        timeoutLinks();

        //increment LLDP clock
        lldpClock = (lldpClock + 1)% LLDP_TO_ALL_INTERVAL;

        if (lldpClock == 0) {
            log.debug("Sending LLDP out on all ports.");
            discoverOnAllPorts();
        }
    }
该线程定时的（每DISCOVERY_TASK_INTERVAL秒执行一次）执行discoverLinks函数，discoverLink函数首先处理那些过时了的link（timeoutLinks），其次，每间隔LLDP_TO_ALL_INTERVAL要求全部交换机在所有端口发送LLDP包（discoverOnAllPorts）
第二个线程是：bddpTask，LLDP广播包。
链路发现服务使用LLDP和广播包（aka BDDPs）发现链路，LLDP的目的MAC地址是01:80:C2:00:00:0E，BDDP的目的地址是FF:FF:FF:FF:FF:FF，LLDP的以太网帧类型是0X88CC，BDDP的以太网帧类型是0X8999，链路发现服务能够正确地认知拓扑图是建立在两个假设上的：①所有的交换机都会销毁link-local packet（LLDP）②Honors layer 2 broadcasts。

链路可以是直连的，也可以是广播。如果LLDP包从一个端口发出去，另外一个端口收到了相同的LLDP包，说明这两个端口是直连的，就会建立一个直连链路；如果一个BDDP包从一个端口发送，在其它的端口接收，说明在两个交换机之间有控制器无法控制的二层交换机，就会建立一个广播链路。

    bddpTask = new SingletonTask(ses, new QuarantineWorker());
    bddpTask.reschedule(BDDP_TASK_INTERVAL, TimeUnit.MILLISECONDS);
第三个线程：updatesTread，该线程主要负责更新消息回调，如果链路或者端口发生变化，则调用已经注册的listener的回调函数，处理这些更新。

    updatesThread = new Thread(new Runnable () {
            @Override
            public void run() {
                while (true) {
                    try {
                        doUpdatesThread();
                    } catch (InterruptedException e) {
                        return;
                    }
                }
            }}, "Topology Updates");
        updatesThread.start();


**HandleportStatus**

处理port状态信息。首先判断是否要删除port或者是要关闭port，如果是这种情况，那么要删除与这个port向连的link，调用deleteLinksOnPort()函数。在该函数中，从switchLink中删除、PortLinks中删除、还有Links中删除。如果是端口修改，找到跟这个端口对应的link，然后修改对应的端口状态。
