# 第十一章 Android 的线程和线程池  

- 线程在Android中从用途上分为*主线程*和*子线程*

## 主线程和子线程
- Java中默认情况一个进程只有一个线程即主线程。  
子线程也叫工作线程，除了主线程以外的线程都是子线程
- Android中主线程的作用运行四大组件以及处理他们和用户的交互
- Android 3.0之后系统要求网络访问必须在子线程中进行，  
否则会抛出*NetworkOnMainThreadException*的异常，  
这样做是为了避免主线程被耗时操作阻塞出现ANR现象

## Android 中的线程形态
### AsyncTask
- AsyncTask是轻量级的异步任务类  
- AsyncTask封装了Thread和Handler  
- AsyncTask并不适合进行特别耗时的后台任务，耗时任务建议使用线程池  
- AsyncTask是抽象的泛型类  
```
public abstract class AsyncTask<Params, Progress, Result> {  
  Params：参数类型  
  Progress：后台任务执行进度的类型  
  Result：后台任务返回结果类型 

@MainThread
protected void onPreExecute() {
	// 主线程执行，在异步任务执行之前会被调用，一般做一些准备工作
}

@WorkerThread
protected abstract Result doInBackground(Params... params);
	// 在线程池中执行，执行异步任务
	// params：异步任务输入的参数
	// 通过publicProgress更新任务进度，会调用onProgressUpdate方法
	// 需要返回执行结果给onPostExecute方法

@MainThread
protected void onProgressUpdate(Progress... values) {
	// 主线程执行，执行进度改变时被调用
}

@MainThread
protected void onPostExecute(Result result) {
	// 主线程执行，异步任务执行之后，result是后台任务返回值
}

@MainThread
protected void onCancelled(Result result) {
    onCancelled();
    // 异步任务被取消时被调用
}
```
- AsyncTask使用条件限制：  
1. AsyncTask的类必须在主线程中加载（4.1及以上系统自动完成）  
2. AsyncTask的对象必须在主线程中创建  
3. execute方法必须在UI线程中调用  
4. 不要在程序中直接调用onPreExecute、onPostExecute、doInBackGround、onProgressUpdate方法  
5. 一个AsyncTask对象只能调用一次execute方法，否则报错（the task is already running.）  
6. 1.6之前是串行执行，1.6时是并行任务，3.0之后为了避免并发错误又采用串行执行，但可通过executeOnExecutor并行执行  

### AsyncTask的工作原理
- AsyncTask中有两个线程池(SerialExecutor和THREAD_POOL_EXECUTOR)和一个Handler(InternalHandler)  
SerialExecutor用于任务的排队，实现串行执行  
THREAD_POOL_EXECUTOR用于真正的执行任务  
InternalHandler用于将执行环境从线程池切换到主线程

### HandlerThread
- HandlerThread继承了Thread，是可以使用Handler的Thread
```
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}
```

### IntentService
- IntentService是特殊的Service，抽象类，任务执行后会自动停止  
比单纯的线程优先级高很多  
IntentService封装了HandlerThread和Handler
```
@Override
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.

    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();

    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
}

private final class ServiceHandler extends Handler {
	public ServiceHandler(Looper looper) {
	    super(looper);
	}

	@Override
	public void handleMessage(Message msg) {
	    onHandleIntent((Intent)msg.obj);
	    stopSelf(msg.arg1);
	}
}

@WorkerThread
protected abstract void onHandleIntent(@Nullable Intent intent);

@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}

/**
 * You should not override this method for your IntentService. Instead,
 * override {@link #onHandleIntent}, which the system calls when the IntentService
 * receives a start request.
 * @see android.app.Service#onStartCommand
 */
@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}

@Override
public void onDestroy() {
    mServiceLooper.quit();
}
```

## Android 中的线程池
- 线程池的有点：  
1. 重用线程池中的线程，避免因线程的创建和销毁带来性能开销  
2. 能有效控制线程池的最大并发数，避免大量的线程之间因互相抢占系统资源而导致阻塞现象  
3. 能够对线程进行简单的管理，并提供定时执行以及制定间隔循环执行等功能  

Executor是一个接口,真正的实现为ThreadPoolExecutor
```
public interface Executor {
    void execute(Runnable command);
}
```

### ThreadPoolExecuter
```
public ThreadPoolExecutor(int corePoolSize,
	                      int maximumPoolSize,
	                      long keepAliveTime,
	                      TimeUnit unit,
	                      BlockingQueue<Runnable> workQueue,
	                      ThreadFactory threadFactory,
	                      RejectedExecutionHandler handler) {
- corePoolSize:线程池核心线程数，默认一直核心线程一直存活，当allowCoreThreadTimeOut设置为TRUE，  
  核心线程等待新任务时会有超时策略，由keepAliveTime决定  
- maximumPoolSize:线程池所能容纳最大线程数，活动线程达到这个数值后，后续新任务会被阻塞  
- keepAliveTime:非核心线程闲置超时时长  
- unit:keepAliveTime参数的时间单位，常用的有TimeUnit.MILLISECONDS、TimeUnit.SECONDS、TimeUnit.MINUTES  
- workQueue:线程池中的任务队列，通过execute提交的Runnable对象会存储在这里  
- threadFactory:线程工厂，为线程池提供创建新线程功能  
- handler:抛出RejectedExecutionException异常，RejectedExecutionHandler提供了几个可选值：  
  CallerRunsPolicy、AbortPolicy、DiscardPolicy、DisCardOldestPolicy  
```
- ThreadPoolExecutor执行任务时大致遵循如下规则：  
1. 如果线程池中的线程数量未达到核心线程数量，那么会直接启动一个核心线程来执行任务。  
2. 如果线程池中的线程数量已达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行。  
3. 如果在步骤2中无法将任务插入到任务队列中，往往因为任务队列已满，这时候如果线程数量未达到线程规定的最大值  
   那么会立即启动一个非核心线程来执行任务。  
4. 如果步骤3中线程数量已经达到线程池规定最大值，那么就拒绝执行此任务，  
   ThreadPoolExecutor会调用rejectedExecution方法通知调用者  


### 线程池的分类
1. FixThreadPool  
```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```
2. CachedThreadPool  
适合执行大量的耗时较少的任务
```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
3. ScheduledThreadPool  
```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```
4. SingleThreadExecutor
```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

- 以上四种线程池典型使用方法：
```
Runnable command = new Runnable() {
    @Override
    public void run() {
        SystemClock.sleep(2000);
    }
};

ExecutorService fixedThreadPool = Executors.newFixedThreadPool(4);
fixedThreadPool.execute(command);

ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
cachedThreadPool.execute(command);

ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(4);
// 2000ms后执行command
scheduledExecutorService.schedule(command, 2000, TimeUnit.MILLISECONDS);
// 延迟10ms后， 每隔1000ms执行一次command
scheduledExecutorService.scheduleAtFixedRate(command, 10, 1000, TimeUnit.MILLISECONDS);

ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
singleThreadExecutor.execute(command);
```