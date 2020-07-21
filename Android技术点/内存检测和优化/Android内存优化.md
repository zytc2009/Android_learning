[TOC]
### Android内存优化

今天这篇文章希望能够带大家一起来掀开内存优化的神秘面纱，让我们对内存整体有一个较清晰的认识。

#### 查看内存：

1.查看指定内存使用情况

使用命令：adb shell，dumpsys meminfo 应用包名

或者：adb shell showmap -a PID号 (adb shell showmap -a 2786)

2.查看cpu使用情况：         

​    输入命令：top -m 10 -s cpu（-m显示最大数量，-s 按指定行排序）

3.代码查看

```java
ActivityManager activityManager = (ActivityManager) getSystemService(ACTIVITY_SERVICE);
        //最大分配内存
        int memory = activityManager.getMemoryClass();
        //最大分配内存获取方法2
       float maxMemory = (float) (Runtime.getRuntime().maxMemory() * 1.0/ (1024 * 1024));
        //当前分配的总内存
        float totalMemory = (float) (Runtime.getRuntime().totalMemory() * 1.0/ (1024 * 1024));
        //剩余内存
        float freeMemory = (float) (Runtime.getRuntime().freeMemory() * 1.0/ (1024 * 1024));
```



#### 6G内存一定比4G内存更好么？

这个问题可能大部分人一时想不好应该怎么回答，回答这个问题之前，我们先了解下手机内存，手机内存就像是PC的内存一样，但是考虑到功耗和体积，手机使用的是LPDDR RAM，全称是低功耗双倍速率内存。

以 LPDDR4 为例，带宽 = 时钟频率 × 内存总线位数 ÷ 8，即 1600 × 64 ÷ 8 = 12.8GB/s，因为是 DDR 内存是双倍速率，所以最后的带宽是 12.8 × 2 = 25.6GB/s。

![img](https://static001.geekbang.org/resource/image/2f/44/2f26e93ac941f30bb4037648640aca44.png)

可以看出DDR4的性能是DDR3的两倍，而DDR4X的电压比DDR4更低，所以也比DDR4更加省电（20%~40%）。

那接下来我们再看看刚才的问题，手机内存**是不是越大越好**呢？

如果一个手机使用的是 4GB 的 LPDDR4X 内存，另外一个使用的是 6GB 的 LPDDR3 内存，那么无疑选择 4GB 的运行内存手机要更加实用一些。

其实内存不是一个孤立的概念，它和操作系统、应用生态这些因素都有关。同样都是2GB的话，使用Android 9.0系统会比Android 5.0系统更加流畅，使用更加封闭的IOS系统也会比更加“狂野”的Android系统更好。比如苹果的iphone XR和iphone XS使用的都是LPDDR4X，不过他们的内存只有3GB和4GB。




#### 为什么要对内存进行优化

- 1，引发卡顿。
- 2，导致GC频繁
- 3，内存泄漏
- 4，OOM

**内存抖动的定位**：找循环或者频繁调用的地方

#### 如何对内存进行优化

##### 设备分级

大家在开发过程中肯定遇到过这个问题，就是同样的APP在4G内存手机上运行的比较流畅，但是在1GB内存的设备上就有些卡顿，而且在系统空闲和繁忙的时候也不一样。

**内存优化需要根据设备来考虑**，不要一杆子打死，也并不是“内存分配越少越好”，我们可以让高性能的设备分配更多的内存以保证流畅体验，根据设备使用不同的分配和回收策略。

这里就需要我们设计一套良好的架构系统了，这里我们可以考虑以下几方面：

- **设备分级：**使用类似 device-year-class 的策略对设备分级，低端机用户关闭复杂的动画，或者是某些功能；使用 565 格式的图片，使用更小的缓存内存等。在现实环境下，不是每个用户的设备都跟我们的测试机一样高端，在开发过程我们要学会思考功能要不要对低端机开启、在系统资源吃紧的时候能不能做降级。

  举个例子：

  device-year-class 根据手机的内存、CPU 核心数和频率等信息决定设备属于哪一个年份。

  对于10年以前的设备不做任何动画；对于10~13年的设备可以做些简单动画，对于13年以上的可以做完整体验的效果。

  ```
  if (year >= 2013) {
      // Do advanced animation
  } else if (year >= 2010) {
      // Do simple animation
  } else {
      // Phone too slow, don't do any animations
  }
  ```

- **缓存管理：**

  我们要建立统一的缓存管理机制，可以适当地使用内存；当“系统有难”时，也要义不容辞地归还。我们可以使用 OnTrimMemory 回调，根据不同的状态决定释放多少内存。对于大项目来说，可能存在几十上百个模块，统一缓存管理可以更好地监控每个模块的缓存大小。

- **进程模型：**

  一个空的进程也会占用 10MB 的内存，而有些应用启动就有十几个进程，甚至有些应用已经从双进程保活升级到四进程保活，所以减少应用启动的进程数、减少常驻进程、有节操的保活，对低端机内存优化非常重要。

- **安装包大小：**

  安装包中的代码、资源、图片以及 so 库的体积，跟它们占用的内存有很大的关系。一个 80MB 的应用很难在 512MB 内存的手机上流畅运行。这种情况我们需要考虑针对低端机用户推出 4MB 的轻量版本，例如 Facebook Lite、今日头条极速版都是这个思路。

  安装包中的代码、图片、资源以及 so 库的大小跟内存究竟有哪些关系？你可以参考下面的这个表格:

  ![img](https://static001.geekbang.org/resource/image/0b/a9/0bbcbc6862d0d5f86b8e42d25231b5a9.png)

##### Bitmap优化

为什么要把它单独拎出来说呢，谈到内存优化，bitmap是避不开的一块，也是占用资源最多的一块，bitmap的优化可以说是内存优化的 “永恒主题”。

即使把**所有的 Bitmap 都放到 Native 内存**，并不代表图片内存问题就完全解决了，这样做只是提升了系统内存利用率，减少了 GC 带来的一些问题而已。那我们回过头来看看，到底该如何优化图片内存呢？我给你介绍两种方法。

**方法一，统一图片库：**

图片内存优化的前提是**收拢图片的调用**，这样我们可以做整体的控制策略。例如低端机使用 565 格式、更加严格的缩放算法，可以使用 Glide、Fresco 或者采取自研都可以。而且需要进一步将所有 Bitmap.createBitmap、BitmapFactory 相关的接口也一并收拢。

**方法二，统一监控：**

在统一图片库后就非常容易监控 Bitmap 的使用情况了，这里主要有三点需要注意。

- **大图片监控**。我们需要注意某张图片内存占用是否过大，例如长宽远远大于 View 甚至是屏幕的长宽。在开发过程中，如果检测到不合规的图片使用，应该立即弹出对话框提示图片所在的 Activity 和堆栈，让开发同学更快发现并解决问题。在灰度和线上环境下可以将异常信息上报到后台，我们可以计算有多少比例的图片会超过屏幕的大小，也就是图片的**“超宽率”**。
- **重复图片监控**。重复图片指的是 Bitmap 的像素数据完全一致，但是有多个不同的对象存在。这个监控不需要太多的样本量，一般只在内部使用，可以参考HAHA。
- **图片总内存**。通过收拢图片使用，我们还可以统计应用所有图片占用的内存，这样在线上就可以按不同的系统、屏幕分辨率等维度去分析图片内存的占用情况。在 OOM 崩溃的时候，也可以把图片占用的总内存、Top N 图片的内存都写到崩溃日志中，帮助我们排查问题。

讲完设备分级和 Bitmap 优化，我们发现**架构和监控需要两手抓**，一个好的架构可以减少甚至避免我们犯错，而一个好的监控可以帮助我们及时发现问题。

##### 内存泄漏

内存泄漏简单来说就是没有回收不再使用的内存，排查和解决内存泄漏也是内存优化无法避开的工作之一。

内存泄漏主要分两种情况，一种是同一个对象泄漏，还有一种情况更加糟糕，就是每次都会泄漏新的对象，可能会出现几百上千个无用的对象。

很多内存泄漏都是框架设计不合理所导致，各种各样的单例满天飞，MVC 中 Controller 的生命周期远远大于 View。优秀的框架设计可以减少甚至避免程序员犯错，当然这不是一件容易的事情，所以我们还需要对内存泄漏建立持续的监控。

- **Java 内存泄漏**。建立类似 LeakCanary 自动化检测方案，至少做到 Activity 和 Fragment 的泄漏检测。在开发过程，我们希望出现泄漏时可以弹出对话框，让开发者更加容易去发现和解决问题。内存泄漏监控放到线上并不容易，我们可以对生成的 Hprof 内存快照文件做一些优化，裁剪大部分图片对应的 byte 数组减少文件大小。**比如一个 100MB 的文件裁剪后一般只剩下 30MB 左右，使用 7zip 压缩最后小于 10MB，增加了文件上传的成功率**。
- **OOM 监控**。美团有一个 Android 内存泄露自动化链路分析组件[Probe](https://static001.geekbang.org/con/19/pdf/593bc30c21689.pdf)，它在发生 OOM 的时候生成 Hprof 内存快照，然后通过单独进程对这个文件做进一步的分析。不过在线上使用这个工具风险还是比较大，在崩溃的时候生成内存快照**有可能会导致二次崩溃**，而且部分手机生成 Hprof 快照可能会耗时几分钟，这对用户造成的体验影响会比较大。另外，部分 OOM 是因为虚拟内存不足导致，这块需要具体问题具体分析。
- **Native 内存泄漏监控**。上一期我讲到 Malloc 调试（Malloc Debug）和 Malloc 钩子（Malloc Hook）似乎还不是那么稳定。在 WeMobileDev 最近的一篇文章《[微信 Android 终端内存优化实践](https://mp.weixin.qq.com/s/KtGfi5th-4YHOZsEmTOsjg)》中，微信也做了一些其他方案上面的尝试。

开发过程中内存泄漏排查可以使用 **Androd Profiler 和 MAT** 工具配合使用，而日常监控关键是成体系化，做到及时发现问题。

坦白地说，除了 Java 泄漏检测方案，目前 OOM 监控和 Native 内存泄漏监控都只能做到实验室自动化测试的水平。微信的 Native 监控方案也遇到一些兼容性的问题，如果想达到灰度和线上部署，需要考虑的细节会非常多。Native 内存泄漏检测在 iOS 会简单一些，不过 Google 也在一直优化 Native 内存泄漏检测的性能和易用性，相信在未来的 Android 版本将会有很大改善。

#### 内存监控手段

前面我也提了内存泄漏的监控存在一些性能的问题，一般只会对内部人员和极少部分的用户开启。在线上我们需要通过其他更有效的方式去监控内存相关的问题。

##### 1、采集方式

用户在前台的时候，可以每 5 分钟采集一次 PSS、Java 堆、图片总内存。我建议通过采样只统计部分用户，需要注意的是要按照用户抽样，而不是按次抽样。简单来说一个用户如果命中采集，那么在一天内都要持续采集数据。

##### 2、计算指标

通过上面的数据，我们可以计算下面一些内存指标。

**内存异常率**：可以反映内存占用的异常情况，如果出现新的内存使用不当或内存泄漏的场景，这个指标会有所上涨。其中 PSS 的值可以通过 Debug.MemoryInfo 拿到。

```
内存 UV 异常率 = PSS 超过 400MB 的 UV / 采集 UV
```

**触顶率**：可以反映 Java 内存的使用情况，如果超过 85% 最大堆限制，GC 会变得更加频繁，容易造成 OOM 和卡顿。

```
内存 UV 触顶率 = Java 堆占用超过最大堆限制的 85% 的 UV / 采集 UV
```

其中是否触顶可以通过下面的方法计算得到。

```
//APP可用的最大内存
long javaMax = runtime.maxMemory();
//APP已占用的内存
long javaTotal = runtime.totalMemory();
//APP已占用，但未使用的内存
long javaFree = runtime.freeMemory();
long javaUsed = javaTotal - javaFree;
// Java 内存使用超过最大限制的 85%
float proportion = (float) javaUsed / javaMax;
```

一般客户端只上报数据，所有计算都在后台处理，这样可以做到灵活多变。后台还可以计算平均 PSS、平均 Java 内存、**平均图片占用**这些指标，它们可以反映内存的平均情况。通过平均内存和分区间内存占用这些指标，我们可以通过版本对比来监控有没有新增内存相关的问题。

![img](https://static001.geekbang.org/resource/image/65/ec/65e0b02933c1f7fe181b83d69587e7ec.jpg)

因为上报了前台时间，我们还可以按照时间维度看应用内存的变化曲线。比如可以观察一下我们的应用是不是真正做到了“用时分配，及时释放”。如果需要，我们还可以实现按照场景来对比内存的占用。

##### 3、GC监控

在实验室或者内部试用环境，我们也可以通过 Debug.startAllocCounting 来监控 Java 内存分配和 GC 的情况，需要注意的是这个选项对性能有一定的影响，虽然目前还可以使用，但已经被 Android 标记为 deprecated。

通过监控，我们可以拿到内存分配的次数和大小，以及 GC 发起次数等信息。

```
long allocCount = Debug.getGlobalAllocCount();
long allocSize = Debug.getGlobalAllocSize();
long gcCount = Debug.getGlobalGcInvocationCount();
```

上面的这些信息似乎不太容易定位问题，在 Android 6.0 之后系统可以拿到更加精准的 GC 信息。

```
// 运行的GC次数
Debug.getRuntimeStat("art.gc.gc-count");
// GC使用的总耗时，单位是毫秒
Debug.getRuntimeStat("art.gc.gc-time");
// 阻塞式GC的次数
Debug.getRuntimeStat("art.gc.blocking-gc-count");
// 阻塞式GC的总耗时
Debug.getRuntimeStat("art.gc.blocking-gc-time");
```

需要特别注意阻塞式 GC 的次数和耗时，因为它会暂停应用线程，可能导致应用发生卡顿。我们也可以更加细粒度地分应用场景统计，例如启动、进入详情页、进入播放页面等关键场景。

##### 4、ARTHook监控

  可以用epic，也能实现类监控功能，主要是线下监控    

```
以ImageView的监控为例，同理可以写Thread监控
class ThreadMethodHook extends XC_MethodHook{
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        super.afterHookedMethod(param);
        ImageView imageView = (ImageView) param.thisObject;
        checkBitmap(imageView);
    }
}
//Hook构造方法
DexposedBridge.hookAllConstructors(ImageView.class, new XC_MethodHook() {
    @Override
    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
        super.afterHookedMethod(param);
        //hook setImageBitmap方法，因为setImageBitmap有参数，所以指定参数类型
        DexposedBridge.findAndHookMethod(ImageView.class, "setImageBitmap",Bitmap.class, new ThreadMethodHook());
    }
});
```



**线上监控方案**：

1.内存Dump文件，回传，分析：文件大，需裁剪；上传失败率高，分析困难；配合一定策略，有一点效果

2.**leakCanary**：

   带到线上；预设泄漏怀疑点；发现泄漏回传；缺点是分析耗时，也容易OOM

   **怎么使用**：

​    监控生命周期，onDestroy添加RefWatcher检测

​    二次确认断定发生内存泄漏

​     分析泄漏，找到引用链

​     监控组件+分析组件

​     **定制方向**：

​      预设怀疑点-》自动找怀疑点

​      分析泄漏链路慢 --》分析Retain size打的对象

​      分析OOM -- 》对象裁剪，不全部加载到内存   

​     **监控方案**:

​      待机内存、重点模块内存、OOM率

​      整体及重点模块gc次数，GC时间

​      增强的LeakCanary自动化内存写泄漏分析

**优化大方向**：内存泄漏，内存抖动，Bitmap

**优化细节**：

LargeHeap属性，onTrimMemory，使用系统优化后的集合

谨慎使用SharePreference，外部库

业务架构设计合理

**总结**：

分析现状，确认问题

针对性优化：Memery Profiler，MAT 分析，定位

效率提升：ARTHook，非侵入，比自定义好

技术优化要结合业务：多个图片库会产生的问题

系统化完善解决方案



#### 参考文档

极客时间-Android开发高手课。

玩转内存分析和优化