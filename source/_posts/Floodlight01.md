title: Floodlight No.1
date: 
tags: 
- Floodlight
- Controller
categories: Floodlight
---


Floodlight 学习笔记--No.1

Floodlight采用模块化的方法加载各种服务。

在各个模块中，FloodlightProvider模块，起到统领的作用，提供IFloodlightProviderService服务，具体来说就是提供controller服务。

在main函数中，在加载完毕各种模块之后，获取FloodlightProviderService对象，即controller对象，执行其run函数。

本系列博客文章针对版本号为0.90的Floodlight源码进行分析，旨在学习基本的SDN控制器的结构组成，运行原理，并与读者分享与讨论。鉴于能力有限，欢迎指出错误并指正。