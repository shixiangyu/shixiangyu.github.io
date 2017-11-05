---
title: 浅析android的OOM问题
date: 2017-11-03 16:17:36
categories: android
tags:
   - OOM
---

### 1、 Android的 java程序为什么容易出现OOM
这个问题要涉及到android系统的设置方面了，主要因为Android系统对 dalvik
 的 vm heapsize 作了硬性限制，当java进程申请的java空间超过阈值时，就会抛出OOM异常（这个阈值可以是48M、24M、16M等，视机型而定），可以通过命令`adb shell getprop | grep dalvik.vm.heapgrowthlimit`查看此值。也可在代码中通过api来获取该数值
 
<!--more-->


	ActivityManager activityManager =(ActivityManager)context.getSystemService(Context.ACTIVITY_SERVICE);
	activityManager.getMemoryClass();


以上方法会返回以M为单位的数字，不同的系统平台或设备上的值都不太一样。
而查看一个进程的内存使用情况，可以通过`dumpsys meminfo`命令可以查看，如`adb shell dumpsys meminfo com.test.wonder` ,后面为应用的包名。

也就是说，程序发生OMM并不表示RAM不足，而是因为程序申请的 java heap 对象超过了 dalvik vm heapgrowthlimit 。所以即使在RAM充足的情况下，也可能发生OOM问题。

### 2、导致OOM问题的原因有哪些？
[Android内存泄漏的八种可能（上）](http://www.jianshu.com/p/ac00e370f83d)
[Android防止内存泄漏的八种方法（下）](http://www.jianshu.com/p/c5ac51d804fa)


ps:这里介绍几个比较常见的：

#### 1 、应用中需要加载大对象，例如Bitmap

在android 2.3和以前的版本，bitmap对象的像素数据都是分配在native heap中的，所以我们在调试过程中这部分内存是在java heap中看不到的，不过在android 3.0之后，bitmap对象就直接分配在java heap上了，这样便于调试和管理。因此在3.0之后我们可以复用bitmap的内存，而不必回收它，不过新的bitmap对象要大小和原来的一样，到了android 4.4之后，就只要高宽不超过原来的就行了。


Android中一张图片（BitMap）占用的内存的计算: Android中一张图片（BitMap）占用的内存主要和以下几个因数有关：*图片长度*，*图片宽度*，*单位像素占用的字节数*。
一张图片（BitMap）占用的内存 = 图片长度 × 图片宽度 × 单位像素占用的字节数，其单位像素占用的字节数由其参数[BitmapFactory.Options](http://developer.android.com/reference/android/graphics/BitmapFactory.Options.html#inScaled)的inPreferredConfig变量决，inPreferredConfig为[Bitmap.Config](http://developer.android.com/reference/android/graphics/Bitmap.Config.html)类型，[Bitmap.Config](http://developer.android.com/reference/android/graphics/Bitmap.Config.html)类是个枚举类型，有四种取值方式，Android手机上一个BitMap的默认格式为`ARGB_8888`格式，也是主要使用格式,它的单位像素占用四个字节。

比如，一张在pc机上用的1024*768图片，如果直接用在手机屏幕这种小屏幕上，不仅没有提高显示质量，还容易使内存吃紧。假设照片是用ARGB_8888格式，那么一张1024×768的图片需要占用3M的内存， 4-5张就OOM了。bitmap分辨率越高，所占用的内存就越大，这个是以2为指数级增长的。

#### 2 、不合理使用Context 导致的Memory leak

android 中很多地方都要用到context,连基本的Activty 和 Service都是从Context派生出来的，我们利用Context主要用来加载资源或者初始化组件，在Activity中有些地方需要用到Context的时候，我们经常会把context给传递过去了，将context传递出去就有可能延长了context的生命周期，最终导致了内存泄漏。例如 我们将activty context对象传递给一个后台线程去执行某些操作，如果在这个过程中因为屏幕旋转而导致activity重建，那么原先的activity对象不会被回收，因为它还被后台线程引用着，如果这个activity消耗了比较多的内存，那么新建activity或者后续操作可能因为旧的activity没有被回收而导致内存泄漏。所以，遇到需要用到context的时候，我们要合理选择不同的context，对于android应用来说还有一个单例的Application Context对象，该对象生命周期和应用的生命周期是绑定的。选择context应该考虑到它的生命周期，如果使用该context的组件的生命周期超过该context对象，那么我们就要考虑是否可以用application context。如果真的需要用到该context对象，可以考虑用弱引用来WeakReference来避免内存泄漏。


#### 3、 非静态内部类导致的Memory leak

 非静态的内部类会持有外部类的一个引用，所以和前面context说到的一样，如果该内部类生命周期超过外部类的生命周期，就可能引起内存泄露了，如AsyncTask ，Thread ，TimerTask和Handler，[这里有篇关于Handler使用不当而导致oom的文章，讲的很清楚](http://www.jianshu.com/p/63aead89f3b9)。
前一段时间专门对项目中使用到的Handler进行了盘查改善来修改和预防oom问题，我们是在代码中是通过软引用来实现的：

	public interface SafeHandlerListener {
	    void handMessage(Message msg);
	}


再看这个SafeHandlerListener:


	public class SafeHandler extends Handler {
	
	    private final WeakReference<SafeHandlerListener> ref;
	
	    public SafeHandler(SafeHandlerListener listener) {
	        ref = new WeakReference<>(listener);
	    }
	
	    @Override
	    public void handleMessage(Message msg) {
	        SafeHandlerListener caller = ref.get();
	        if (caller != null){
	            caller.handMessage(msg);
	        }
	    }
	}

由于在Activity中我们经常会用到内部类，所以要小心管理其生命周期。 如果明确生命周期较外部类长的话，那么应该使用静态内部类或者考虑用弱引用WeakReference来避免内存泄漏。