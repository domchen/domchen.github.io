---
author: dom
comments: true
date: 2012-08-15 17:08:07+00:00
layout: post
slug: as3%e5%a4%9a%e7%ba%bf%e7%a8%8b%e5%bf%ab%e9%80%9f%e5%85%a5%e9%97%a8%e4%b8%80%ef%bc%9ahello-world%e8%af%91
title: AS3多线程快速入门(一)：Hello World[译]
wordpress_id: 319
categories:
- ActionScript
- Flash
tags:
- Flash Player11.4
- 多线程
---

<blockquote>原文链接：[http://esdot.ca/site/2012/intro-to-as3-workers-hello-world](http://esdot.ca/site/2012/intro-to-as3-workers-hello-world)</blockquote>


随着AIR3.4和Flash Player11.4的测试版发布，Adobe终于推出了多年来被要求最多的API：多线程！

如今，使用AS3 Workers让创建真正的多线程应用变得非常简单，只需要几行代码即可。这个API相当的简单易懂，而且他们还做了一些非常有用的事：比如ByteArray.shareable这个属性，能实现在worker之间共享内存；还有一个新的BitmapData.copyPixelsToByteArray方法，用来快速转换bitmapData到ByteArray。

在本文中，我将全部过一遍Workers这个API的各个组成部分。然后我们再看一个简单的小程序：HelloWorker。

如果你想要跟着做，你可以先下载下面的Flash Builder项目文件：
[FlashBuilder项目文件<!-- more -->](http://blog.domlib.com/wp-content/uploads/2012/08/HelloWorldWorker.zip)

如果你只是想看看代码而已，废话就不多说了，下载这个看看：[HellWorldWorker.as](http://esdot.ca/examples/HelloWorldWorker.as)

**那Worker是什么呢？**

简单说，Worker就是在你主SWF里运行的另一个SWF程序。想要更深入一点的了解，请移步：[Thibault Imbert的这篇牛文](http://www.bytearray.org/?p=4423)。

要创建一个worker，你需要调用WorkerDomain.current.createWorker()方法并传入一个SWF文件的字节流。

共有3种方式可以生成那些SWF字节流。

1.使用主SWF应用程序自身的loaderInfo.bytes属性，然后在文档类的构造函数里判断下Worker.current.isPrimordial标志(true代表当前是主线程)。这是创建Worker最快的方式，而且在即时调试周期上也有优势。

2.发布一个SWF文件，然后用[Embed]标签嵌入它到你的主程序。这种方式在Flash Builder4.7里将会有工具支持，但在工具还没出来前这种方式还是太不灵活了。每次改变worker里的代码，你都必须重新导出一次SWF，这会让人很快就厌烦的！

使用[Worker From Class](https://github.com/bortsen/worker-from-class3),这个新的类库能让你直接从类创建workers。这看起来似乎是最优的解决方案，但是它让你的项目引入了一系列的外部依赖。

在这个初始的教程中，我将集中精力在第一种方式上，使用自身的loaderInfo.bytes字节流。这是一个快速但有一定瑕疵的方式。在接下来的教程中，我将试试“Worker From Class”类库方式。

对于第2种方式，已有一个非常好的教程，可以去看看Lee Brimelow的视频教程[part 1](http://www.gotoandlearn.com/play.php?id=162)和[part 2](http://www.gotoandlearn.com/play.php?id=163).

**让我们交流吧**

在任何多线程使用场景中通信都是关键。从一个线程直接传送内存数据到另一个线程的代价是昂贵的，而内存共享又必需细致设计和周全考虑才行。在实现Worker系统过程中，大多数的挑战都来自寻求一种正确的设计架构，能让数据在你的Worker之间彼此共享。

为了帮我们解决这个问题，Adobe给了我们一些简单（但灵活）的方式来发送数据。

**(1)worker.setSharedProperty()和worker.getSharedProperty()**

这是传递数据最简单但是功能也最有限的方式。你可以调用worker.setSharedProperty(“key”, value)来设置数据，然后在另一边用WorkerDomain.current.getSharedProperty(“key”)来获取它们。示例代码：

    
    [as3]
    //在主线程里
    worker.setSharedProperty("foo", true);
    
    //在worker线程里
    var foo:Boolean = Worker.current.getSharedProperty("foo");
    
    [/as3]


你在这里存储简单或者复杂对象都可以，但对于多数情况下，存储的数据都是被序列化过的，它并不是真的被共享着。如果一个数据对象在一边改变了，在另一边并不会同步更新，直到等你再次调用了set/get方法后它才会更新。

有俩例外情况是能共享的，如果你传递的对象是ByteArray且shareable属性为true，或者是一个MessageChannel对象。巧合的是，这些正是其他两种通信方法

**(2)MessageChannel**

MessageChannels就像是一个worker到另一个的单向通行管道。它们结合使用了事件机制和简单队列系统。你在一端调用channel.send()方法，在另一端一个就会抛出一个Event.CHANNEL_MESSAGE事件。在事件的处理函数里，你可以调用channel.receive()来接收发送过来的数据。就像我提到的，这个方法类似队列。所以，你可以在两边send()或receive()多次。示例代码：

    
    [as3]
    //在主线程里
    mainToWorker.send("ADD");
    mainToWorker.send(2);
    mainToWorker.send(2);
    
    //在worker线程里
    function onChannelMessage(event:Event):void {
    
        var msg:String = mainToWorker.receive();
        if(msg == "ADD"){
            var val1:int = workerToMain.receive();
            var val2:int = workerToMain.receive();
            var result:int = val1 + val2;
             //发送计算结果给主线程
            workerToMain(result);
        }
    
    }
    [/as3]


MessageChannels对象使用worker.setSharedProperty()来实现共享，所以你想在多少个worker之间共享它都是可以的。不过它们都只能有一个发送终点。这样，约定以它们的接收者来命名似乎不错。例如channelToWorker或者channelToMain。

示例代码：

    
    [as3]
    //在主线程里
    workerToMain = worker.createMessageChannel(Worker.current);
    worker.setSharedProperty("workerToMain", workerToMain);
    
    mainToWorker = Worker.current.createMessageChannel(worker);
    worker.setSharedProperty("mainToWorker", mainToWorker);
    
    //在worker线程里
    workerToMain = Worker.current.getSharedPropert("workerToMain");
    mainToWorker= Worker.current.getSharedPropert("mainToWorker");
    [/as3]
    
    对于MessageChannel和sharedProperties通信方式都有一个重要的限制，当数据发送时它们都被序列化了。这意味它们需要被解析，传输然后在另一端还原。这个的性能开销是比较大的。由于这个限制，这两种通信方式最好是应用于间隙性地传递小规模数据上。
    
    那如果你确实要共享一个庞大的数据块怎么办？答案是使用共享的ByteArray。
    
    <strong>(3)byteArray.shareable</strong>
    
    传输数据最快的方式就是根本不传输它！
    
    感谢Adobe给了我们可以直接共享ByteArray对象的途径。这是非常强大的，因为我们几乎可以在ByteArray里存储任何数据。要共享一个byteArray对象，你需要设置byteArray.shareable=true，然后调用messageChannel.send(byteArray)或者worker.setSharedPropert(“byteArray”, byteArray)方法来共享它。
    
    一旦你的byteArray对象是共享的，你可以直接对它写入数据，然后在另一端从它上面直接读取，太棒了.
    
    基本上这就是通信的所有方式了。正如你看到的，有3种截然不同的共享数据的方式。你可以用各种方式结合使用它们。已经过完了一遍基础结构，现在让我们来看看一个简单的Hello World的例子吧。
    
    <strong>示例应用程序</strong>
    
    首先，为了让你的工程能够正确编译运行，你需要做以下的事：
    
    1.下载最新的<a href="http://labs.adobe.com/technologies/flashplatformruntimes/air3-4/" target="_blank">AIR3.4 SDK和playerglobal.swc</a>文件，并确保配置进你的项目里。
    
    2.添加编译参数："swf-version=17"到你的项目属性里。
    
    3.安装FlashPlayer 11.4 standalone debugger版本。
    
    做了以上步骤，你现在应该能看到各种Worker相关的API比如Worker，WorkerDomain,MessageChannel等等了。如果你在以步骤作中遇到困难，请参考<a href="http://www.gotoandlearn.com/play.php?id=162" target="_blank">Lee Brimelow的视频教程 </a>，最开始的几分钟有介绍这个。
    
    你也可以直接下载<a href="http://blog.domlib.com/wp-content/uploads/2012/08/HelloWorldWorker.zip" target="_blank">我Flash Builder项目</a>的一个副本，应该能运行起来。
    
    <strong>第一步：文档类</strong>
    
    首先，我们需要创建我们的文档类：



    
    [as3]
    public class HelloWorldWorker extends Sprite{
    
    protected var mainToWorker:MessageChannel;
    protected var workerToMain:MessageChannel;
    protected var worker:Worker;
    
    public function HelloWorldWorker()
    {
    	/** 
    	 * 启动主线程
    	 **/
    	if(Worker.current.isPrimordial){
    		//从我们自身的loaderInfo.bytes创建
    		worker = WorkerDomain.current.createWorker(this.loaderInfo.bytes);
    
    		//为两个方向通信分别创建MessagingChannel
    		mainToWorker = Worker.current.createMessageChannel(worker);
    		workerToMain = worker.createMessageChannel(Worker.current);
    
    		//注入MessagingChannel实例作为一个共享对象
                      worker.setSharedProperty("mainToWorker", mainToWorker);
    		worker.setSharedProperty("workerToMain", workerToMain);
    
    		//监听来自Worker的事件
    		workerToMain.addEventListener(Event.CHANNEL_MESSAGE, onWorkerToMain);
    
    		//启动worker(重新实例化文档类)
    		worker.start();
    
    		//设置时间间隔给worker发送消息
    		setInterval(function(){
    			mainToWorker.send("HELLO");
    			trace("[Main] HELLO");
    		}, 1000);
    
    	} 
    	else 
             {
    		//在worker内部，我们可以使用静态方法获取共享的messgaeChannel对象
    		mainToWorker = Worker.current.getSharedProperty("mainToWorker");
    		workerToMain = Worker.current.getSharedProperty("workerToMain");
    
    		//监听来自主线程的事件		
    		mainToWorker.addEventListener(Event.CHANNEL_MESSAGE, onMainToWorker);
    	}
    }
    }
    [/as3]


跟着下面步骤一步一步来：

1.我们使用Worker.current.isPrimordial属性来区分我们是主线程还是Worker。
2.在主线程里，我们创建MessageChannels对象并且跟worker共享它。
3.我们启动worker线程，这导致文档类再次被实例化。
4.worker被创建了，然后获取共享的MessageChannels对象的引用。

我还设置了一个小循环：每隔1000ms给worker发送一个"HELLO"字符串信息。接下来，我们将让worker返回一个“WORLD”给主线程并打印出来。

**第二步：处理事件**

现在我们已经有了共享的MessageChannels对象，我们可以轻松地在worker之间通信了。你可以看到在上面的代码中，我们已经给对象引用设置了事件监听了。所以剩下的事就是创建它们：

    
    [as3]
    //从主线程接收信息
    protected function onMainToWorker(event:Event):void {
    	var msg:* = mainToWorker.receive();
    	//当主线程给我们发送"HELLO"时，我们返回"WORLD"
    	if(msg == "HELLO"){
    		workerToMain.send("WORLD");
    	}
    }
    
    //从worker线程接收信息
    protected function onWorkerToMain(event:Event):void {
    	//打印输出worker里接收到的任何消息
    	trace("[Worker] " + workerToMain.receive());
    }
    [/as3]


如果你现在运行这个程序，将会看见每隔1000ms“HELLO”和“WORLD”就被打印一次。祝贺你完成了你的第一个多线程应用程序！

**第三步:再多一些？**

好吧，不得不承认，这个程序看起来有点没啥用处。让我们稍微加强下我们的测试程序，让它做一些数学运算。2+2的咋样？

首先，让我们修改interval函数，让它看起来像这样：

    
    [as3]
    //设置一个时间间隔让Worker线程做一些数学计算
    setInterval(function(){
    	mainToWorker.send("ADD");
    	mainToWorker.send(2);
    	mainToWorker.send(2);
            trace("[Main] ADD 2 + 2?");
    }, 1000);
    [/as3]


然后我们再修改worker里的事件监听函数，来获取这些值：

    
    [as3]
    protected function onMainToWorker(event:Event):void {
    	var msg:* = mainToWorker.receive();
    	if(msg == "ADD"){
    		//接收到两个值，然后把它们相加
                    var val1:int = mainToWorker.receive();
                    var val2:int = mainToWorker.receive();
    		//返回计算结果给主线程
    		workerToMain.send(val1 + val2);
    	}
    }
    [/as3]


就是这样！如果你现在运行这个程序，你将会看见“[Main] ADD 2 + 2?”，然后正确答案“4″会被输出。

如果你想要查看完整的类。

点击此处：[HellWorldWorker.as](http://esdot.ca/examples/HelloWorldWorker.as)

在下一篇教程中，我们将一步一个脚印，看看如何利用多线程做一些图像处理的事。
