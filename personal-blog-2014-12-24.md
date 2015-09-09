Title: 终于把个人Blog搭起来了
Date: 2014-12-24
Category: 杂谈

之前一直想搭建一个个人的技术Blog，无奈拖延症严重，拖到平安夜才动手，花了3个小时，基本上把Blog搭起来了，简单记录搭建过程如下

###域名

之前在 GoDaddy 买了一个域名，100多块一年，价格还好，当然最近很多人似乎开始用 namecheap 的域名服务了，没用过，以后有机会试试

###虚拟主机

虚拟主机本来准备选 hi-pda 看到的99块包年的 OpenVZ 主机，后来搜了一下发现搬瓦工的评价也不错，512M RAM + 5G SSD 的 VPS 才9.99美元一年，于是果断入了一个，机房位于西海岸洛杉矶，ping 值在300左右，用起来感觉不错，超出我的预期

另外我刚在 PayPal 上支付了，交行风险控制中心的电话就打过来了，跟我确认交易金额，不过用一个027开头的武汉号码很容易让人以为是电信诈骗电话...

###DNS解析

DNS解析的话，当然是用烟台本地的良心企业 DNSPod 啦，免费版的解析就很不错

###静态博客生成

作为一个 Python 用户，当然要使用 Pelican 来生成静态博客了，之前试过 Jekyll， 不过 Jekyll 是用 Ruby 编写的，修改起来总归不是很顺手，因此最终还是选择了熟悉的 Python 编写的 Pelican

###搭建过程

首先设置 VPS 的登录密码，然后 SSH 登录

安装 pip

	wget https://bootstrap.pypa.io/get-pip.py

	python get-pip.py

安装 Pelican 和 Markdown

	pip install pelican markdown

配置 Pelican

	mkdir /path/to/your/blog

	cd /path/to/your/blog

	pelican-quickstart

回答几个问题之后，Pelican 的目录就基本配置好了，后续要修改配置的话直接更改 pelicanconf.py 文件即可

测试 Pelican

在 content 目录新建一个 test.md 文件，填入以下内容：

	Title: 文章标题
	Date: 2014-12-24
	Category: 文章类别
	Tag: 标签1， 标签2

	这里是 Pelican 测试文件

然后切换至 Pelican 目录，并生成 HTML 文件

	cd ..
	pelican content

测试生成的 HTML 文件

	python -m SimpleHTTPServer 80

直接通过 IP 地址访问 VPS，即可看到生成的页面

为了以后可以直接上传本地编写好的 markdown 文件，搭建一个 FTP 服务器，选用久经考验的 vsftp 即可

	sudo apt-get update
	sudo apt-get install vsftpd

添加用于 FTP 服务的用户，并指定家目录

	mkdir /home/ftp
	useradd -m /home/ftp -M ftp
	passwd ftp

对于 /etc/vsftpd.conf 的配置，直接按照网上的教程即可

	listen=YES
	anonymous_enable=YES
	local_enable=YES
	write_enable=YES
	
不过目录权限这块需要注意一下

	mkdir /home/ftp/wrie
	chmod 755 /home/ftp
	chmod 777 /home/write

然后在 /etc/vsftpd.conf 的结尾加一句

	local_root=/home/ftp/write

这里不能直接将 /home/ftp 目录的权限设置为777，否则会报错

然后用 FileZilla 测试 FTP 服务器，工作正常

到这里为止，博客系统已经可以通过 IP 地址正常访问了，最后还需要把购买的域名解析到 VPS 的 IP 地址

登录 DNSPod，为 VPS 的 IP 地址添加一条 A 记录，记录值指向 VPS 的 IP 地址，其余保持默认

然后登录 GoDaddy 后台，将 DNSPod 提供的两个域名解析服务器的地址 f1g1ns1.dnspod.net f1g1ns2.dnspod.net 填写到我们购买的域名的 nameserver 中，并将原来默认的 GoDaddy 自带的 nameserver 删除

等待一段时间，DNS 记录扩散到全网，博客就可以通过域名访问啦