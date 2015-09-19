title: python-setdefault
date: 2014/11/26 21:43:19 
tags: 
- Python
categories: Python
---

python 字典的setdefault

这个函数很好用
当我们向字典添加元素时，可能会遇到如果key存在和不存在时，进行的赋值不一样。我过去的做法时

	d = dict()
	a='key'
    if a in d.keys():
		pass
    else:
		d[a] = 1

现在一行搞定

    d.setdefault(a,1)