Title: SDN 网络新人学习计划
Date: 2015-9-15
Category: 杂谈

老板让我给新进组的研究生写一个简单的 SDN 学习计划，于是就大概写了一点，现在的版本比较简单，后期可能会不断补充

##基础知识

###网络原理

* 网络基础：[《计算机网络——自顶向下方法》](http://book.douban.com/subject/1391207/)
* 网络场景：[《腾云：云计算和大数据时代的网络技术揭秘》](http://book.douban.com/subject/21966988/)

####编程语言

* Python：[《Python基础教程》](http://book.douban.com/subject/4866934/) [《Python Cookbook》](http://book.douban.com/subject/4828875/) [PyPI](https://pypi.python.org/pypi) [Virtualenv](https://virtualenv.pypa.io/en/latest/)
* C：[《C程序设计语言》](http://book.douban.com/subject/1139336/) [《C和指针》](http://book.douban.com/subject/3012360/)
* Java：[《Java编程思想》](http://book.douban.com/subject/2130190/)

####必备工具

* Linux：[《鸟哥的Linux私房菜》](http://book.douban.com/subject/4889838/)
* Git：[《Pro Git》](http://git-scm.com/book/zh/v1/)
* Sublime Text：[可能是世界上最好用的编辑器](http://www.sublimetext.com/)
* PyCharm：[可能是最好用的Python IDE](https://www.jetbrains.com/pycharm/)
* VirtualBox：[VirtualBox官网](https://www.virtualbox.org/)
* Source Insight：[可能是最好用的C语言源码阅读工具](http://www.sourceinsight.com/)

####常用网站

* Google：[世界上最好用的搜索引擎](https://www.google.com.hk)
* StackOverflow：[世界上最好用的编程问答网站](http://www.stackoverflow.com)
* GitHub：[无尽的开源代码宝库](https://github.com)

##SDN/OpenFlow 相关知识

####概览与基本概念

* [《深度解析SDN——利益、战略、技术、实践》](http://book.douban.com/subject/25779099/)
* [《SDN核心技术剖析和实战指南》](http://book.douban.com/subject/25723063/)

####SDN/OpenFlow 常用网站

* [SDNLAB](http://www.sdnlab.com)
* [SDNAP](http://www.sdnap.com)

####SDN/OpenFlow 相关开源项目

* OpenFlow 协议：[OpenFlow Specification 1.3.0](http://www.sdnap.com/wp-content/uploads/openflow/openflow-spec-v1.3.0-SDNAP_CN.pdf)
* OpenFlow 仿真环境：[Mininet](http://mininet.org)
* OpenFlow 虚拟交换机：[Open vSwitch](http://openvswitch.org/)
* OpenFlow 控制器：[POX](http://www.noxrepo.org/pox/about-pox/) [Ryu](http://osrg.github.io/ryu/) [Floodlight](http://www.projectfloodlight.org/floodlight/) [OpenDaylight](https://www.opendaylight.org/)
* OpenFlow 相关前沿技术：[OpenStack](http://www.openstack.org/) [Docker](https://www.docker.com/)

##学习计划

####第一阶段

学习目标：

* 学习网络基础
* 学习 Python
* 阅读 OpenFlow 协议

学习任务：

* 熟悉基本的网络原理，如二层交换、三层路由、跨网段通信、ARP/ICMP/TCP/DNS 等的工作过程
* 熟悉 OpenFlow 架构中控制器、交换机、安全通道的作用及交互过程

####第二阶段

学习目标：

* 搭建 OpenFlow 实验环境
* 通过操作，熟悉 OpenFlow 的工作过程

学习任务：

* 使用 Ubuntu+ Ryu + Mininet，搭建 OpenFlow 实验环境，进行基本的 Ping 等测试
* 使用 Wireshark 或者 tcpdump，抓取网络数据包，实际观察 OpenFlow 协议工作过程

####第三阶段

学习目标：

* 阅读 OpenFlow 相关开源项目源码，深入 OpenFlow 实现

学习任务：

* 阅读 POX/Ryu 中自带的 OpenFlow 模块源码，并可总结其工作过程
* 阅读 OpenvSwitch 中相关源码，熟悉其调用流程及模块间关系（较难）