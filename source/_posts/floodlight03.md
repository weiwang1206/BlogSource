title: Floodlight No.3
date: 2014/10/30 21:46:20 
tags: 
- Floodlight
- Controller
categories: Floodlight
---

## Controller (core/internal/Controller.java)(1) ##


Controller类实现了IFloodlightProviderService服务。FloodlightProrider类中定义了controller的对象，作为一个模块出现。IFloodlightProviderService定义了controller和switch通信的接口。Controller类实现了控制器和交换机之间的交互。

**IFloodlightProviderService服务接口**
<!--more-->
首先，在该接口中定义了控制器的角色，包括：EQUAL, MASTER, SLAVE。

    public static final FloodlightContextStore<Ethernet> bcStore = new FloodlightContextStore<Ethernet>();
新建FloodlightContextStore 对象，bcStore对象提供了FloodlightContext对象的操作方法，包括get，put，remove。而FloodlightContext对象实际上是一个map对象，用来存储事件的上下文信息（如packet-in payload）。

    public synchronized void addOFMessageListener(OFType type, IOFMessageListener listener) {
        ListenerDispatcher<OFType, IOFMessageListener> ldd = 
            messageListeners.get(type);
        if (ldd == null) {
            ldd = new ListenerDispatcher<OFType, IOFMessageListener>();
            messageListeners.put(type, ldd);
        }
        ldd.addListener(type, listener);
    }
该函数向messageListener（map）中添加type和对应的message listener，各个模块通过该函数向控制器注册各种其关心的消息类型，当控制器收到该类型的消息后，将调用其对应的函数。

    public Map<OFType, List<IOFMessageListener>> getListeners() 
getListeners（）函数获取所有listener。

    public Map<Long, IOFSwitch> getSwitches() {
        return Collections.unmodifiableMap(this.activeSwitches);
    }
getSwitches函数获取所有active交换机，但是如果controller处于slave角色，将将不包含任何switch。

    public void addOFSwitchListener(IOFSwitchListener listener) {
        this.switchListeners.add(listener);
    }
    public void removeOFSwitchListener(IOFSwitchListener listener) {
        this.switchListeners.remove(listener);
    }
添加与删除switch的listener

    public synchronized void terminate() {
        log.info("Calling System.exit");
        System.exit(1);
    }
退出controller进程。

    public boolean injectOfMessage(IOFSwitch sw, OFMessage msg) {
        // call the overloaded version with floodlight context set to null    
        return injectOfMessage(sw, msg, null);
    }
    ……
    public boolean injectOfMessage(IOFSwitch sw, OFMessage msg,
                                   FloodlightContext bc) {
        if (sw == null) {
            log.info("Failed to inject OFMessage {} onto a null switch", msg);
            return false;
        }
        if (!activeSwitches.containsKey(sw.getId())) return false;
        try {
            // Pass Floodlight context to the handleMessages()
            handleMessage(sw, msg, bc);
        } catch (IOException e) {
            log.error("Error reinjecting OFMessage on switch {}", 
                      HexString.toHexString(sw.getId()));
            return false;
        }
        return true;
    }
重新为一个switch注入一个msg，并调用handleMessage函数处理该消息。

    public void handleOutgoingMessage(IOFSwitch sw, OFMessage m, FloodlightContext bc) {
        if (log.isTraceEnabled()) {
            String str = OFMessage.getDataAsString(sw, m, bc);
            log.trace("{}", str);
        }
        List<IOFMessageListener> listeners = null;
        if (messageListeners.containsKey(m.getType())) {
            listeners = 
                    messageListeners.get(m.getType()).getOrderedListeners();
        }            
        if (listeners != null) {                
            for (IOFMessageListener listener : listeners) {
                if (listener instanceof IOFSwitchFilter) {
                    if (!((IOFSwitchFilter)listener).isInterested(sw)) {
                        continue;
                    }
                }
                if (Command.STOP.equals(listener.receive(sw, m, bc))) {
                    break;
                }
            }
        }
    }
处理发向switch的outgoing包，调用注册了该类型OFType的listener。

    public BasicFactory getOFMessageFactory();
返回factory，该factory为BasicFactory类对象，实现了OFMessageFactory, OFActionFactory, OFStatisticsFactory, OFVendorDataFactory等，用来生成OF的message和actions。

最后还有一些简单的函数

    public void addInfoProvider(String type, IInfoProvider provider); //添加各种类型的信息的provider，获取服务的信息
    public void removeInfoProvider(String type, IInfoProvider provider); //删除provider
    public Map<String, Object> getControllerInfo(String type); //获取一种特定类型的控制器信息
    public long getSystemStartTime();//获取系统时间
    public void setAlwaysClearFlowsOnSwAdd(boolean value);//设置当交换机连接到控制器时，清除其流表
