最近在做一些有关art的研究工作，碰到一些有趣的东西，准备写一篇介绍art和jvm的成体系的文章，由于文章计划篇幅较大，因此先写一些知识点的介绍。

本篇文章主要由一个有趣的虚函数调用欺骗的demo小程序出发，介绍art虚拟机本地方法调用的动态绑定和静态绑定机制，以及编译器对于虚函数调用的优化，同时从这个小demo展示android N上的方法内联，由于目前网络上存在的介绍art执行类方法过程的文章都缺少本地方法调用的动态绑定和静态绑定的分析，导致很多读者看完后很难理解类方法在*Marshmallow*之前的运行机制，因此本文主要介绍这方面的内容。

**1.本地方法调用的动态绑定和静态绑定机制以及编译优化**

如图1所示，我们编写了一个demo小程序，这个demo中有个call方法接收一个ClassChildA类型参数，在方法体中，new了一个Invoker类型的对象，并将这个对象的shadow$_klass_通过反射的方式设置为了另外一个名为patch的ClassChildB.class，替换shadow$_klass_主要用于虚函数欺骗。
```
public void call(ClassChildA paramClassChildA) {
    if (this.patch != null) {
      try {
        Invoker invoker = new Invoker();
        Field localField = Class.forName("java.lang.Object").getDeclaredField("shadow$_klass_");
        localField.setAccessible(true);
        localField.set(invoker, patch); //patch is ClassChildB.class
        Object localObject = invoker.a(paramClassChildA);
        Toast.makeText(this,(String)localObject,Toast.LENGTH_SHORT).show();
        return;
      } catch (Exception localException) {

      }
    }
    paramClassChildA.printName(this);
```
invoker对象初始化后便调用了方法a，其中方法a是一个public类型的方法，也就是一个虚方法，属于动态绑定，对于面向对象的语言，多态也就是动态绑定是三大主要特性之一，也是最核心机制，那么什么是动态绑定呢，绑定就是确定方法与类的关系，绑定分为动态绑定和静态绑定，静态绑定就是编译期就能确定方法属于哪个类，例如java中的private方法，final方法，static方法，init方法以及super调用，这些方法都是静态绑定，而interface，protected和public方法，由于可以被override（为了避免理解混乱还是英文最直接:)），这些方法需要运行时根据调用的对象来确定方法属于子类还是父类，因此这些方法被称为虚方法，属于动态绑定。有关动态绑定和虚函数的更多知识请参阅：[动态绑定 vs 静态绑定](http://www.importnew.com/7751.html),[蘑菇街Android热修复探索之路](https://mp.weixin.qq.com/s/GuzbU1M1LY1VKmN7PyVbHQ)。

对于虚函数的调用，android的虚拟机典型指令是Invoke-virtual，虽然android使用的java的api作为开发语言，但实际上android的基础库实现和oracle并不相同，虚拟机的指令集和实现也完全不同，android虚拟机包括dalvik和art都是基于寄存器的三地址指令集架构，而jvm，hotspot采用的是基于栈的零地址指令集架构，Dalvik VM的主要设计者[Dan Bornstein](http://www.milk.com/home/danfuzz/)在Google I/O 2008上做过一个[关于dalvik内部实现](http://sites.google.com/site/io/dalvik-vm-internals)的演讲，介绍了dalvik这样设计的原因，有兴趣的同学可以去研究一下。

在mashmellow之前的art，app安装之后会将字节码预先编译为机器码，而在Nougat上，google为art设计了interpret，jit，和aot三种运行时，我们首先看一下虚函数调用例如invoke-virtual指令和静态绑定例如invoke-direct指令编译成机器码分别是什么样子。
```
3: void bear.mylaboratory.MainActivity.call(bear.mylaboratory.ClassChildA) (dex_method_idx=17154)
    DEX CODE:
      0x0000: 5453 e119                	| iget-object v3, v5, Ljava/lang/Class; bear.mylaboratory.MainActivity.patch // field@6625
      0x0002: 3803 2c00                	| if-eqz v3, +44
      0x0004: 2200 3508                	| new-instance v0, bear.mylaboratory.Invoker // type@2101
      0x0006: 7010 f942 0000           	| invoke-direct {v0}, void bear.mylaboratory.Invoker.<init>() // method@17145
      0x0009: 1a03 7f2f                	| const-string v3, "java.lang.Object" // string@12159
      0x000b: 7110 5d43 0300           	| invoke-static {v3}, java.lang.Class java.lang.Class.forName(java.lang.String) // method@17245
      0x000e: 0c03                     	| move-result-object v3
      0x000f: 1a04 8743                	| const-string v4, "shadow$_klass_" // string@17287
      0x0011: 6e20 6343 4300           	| invoke-virtual {v3, v4}, java.lang.reflect.Field java.lang.Class.getDeclaredField(java.lang.String) // method@17251
      0x0014: 0c01                     	| move-result-object v1
      0x0015: 1213                     	| const/4 v3, #+1
      0x0016: 6e20 0144 3100           	| invoke-virtual {v1, v3}, void java.lang.reflect.Field.setAccessible(boolean) // method@17409
      0x0019: 5453 e119                	| iget-object v3, v5, Ljava/lang/Class; bear.mylaboratory.MainActivity.patch // field@6625
      0x001b: 6e30 0044 0103           	| invoke-virtual {v1, v0, v3}, void java.lang.reflect.Field.set(java.lang.Object, java.lang.Object) // method@17408
      0x001e: 6e20 fa42 6000           	| invoke-virtual {v0, v6}, java.lang.Object bear.mylaboratory.Invoker.a(java.lang.Object) // method@17146
      0x0021: 0c02                     	| move-result-object v2
      0x0022: 1f02 8008                	| check-cast v2, java.lang.String // type@2176
      0x0024: 1203                     	| const/4 v3, #+0
      0x0025: 7130 ec42 2503           	| invoke-static {v5, v2, v3}, android.widget.Toast android.widget.Toast.makeText(android.content.Context, java.lang.CharSequence, int) // method@17132
      0x0028: 0c03                     	| move-result-object v3
      0x0029: 6e10 ee42 0300           	| invoke-virtual {v3}, void android.widget.Toast.show() // method@17134
      0x002c: 0e00                     	| return-void
      0x002d: 0d03                     	| move-exception v3
      0x002e: 6e20 f442 5600           	| invoke-virtual {v6, v5}, void bear.mylaboratory.ClassChildA.printName(android.content.Context) // method@17140
      0x0031: 28fb                     	| goto -5
```

上图展示了call方法的字节码，new Invoke这条语句主要包括两条指令，一个是new-instance分配内存，主要由pAllocObject函数实现，另外一个是invoke-direct指令调用init方法进行初始化，我们首先看一下执行init这条指令的机器码
```
0x005ba8c4: aa0003f6	mov x22, x0				::x22=invoker
0x005ba8c8: aa1603e1	mov x1, x22				::x1=x22
0x005ba8cc: aa1303e0	mov x0, x19				::x0=call方法对应的artmethod指针
0x005ba8d0: b9400400	ldr w0, [x0, #4]		::w0 = [x0 + 4] = artmethod中dex_cache_resolved_methods_字段,w0=x0+dex_cache_resolved_methods_offset
0x005ba8d4: d282fb08	mov x8, #0x17d8			::
0x005ba8d8: f2a00048	movk x8, #0x2, lsl #16	::x8=<init>方法在dex_cache_resolved_methods_中的偏移
0x005ba8dc: f8686800	ldr x0, [x0, x8]		::x0=<init>方法对应的artmethod对象，x0+dex_cache_resolved_methods_offset+dex_cache_resolved_methods_offset_of(<init>)
0x005ba8e0: f940181e	ldr lr, [x0, #48]		::lr=<init>的artmethod的entry_point_from_quick_compiled_code_字段48
0x005ba8e4: d63f03c0	blr lr					::执行init方法
```

从汇编代码可以看出，invoke-direct指令执行的过程是，直接从dex_cache中通过固定的偏移找到了init方法，也就是说并没有用到invoker对象的class类型，属于静态绑定。

我们在看一下最后一条语句paramClassChildA.printName(this);的机器码，printName一个虚方法
```
0x005baae4: aa1503e2	mov x2, x21				::x2=this,x21中是call方法第一个参数this
0x005baae8: aa1903e1	mov x1, x25				::x1=paramClassChildA,x25中是call方法的第二个参数paramClassChildA
0x005baaec: b9400020	ldr w0, [x1]			::w0=paramClassChildA.shadow$_klass_，0是java对象中shadow$_klass_的偏移量，指向Class对象
suspend point dex PC: 0x003f
GC map objects:  v6 (r21), v7 (r25)			
0x005baaf0: f9418400	ldr x0, [x0, #776]		::x0=printName方法的artmethod=shadow$_klass_+vtable_offset+vtable_offset_of(printName)
0x005baaf4: f940181e	ldr lr, [x0, #48]		::lr=printName的artmethod的entry_point_from_quick_compiled_code_入口
0x005baaf8: d63f03c0	blr lr					::执行printName方法
```
从汇编代码可以看出，虚方法在执行过程中，也就是invoke-virtual指令编译成的机器码，是根据运行时对象获得其shadow$_klass_然后通过虚表查找的方式获得的artmethod，这与前面的invoke-direct指令不同，接下来我们看一下同为虚方法调用的Object localObject = invoker.a(paramClassChildA);这条语句的汇编代码
```
0x005ba904: aa1603e1	mov x1, x22                 ::x1=invoker
0x005ba908: aa1303e0	mov x0, x19					::x0=call方法对应的artmethod指针
0x005ba90c: b9400400	ldr w0, [x0, #4]			::w0 = [x0 + 4] = artmethod中dex_cache_resolved_methods_字段,w0=x0+dex_cache_resolved_methods_offset
0x005ba910: aa1903e2	mov x2, x25					::x2=paramClassChildA,x25中是call方法的第二个参数paramClassChildA
0x005ba914: d282fc0e	mov x14, #0x17e0
0x005ba918: f2a0004e	movk x14, #0x2, lsl #16		::x15=a方法在dex_cache_resolved_methods_中的偏移
0x005ba91c: f86e6800	ldr x0, [x0, x14]			::x0=a方法对应的artmethod对象，x0+dex_cache_resolved_methods_offset+dex_cache_resolved_methods_offset_of(a)
0x005ba920: f940181e	ldr lr, [x0, #48]			::lr=Invoker.a的artmethod的entry_point_from_quick_compiled_code_入口
0x005ba924: d63f03c0	blr lr						::执行a方法
```
我们惊奇的发现，调用a方法汇编码居然和调用printName方法的不同，而是和调用init方法的一样，并没有使用invoker对象的Class，属于静态绑定，经过研究后我们发现，art的编译器根据前面new Invoker的语句确定了invoker对象的类型，因此将a方法的调用优化成了静态绑定，为了验证这个结论，我们将call方法更改为下面代码

```
  public void call(ClassChildA paramClassChildA,Invoker invoker) {
    if (this.patch != null) {
      try {
        Field localField = Class.forName("java.lang.Object").getDeclaredField("shadow$_klass_");
        localField.setAccessible(true);
        localField.set(invoker, patch);//patch is CLassChildB.class
        Object localObject = invoker.a(paramClassChildA);
        Toast.makeText(this,(String)localObject,Toast.LENGTH_SHORT).show();
        return;
      } catch (Exception localException) {

      }
    }
    paramClassChildA.printName(this);
  }
```
然后我们再看一下Object localObject = invoker.a(paramClassChildA);这条语句的汇编代码
```
0x005baa14: aa1803e2	mov x2, x24
0x005baa18: aa1903e1	mov x1, x25
0x005baa1c: b9400020	ldr w0, [x1]
suspend point dex PC: 0x0019
GC map objects:  v0 (r22), v2 (r20), v3 (r26), v4 (r21), v5 (r24), v6 (r25)
0x005baa20: f9417c00	ldr x0, [x0, #760]
0x005baa24: f940181e	ldr lr, [x0, #48]
0x005baa28: d63f03c0	blr lr
```
果然如我们所料，变成了动态绑定，其中x25是invoker的对象指针，x24是paramClassChildA参数。

对于解释执行情况下的绑定机制，由于已经有一些文章做过介绍，本文不再赘述。

**2.Android N上的方法内联**

我们知道在marshmellow之前，android的编译器默认是Quick类型，nougat之后是Optimizing类型，Quick类型编译器的内联条件主要需满足以下几条：

> 1.App不是Debug版本的；
> 
> 2.被调用方法的实现满足下列条件之一：
> 
> 2.1\. 空方法；
> 
> 2.2\. 仅返回方法参数；
> 
> 2.3\. 仅返回一个方法内声明的常量或null；
> 
> 2.4\. 从被调用方法所在类的非静态成员获取并返回获取的值；（注意，static final成员会被优化成常量，此时要参照2.3）
> 
> 2.5\. 仅设置了被调用方法所在类的非静态成员的值；
> 
> 2.6\. 仅设置了被调用方法所在类的非静态成员的值，并返回一个方法内声明的常量或null

Optimizing类型编译器的内联条件主要需满足以下几条：

> 1、App不是Debug版本的；
> 
> 2、被调用的方法所在的类与调用者所在的类位于同一个Dex；（注意，符合Class N命名规则的多个Dex要看成同一个Dex）
> 
> 3、被调用的方法的字节码条数不超过dex2oat通过--inline-max-code-units指定的值，6.x默认为100，7.x默认为32；
> 
> 4、被调用的方法不含try块；
> 
> 5、被调用的方法不含非法字节码；
> 
> 6、对于7.x版本，被调用方法还不能包含对接口方法的调用。（invoke-interface指令）

从上面的条件对比可以看出，google在nougat之后采用了更加激进的内联策略，方法内联可以跨多级方法调用进行，最多可以达到5层。

接下来我们还是继续看call方法，我们首先看一下Invoker以及a的方法体
```
public class Invoker extends ClassParrent{
  private String childname = "childA";
  public  Object a(Object object) {
    return "";
  }
}
```
a方法很简单，仅仅返回一个空的字符串，在call方法中，我们将invoker的shadow$_klass_更改为了ClassChildB（patch是通过dex加载的ClassChildB），我么看一下ClassChildB及ClassChildB中的a方法（涉及到vtable的知识，ClassChildB中的a可以叫任何名字，只要保证和Invoker中的a的vtable索引相同即可），
```
public class ClassChildB extends ClassParrent {
  private String childname = "childB";

  public  Object a(ClassChildB object) {
    return "classb:"+getClassName()+"::::"+getChildname();
  }

  private String getChildname() {
    String str = "bbbbbbbb";

    for(int i = 0; i < 20; ++i) {
      str = str + "s";
    }

    childname = str;
    return childname;
  }

  public String getClassName() {
    return "bbb0b" + super.getName();
  }

  public void printName(Context paramContext) {
    Log.d(ClassChildB.class.getSimpleName(),getClassName());
    Toast.makeText(paramContext, this.getClassName(), Toast.LENGTH_SHORT).show();
  }

  public void setChildname(String paramString) {
    this.childname = paramString;
  }
}
```
我们通过点击call按钮来调用call方法看一下结果
![8959639-a25da8760f839465.png](http://upload-images.jianshu.io/upload_images/8959639-cc41d45e1f3d1582.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第一次patch为空，调用了最后一句paramClassChildA.printName(this);ClassChildA的printName方法与ClassChildB几乎相同，只不过打印的是aa开头
```
public String getClassName() {
    return "aa0a" + super.getName();
  }

  public void printName(Context paramContext) {
    Log.d(ClassChildA.class.getSimpleName(),getClassName());
    Toast.makeText(paramContext, this.getClassName(), Toast.LENGTH_SHORT).show();
  }
```
接下来我们点击load按钮，将patch对象设置为ClassChildB.class，然后我们再次点击call按钮
![8959639-be2b290899b275c0.png](http://upload-images.jianshu.io/upload_images/8959639-99e9ccd6d646dbba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们发现toast的内容，也就是Invoker的a方法的返回值，并不是“”这个空字符串，其实这是因为我们通过shadow$_klass_将Invoker的对象类型设置为了ClassChildB，由于Invoker的a方法是一个虚方法，属于动态绑定，因此他需要使用invoker对象的class类型确定方法，所以导致虽然调用的是Invoker的a方法，但实际执行的是ClassChildB的a方法，并且运行时仅仅对Invoker的a方法进行了参数校验，并没有ClassChildB的a方法进行入参校验。

接下来我们点击数次call按钮，大约20多次吧，输出结果变成了下面这样
![8959639-41974f395f5fd141.png](http://upload-images.jianshu.io/upload_images/8959639-a79a85c0413b21ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们惊奇的发现，那个被取代的Invoker中的a方法，又神奇的执行了，回到刚才我们介绍的nougat之后的Optimizing类型的编译器的内联策略，我们怀疑a方法被 jit 内联了，通过前面的内容我们知道，含有try块的方法是不会被内联的，于是我们将Invoker的a方法更改为下面这样
```
public class Invoker extends ClassParrent{
  private String childname = "childA";
  public  Object a(Object object) {
    try{
      
    }catch (Exception e){

    }
    return "";
  }
}
```
不出所料，内联现象消失了。

通过上面的例子我们发现，在n之后的jit编译器上，jit会收集方法调用的热度，对满足内联条件的方法进行内联，这些内联策略会影响某些热修复技术，tinker的技术团队做过相应的介绍：[ART 下的方法内联策略及其对 Android 热修复方案的影响分析](https://www.qcloud.com/community/article/590725)。

通过对不同版本art虚拟机以及jvm的实现进行研究，我们发现不同的java虚拟机以及不同版本的art实现差别很大，很多android上的基于native的热修复技术都利用了虚拟机的实现机制，然而art还未成为一种稳定实现的虚拟机，google对于art的不同版本经常会做比较大的改动，并且art与jvm不同的是，为了提高art的执行速度，google将dex看做一种不会更改，并且被优化后不可移植的代码，基于这个思想做了很多的编译优化，这给热修复技术带来了很大的难度，因此epic项目的作者给其最新的文章起名叫《我为Dexposed续一秒——论ART上运行时 Method AOP实现》，“续一秒”这个词我认为非常形象的描述了当前基于虚拟机实现机制的热修复或者hook技术特点，这种技术不会受到公开规范的保证，一旦art实现变化，很可能变得出现问题甚至不可用，因此instant run的团队采用了完全符合java规范的实现机制。

最后感谢RednaxelaFx大牛的博客以及知乎问答，让我学到了很多VM的基础知识