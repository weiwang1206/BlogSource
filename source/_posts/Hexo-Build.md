title: Github搭建Hexo
date: 2013-11-30 16:08:28
tags: 
- Hexo
- github
categories: Others
---



搞了一天终于搞定了Hexo的搭建，今后把博客开在这里了，来记录我工作学习和生活。

<!--more-->

之前就一直想写博客，记录我的学习历程，最开始搭建在GAE上，后来总是要翻墙，比较麻烦，就停止了。再后来转战Evernote了，记录在本地了，后来发现这样无法与广大学者交流学习，提高比较慢，分享的也少，所以就又筹划着开博客了。

然后就，我的前提是不买VPS，不买域名，反正就是不花钱（本人是穷PhD一枚），Google了一下，发现Githug Pages还不错，它的初衷是给github上的项目提供一个介绍页的，避免大家直接看到烦乱的Code。但是也可以作为一个host，搭建我们的Blog（程序员们太聪明了）。而且可以用git的方式更新，太方便了。

Host有了，用什么搭建呢？Google下发现，Hexo利用node.js，而且和github结合的比较紧密，搭建方便。

我是在windows上搭建的，估计大多数朋友写博客也是在window吧,，闲话少说，切入正题.

**准备工作**

首先需要一个github的账号

下载安装git，下载地址 [http://msysgit.github.io/](http://msysgit.github.io/)
安装git的时候最好选上那个在CMD中也可以运行的选项，否则以后更新博客的时候必须得用那个git bash

下载安装node.js，下载地址 [http://nodejs.org/](http://nodejs.org/)

安装hexo

在命令行中输入

    npm install -g hexo
    hexo init <your folder> #博客在本地将来要放的地方

然后就在指定的目录中生成了一堆的文件，此时在本地已经有了基本的blog的框架了。

**使用与配置**

可以在本地预览一下默认的blog的样子，在你指定的目录中（必须是在那个目录下）输入：

    hexo server

启动本地服务器，然后在浏览器打开 [localhost:4000](localhost:4000)，就可以预览了

接下来就要将你的本地blog部署到Github上，但在这之前需要做一个重要的工作，就是使用ssh-key可以登录github，没有这个的话后面会遇到很多access deny的麻烦，方法很简单

ssh-keygen -t rsa

连续按三次回车就会生成，~/.ssh/id_rsa.pub，用记事本打开，将其复制到github->accounting setting->SSH Key->Add SSH Key 的Key一栏中，名称自己随便取。之后会让你填一下github的密码。

然后在cmd中输入

    ssh -vT git@github.com

来测试是否成功

成功后，修改*_config.yml* 文件的deploy项，注意type前面的缩进和冒号后面的空格

	deploy:
  		type: github
  		repository: git@github.com:weiwang1206/weiwang1206.github.io.git
  		branch: master

这里需要注意的是，首先得有一个project在github中，而且名字**必须**为 *username*.github.io

生成并部署

	hexo deploy --generate


之后我们就可以在浏览器打开 *username*.github.io 来访问网页，可能需要等一会儿。

然后具体hexo如何写blog可以参加hexo的[官方网站](http://zespia.tw/hexo/)来学习