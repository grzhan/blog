title: 舰娘SWF反编译指南
date: 2016-03-15 23:06:19
tags: 
	- 舰娘
	- 逆向工程
	- SWF反编译
categories: 黑科技
thumbnailImage: yubari.png
---

{% asset_img "yubari.png" %}

> 2016-04-11: 有邮件跟我联系说文中的方法已经失效了，有需求的同学可以去[这个链接](http://dazzyd.org/blog/2016/03/17/kantai-collection-obfuscation-anti-obfuscation/)，里面有直接反编译`Core.swf`的python脚本，截止目前是有效的。

这是一篇舰娘SWF的反编译指南。

大概是去年（前年？）开始，角川对舰娘的flash游戏本体做了加密处理，以打击各类挂机脚本和偷跑。角川先是混淆了校验参数`api_port`的生成函数（在游戏主体`Core.swf`中），后来干脆把整个`Core.swf`文件混淆了，给广大研发黑科技的提督们添了不少麻烦 \_(:зゝ∠)\_。

最近我在开发[POI](https://github.com/poooi/poi)的[语音字幕插件](https://github.com/kcwikizh/poi-plugin-subtitle)，发现如今舰娘的语音文件也是经过加密的（例如：`1.mp3`变成`163563.mp3`），而加密算法恰好就在`Core.swf`中，直接使用反编译软件是导不出来源码的。虽然POI开发组的菊苣已经成功逆向出了[语音的加密算法](https://github.com/poooi/plugin-secretary/commit/ea56456daee13e11a24ab8fc8bf62137d59fa275)（`Core.swf/common.util.SoundUtil`），但角川加密算法的key似乎会不定期更新（F**k!）。我总觉得不能每次更新了就腆着脸找人要key，加上网上有关反编译舰娘SWF文件的资料确实不多，就试着填下坑。

经过一个半晚上总算把`Core.swf`搞了出来，中间还是走了不少弯路（走了哪些弯路我会在之后的【吐槽】提到），不过总结之后的解决方案操作起来其实还是很简单的。

这篇文章是写给那些想要搞舰娘黑科技但跟我一样**缺乏Flash开发基础**的提督的，至于之前就是Flash开发者的提督，我相信自己随便逆逆改一下就出来了，也不需要看这种东西。

本文操作环境是`OSX Yosemite`，并涉及这两个工具软件：

+ [JPEXS](https://www.free-decompiler.com/flash/), 又名`FFDEC`，SWF文件的反编译软件。就逆向SWF而言感觉比Sothink（硕思）好用多了，而且开源、免费、跨平台。
+ [Adobe Flash CC](https://www.baidu.com/s?wd=adobe%20flash%20cc)，Flash的开发环境。本文使用的是Mac上的`Adobe Flash CC 2014`破解版，破解过程的话就是从`Adobe PhotoShop CS 6`的破解版里拿出`amtlib.framework`放进去，有需要的同学可以发邮件联系我。

由于之后我们需要修改舰娘Flash文件的ActionScript代码，如果有编程基础（会Javascript更好）读起来不会有什么问题。

<!-- toc -->

## 反编译

我们简单分析一下舰娘游戏的swf加载过程，我们首先看到的是个舰船动画，对应的文件是`mainD2.swf`。

{% asset_img screenshot_1.png %}

通过JPEXS打开`mainD2.swf`我们就可以把资源和AS脚本文件都提取出来，而要关注的脚本文件则是其中的`mainD2.as`。

{% asset_img screenshot_2.png %}

（要分析的`mainD2.as`源码在[这里](https://gist.github.com/grzhan/0647a162dc728d1d30b2)可以看到）

`mainD2.as`中定义的是`mainD2`类，而这个类中有这么几个方法是需要了解的：

+ `mainD2`，即`mainD2`的构造方法，各类初始化操作都在这里进行。
+ `_handleAddToStage`，`addedToStage`事件的回调方法。里面有请求获取`Core.swf`字节流的逻辑（对应`uclLoader`加载完成事件）。事实上我不知道也不需要知道`addedToStage`事件是啥以及怎么触发，之后我们只需要把事件调用删掉，改成在构造函数直接调用这个方法就好了。
+ `_handleLoadComplete`，UrlLoader加载完成事件的回调方法。该方法会获得之前在`_handleAddToStage`打算加载的`Core.swf`的字节数据，利用类中的反混淆方法（`___`）对数据进行解密，再将解密后的字节数据通过`flash.display.Loader`的`loadBytes`方法作为SWF文件加载。
+ `_handleLoadComplete2`，`Loader.loadBytes`方法加载`Core.swf`完成事件的回调函数。这里是对已经加载成功的`Core.swf`进行设置属性等操作了，我们将使用`FileReference.save`函数将已经成功载入的`Core.swf`文件导出，完成整个`Core.swf`的反混淆工作。

总结一下，就知道`mainD2.as`做了这么件事情：读取`Core.swf`的字节数据，将它们进行反混淆，再作为SWF文件加载到Flash中。那么接下来我们要做的就是改写`mainD2.as`文件，并编译一个新的`mainD2.swf`，将反混淆后的`Core.swf`导出。

## Hack MainD2.as

修改`mainD2.as`主要是要做这么几件事情（注意JPEXS的编辑功能还很不完善，所以不要直接在JPEXS修改ActionScript代码）：

1. 删除一些不必要的代码，例如在构造函数中与`_anim`有关的代码（即那个舰船浮动的动画），以及在`_handleAddedToStage`中的stage部分代码
2. 将`addedToStage`的事件调用改成在构造函数中的直接调用`_handleAddToStage`
3. 在`_handleLoadComplete`中，有个正则判断，当以本地文件系统调用时（URL以`file:/`开头）不会调用反混淆函数，所以这部分也需要改掉
4. 保险起见，我把调用`SwfVer.as`的`getSWFVersionsObject`方法直接加到了`mainD2.as`文件里
5. 在`_handleLoadComplete2`添加Flash导出语句，使用`LaderInfo.bytes`获得`Core.swf`字节数据，并用`FileReference`类的`save`方法保存反混淆后的文件。注意，是在`_handleLoadComplete2`中导出文件而不是在`_handleLoadComplete`中立刻把`_loc3_`导出，事实上刚经过反混淆过的字节数据并不是SWF文件格式（推断应该是缺少SWF文件头之类的），只有经过Loader载入后的字节数据才是有效的。

将改完的文件保存为`mainD2Hacked.as`，具体的Hack文件在[这里](https://gist.github.com/grzhan/ffd3a30b7fe10406907b)，可以直接下载下来用，如果日后`mainD2.swf`的加密算法发生变化，用新的`___`方法替换掉就行了。

（现在想想有些地方应该是不用改或者不用删的\_(:зゝ∠)\_）

## 编译导出Core.swf

有了`mainD2Hacked.as`之后我们就需要重新编译一个新的`mainD2.swf`文件了。

接下来使用JPEXS的【Export to FLA】功能，将所有的资源以FLA格式导出，再使用`Adobe Flash CC`打开。

打开Adobe Flash之后，在右侧的【Properties】面板中，有个Class选项，点击铅笔图案的编辑按钮，就可以对`mainD2.as`进行编辑

{% asset_img screenshot_3.png %}

我们用之前改写的`mainD2Hacked.as`文件将`mainD2.as`覆盖掉，然后点击菜单栏上的【Debug】==》【Debug】调试按钮，就可以进行`mainD2.swf`的编译了。

编译完成后开发环境就会以调试模式运行`mainD2.swf`，很快就会跳出请求保存`result.swf`的对话框，这就是反混淆后的`Core.swf`文件。

{% asset_img screenshot_4.png %}

再将`result.swf`丢进JPEXS里，就可以看到`Core.swf`的反编译结果了wwww

至此，我们已经拿到`Core.swf`的代码，之后想拿Key的拿Key，想作死的作死……（快够）。

由于本文具有时效性，可能过个一年半载文章的内容就会过期了，不过有问题的话仍然欢迎通过在本文评论或者邮件联系我。

下面是吐槽，跟反编译的解决方案可以说没什么大关系，所以大可以跳过不看（。


## 吐槽

这里说下在填坑过程中走的弯路，其实其中还有很多问题没有解决的，如果有懂这方面的菊苣可以回复解答一下。

一开始填这个坑的时候我并没有打算用破解版的`Adobe Flash CC 2014`，而是想直接用Flex的命令行编译器来解决问题。

我去Adobe官网下了Flex的SDK，然后发现这个SDK竟然依赖32Bit的Java（64位不行哦），而在Oracle官网里竟然MacOS只有64位的JDK……据说苹果官网有老版的32位，但找了半天没找到，后面就干脆开了个Ubuntu的32位虚拟机，装下了32位JDK和Flex SDK……

接着，Parallel Desktop开的虚拟机莫名其妙文件共享就不能用了（剪贴板也不共享），我就傻逼呵呵地借了室友的优盘，把主机里的数据拷进优盘，再重插将数据拷进虚拟机……

之后就折腾`mxmlc`的命令行选项，由于安全策略和编译器默认值的关系所以也翻man翻了半天，最后的`mxmlc`命令是这样的：

	mxmlc -use-network=false -debug=true -omit-trace-statements=false mainD2.as

结果使用`Flashplayer Debugger`，`Core.swf`的加载完成事件不知为何没法触发（难道是安全策略问题？），而光是在`_handleLoadComplete`中只经过反混淆而尚未作为SWF加载的字节数据也是没法直接用的……导致我还以为JPEXS反编译出来的混淆函数是错的（因为实际分析这个混淆函数的行为感觉还是挺奇怪的，join整个mainD2对象什么的）

后来发现如果开个服务器（`python -m SimpleHTTPServer 80`，还得用80端口才行）`Core.swf`可以被`mainD2.swf`加载。这算是个好消息，证明反编译出的混淆函数是可以用的。然后我就打算直接用这个办法调试……就遇到无法保存的问题：`Flex Error #2176`，查了下这是`Flash Player 10`开始的UIA安全策略，在Web浏览器中限制了`FileReference.save`的使用，必须由用户点击触发才行……下面的一篇文章给出用Alert警告框来强制触发用户点击的办法……

在虚拟机下用命令行搞SWF算是消磨掉了我的耐心，之后想想干脆就在Mac下个`Flash CC`的破解版，没想到一试就行……

当然其实还是有蛮多备选的解决方案，包括直接改SWF的PCODE、将Bytes数据通过POST上传给自己写的PHP脚本（类似文件上传机制）……诸如此类。

回过头看，也难怪网上没什么舰娘反编译的资料，因为操作起来确实简单，根本不值一提，也就像我这样的会莫名纠结半天。果然自己水平不足，还需要多多努力才行qwq（。

## 参考资料

+ [「艦隊これくしょん」混淆与反混淆](http://dazzyd.org/blog/2016/03/17/kantai-collection-obfuscation-anti-obfuscation/)
+ [舰娘之混淆代码分析](http://blog.mukio.org/articles/remove-kancolle-obfus/)
+ [$0 Flash Development: Quick Intro to MXMLC](http://gamedevlessons.com/lessons/mxmlc-tut/)
+ [How to write your own AS3 project using VI and how to compile it under Ubuntu linux ](https://vrichard86.wordpress.com/2012/12/02/how-to-write-your-own-as3-project-using-vi-and-how-to-compile-it-under-ubuntu-linux-actionscript-3-0/)
+ [Flex error #2176 when using FileReference](http://www.bogdanmanate.com/2010/05/12/flex-error-2176-when-using-filereference/)
+ [ant编译swf的选项（mxmlc的简单编译）](https://gist.github.com/zrong/944712)

<style>
@media screen and (min-width: 768px) {
	.post-content img {
		max-width: 50% !important;
	}
}
</style>