title: python-inspect模块
date: 2014/11/26 21:24:06 
tags: 
- Python
categories: Python
---


python inspect模块

今天看RYU源码时，发现一个inspect模块，RYU使用了该模块的getmembers函数来获取ryu app的app类。

函数原型是 inspect.getmembers(object[, predicate])

功能： 从一个Object中获取符合predicate的元素的list，元素的形式是（name，value）
predicate可以是ismodule(), isclass(), ismethod(), isfunction(), isgeneratorfunction(), 
isgenerator(), istraceback(), isframe(), iscode(), isbuiltin(),isroutine()，这些验证函数。

<!--more-->

看一个例子

    import inspect

    class C():

		class CC():
			def foo3():
			print "foo3"

	    def foo():
	    	print "foo"
		
		def foo2():
			print "foo2"
		
		
	cls = inspect.getmembers(C,inspect.ismethod)
	print cls


执行的结果是
![phase](/img/python-inspect-result.jpg)
