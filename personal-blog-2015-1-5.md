Title: 个人Blog的一些额外工作
Date: 2015-1-5
Category: 杂谈

上次将个人Blog简单搭建起来之后，由于临近年底其他事情比较忙，就暂时搁置了一段时间，今晚抽了点时间把个人Blog完善了一下，记录在这里

###配置Web服务器

之前用的是Pelican自带的测试服务器，毕竟不是长久之计，于是安装配置了一下Apache作为Web Server

	sudo apt-get update && sudo apt-get install apache2

然后编辑Apache配置文件
	
	sudo touch /etc/apache2/sites-available/deepurle-info.conf
	sudo vim /etc/apache2/sites-available/deepurple-info.conf

配置文件如下

	# domain: deepurple.info
	<VirtualHost *:80>
    	# Admin email, Server Name (domain name) and any aliases
    	ServerAdmin webmaster@deepurple.info
    	ServerName  deepurple.info
    	ServerAlias www.deepurple.info

    	# Index file and Document Root (where the public files are located)
    	DirectoryIndex index.php index.html
    	DocumentRoot /home/blog/output
	</VirtualHost>

然后编辑/etc/hosts文件，添加一条记录

	127.0.0.1 deepurple.info

重启Apache

	sudo a2ensite deepurple-info
	sudo service apache2 reload

####2015-3-4补充

之前这里做的有问题，按照上面这几个步骤做完之后是无法正常访问的，临时把/home/blog/output做了一个软链接飞到/var/www然后凑合着用了

这是因为我们配置了不同的VirtualHost，并且有某个VirtualHost的DocumentRoot不在全局的DocumentRoot目录下，这时必须在全局种单独增加对该目录的Directory 项进行设置，否则该VirtualHost下的所有访问均会出现403 forbidden错误

要正常访问的话，还需要修改一下apache2的全局配置文件，位于/etc/apache2/apache2.conf，增加如下的Directory配置项

	<Directory /home/blog/output/>
		Options Indexes FollowSymLinks
		AllowOverride None
		Require all granted
	</Directory>

然后重启一下apache2即可

###防御SSH端口扫描

基本上公网上的VPS，只要开了22端口，都会被疯狂扫描，搬瓦工在购买的时候就会分配一个随机SSH端口，这一点很贴心，当然为了保证SSH安全性，还需要一些额外的配置

#####修改SSH配置文件

SSH配置文件位于/etc/ssh/sshd_config，编辑之

	sudo vim /etc/ssh/sshd_config

只允许Key登录，不允许用密码登录

不允许空密码

	PermitEmptyPasswords no

设置Allowusers为当前用户

#####安装fail2ban

fail2ban是Python实现的一个安全防御工具，它可以自动扫描log文件，并且对于使用错误密码尝试登录SSH超过一定次数的IP进行封禁

安装fail2ban

	sudo apt-get install fail2ban

fail2ban的配置文件主要有/etc/fail2ban/fail2ban.conf和/etc/fail2ban/jail.conf两个，默认配置是密码错误超过6次即封禁该IP，保持默认配置即可