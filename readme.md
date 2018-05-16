## 20180320

最近集中将代码迁移到github，顺便将blog也逐步迁移过来，平时较忙，除了感觉有些创新的，写的文章也很少               
国内git带宽怎么这么低啊，图片都显示不粗来~~~

---

### 20180410
今天听闻Android P开始限制私有API的访问权限，这导致我们项目的基础架构插件框架无法使用，思考其影响，其他的都好说，最重要的是资源加载这块，可以说一个apk的代码分为两种，java代码和xml这种dsl代码，java代码通过类加载器加载dex加载，由于android没有ios的类加载限制，签名校验自己负责，因此java代码加载目前没有问题，而视图代码也就是resource那部分就不同了，使用AssertManager加载，由于android studio相比ios的开发工具好用很多，所以开发视图方面不像ios，android基本都是用xml这种dsl模板开发，这导致如果限制资源加载，那么庞大的视图代码无法动态加载，头疼，本来处理context代理以及资源id固定已经让插件机制十分脆弱，现在更是雪上加霜
### 20180420
今天尝试使用伪造android包名的方式访问framework中的protected或者default属性和成员，失败，查找原因[JVM对于不同classLoader加载的对象之间default或protected字段存在访问限制](https://www.iflym.com/index.php/code/jvm-constraint-for-default-or-protected-field-between-classloaders.html)，也就是说这两个字段的真正意义在于，保护JRE或者art中的类访问权限，只能说自己太naive了
### 20180421
[“Java泛型有这么一种规律： 
位于声明一侧的，源码里写了什么到运行时就能看到什么； 
位于使用一侧的，源码里写什么到运行时都没了”
——RednaxelaFX](http://rednaxelafx.iteye.com/blog/586212)

吐血，再吐三公升血
### 20180421
不得不说，网络上的技术水文到了影响别人学习的地步，差点的是虎头蛇尾，说一通网上到处都是的技术点，文章不是文章，笔记不是笔记，好点的是人云亦云，写一堆别人都写烂的东西，搜一个知识点，写的内容都一样，真是不明白这些人怎么那么多时间，刷这些水文
### 20180422
今天做个小需求，要在Activity启动的时候弹个浮层，用到了多年不用的PopupWindow和View.post()方法，想起了以前用View.post()是为了获得控件高度，但仔细想了想，ActivityThread在handleLaunchActivity、handleResumeActivity这两个同步操作之后，只是把view add到WMS中，真正的测绘需要Choreographer下一个Vsync同步信号到来时才会执行，虽然ViewRootImpl加入了SyncBarrier，但那么View.post()是在加入屏障之前的任务，不会受到屏障影响，View.post放入handler的任务应该在测绘之前就执行了啊，不解，遂先Google之实在不行再看代码，[果然有细心的同学已经研究过了](http://androidperformance.com/2015/12/29/Android%E5%BA%94%E7%94%A8%E5%90%AF%E5%8A%A8%E4%BC%98%E5%8C%96-%E4%B8%80%E7%A7%8DDelayLoad%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%92%8C%E5%8E%9F%E7%90%86-%E4%B8%8B%E7%AF%87.html)，我一直以为View.post只是把任务简单抛到主线程的handler，原来并没有那么简单，View.post是委托给ViewRootImpl做了更多的同步操作，这位同学是一个细心认真的同学，不过有一点写错了，performTraversals一开始确实会执行两次，但都是同步的，和executeActions里面那个Post无关，那个post只是为了在同步任务执行完毕后执行。

今天写的多点，[看到一篇文章](https://blog.csdn.net/cdecde111/article/details/54670136)，这篇文章的作者是个不错的同学，利用SyncBarrier实现预加载，还在github上实现了完整的tools，但这位同学把这件事情弄复杂了，实际上在onCreate或者onResume里本来就是要让你加载数据的，你把数据加载了，后面的测绘(layout measure draw)才不会在数据加载完后再重复执行，并且异步数据请求，最终都是要抛到handler任务队列里，既然都是要抛到主线程队列，那么在onCreate这样一个方法内部再做preload、readload岂不是多此一举，你只要load，数据返回时自然onCreate之类的已经执行完毕了，如果说要在startActivity之前做预加载，那么做个双向观察者就可以了，还有[Bugly公众号推荐的这个文章](https://mp.weixin.qq.com/s/KpeBqIEYeOzt_frANoGuSg)，在向Choreographer post一个Vsync回调帧之后，要等到Vsync到来才会往handler中post一个测绘任务，在这中间handler中的msg queue如果空闲，就会先执行idleHandler，当然由于文章中的数据请求本身就是异步任务，所以addIdleHandler只有极特殊情况下才会先于Vsync信号执行，或者说在Vsync回调之前msg queue根本不会为空，但Choreographer的postFrameCallback确实不等同于handler.post,因为回调帧本身就是异步任务，其实google在测绘开始前加入SyncBarrier，只要通过View.post或者handler.post(post())是可以达到绘制完成的回调效果的,可以这么说View.post是回调view绘制完成的最佳手段
### 20180422
Window.Callback 有意思 :)，所有的window事件，都从ViewRootImpl收到，然后交个DecorView，DecorView先给Activity预览一下，之后分给小弟们
### 20180424
今天研究bugly上一个listview数组溢出问题，重新翻了翻view的重绘和relayout流程，突然发现以前一直有个盲点，就是在view invalidate冒泡的过程中，会不断的向parent提交自己的dirty区域，但各种各样的文章，例如[这个](https://blog.csdn.net/litefish/article/details/52859300)、[这个](https://juejin.im/entry/57abeac5a341310060dbd7fd)，都在说parent会把这个dirty和自己做union，也有说这个union是交集也有说并集，要说交集可以理解，但并集就匪夷所思了，并集到最后，不就是整个DecorView吗，但是union这个方法，让我相信是交集，打死我也不信，于是先看看官方doc，“Update this Rect to enclose itself and the specified rectangle.”，“enclose”确定是并集无疑了，但并集不符合逻辑啊，遂查看ViewRootImpl和ViewGroup代码，ViewRootImpl确实将mTmpInvalRect这个计算结果和自己的mDirty做了union，这个比较合理，因为一次Vsync间隔内或多次invalidate可能多个区域需要重绘，ViewGroup中invalidateChildInParent方法内容就比较多了，但归根到底是那些人忽视了FLAG_CLIP_CHILDREN这个flag，看到代码中有个union想当然的按自己的想法理解，实际上FLAG_CLIP_CHILDREN这个flag对dirty的计算很重要，这个flag的意思是子view是否可以超出parent区域，由于view的绘制是从根到叶，child最后绘制，所以默认情况下这个flag是被true，也就是对child进行clip，而invalidateChildInParent方法中计算dirty的时候，会根据如果这个flag是true就进行dirty.intersect，这才是交集，而如果flag是false，才会进行union，为什么union显而易见，因为子view会超出parent区域，所以只能union不能intersect，不知道那些作者再说父窗口会根据这个rect和父窗口本身做union的时候，有没有思考过union是什么意思
### 20180430
一直以来都以为AMS WMS PMS等系统服务都是独立进程，后来碰到此文[Binders & Window Tokens](https://www.androiddesignpatterns.com/2013/07/binders-window-tokens.html)，好奇binder是如何在wms和ams以及app process三个之间实现a unique identity的，ams和app之间可以理解，本来就是一个server对应一个client，但wms是如何知道ams和app传递给他们的token是同一个呢？网络上介绍binder的文章非常多，介绍token唯一性的也不少，但对于具体的实现细节，却几乎没有，其实这也不是什么复杂的东西，就是binder的基础内容，简单来说就是binder也就是BpBinder和BBinder在IPC传递的时候，binder驱动会对这个对象进行特殊处理，保证同一个进程同一个编号的binder只有一个,或是BBinder或是BpBinder，这样在wms使用binder作为hashmap的key就能保证这个编号的binder进程内对象的唯一性，还有一个有趣的事情就是，原来wms ams等服务都是运行在SystemServer进程中，仅仅是不同的workthread，也就是说ams传递给wms的token实际上是同一个BBinder对象
### 20180431
最近三端是不是到了瓶颈了，各种跨平台技术层出不穷，甚至有用cocos2d搞跨平台的，有趣的是Google又搞了个Flutter，虽说是为了跨平台吧，这和framework的UI tookit团队算是左右手互搏呢还是两条腿走路呢，当然Google有的是人，把Flutter丰富到和framework一样的水平，也是非常nice的，但从原理上讲，Flutter要比RN以及RN的copy版weex这些高一个档次，RN相当于是复用了现成的前端布局体系，然后bridge到各个Platform上，核心点只有一个，那就是让前端工程师可以开发native页面，但弊端也很明显，需要为每个Platform实现不同的bridge，再加上FB投入的人力有限，无法实现大量丰富的bridge，导致RN只支持部分css，满足不了复杂页面的需求，而Flutter完全抛弃了这种做法，第一个就是重新定义了一套新的布局体系，当然这和跨平台无关，第二就是基于更底层的渲染引擎构建，这是Flutter跨平台的方式，这使得Flutter成为真正意义上的跨平台视图框架，当然Flutter的学习成本也是非常高的，一套新的布局体系导致无论是前端还是android iOS都需要学习新的技术，并且Flutter还使用Dart，这导致三端又都需要学习新的语言，可以说Flutter的前景很美好，使得三端真正打通，相当于又做了个H5+webKit，当然推广的难度也更大
### 20180501
闲来无事扯一扯跨平台，可以这么说，其实今天大前端走的路线，都是历史重演，跨平台这个事怎么说呢，要说一份代码多平台运行，那么java本身就可以，当年java在server端风风火火，于是想把触手伸到客户端，大有一统江湖的气势，不过结果却不怎么好，主要就是因为java遇到了gui这个硬骨头，java在gui上的努力经历了awt、swing、swt几个阶段，由于gui在各个平台完全不同，所以awt的时候java还想的比较简单，只支持相同的，结果可想而知，慢是其次，关键需求都做不出来，谁用啊，java一看不行，又搞出了swing，swing和flutter flash等思想一致，就是所有控件都是自己画，关键自己画你就画全点啊，画那么几个简陋的控件，design又不能与时俱进，显然又是被淘汰的命运，java一看这样也不行，自己穷雇不了那么多人画控件，那就awt+swing，能复用原生的复用原生的，其实也就是react native的思路，关键是适配工作量也不小啊，结果不用说大家也知道了。

说到这里，其实RN Flutter weex以及其他一些跨平台的思路也比较清晰了，接下来说说Android自身，Android体系主要包括：linux kernel(提供基础的驱动适配、进程、内存、文件系统等基础设施)、native library(主要包括openGL、Skia以及音频等基础库)、然后就是我们接触最多了VM，VM主要加载libcore和framework core，libcore是java核心类库，为runtime提供java api功能，framework core主要是android的核心类库，主要包括两种，一种是ams、wms、pms这些系统服务，负责调度系统资源，包括进程管理、内存管理、屏幕显示和输入管理以及一些琐碎的组件管理等，另外一种是GUI或者说View System，这部分toolkit是应用程序使用最多的类库，也是跨平台技术主要处理的部分，主要是为app提供丰富的布局体系，大部分情况下可以避免app直接操作GI从而减小开发难度。像刚才说的那样，ReactNative目前的实现类似于swt，通过适配系统GUI来达到跨平台的目的，而FLutter采用的是swing和flash的思路，抛弃GUI从skia那里自己动手画，抛弃性能不谈，开发者其实不关心你怎么实现，只关心你是否能够提供丰富便捷的布局体系，如果能像RN一样复用现有的CSS那更好，Flutter那样自己再弄一套也行，不过话说回来，虽然之前java的尝试失败了，但不代表移动端的尝试也会失败，没准Google财大气粗能对Flutter和dart提供强有力的支持，再加上自己掌握android作为推广，或许能够使得Flutter成功，Facebook这个前端派也想一桶江湖，不过目前看其在RN的表现不算十分给力，当然，其实最后可能还有另外一种结果，那就像windows那样，一个平台获胜，那就不用跨平台喽 :)

### 20180502
“面向钱编程”，这笑话让我笑了一天 :)

### 20180503
今天思考一个有趣的问题，不管是系统还是组件，之前都是关注启动过程，那么结束过程呢？如果在一个方法中同时调用performClick和finish(),或者恰好用户点击屏幕之后，wms还没来得及分发事件，就调用了finish(),这个事件会被执行吗？结果一看源码，发现没那么复杂，在finish()方法中，调用AMS是个同步方法，也就是说在AMS把所有的Activity资源断开之前，主线程是挂起的。。。。,姑且猜测App进程调用AMS应该都是同步的（除了start组件这种），而AMS调用APP都要经过handler，都是异步的，一方面避免挂起AMS，另一方面也是因为App进程的单线程模型

```
if (ActivityManager.getService().finishActivity(mToken, resultCode, resultData, finishTask)) {
                    mFinished = true;
}
```
### 20180504
呜呼，看着那些优秀的开源代码结构设计，什么时候自己才能参与设计这样的项目，或者像RednaxelaFX一样成为VM界的大牛，被人膜拜！

还是说正题吧 :) ,昨日研究起activity结束情景来，一发不可收拾，产生了各种各样的疑问，在回忆起以前解决过的插件框架因为Activity重建导致的问题，遂深入研究一下，发现了一篇好文章，[Android后台杀死系列之二：ActivityManagerService与App现场恢复机制](https://www.jianshu.com/p/e3612a9b1af3)，这篇文章详细描述了android进程回收和恢复机制，写的很好，我就不转载了，从这篇文章主要了解到三个知识：AMS恢复app栈或者activity是通过ActivityRecord、binder服务端被杀客户端会收到讣告、activity保存的state会跨进程传输到AMS。第一点我之前也是这样猜测的，第二点属于纯新的知识，最是这第三点让我惊讶，我之前一直以为activity状态会序列化到磁盘上存储，没想到是跨进程传输到AMS，这让我很吃惊，那么很显然出现了一个问题，如果这个state超过binder限制1M，不就崩了吗？于是实验验证，果然崩了。。。，Google你这不是坑爹吗，不过API 21之后好像增加了一个新的方法onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState)，可以进行持久化存储
### 20180509
Google I/O大会，“有35%的专业Android开发者在使用Kotlin”，我想说那你的Dart呢，Flutter用dart，Kotlin又是Android 开发一级语言，到底闹哪样，ios那边推swift，冲你们这瞎搞的状态，只能让我感觉Html5才最有可能成为未来三端统一架构的开发语言，只要未来Google和Apple不要给使绊子就行，别拦着我，我要投身前端阵营了 :)
### 20180510
NewRelic真傻啊，居然不知道混淆自己代码保护自己的知识产权，国内什么OneAPM、GrowingIO、redware、twoapm各种抄袭人家的源码，甚至包括我们网易的APM也是参考GrowingIO，不过这样也挺好的，可以让程序员们共享技术成果，避免出现Oracle那样的专利流氓，中国的程序员在技术分享方面还是很赞的
### 20180512
周六开始学习Html和JavaScript的基础知识，突然冒出一个有意思的想法，Flutter是在skia canvas的api基础上封装的布局框架，但开发语言是dart，所以需要dart虚拟机的支持，并且Flutter貌似只支持iOS和Android平台，那么既然Html5也暴露 了Canvas API，我们可不可以基于Html5的canvas 元素和Flutter的布局体系，封装一套前端的Flutter布局框架呢？这样前端开发就可以抛弃DOM和CSS那套比较老的UI操作方式

```
var w=window.innerWidth
|| document.documentElement.clientWidth
|| document.body.clientWidth;

var h=window.innerHeight
|| document.documentElement.clientHeight
|| document.body.clientHeight;
```
前端这历史包袱，够沉重的。。。。
### 20180513
今天负责需求的同学跟我反馈灰度包出了很多jsoup ClassNotFound的崩溃，追查问题后发现是因为一位小同学在将支付代码插件化的时候，把一个第三方lib遗留在了主项目中，而这个lib依赖的jsoup库却在插件apk中，他们虽然觉得和这个有关，但说不清楚为什么，其实主要还是类加载机制不熟悉，我们的插件是用从主项目派生的类加载器加载，第三方lib是被主项目的classloader加载，当他调用jsoup的时候必然会从自己的classloader也就是主项目的Cl向上查找，而jsoup实际在插件中，插件的类加载器是主项目派生的，显然找不到了，甚至有一位同学认为：classA(classloaderChild 派生自 classloaderParent)调用classB(classloaderParent)调用classC，这种调用关系情况下，虚拟机也会去classA的类加载器classloaderChild里去寻找classC,
### 20180514
今天又来了新问题，一个同事问我listview嵌套grideview怎么整，一下把我问懵逼了，我说：你说啥?，小同事给我描述了一下需求，也说了以下网上最普遍的两种解决方案：

```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int expandSpec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2,
                MeasureSpec.AT_MOST);
    super.onMeasure(widthMeasureSpec, expandSpec);
}
```
第二种就不贴代码了，[ScrollView嵌套两个及以上ListView的解决方案](https://www.jianshu.com/p/bae7c596a1c6)
，第一种一看就被我pass了，你这不就成gridlayout了吗，第二种好点，但一是比较复杂，二是也会绘绘出一屏，既然俩都不行，我又思考了一下，发现陷入了他的思维误区，哪有什么listview嵌套GridView，根本不需要里面那个GridView，不就是外面那个listview多了几种view type吗，犯这种错误还是对listview没有完全理解
### 20180515
前段时间两位同事去西二旗面试新人，聊起了单例的问题，说起单例可能是码农界最常见的问题了，但两位同事对单例的看法却让我很是吃惊，他们认为，如果你写一个饿汉式的单例，显然是不过关的，而我对此却持不同的看法，实际上根据我的经验，在没有静态方法或者静态函数的单例上，使用单例必须获得单例的实例（在这种情况下实际上会触发类的加载,而类的加载和初始化是同步的），在这种情况下，你使用加锁的getInstance获得单例与饿汉式并没有明显区别，反而因锁的存在降低程序性能，假如你的单例真的有静态成员，那么你使用静态内部类的方式创造单例也是可以的，总之我认为那些双重检查或者加锁的单例模式都是不可取的。

最近面试也问到了广播的一些东西，虽然大家平时用广播基本上也就是普通广播和Local广播，但某些知识在某些场合，还是有些用处，因此又重新复习了一下广播相关的东西，当然还是老套路，先看使用方法，再看系统源码，过程不提了，没什么新意，贴两个还不错的文章做个记录吧，[BroadcastReceiver的原理和使用](https://blog.csdn.net/yueqian_scut/article/details/51298996)，[Android广播工作过程分析](https://jdqm.github.io/2017/09/26/broadcast-analysis/)、[android 限制广播消息的接收者](https://blog.csdn.net/mingli198611/article/details/17762149)。

sharedUid与uid，挺有意思，原来ContentProvider可以利用这玩意做multiprocess，知乎上的这个讨论也挺有深度，[app覆盖安装的时候，会改变UID吗?](https://www.zhihu.com/question/31800220)

如果在一个同步过程中调用多次startActivity，这些新启动的Activity生命周期会怎么样? AMS只会安排启动最顶层的那个Activity，当这个Activity finish以后，下面的Activity才会走onCreate onResume之类的流程，也就是说，startActivity之后，这个Activity虽然在栈中有记录，但在APP进程真不一定会启动
### 20180516
今天学了一招，触摸事件触发过程中，如果move事件超出了TouchTarget的范围，那么点击事件不会触发，详细过程如下：

```
case MotionEvent.ACTION_MOVE:
    if (clickable) {
        drawableHotspotChanged(x, y);
    }
    // Be lenient about moving outside of buttons
    if (!pointInView(x, y, mTouchSlop)) {
    // Outside button
    // Remove any future long press/tap checks
		removeTapCallback();
		removeLongPressCallback();
		if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
			setPressed(false);
		}
		mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
	}
    break;
```
pointInView会再次判断触摸事件区域是否在自己坐标内，如果不是，会在removeTapCallback等方法中清除press状态