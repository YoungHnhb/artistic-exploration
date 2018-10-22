# 第八章 理解Window和WindowManager
Window是抽象类：public abstract class Window
*The only existing implementation of this abstract class is
 android.view.PhoneWindow, which you should instantiate when needing a
 Window.*
WindowManager和WindowManagerService的交互是一个IPC过程

## Window和WindowManger
```
Button mFloatingButton = new Button(this);
mFloatingButton.setText("button");
LayoutParams mLayoutParams = new LayoutParams(
        LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT, 0, 0, PixelFormat.TRANSPARENT);
mLayoutParams.flags = LayoutParams.FLAG_NOT_TOUCH_MODAL
        | LayoutParams.FLAG_NOT_FOCUSABLE
        | LayoutParams.FLAG_SHOW_WALLPAPER;
mLayoutParams.gravity = Gravity.LEFT |Gravity.TOP;
mLayoutParams.x = 100;
mLayoutParams.y = 300;
getWindowManager().addView(mFloatingButton, mLayoutParams);
```
- Flags控制Window的显示特性
1. FLAG_NOT_FOCUSABLE：不接收各种输入事件，向下传递
2. FLAG_NOT_TOUCH_MODAL：内层自己处理，外层传递
3. FLAG_SHOW_WHEN_LOCKED：出现在锁屏界面
- type表示window的类型
1. 应用window：1-99
2. 子window：1000-1999
3. 系统window：2000-2999
有些情况需要配置<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>等

WindowManager提供了三个方法在ViewManager里，
而WindowManger继承了ViewManager
```
public interface ViewManager {
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```
## Window的内部机制
一个window对应一个View和一个ViewRootImpl，
window和view通过ViewRootImpl建立联系
### Window的添加过程
```
public interface WindowManager extends ViewManager

public final class WindowManagerImpl implements WindowManager {
	@Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
}
```
WindowManagerImp将实现全交给WindowManagerGlobal处理（桥接模式）
1. 检查参数是否合法，如果是子window还需要调整一些布局参数
2. 创建ViewRootImpl并将View添加到列表中
3. 通过ViewRootImpl来更新界面并完成Window的添加过程
WindowSession最终完成window添加过程
WindowSession类型是IWindowSession是一个Binder对象，真正的实现类是Session。
Window添加过程是一次IPC调用
Session内部会通过WindowManagerService来实现Window的添加

### Window的删除过程
window的删除过程和添加过程一样，先通过WindowManagerImpl，
再进一步通过WindowManagerGlobal的removeView实现
removeView通过findViewLocked查找待删除View的索引，
再调用removeViewLocked进一步的删除
removeViewLocked通过ViewRootImpl来完成删除操作
```
	@Override
    public void removeView(View view) {
    	// 异步删除
        mGlobal.removeView(view, false);
    }

    @Override
    public void removeViewImmediate(View view) {
    	// 同步删除
        mGlobal.removeView(view, true);
    }
```

### Window的更新过程
ViewRootImpl会通过WindowSession来更新Window视图，
最终由WindowManagerService的relayoutWindow()实现，也是IPC过程

## Window的创建过程
### Activity的Window创建过程
1. 如果没有DecorView，那么就创建他

2. 将View添加到DecorView的mContentParent中

3. 回调Activity的onContentChanged 方法通知Activity视图已经发生改变

### Dialog的Window创建过程
1. 创建window

2. 初始化DecorView并将Dialog的视图添加到DecorView中

3. 将DecorView添加到Window中并显示

- 当Dialog被关闭时，他会通过WindowManger来移除DecorView：
	mWindowManager.removeViewImmediate(mDecor);
- 普通的Dialog有特别之处，必须采用Activity的Context，
如果采用Application的Context则会报错：
```
BadTokenException:Unable to add window -- token null is not for an application
```
- 另外，系统window不需要token，也可将对话框的window设为系统类型，
但不能忘记添加uses-permission

### Toast的Window创建过程
- Toast具有定时取消功能，所以系统采用了Handler
- Toast内部有两类IPC过程
1. Toast访问NotificationManagerService
2. NotificationManagerService回调Toast里的TN接口
- Toast属于系统Window
内部的视图有两种方式制定：一、系统默认样式；二、通过setView制定自定义View
- Toast的show(), cancel()内部是一个IPC过程
```
public void show() {
    if (mNextView == null) {
        throw new RuntimeException("setView must have been called");
    }

    INotificationManager service = getService();
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    tn.mNextView = mNextView;

    try {
        service.enqueueToast(pkg, tn, mDuration);
    } catch (RemoteException e) {
        // Empty
    }
}

TN.class
public void hide() {
    if (localLOGV) Log.v(TAG, "HIDE: " + this);
    mHandler.obtainMessage(HIDE).sendToTarget();
}

public void handleShow(IBinder windowToken) {
    if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
            + " mNextView=" + mNextView);
    // If a cancel/hide is pending - no need to show - at this point
    // the window token is already invalid and no need to do any work.
    if (mHandler.hasMessages(CANCEL) || mHandler.hasMessages(HIDE)) {
        return;
    }
    if (mView != mNextView) {
        // remove the old view if necessary
        handleHide();
        mView = mNextView;
        Context context = mView.getContext().getApplicationContext();
        String packageName = mView.getContext().getOpPackageName();
        if (context == null) {
            context = mView.getContext();
        }
        mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
        ·····
        mParams.token = windowToken;
        if (mView.getParent() != null) {
            if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
            mWM.removeView(mView);
        }
        if (localLOGV) Log.v(TAG, "ADD! " + mView + " in " + this);
        // Since the notification manager service cancels the token right
        // after it notifies us to cancel the toast there is an inherent
        // race and we may attempt to add a window after the token has been
        // invalidated. Let us hedge against that.
        try {
            mWM.addView(mView, mParams);
            trySendAccessibilityEvent();
        } catch (WindowManager.BadTokenException e) {
            /* ignore */
        }
    }
}

public void handleHide() {
    if (localLOGV) Log.v(TAG, "HANDLE HIDE: " + this + " mView=" + mView);
    if (mView != null) {
        // note: checking parent() just to make sure the view has
        // been added...  i have seen cases where we get here when
        // the view isn't yet added, so let's try not to crash.
        if (mView.getParent() != null) {
            if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
            mWM.removeViewImmediate(mView);
        }

        mView = null;
    }
}
```
mToastQueue最多存在50个ToastRecord,是为了防止DOS(Denial of Service拒绝服务攻击)攻击

Q:一个应用到底有多少个Window