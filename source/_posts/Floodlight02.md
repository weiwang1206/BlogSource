title: Floodlight No.2
date: 2014/10/30 21:16:47 
tags: 
- Floodlight
- Controller
categories: Floodlight
---

## 代码结构 ##

Floodlight代码总体上分为两部分：floodlight controller和openflow协议。floodlightcontroller部分包含了控制器的核心部件(core)，各个服务模块(module)以及相关工具组件。Openflow部分主要定义了openflow协议标准。Floodlight由各个模块组成，每个模块又可以提供不同的服务。


## 控制器核心组件（core） ##
Floodlight通过模块的方式来提供控制器的扩展性，core组件负责组件核心框架，包括网络通信，模块接口等，是其他模块的基础。首先，从main函数开始。
<!--more-->
## Main函数 ##

    public static void main(String[] args) throws FloodlightModuleException {
        // Setup logger
        System.setProperty("org.restlet.engine.loggerFacadeClass", 
                "org.restlet.ext.slf4j.Slf4jLoggerFacade");
        
        CmdLineSettings settings = new CmdLineSettings();
        CmdLineParser parser = new CmdLineParser(settings);
        try {
            parser.parseArgument(args);
        } catch (CmdLineException e) {
            parser.printUsage(System.out);
            System.exit(1);
        }
        
        // Load modules
        FloodlightModuleLoader fml = new FloodlightModuleLoader();
        IFloodlightModuleContext moduleContext = fml.loadModulesFromConfig(settings.getModuleFile());
        // Run REST server
        IRestApiService restApi = moduleContext.getServiceImpl(IRestApiService.class);
        restApi.run();
        // Run the main floodlight module
        IFloodlightProviderService controller =
                moduleContext.getServiceImpl(IFloodlightProviderService.class);
        // This call blocks, it has to be the last line in the main
        controller.run();
    }
	



在main函数中，首先读取并解析命令行参数。然后从默认配置文件中加载需要加载的模块(fml.loadModulesFromConfig(settings.getModuleFile()))。默认配置文件为config/floodlight.properties，该文件保存着需要加载的模块的类的路径。追踪该函数可以发现，最终调用loadModuleFromList函数,该函数首先调用findAllModules(configMods)函数，该函数建立三个map

- serviceMap -> Maps a service to a module
- moduleServiceMap -> Maps a module to all the services it provides
- moduleNameMap -> Maps the string name to the module

其次，遍历加载每一个模块及其依赖的模块，如果该模块中提供的service被设置为ignored，则不加载该模块。

在loadModulesFromList的最后调用parseConfigParameters,initModues和startpuModules函数,进行模块的配置、初始化、和启动。

    parseConfigParameters(prop);
    initModules(moduleSet);
    startupModules(moduleSet);

InitModules函数加载各个模块的service，并加入floodlightModuleContext中,最后调用每个模块的init函数
startpuModules函数调用每个模块的startup函数。


Main函数，然后启动REST服务器，接收RESTful API调用。最后从moduleContext获取IFloodlightProviderService的实现类，实际上controller类（core/internal/Controller.java）实现了该接口。最后调用run函数启动controller。
