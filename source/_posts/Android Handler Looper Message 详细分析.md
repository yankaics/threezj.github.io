title: Android Handler Looper Message 详细分析
date: 2016-01-23 15:13:06
tags: ["Android"]
---
## Android异步消息机制架构

Android异步消息处理架构，其实没那么复杂。简单来说就是`looper`对象拥有`message queue`，并且负责从`message queue`中取出消息给`handler`来处理。同时`handler`又负责发送`message`给`looper`,由`looper`把`message`添加到`message queue`尾部。就一个圈儿。下面给出图解，应该不难吧？

![](http://ww4.sinaimg.cn/mw690/87dacc16gw1f09d4z9fsgj20ij0csmxo.jpg)

所以很明显`handler`和`looper`是来联系在一起的。需要说明的是，多个`message`可以指向同一个`handler`，多个`handler`也可以指向同一个`looper`。

还有一点很重要，普通的线程是没有`looper`的，如果需要`looper`对象，那么必须要先调用`Looper.prepare()`方法，而且一个线程只能有一个`looper`。调用完以后，此线程就成为了所谓的`LooperThread`,若在当前`LooperThread`中创建`Handler`对象，那么此`Handler`会自动关联到当前线程的`looper`对象，也就是拥有`looper`的引用。
<!--more-->

## Looper

简而言之，`Looper`就是一个管理`message queue`的类。我们直接来看下源码好了。

```java
public class Looper {
    ......

    private static final ThreadLocal sThreadLocal = new ThreadLocal();

    final MessageQueue mQueue;//拥有的消息队列

    ......

    /** Initialize the current thread as a looper.
    * This gives you a chance to create handlers that then reference
    * this looper, before actually starting the loop. Be sure to call
    * {@link #loop()} after calling this method, and end it by calling
    * {@link #quit()}.
    */
    //创建新的looper对象，并设置到当前线程中
    public static final void prepare() {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper());
    }
    ·····
    /**
    * Return the Looper object associated with the current thread.  Returns
    * null if the calling thread is not associated with a Looper.
    */
    //获取当前线程的looper对象
    public static final Looper myLooper() {
        return (Looper)sThreadLocal.get();
    }

    private Looper() {
        mQueue = new MessageQueue();
        mRun = true;
        mThread = Thread.currentThread();
    }

    ......
}
```

由源码可知，`looper`拥有`MessageQueue`的引用，并且需要调用`Looper.prepare()`方法来为当前线程创建`looper`对象。Android注释很详细的说明了这个方法。

> This gives you a chance to create handlers that then reference
>   this looper, before actually starting the loop. Be sure to call
>   `loop()` after calling this method, and end it by calling
>   `quit()`

也就是说只用在调用完`Looper.prepare()`之后，在当前的线程创建的`Handler`才能用有当前线程的`looper`。然后调用`loop()`来开启循环，处理`message`.来简单看一下`loop()`的源码

```java
public static void loop() {
        final Looper me = myLooper();//获取looper对象
        if (me == null) {
            //若为空则说明当前线程不是LooperThread，抛出异常
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue; 获取消息队列
        .....
        for (;;) {
            //死循环不断取出消息
            Message msg = queue.next(); // might block 
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            //打印log，说明开始处理message。msg.target就是Handler对象
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            //重点！！！开始处理message，msg.target就是Handler对象
            msg.target.dispatchMessage(msg);

            //打印log，处理message结束
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
            .....
        }
    }
```

很明显，就是一个大的循环，不断从消息队列出取出消息。然后调用一个很关键的方法`msg.target.dispatchMessage(msg)`开始处理消息。`msg.target`就是`message`对应的`handler`.其实我一直在重复文章开头的概念。`looper`对象管理`MessageQueue`，从中取出`message`分配给对应的`handler`来处理。

在分析`msg.target.dispatchMessage(msg)`方法之前，先让大家了解下`message`和`handler`

## Message

`Message` 就是一些需要处理的事件，比如访问网络、下载图片、更新ui界面什么的。`Message`拥有几个比较重要的属性。

*   public int what   标识符，用来识别`message`
*   public int arg1,arg2    可以用来传递一些轻量型数据如int之类的
*   public Object obj    `Message`自带的Object类字段，用来传递对象
*   Handler target   指代此`message`对象对应的`Handler`

如果携带比价复杂性的数据，建议用`Bundle`封装，具体方法这里不讲了。

值得注意的地方是,虽然`Message`的构造方法是公有的，但是不建议使用。最好的方法是使用`Message.obtain()`或者`Handler.obtainMessage()` 能更好的利用循环池中的对象。一般不用手动设置`target`,调用`Handler.obtainMessage()`方法会自动的设置`Message`的`target`为当前的`Handler`。

得到`Message`之后可以调用`sendToTarget()`,发送消息给`Handler`，`Handler`再把消息放到`message queue`的尾部

## Handler
```java
public Handler(Looper looper, Callback callback, boolean async) {
       mLooper = looper;
       mQueue = looper.mQueue;
       mCallback = callback;  //handleMessage的借口
       mAsynchronous = async;
   }
```

上面是`Handler`构造器,由此可知，它拥有`looper`对象，以及`looper`的`message queue`。

要通过`Handler`来处理事件，可以重写`handleMessage(Message msg)`,也可以直接通过`post(Runnable r)`来处理。这两个方法都会在looper循环中被调用。

还记得刚在loop循环中处理信息的`msg.target.dispatchMessage(msg)`方法。现在我们来看下这个方法的源码。
```java
public void dispatchMessage(Message msg) {
    //注意！这里先判断message的callback是否为空，否则就直接处理message的回调函数
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            //正是在这调用我们平常重写handleMessage
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
没错，正是在这个方法中调用了`handleMessage(Message msg)`。值得注意的是，再调用`handleMessage(Message msg)`之前，还判断了message的callback是否为空。这是为什么？

很简单，这就是`post(Runnable r)`方法也能用来处理事件的原因。直接看它的源码。我就不赘述了。相信已经很清晰了。
```java
public final boolean post(Runnable r)
{
    //获取消息并发送给消息队列
    return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    //创建消息，且直接设置callback
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
## 总结

现在一切都弄清楚了。先调用`Looper.prepare()`使当前线程成为`LooperThread`,`Looper.loop()`开启循环之后，然后创建`message`,再由`Handler`发送给消息队列，最后再交由`Handler`处理。下面是官方给出的LooperThread最标准的用法。
```java
class LooperThread extends Thread {
    public Handler mHandler;
  
    public void run() {
        Looper.prepare();
  
        mHandler = new Handler() {
            public void handleMessage(Message msg) {
                // process incoming messages here
            }
        };
  
        Looper.loop();
    }
```
最后的最后，再提一下，平时在说的主线程也就是UIThread，也是一个LooperThread。这也是为什么我们能在子线程中发送消息，然后在主线程中更新ui。因为`Handler`是在主线程创建的，所以`Handler`关联的是主线程的`looper`，最后给一个更新ui的实例。

```java
private static Handler handler=new Handler();

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.message_activity);  
    new Thread(new Runnable() {                    
        @Override
        public void run() {
            // tvTest.setText("hello word");
            // 以上操作会报错，无法再子线程中访问UI组件，UI组件的属性必须在UI线程中访问
            // 使用post方式设置message的回调
            handler.post(new Runnable() {                    
                @Override
                public void run() {
                    tvTest.setText("hello world");                        
                }
            });                                
        }
    }).start();
}
```

## 声明

转载写明出处，谢谢。

这是我的微博 有任何问题都可以联系我。[Three_ZJ](http://weibo.com/zjthree)
