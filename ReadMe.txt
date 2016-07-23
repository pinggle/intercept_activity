intercept activity

在 Java 平台要做到动态运行模块和热拔插可以使用 ClassLoader 技术进行动态类加载,
比如使用 OSGi 技术.
在 Android 上当然也可以使用动态加载技术, 但是仅仅把类加载进来还不够, Activity,
Service 等组件是有生命周期的, 它们统一由系统服务 AMS 管理; 仅仅使用 ClassLoader 
来加载动态类是不够的, 我们要想办法将加载进来的 Activity 等组件交给系统管理,让
AMS 赋予组件生命周期.

我们现在是要动态启动一个 Activity, 那么就 Android 开发而言, 要成功启动一个Activity,
必须要在 AndroidManifest.xml 文件中进行显示声明, 否则就会在跳转Activity时报错.

我们采用的大致思路时这样的:
首先在 AndroidManifest.xml 文件中声明一个替身, 当需要启动插件的某个 Activity 的时候,
先让系统启动替身Activity, 待替身Activity经过了"系统安检口", 我们将它劫持,换回我们
真正需要启动的Activity.

[[ 分析 Activity 启动过程 ]]  -- Android6.0 为例;
// A 用户调用接口API启动Activity: Activity::startActivity
	public void startActivity(Intent intent) {
	  this.startActivity(intent, null);
 	}
// B ==> startActivityForResult ==> Instrumentation::execStartActivity
// C ==> ActivityManagerNative.getDefault().startActivity
// D ==> 通过 Binder IPC 到 AMS 所在进程调用 AMS 的 startActivity 方法;
// E ==> ActivityManagerService::startActivity ==> AMS::startActivityAsUser
// F ==> ActivityStackSupervisor::startActivityMayWait
	在这个方法内对传进来的 Intent 进行了解析, 并尝试从中取出关于启动 Activity 的信息;
	这个方法调用了 startActivityLocked 方法; 在 startActivityLocked 方法内部进行了一系列
	重要的检查: 比如权限的检查,Activity的exported属性检查;检查是否在 Manifest 中显示声明;
	        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
			// We couldn't find the specific class specified in the Intent.
			// Also the end of the line.
			err = ActivityManager.START_CLASS_NOT_FOUND;
		}
	(这个校验过程在 AMS 所在的进程 system_server)
// G ==> ActivityStackSupervisor::realStartActivityLocked  ("真正的启动Activity")
// H ==> ApplicationThread::scheduleLaunchActivity (具体实现是在 ActivityThread.java 中的 ApplicationThread 类)
	ApplicationThread 实际上是一个Binder对象,是APP所在的进程与AMS所在进程system_server通信的桥梁;
// I ==> 通过一个消息转发给主线程: sendMessage(H.LAUNCH_ACTIVITY, r);
	// H 是一个 Handler
		----------------------------------------------------------------------
		Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
		final ActivityClientRecord r = (ActivityClientRecord) msg.obj;
		r.packageInfo = getPackageInfoNoCheck(
			r.activityInfo.applicationInfo, r.compatInfo);
		handleLaunchActivity(r, null);
		Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
		----------------------------------------------------------------------
// J ==> ActivityThread::handleLaunchActivity  ==> Activity a = performLaunchActivity(r, customIntent);
// K ==> ActivityThread::performLaunchActivity 
	1.使用 ClassLoader 加载并通过反射创建 Activity 对象;
		----------------------------------------------------------------------
		java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
		activity = mInstrumentation.newActivity(
		        cl, component.getClassName(), r.intent);
		StrictMode.incrementExpectedActivityCount(activity.getClass());
		r.intent.setExtrasClassLoader(cl);
		r.intent.prepareToEnterProcess();
		----------------------------------------------------------------------
	2.如果 Application 还没有创建,那么创建 Application 对象并回调相应的生命周期方法;
		----------------------------------------------------------------------
		Application app = r.packageInfo.makeApplication(false, mInstrumentation);
		if (r.isPersistable()) {
		    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
		} else {
		    mInstrumentation.callActivityOnCreate(activity, r.state);
		}
		----------------------------------------------------------------------
// L ==> 小结: AMS管理整个系统的Activity堆栈,Activity的生命周期回调都是由 AMS 所在的系统进程
		system_server 帮开发者完成的; Android的Framework层帮忙完成了诸如生命周期管理等
		繁琐复杂的过程,简化了应用层的开发.

我们简化一下上面的流程:
1.APP->startActivity: ABCDE ;
2.system_server(AMS) 权限及真实性校验: F ;
3.APP->真正创建Activity对象: GHIJK ;

1,3是在APP进程内; 2在系统进程内;
我们可以在第1步启动一个已经在 AndroidManifest.xml 里面声明过的替身Activity;让它进入第2个环节接受AMS的检验;
最后在第3步的时候换成我们需要启动的Activity即可. (这样就成功欺骗AMS进程)


[[ 实战启动我们的Activity ]]
战略:
1.在 AndroidManifest.xml 文件中声明我们的替身Activity : StubActivity;
2.在 应用中直接启动我们的目标Activity: startActivity(new Intent(MainActivity.this, TargetActivity.class));
3.拦截 ActivityManagerService::startActivity 方法,在这里将 TargetActivity 替换为 StubActivity;
  ( 使用替身躲过AMS检测 )
4.替换ActivityThread中 H(Handler) 的 mCallback 变量, 这个 mCallback 由我们伪造, 在这里将 替身 StubActivity 恢复成我们的真身 TargetActivity;
  ( 现在是刚从AMS回到我们的APP进程,拦截下来,替换真身,创建Activity )

这里为什么要替换 Handler 的 mCallBack 变量, 因为 Handler 类消息分发的过程如下:
1.如果传递的 Message 本身就有 callback, 那么直接使用 Message 对象的 callback 方法;
2.如果 Handler 类的成员变量 mCallback 存在, 那么首先执行这个 mCallback 回调;
3.如果 mCallback 的回调返回 true, 那么表示消息已经成功处理; 直接结束;
4.如果 mCallback 的回调返回 false,那么表示消息没有处理完毕,会继续使用 Handler 类的 handleMessage 方法处理消息;





[[ 分析 Activity 的 onDestroy 函数 ]]
==> Activity::finish()
==> ActivityManagerNative.getDefault().finishActivity();
==> 通过 Binder IPC 到 AMS 所在进程调用 AMS 的 finishActivity 方法;
==> ActivityManagerService::finishActivity
// 还真没理清这个路线怎么走.
// ... ...
==> ActivityThread::scheduleDestroyActivity
==> ActivityThread::performDestroyActivity
	ActivityClientRecord r = mActivities.get(token);
	mInstrumentation.callActivityOnDestroy(r.activity);
	这里通过 mActivities 拿到了一个 ActivityClientRecord, 然后直接把这个 record 里面的 Activity 交给
	Instrument 类完成了 onDestroy 的调用;
	r.activity 是替身Activity还是真身Activity, 一切的秘密在 token 里面.
	AMS 与 ActivityThread 之间对于Activity的生命周期的交互,并没有直接使用Activity对象进行交互,而是使用一个 token 来标识,
	这个token是binder对象,因此可以方便滴跨进程传递. Activity里面有一个成员变量mToken代表的就是它, token可以唯一地标识一个
	Activity对象, 它在Activity的 attach 方法里面初始化;

	在APP进程里面, token 对应的是 真身Activity.
	回到代码, ActivityClientRecord 是在 mActivities 里面取出来的, 确实是根据token取; 那么这个 token 是什么时候添加进去的呢?
	我们看 performLaunchActivity 就明白了: 它通过 classloader 加载了 TargetActivity, 然后完成一切操作以后把这个 activity 添加
	进了 mActivities! 另外, 在这个方法里面我们还能看到对 Activity attach 方法的调用, 它传递给了新创建的 Activity 一个token对象,
	而这个token是在 ActivityClientRecord 构造函数里面初始化的.

	结论: 通过这种方法启动的Activity有它自己完整而独立的生命周期!

作业: 
1.支持动态启动不同(LaunchMode)模式的Activity;
  参考 DroidPlugin 的 com.morgoo.droidplugin.stub 包下面;
2.替身Activity数量如何解决?
  我们要启动50个Activity,难道要我们声明50个替身Activity吗?
3.如何启动一个外部的Activity;




















