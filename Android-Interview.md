# Android 基础
1. 四大组件是什么？  
	Activity:包含用户界面的组件，主要用于和用户进行交互  
	Service：Android中实现程序后台运行的解决方案  
	Broadcast Receiver：广播接收器，广播分标准广播，有序广播  
	Content Provider：用在不同应用程序之间时间数据共享功能  
	四大组件通过Intent传递数据
2. Activity的生命周期
	- 正常进入A activity及按back键退出  
	E/TT: A onCreate->
	E/TT: A onStart->
	E/TT: A onResume->  
	E/TT: A onPause->
	E/TT: A onStop->
    E/TT: A onDestroy->
    - 启动B activity后按back键退出  
    E/TT: A onCreate->
	E/TT: A onStart->
	E/TT: A onResume->
	E/TT: A onPause->  
	E/TT: B create->
	E/TT: B Start->
	E/TT: B resume->
	E/TT: A onSaveInstanceState->
    E/TT: A onStop->  
    E/TT: B pause->
	E/TT: A onRestart->
	E/TT: A onStart->
    E/TT: A onResume->  
	E/TT: B stop->
    E/TT: B destory->
    E/TT: A onPause->
	E/TT: A onStop->
    E/TT: A onDestroy->
    - 启动A activity后按home键后再进入A  
    E/TT: A onCreate->
	E/TT: A onStart->
	E/TT: A onResume->
	E/TT: A onPause->
	E/TT: A onSaveInstanceState->  
	E/TT: A onStop->
	E/TT: A onRestart->
	E/TT: A onStart->
	E/TT: A onResume->

3. Activity之间的通信方式  
	通过Intent发送消息，有显示调用，隐式调用

4. Activity各种情况下的生命周期


5. 横竖屏切换时Activity的生命周期
	一般情况会触发onDestory->onCreate->onStart->onResume

	配置

6. 前台切换到后台，然后再回到前台时Activity的生命周期

7. 弹出Dialog的时候按Home键时Activity的生命周期

8. 两个Activity之间跳转时的生命周期

9. 下拉状态栏时Activity的生命周期

10. Activity与Fragment之间生命周期比较

11. Activity 的四种 LaunchMode（启动模式）的区别？
12. Activity 状态保存与恢复？
13. Fragment 各种情况下的生命周期？
14. [Activity 和 Fragment 之间怎么通信， Fragment 和 Fragment 怎么通信？](https://github.com/jeanboydev/Android-Interview/blob/master/Android.md#android_base_14)
15. Service 的生命周期？
16. Service 的启动方式？
17. Service 与 IntentService 的区别?
18. [Service 和 Activity 之间的通信方式？](https://github.com/jeanboydev/Android-Interview/blob/master/Android.md#android_base_18)
19. 对 ContentProvider 的理解？
20. ContentProvider、ContentResolver、ContentObserver 之间的关系？
21. 对 BroadcastReceiver 的了解？
22. 广播的分类？使用方式和场景？
23. [动态广播和静态广播有什么区别？](https://github.com/jeanboydev/Android-Interview/blob/master/Android.md#android_base_23)
24. AlertDialog、popupWindow、Activity 之间的区别？
25. Application 和 Activity 的 Context 之间的区别？
26. Android 属性动画特性？
27. LinearLayout、RelativeLayout、FrameLayout 的特性对比及使用场景？
28. 对 SurfaceView 的了解？
29. Serializable 和 Parcelable 的区别？
30. Android 中数据存储方式有哪些？
31. 屏幕适配的处理技巧都有哪些?
32. Android 各个版本 API 的区别？
33. 动态权限适配方案，权限组的概念？
34. 为什么不能在子线程更新 UI？
35. ListView 图片加载错乱的原理和解决方案？
36. 对 RecycleView 的了解？
37. Recycleview 和 ListView 的区别？
38. RecycleView 实现原理？
39. Android Manifest 的作用与理解？
40. 多线程在 Android 中的使用？