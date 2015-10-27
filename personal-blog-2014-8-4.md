Title: libnet和libpcap安装记录
Date: 2014-8-4
Category: 折腾

###安装libnet

首先下载libnet的源码包，这里要注意，最早的libnet在2004年之后就不维护了，现在的libnet全名叫作libnet-dev，由[Sam Roberts](vieuxtech@gmail.com)维护，代码托管在[GitHub](https://github.com/sam-github/libnet)上

报错

	./autogen.sh: 8: ./autogen.sh: autoreconf: not found

解决：安装automake

报错

	Libtool library used but 'LIBTOOL' is undefined

解决：安装libtool

报错：

	/bin/bash: doxygen: 未找到命令

解决：安装doxygen

安装完成，进sample目录编译测试代码进行测试

	cd sample
	gcc -Wall icmp_echo_cq.c -o test -lnet
	sudo ./test
	./test: error while loading shared libraries: libnet.so.9: cannot open shared object file: No such file or directory

报了无法找到共享库的错误，然后去/usr/local/lib下看，发现已经有libnet.so.9和libnet.so.9.0.0两个文件了，其中libnet.so.9.0.0其实就是libnet.so.9的软链

然后就有两种方法，简单粗暴的是直接把共享库拷贝到/usr/lib下面

	sudo cp /usr/local/lib/libnet.so.9 /usr/local/lib/libnet.so.9.0.0 /usr/lib/

另外，修改ld.so.conf中的共享库加载路径

	vim /etc/ld.so.conf

发现里面就一句话：

	include /etc/ld.so.conf.d/*.conf

原来ld.so.conf只是负责把/etc/ld.so.conf.d目录下所有的conf文件中的共享库路径加载进来，于是进/etc/ld.so.conf.d看看

	总用量 12
	-rw-rw-r-- 1 root root  36  3月 24 13:21 fakeroot-i386-linux-gnu.conf
	lrwxrwxrwx 1 root root  41  8月  3 22:06 i386-linux-gnu_EGL.conf -> /etc/alternatives/i386-linux-gnu_egl_conf
	lrwxrwxrwx 1 root root  40  8月  3 22:06 i386-linux-gnu_GL.conf -> /etc/alternatives/i386-linux-gnu_gl_conf
	-rw-r--r-- 1 root root 108  4月 12 18:40 i686-linux-gnu.conf
	-rw-r--r-- 1 root root  44  8月 10  2009 libc.conf

挨个打开看看，发现libc.conf里已经包括了/usr/local/lib的共享库路径了

以下引用[ChinaUnix帖子](http://os.chinaunix.net/a2011/1031/1266/000001266131.shtml)

> 搜索路径的设置方式对于程序连接时的库(包括共享库和静态库)的定位已经足够了，但是对于使用了共享库的程序的执行还是不够的，这是因为为了加快程序执行时对共享库的定位速度，避免使用搜索路径查找共享库的低效率，所以是直接读取库列表文件 /etc/ld.so.cache 从中进行搜索的
> 
> /etc/ld.so.cache 是一个非文本的数据文件，不能直接编辑，它是根据 /etc/ld.so.conf 中设置的搜索路径由 /sbin/ldconfig 命令将这些搜索路径下的共享库文件集中在一起而生成的(ldconfig 命令要以 root 权限执行)，因此为了保证程序执行时对库的定位，在 /etc/ld.so.conf 中进行了库搜索路径的设置之后，还必须要运行 /sbin/ldconfig 命令更新 /etc/ld.so.cache 文件之后才可以
> 
> ldconfig ,简单的说，它的作用就是将/etc/ld.so.conf列出的路径下的库文件缓存到/etc/ld.so.cache 以供使用，因此当安装完一些库文件，(例如刚安装好glib)，或者修改ld.so.conf增加新的库路径后，需要运行一下 /sbin/ldconfig使所有的库文件都被缓存到ld.so.cache中，如果没做，即使库文件明明就在/usr/lib下的，也是不会被使用的，结果编译过程中抱错，缺少xxx库，去查看发现明明就在那放着，搞的想骂人

于是sudo ldconfig刷新ls.so.cache，齐活

###安装libpcap

libpcap的安装就要简单许多了，首先从[tcpdump.org](www.tcpdump.org)下载libpcap源码包，最新版本是libpcap-1.6.1.tar.gz

然后根据INSTALL.txt，经典的./configure && make && make install三部曲就好了