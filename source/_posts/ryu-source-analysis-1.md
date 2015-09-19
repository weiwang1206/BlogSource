title: Ryu代码解析（一）
date: 2014/11/27 22:21:10 
tags: 
- Ryu
- Controller
categories: Ryu
---

## RYU main函数 ##
Ryu的main函数位于cmd\manager.py文件中，main函数如下：

	def main(args=None, prog=None):
	    try:
	        CONF(args=args, prog=prog,
	             project='ryu', version='ryu-manager %s' % version,
	             default_config_files=['/usr/local/etc/ryu/ryu.conf'])
	    except cfg.ConfigFilesNotFoundError:
	        CONF(args=args, prog=prog,
	             project='ryu', version='ryu-manager %s' % version)
	
	    log.init_log()
	
	    if CONF.pid_file:
	        import os
	        with open(CONF.pid_file, 'w') as pid_file:
	            pid_file.write(str(os.getpid()))
	
	    app_lists = CONF.app_lists + CONF.app //从用户命令行获取的app列表
	    # keep old behaivor, run ofp if no application is specified.
	    if not app_lists:
	        app_lists = ['ryu.controller.ofp_handler'] //添加默认的应用
	
	    app_mgr = AppManager.get_instance() //获取appManager实例，是单体类
	    app_mgr.load_apps(app_lists) //加载所有app
	    contexts = app_mgr.create_contexts() //加载context
	    services = []
	    services.extend(app_mgr.instantiate_apps(**contexts)) //以context作为参数，实例化app
	
	    webapp = wsgi.start_service(app_mgr)
	    if webapp:
	        thr = hub.spawn(webapp)
	        services.append(thr)
	
	    try:
	        hub.joinall(services)
	    finally:
	        app_mgr.close()
<!--more-->
## AppManager ##
AppManager中get_instance()函数，典型的单体类实现方法

	 def get_instance():
        if not AppManager._instance:
            AppManager._instance = AppManager()
        return AppManager._instance


load\_app函数：加载app，本质是找到app名字对应的类
用到的python 基础：[inspect.getmembers()](https://docs.python.org/2/library/inspect.html#inspect.getmembers)和[inspect例子](http://geekwei.com/2014/11/26/python-inspect/)

	def load_app(self, name):
        mod = utils.import_module(name)
        clses = inspect.getmembers(mod,
                                   lambda cls: (inspect.isclass(cls) and
                                                issubclass(cls, RyuApp) and
                                                mod.__name__ ==
                                                cls.__module__)) //过滤函数是lambda表达式
        if clses:
            return clses[0][1] //只与符合条件的第一个！！！文档里就这么要求的
        return None

load\_apps函数：加载app，说是加载，其实就是找到这个类，并把其存到applications\_cls中，key是app类的名字，值就是这个类，加载的过程中顺便加载每个app依赖的app类，而且把每个app的context保存到context\_cls中。

python 基础 [python map函数浅析](http://my.oschina.net/zyzzy/blog/115096)

	def load_apps(self, app_lists):
        app_lists = [app for app
                     in itertools.chain.from_iterable(app.split(',')
                                                      for app in app_lists)]
        while len(app_lists) > 0:
            app_cls_name = app_lists.pop(0)

            context_modules = map(lambda x: x.__module__,
                                  self.contexts_cls.values()) //获取所有context类所在的模块
            if app_cls_name in context_modules: //不加载属于context的类，因为之后在create_context函数中会加载
                continue

            LOG.info('loading app %s', app_cls_name)

            cls = self.load_app(app_cls_name) //找到app的类，cls就是这个app的类
            if cls is None:
                continue

            self.applications_cls[app_cls_name] = cls //将app添加到字典

            services = []
            for key, context_cls in cls.context_iteritems(): //便利这个app类的context，将其context加入contexts_cls字典
                v = self.contexts_cls.setdefault(key, context_cls)
                assert v == context_cls
                context_modules.append(context_cls.__module__)

                if issubclass(context_cls, RyuApp): //感觉这里写错了，其实context的类在之后会加载，和前面的描述不符
                    services.extend(get_dependent_services(context_cls))

            # we can't load an app that will be initiataed for
            # contexts.
            for i in get_dependent_services(cls): //加载这个app的依赖app，依赖app不在context_modules字典中
                if i not in context_modules:
                    services.append(i)
            if services:
                app_lists.extend([s for s in set(services)
                                  if s not in app_lists]) //将依赖的app都加入app_lists,在之后的循环中加载

create\_contexts函数：实例化context\_cls中的context，如果这个context是app，就实例化该app。记得在load_apps中没有管属于context\_cls的app吗，在这里直接初始化了。

	 def create_contexts(self):
        for key, cls in self.contexts_cls.items():
            if issubclass(cls, RyuApp):
                # hack for dpset
                context = self._instantiate(None, cls) //初始化app
            else:
                context = cls() //实例化context
            LOG.info('creating context %s', key)
            assert key not in self.contexts
            self.contexts[key] = context //加入contexts字典
        return self.contexts //返回contexts字典

_instantiate函数：

	def _instantiate(self, app_name, cls, *args, **kwargs):
        # for now, only single instance of a given module
        # Do we need to support multiple instances?
        # Yes, maybe for slicing.
        LOG.info('instantiating app %s of %s', app_name, cls.__name__)

        if hasattr(cls, 'OFP_VERSIONS') and cls.OFP_VERSIONS is not None:
            ofproto_protocol.set_app_supported_versions(cls.OFP_VERSIONS)

        if app_name is not None:
            assert app_name not in self.applications
        app = cls(*args, **kwargs) //实例化app类，kwargs参数是context字典
        register_app(app) //注册app对象，主要是通过inspect组件获取其中的handler，详见下文
        assert app.name not in self.applications
        self.applications[app.name] = app //将app对象保存到applications字典中
        return app

register\_app函数中调用regester\_instance函数（该函数在handler.py文件中）,在该函数中通过inspect.getmembers函数，获取app类中的方法，并通过app的register_handler将handler注册到app的event_handlers成员变量中

	def register_instance(i):
	    for _k, m in inspect.getmembers(i, inspect.ismethod):
	        # LOG.debug('instance %s k %s m %s', i, _k, m)
	        if _has_caller(m):
	            for ev_cls, c in m.callers.iteritems():
	                i.register_handler(ev_cls, m)

到这儿可能还比较迷惑，handler是如何挂到每个事件上的，将在后面的[Ryu事件处理函数的挂接方式分析](http://geekwei.com/2014/12/01/ryu-source-analysis-2md/)中专门介绍一下。

instantiate\_apps函数：实例化app，并且启动每个app

	def instantiate_apps(self, *args, **kwargs):
        for app_name, cls in self.applications_cls.items():
            self._instantiate(app_name, cls, *args, **kwargs)

        self._update_bricks() //将app更新到全局变量SERVICE_BRICK中
        self.report_bricks()

        threads = []
        for app in self.applications.values():
            t = app.start() //启动APP
            if t is not None:
                threads.append(t)
        return threads