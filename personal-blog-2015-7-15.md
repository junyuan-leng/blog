Title: 阿里技术保障部内推面试
Date: 2015-7-15
Category: 杂谈

内推面试一共 5 轮技术面 + 1 轮 HR 面，给了 offer

然而后来“拥抱变化”了，233

能回忆起来的面试题基本就下面这些了

##数据中心网络

* 一般数据中心接入架构（接入层，汇聚层，核心层）
* 二层和三层分别的应用场景和优缺点
* 大二层的含义，为什么要用大二层
* 负载均衡项目的实现
* 单点故障在LVS中如何解决（回包不走网关）

##OpenStack

* GRE和VXLAN的用途、实现
* br-int和br-tun的作用
* 每个br上流表规则大致是什么样子
* 数据包的处理流程

##OpenvSwitch

* 哈希表和链表分别的用途
* 为什么要用哈希表和链表来实现双级流表（哈希表作为Cache，模拟交换机行为）

##C/Linux/操作系统

* 一个HelloWorld程序从 源代码到执行，都经过什么过程
* 静态链接和动态链接的区别、优缺点、具体实现
* static和inline的含义与区别
* 可执行文件内存分布，malloc和free在堆上还是栈上，局部变量在堆上还是栈上，全局变量在哪里
* 运行时重定位实现动态库函数调用的过程
* 快排的过程，运用了分治思想，分治的含义，快排选第一个元素作为分界的缺点（选第一个，最后一个，中间一个，选最小的作为分界点）
* Linux用户态和内核态的区别
* 线程和进程的区别
* 用户态线程、用户态进程、内核态线程的联系和区别
* 抢占式调度和非抢占式调度的区别
* 线程调度的时机

##SDN/OpenFlow

* 多控制器瓶颈问题，解决方案有哪些（冗余、分片）
* 冗余和分片这两种方式分别解决什么问题，什么情况下用冗余，什么情况下用分片

##网络

* ARP包的DST MAC（全F）
* 三层网络的子网ICMP包的DST IP和DST MAC（网关MAC）
* BGP工作过程

##参考链接

[http://oss.org.cn/kernel-book/ch05/5.3.2.htm](http://oss.org.cn/kernel-book/ch05/5.3.2.htm)

[http://www.h3c.com.cn/About_H3C/Company_Publication/IP_Lh/2013/04/Home/Catalog/201309/796466_30008_0.htm](http://www.h3c.com.cn/About_H3C/Company_Publication/IP_Lh/2013/04/Home/Catalog/201309/796466_30008_0.htm)

[http://stackoverflow.com/questions/7762731/whats-the-difference-between-static-and-static-inline-function](http://stackoverflow.com/questions/7762731/whats-the-difference-between-static-and-static-inline-function)

[http://itony.me/849.html](http://itony.me/849.html)

[http://www.cquestions.com/2011/02/static-variable-in-c.html](http://www.cquestions.com/2011/02/static-variable-in-c.html)

[http://stackoverflow.com/questions/572547/what-does-static-mean-in-a-c-program](http://stackoverflow.com/questions/572547/what-does-static-mean-in-a-c-program)

[http://www.thegeekstuff.com/2012/08/c-static-variables-functions/](http://www.thegeekstuff.com/2012/08/c-static-variables-functions/)

[http://www.cnblogs.com/kaituorensheng/archive/2013/03/02/2939690.html](http://www.cnblogs.com/kaituorensheng/archive/2013/03/02/2939690.html)

[http://www.h3c.com.cn/About_H3C/Company_Publication/IP_Lh/2012/06/Home/Catalog/201212/769073_30008_0.htm](http://www.h3c.com.cn/About_H3C/Company_Publication/IP_Lh/2012/06/Home/Catalog/201212/769073_30008_0.htm)