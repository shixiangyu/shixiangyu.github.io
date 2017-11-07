---
title: 'Handler, Thread, Looper, MessageQueue之间的关系'
date: 2017-11-05 16:10:30
categories: android
tags:
   - handler
---

Android 是由事件驱动的，在Android中，主线程（也就是UI线程）是不安全的，当在主线程处理消息过长时，非常容易发生ANR（Application Not Responding）问题；其次，如果我们在子线程中尝试进行UI的操作，程序就可能还会直接崩溃，这就是UI线程被阻塞导致的，归根结底其实是Looper被当前事件（也就是当前正在处理的消息）阻塞住导致的。在android系统中Looper通过Looper.loop() 不断地接收事件、处理事件，每一个点击，触摸包括Activity的生命周期都是运行在 Looper.loop() 的控制之下，如果它停止了，应用也就停止了，所有Looper的生命周期是伴随整个应用的。

<!--more-->

#### 我们先来分析一下Handler

要创建一个Handler对象非常的简单明了，直接进行new一个对象即可mHandler0 = new Handler()，但这里其实还隐藏着一个信息，那就是这个mHadler也就是在此刻和当前线程中的Looper进行了隐匿绑定，看下面代码


	public class MainActivity extends ActionBarActivity {
	
	    private Handler mHandler0;
	
	    private Handler mHandler1;
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	
	        mHandler0 = new Handler();
	        new Thread(new Runnable() {
	            @Override
	            public void run() {
	                mHandler1 = new Handler();
	            }
	        }).start();
	    }

这一小段程序代码主要创建了两个Handler对象，其中，一个在主线程中创建，而另外一个则在子线程中创建，现在运行一下程序，则你会发现，在子线程创建的Handler对象竟然会导致程序直接崩溃，提示的错误竟然是Can't create handler inside thread that has not called Looper.prepare()

于是我们按照logcat中所说，在子线程中加入Looper.prepare(),即代码如下：

```
new Thread(new Runnable(){

    @override
    public void run(){
        Looper.prepare();
        mHandler1 = new Handler()
    }
}).start();
```

再次运行一下程序，发现程序不会再崩溃了，可是，单单只加这句Looper.prepare()是否就能解决问题了。我们探讨问题，就要知其然，才能了解得更多。我们还是先分析一下源码吧，看看为什么在子线程中没有加Looper.prepare()就会出现崩溃，而主线程中为什么不用加这句代码？我们看下Handler()构造函数：
```
public Handler() {
    this(null, false);
}
```
构造函数直接调用this(null, false)，于是接着看其调用的函数，

	 public Handler(Callback callback, boolean async) {
	        if (FIND_POTENTIAL_LEAKS) {
	            final Class<? extends Handler> klass = getClass();
	            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
	                (klass.getModifiers() & Modifier.STATIC) == 0) {
	                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
	                klass.getCanonicalName());
	            }
	        }
	
	        mLooper = Looper.myLooper();
	        if (mLooper == null) {
	            throw new RuntimeException(
	                "Can't create handler inside thread that has not called Looper.prepare()");
	        }
	        mQueue = mLooper.mQueue;
	        mCallback = callback;
	        mAsynchronous = async;
	  }

不难看出，源码中调用了mLooper = Looper.myLooper()方法获取一个Looper对象，若此时Looper对象为null，则会直接抛出一个“Can't create handler inside thread that has not called Looper.prepare()”异常，那什么时候造成mLooper是为空呢？那就接着分析Looper.myLooper()，

    public static Looper myLooper() {
         return sThreadLocal.get();
    }

这个方法在sThreadLocal变量中直接取出Looper对象，若sThreadLocal变量中存在Looper对象，则直接返回，若不存在，则直接返回null，而sThreadLocal变量是什么呢？


    static final ThreadLocat<Looper> sThreadLocal = new ThreadLocal<Looper>();

它是本地线程变量，由此看到，我们的判断是正确的，在Looper.prepare()方法中给sThreadLocal变量设置Looper对象，看看源码
  
	public static void prepare() {
	        prepare(true);
	}

直接调用了带个boolean参数的prepare函数

	private static void prepare(boolean quitAllowed) {
	      if (sThreadLocal.get() != null) {
	            throw new RuntimeException("Only one Looper may be created per thread");
	      }
	        sThreadLocal.set(new Looper(quitAllowed));
    }

这样也就理解了为什么要先调用Looper.prepare()方法，才能创建Handler对象，才不会导致崩溃。所以我们可以看到，一个线程在调用Looper的静态方法prepare()时，这个线程会新建一个Looper对象，并放入到线程的局部变量中，而这个变量是不和其他线程共享的，但是，仔细想想，为什么主线程就不用调用呢？不要急，我们接着分析一下主线程，我们查看一下ActivityThread中的main()方法，代码如下：

    public static void main(String[] args) {
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        Security.addProvider(new AndroidKeyStoreProvider());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

代码中调用了Looper.prepareMainLooper()方法，而这个方法又会继续调用了Looper.prepare()方法。

分析到这里已经真相大白，主线程中google工程师已经自动帮我们创建了一个Looper对象了，因此我们不再需要手动再调用Looper.prepare()再创建，而子线程中，因为没有自动帮我们创建Looper对象，因此需要我们手动添加，调用方法是Looper.prepare()，这样，我们才能在子线程中正确地创建Handler对象。



#### 再来分析Looper
在上面的过程中，我们对Looper也了解了不少，一个线程仅有一个Looper对象，主线程中的Looper对象是由系统为我们自动创建的，并通过Looper.loop()方法开启消息循环，不停的从MessageQueue中取出msg，然后通过msg.target.dispatchMessage(msg)方法交给对应的Handler的handleMessage()方法进行处理，而不难看出，此时msg.target就是Handler对象。概括性来说，Looper负责的是创建一个MessageQueue对象，然后进入到一个无限循环体中不断取出消息，而这些消息都是由一个或者多个Handler进行创建处理。

#### 关于MessageQueue
消息池，负责管理消息队列，实际上Message类有一个next字段，会将Message对象串在一起成为一个消息队列。MessageQueue是在Looper被创建的时候被同时创建的，从下面Looper的构造方法可以看出

    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

仔细想一下上面的Handler的构造方法，其中有这么一行`    mQueue = mLooper.mQueue;` 明白了吧，Looper和Handler拥有的是同一个MessageQueue，也就是说这个queue是由它们两个共同来维护的，其实这也是必须的，Handler通过sendMessage（Message msg）等方法发送消息，然后再通过enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis)方法进行消息入队列操作，而Looper.loop()不停的在从该消息队列中取出消息，然后调用msg.target.dispatchMessage(msg)去处理该消息。


#### 最后，是否清楚这个几者之间的对应关系呢，分析一下：
线程和handler是一对多关系，一个线程里可以有一个或者多个Handler；而线程和Looper则是一对一的关系，一个线程里只会存在一个Looper；
那么，从以上两个对应关系，可知，一个线程中的Handler和Looper是多对一的关系，由于一个Looper中有就一个mQueue,那么Handler和MessageQueue的关系也是多对一了,因为handler里的mQueue都是从Looper中拿到的；

#### 那么，消息发送机制就是这样的：

一个线程里的一个或多个handler向mQueue发送消息，线程里的Looper通过Looper.loop()不停的在从该消息队列中取出消息，然后通过该消息的msg.target去获知发送这个消息的handler是谁，之后调用msg.target.dispatchMessage(msg)去处理该消息。