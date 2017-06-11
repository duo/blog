+++
date = "2011-11-11T00:57:00+08:00"
draft = false
title = "TermConveyor"
topics = [ "Development" ]
tags = [ "Python", "Terminal" ]
+++

这段时间，在公司负责升级一个批量给机器重灌 OS 的系统，扩展其功能以及易用性。之前的系统经常遇到的一个问题就是，由于环境的复杂，[Kickstart](http://en.wikipedia.org/wiki/Kickstart_(Linux) "Kickstart") 脚本是无法覆盖到所有情况，有时候就会需要人工来进行干预，这时不得不登陆到跳板机，然后通过 [OOB](http://en.wikipedia.org/wiki/Out-of-band_management "Out-of-band management") 来查看问题和进行一些相应的操作，比较繁琐。于是在新版里，把 [AjaxTerm](https://github.com/antonylesuisse/qweb "AjaxTerm") 改了改加进去，将 [IPMI](http://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface "Intelligent Platform Management Interface") 和 [iLo](http://en.wikipedia.org/wiki/HP_Integrated_Lights-Out "HP Integrated Lights-Out") 的输出转到了 Web 上来进行操作，并与权限系统整合，这样当发现 OS 安装异常后，可以直接就在浏览器里来对相应的主机来排查和解决遇到的问题来排查和解决遇到的问题。

<!--more-->

不过 [AjaxTerm](https://github.com/antonylesuisse/qweb "AjaxTerm") 的功能实在是太弱了，非 [VT100](http://en.wikipedia.org/wiki/VT100 "VT100") 兼容，遇到复杂点的输出容易就出现一堆混乱的字符了，而且还是传统的轮询的方式更新，每 100 毫秒就提交一次 POST 请求，对服务器的压力还是蛮大的... 在当前版本上线后，就开始找有没有替代方案，经过几番比较后，[Shell In A Box](http://code.google.com/p/shellinabox "Shell In A Box") 比较合胃口，终端的模拟是用纯 JavaScript 的实现，[VT100](http://en.wikipedia.org/wiki/VT100 "VT100") 兼容，与服务端的交互采用的是 [Comet](http://en.wikipedia.org/wiki/Comet_(programming) "Comet") 的方式，可以说是全面胜出的解决方案。对于我们来说，唯一美中不足的一点就是，服务器是用 C 写的，这样与一些现有接口的整合的工作量比较大，不过其实还是因为组里熟悉 C 的人太少了，不好维护。于是打算用 Python 把服务端重新实现下，但是工作时间从来都是疲于奔命的应对各种需求，完全没有闲暇的时间好好研究下，业余时间得忙别的，于是就这么搁置下来了...

这么一搁置，就是一个多月过去了，直到上个月底，公司开年会，在杭州待了三天，晚上在酒店那叫一个无聊啊，于是开始写代码打发时间。倒腾了两个晚上，查了查资料，过了过两个项目的代码，然后以 [Tornado](http://www.tornadoweb.org "Tornado") 为框架，按 [AjaxTerm](https://github.com/antonylesuisse/qweb "AjaxTerm") 写了转发 [PTY](http://en.wikipedia.org/wiki/Pseudo_terminal "Pseudo terminal") 输出的服务，和 [Shell In a Box](http://code.google.com/p/shellinabox "Shell In A Box") 的 vt100.js 配合，就成了 [TermConveyor](https://github.com/duo/TermConveyor "TermConveyor")...

项目目前托管在 GitHub，嗯，回来后，继续被各种需求压着，各种催活，看来又有段时间没法更新了，继续挖坑不埋的作风...

