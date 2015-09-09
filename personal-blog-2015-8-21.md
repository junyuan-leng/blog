Title: /proc文件系统之cmdline
Date: 2015-8-21
Category: Python

今天在阅读 OpenStack 源码的时候，看到如下一段代码，位于 neutron/services/vpn/device_drivers/sslvpn.py 中

	def get_status(self):
		pid = self.pid
		if pid is None:
			return False
		cmdline = '/proc/%s/cmdline' % pid
		try:
			with open(cmdline, "r"):
				return constants.ACTIVE
		except IOError:
				return constants.DOWN

代码逻辑倒是很简单，用 pid 拼接了一个 /proc/pid/cmdline 的文件名，然后尝试去读取，如果该文件存在，则认为该进程的 Status 为 Active，如果该文件不存在，则会触发 IOError，从而将该进程的 Status 设置为 Down

这个 /proc/pid/cmdline 的用法第一次接触，于是就查了一下，查到的相关信息大致如下

在 /proc 根目录下，以数字命名的目录表示当前一个运行的进程，目录名即为进程的 pid，其内的目录和文件给出了一些关于该进程的信息，其中的 cmdline 文件中包含的是该进程的命令行参数

编写如下所示的 HelloWorld 程序验证一下

<img src="http://www.deepurple.info/images/2015-8-21/hello.png" alt="image" width="100%">

然后编译，命令行传入 one two 两个参数执行后扔到后台，查看 cat /proc/pid/cmdline 的输出结果（此处 pid 为 24243），如下图所示

<img src="http://www.deepurple.info/images/2015-8-21/cmdline.png" alt="image" width="100%">

可以看到输出结果为 ./helloonetwo，正好是命令行的参数

##参考链接

[Linux内核之旅——proc文件系统探索之以数字命名的目录](http://www.kerneltravel.net/?p=285)