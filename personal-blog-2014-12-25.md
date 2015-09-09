Title: 宅总的编辑器
Date: 2014-12-25
Category: 折腾

晚上刷知乎，偶然发现有人给了一幅美剧《疑犯追踪》的截图，然后问截图里电脑上的是什么文本编辑器，截图如下

<img src="http://www.deepurple.info/images/2014-12-25/snapshot.jpg" width="640")>

宅总似乎就是这部美剧里一个主人公的名字？不懂，不太看美剧

首先，根据截图里左上角的显示，这款软件是“Code Editor”

然后我以“Code Editor”作为关键词在Google、Bing、Baidu进行了搜索，大致浏览了搜索结果，唯一可能有点关系的就是下面这个项目了

[http://savannah.nongnu.org/projects/codeeditor/](http://savannah.nongnu.org/projects/codeeditor/)

项目主页：[Code Editor](http://www.nongnu.org/codeeditor/)

<img src="http://www.deepurple.info/images/2014-12-25/codeeditor.jpg" width="640")>

从主页截图来看风格倒是有点像，不管怎样，先装个试试

从[http://download.savannah.gnu.org/releases/codeeditor/CodeEditor-0.4.4.tar.gz](http://download.savannah.gnu.org/releases/codeeditor/CodeEditor-0.4.4.tar.gz)下载源码包

当我看到源码上次提交的日期是2004年4月8日的时候，整个人都不好了

十年前的代码，能编译成功的概率跟你扔一摞扑克牌然后落下之后正好按顺序排列好的概率一样大

话说我找遍了源码包也没找到安装文档之类的东西，看来还得自己动手

首先，源码路径下面有 GNUmakefile 文件，应该是 GNUstep 的项目，翻了下源码确实是Obj-C

于是 sudo apt-get install gnustep* 装好 GNUstep 环境

第一遍 make，果然报错

<img src="http://www.deepurple.info/images/2014-12-25/1.png" width="640")>

对 Obj-C 不是很熟，但是依稀记得 Obj-C 里有一个 BOOL 类型还有一个 bool 类型，BOOL 类型包括 YES 和 NO 而 bool 类型包括 true 和 false，看来这个编译错误是因为编译器错把 bool 当成类型了

最简单的方法就是直接把 bool 替换掉，sed 大法好

	sed -i "s/bool/my_bool/g" `grep bool -rl ./`

<img src="http://www.deepurple.info/images/2014-12-25/2.png" width="640")>

然后第二遍 make，这一次会报找不到 ZLIB 里某个 reference 的错误，把 zlib1g-dev 装上即可

第三遍 make，似乎顺利编译通过了

<img src="http://www.deepurple.info/images/2014-12-25/3.png" width="640")>

然后打开编译好的 CodeEditor.app

<img src="http://www.deepurple.info/images/2014-12-25/4.png" width="640")>

Critical Error，有空再修吧，不管怎么样起码菜单栏元素出来了（左上角，看不清可以点大图），然后对照菜单栏元素跟题主的截图，发现完全不一样，基本可以断定不是这个玩意了

所以我花了一个小时的时间就编译了这么一套十年前的代码，然后还没有回答开头提出的问题

我不是认真，我就是闲的
