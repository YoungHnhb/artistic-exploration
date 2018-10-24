# 第十章 Android的消息机制
- Handler是Android消息机制的上层接口
- Android的消息机制 == Handler的运行机制
- MessageQueue
- Looper
- ThreadLocal

## Android 的消息机制概述
- Android规定访问UI只能在主线程中进行
> ViewRootImpl 的checkThread对UI操作做了验证
```
// mThread在ViewRootImpl构造方法内赋值
mThread = Thread.currentThread(); 

void checkThread() {
    if (mThread != Thread.currentThread()) {
        throw new CalledFromWrongThreadException(
                "Only the original thread that created a view hierarchy can touch its views.");
    }
}
```
-Handler主要为了解决在子线程无法访问UI的矛盾
Q:为什么不允许子线程访问UI
A:
1. UI空间线程不安全
2. 锁机制会让UI访问的机制变复杂
3. 锁机制会降低UI访问的效率，因为锁会阻塞某些线程的执行
- Handler创建时会采用**当前线程**的Looper来构建内部消息循环系统

## Android 的消息机制分析
### ThreadLocal的工作原理
- ThreadLocal时线程内部的数据存储类
Looper、ActivityThread以及AMS都用到了ThreadLocal
较好的使用场景：作用域时线程且不同线程具有不同的数据副本
```
// private ThreadLocal<Boolean> mBooleanThreadLocal = new ThreadLocal<>();
mBooleanThreadLocal.set(true);
Log.e("TTT", "[Thread#main]mBooleanThreadLocal=" + mBooleanThreadLocal.get());

new Thread("Thread#1") {
    @Override
    public void run() {
        mBooleanThreadLocal.set(false);
        Log.e("TTT", "[Thread#1]mBooleanThreadLocal=" + mBooleanThreadLocal.get());
    }
}.start();

new Thread("Thread#2") {
    @Override
    public void run() {
        // mBooleanThreadLocal.set(null);
        Log.e("TTT", "[Thread#2]mBooleanThreadLocal=" + mBooleanThreadLocal.get());
    }
}.start();
```
> 运行结果：
> E/TTT: [Thread#main]mBooleanThreadLocal=true
> E/TTT: [Thread#1]mBooleanThreadLocal=false
> E/TTT: [Thread#2]mBooleanThreadLocal=null

ThreadLocal的set、get方法
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

### 消息队列的工作原理
MessageQueue
插入数据：enqueueMessage
读取数据：next // 读取伴随着删除操作
- 内部通过**单链表**的数据结构

### Looper的工作原理
- Looper不停从MessageQueue中查看是否有新消息，如果有则立即处理，否则一直阻塞
- Handler工作需要Looper，没有则会报错
- Looper.prepare()
```
new Thread("Thread#2") {
    @Override
    public void run() {
        Looper.prepare();
        Handler handler = new Handler();
        Looper.loop();
    }
}.start();
```
- prepareMainLooper()
- getMainLooper()
- 退出Looper：
1. 直接退出quit()
2. 安全退出quitSafely() // 设定一个退出标记，消息队列已有消息处理完后才退出
- 退出后Handler的send返回false
- msg.target就是Handler对象
```
public void dispatchMessage(Message msg) {
	if (msg.callback != null) {
	    handleCallback(msg);
	} else {
	    if (mCallback != null) {
	        if (mCallback.handleMessage(msg)) {
	            return;
	        }
	    }
	    handleMessage(msg);
	}
}

public interface Callback {
    /**
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public boolean handleMessage(Message msg);
}
```

### Handler的工作原理
> public final boolean sendMessage(Message msg) ↓
> public final boolean sendMessageDelayed(Message msg, long delayMillis) ↓
> public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
>     MessageQueue queue = mQueue;
>     if (queue == null) {
>         RuntimeException e = new RuntimeException(
>                 this + " sendMessageAtTime() called with no mQueue");
>         Log.w("Looper", e.getMessage(), e);
>         return false;
>     }
>     return enqueueMessage(queue, msg, uptimeMillis);
> }
> 
> private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
>     msg.target = this;
> 	  if (mAsynchronous) {
> 	      msg.setAsynchronous(true);
> 	  }
> 	  return queue.enqueueMessage(msg, uptimeMillis);
> }
> 
```
public Handler(Looper looper) {
	this(looper, null, false);	
}

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
```
## 主线程的消息循环
Android 的主线程就是ActivityThread，主线程入口方法为main，
main中调用Looper.prepareMainLooper()创建主线程Looper以及MessageQueue
通过Looper.loop()开启主线程消息循环。
消息循环开始后，ActivityThread需要一个Handler，即ActivityThread.H

ActivityThread通过ApplicationThread和AMS进行进程间通信，
AMS以进程间通信的方式完成ActivityThread的请求后回调ApplicationThread中的Binder方法，
然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中逻辑切换到
ActivityThread中去执行，即切换到主线程中去执行，
这个过程就是主线程的消息循环模型。
