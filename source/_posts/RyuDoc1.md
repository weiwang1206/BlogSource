title: Ryu文档-1-开始
date: 2014/11/5 23:39:29 
tags: 
- Ryu
- Controller
categories: Ryu
---

## 1.1 什么是Ryu ##
Ryu是一套基于组件的软件定义网络的框架。

Ryu提供了一套软件组件和高效的API，使网络开发者在开发网络管理控制应用时更加方便快捷。Ryu支持各种南向接口来控制网络设备，如OpenFlow，Netconf，OF-config等。Ryu支持OpenFlow 1.0，1.2，1.3，1.4版本和Nicira扩展。

Ryu所有源代码都是开源的，遵循 Apache 2.0 license。Ryu完全有Python编写。
<!--more-->
## 1.2 快速入门 ##

安装Ryu非常简单，在linux命令行下输入

    % pip install ryu
pip是Python包管理工具，具体可见[https://pypi.python.org/pypi/pip](https://pypi.python.org/pypi/pip "pip")

如果你热衷于使用源码安装Ryu，可以使用Git下载源码，并安装

    % git clone git://github.com/osrg/ryu.git
    % cd ryu; python ./setup.py install
如果你想了解在Openstack中如何使用Ryu，请参见[详细文档](http://ryu.readthedocs.org/en/latest/using_with_openstack.html)。你可以VLAN使用创建许多隔离的虚拟网络。 Ryu在Openstack的E版本中出现。
如果你系那个要编写你自己的Ryu的应用，可以参考[Writing ryu application](http://ryu.readthedocs.org/en/latest/writing_ryu_app.html)文档。编写完成后，只需要简单的输入

    % ryu-manager yourapp.py
## 1.3 Support ##
Ryu的官方网站是[http://osrg.github.io/ryu/](http://osrg.github.io/ryu/)

----------

本系列文章翻译于Ryu Documentation Release 3.14，并加入自己的理解