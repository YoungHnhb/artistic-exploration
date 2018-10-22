#第一章 Activity的生命周期和启动模式
##1.1 Activity的生命周期全面分析
###①典型情况（用户正常操作）
onCreate：做一些初始化
onRestart：
onStart：可见但未出现在前台
onResume：已可见切出现在前台
onPause：可存储一些数据，但不能做太耗时操作，会影响新activity显示，此方法执行完才会执行新activity的onResume
onStop：可做轻量级回收工作
onDestory：回收、释放资源

新的Activity启动之前，栈顶Activity需要先onPause(因为pauseBackStacks())
之后调用scheduleLaunchActivity()完成新Activity的onCreate、onStart、onResume的调用
E/TT: A create
E/TT: A Start
E/TT: A resume
E/TT: A pause
E/TT: B create
E/TT: B Start
E/TT: B resume
E/TT: A stop

###②异常情况（被回收或者configuration改变导致activity被销毁重建）

比如旋转屏幕若不对Activity做处理
10-21 23:16:58.486 29569-29569/com.liyueyang.demo E/TT: A create
10-21 23:16:58.488 29569-29569/com.liyueyang.demo E/TT: A Start
10-21 23:16:58.490 29569-29569/com.liyueyang.demo E/TT: A resume
10-21 23:17:01.080 29569-29569/com.liyueyang.demo E/TT: A pause
10-21 23:17:01.080 29569-29569/com.liyueyang.demo E/TT: A stop
10-21 23:17:01.080 29569-29569/com.liyueyang.demo E/TT: A destory
10-21 23:17:01.152 29569-29569/com.liyueyang.demo E/TT: A create
10-21 23:17:01.156 29569-29569/com.liyueyang.demo E/TT: A Start
10-21 23:17:01.161 29569-29569/com.liyueyang.demo E/TT: A resume
---
10-21 23:21:08.011 30089-30089/com.liyueyang.demo E/TT: A create
10-21 23:21:08.012 30089-30089/com.liyueyang.demo E/TT: A Start
10-21 23:21:08.016 30089-30089/com.liyueyang.demo E/TT: A resume
10-21 23:21:13.295 30089-30089/com.liyueyang.demo E/TT: A pause
10-21 23:21:13.297 30089-30089/com.liyueyang.demo E/TT: SaveInstanceState
10-21 23:21:13.298 30089-30089/com.liyueyang.demo E/TT: A stop
10-21 23:21:13.300 30089-30089/com.liyueyang.demo E/TT: A destory
10-21 23:21:13.351 30089-30089/com.liyueyang.demo E/TT: A create
10-21 23:21:13.357 30089-30089/com.liyueyang.demo E/TT: A Start
10-21 23:21:13.361 30089-30089/com.liyueyang.demo E/TT: RestoreInstanceState
10-21 23:21:13.363 30089-30089/com.liyueyang.demo E/TT: A resume
---
对Activity配置configChanges重写onConfigurationChanged()实现不重建Activity
---
##1.2 Activity的启动模式
###启动模式LaunchMode
1.standard 
*非Activity的Context没有所谓任务栈，需要为待启动Activity指定FLAG_ACTIVITY_NEW_TASK,
此时实际上是以singleTask模式启动*
2.singleTop
*位于栈顶且不会被重新创建，同时onNewIntent会被回调*
3.singleTask
4.singleInstance

*查看栈信息
adb shell dumpsys activity*
###标志位Flags
FLAG_ACTIVITY_NEW_TASK
FLAG_ACTIVITY_SINGLE_TOP
FLAG_ACTIVITY_CLEAR_TOP
FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS
*此标记不会出现在历史Activity列表等同android:excludeFromRecents="true"*
## IntentFilter的匹配规则
隐式调用Activity
需要Intent能够匹配目标的IntentFilter中的action、category、data
*一个IntentFilter可以有多个action、category、data且需要同时匹配action类别、category类别、data类别才算完全匹配(且)
一个Activity可以有多个IntentFilter，且只需匹配一个(或)*
	<activity android:name=".SecondActivity">
	    <intent-filter>
	        <action android:name="android.intent.action.SEND"/>
	        <!--public static final String ACTION_SEND = "android.intent.action.SEND";-->
	        <action android:name="android.intent.action.SEND_MULTIPLE"/>
	        <category android:name="android.intent.category.APP_BROWSER"/>
	        <!--public static final String CATEGORY_DEFAULT = "android.intent.category.DEFAULT";-->
	        <category android:name="android.intent.category.ALTERNATIVE"/>
	        <data android:mimeType="text/plain"/>
	    </intent-filter>
	</activity>
	Intent intent = new Intent("android.intent.action.SEND");
    intent.addCategory("android.intent.category.APP_BROWSER");
    intent.setDataAndType(Uri.parse("file://abc"), "text/plain");
    查找是否有匹配的Activity
    List<ResolveInfo> resolveInfos = mContext.getPackageManager()
                .queryIntentActivities(intent, PackageManager.MATCH_SYSTEM_ONLY);
    ResolveInfo resolveInfo = mContext.getPackageManager().resolveActivity(homeIntent, 0 /* flags */);