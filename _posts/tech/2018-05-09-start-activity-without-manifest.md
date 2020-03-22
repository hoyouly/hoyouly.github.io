---
layout: post
title: 启动一个不在 AndroidManifest 中注册的 Activity
category: 技术
tags: AndroidManifest
---
* content
{:toc}

这个也是一个面试题，但是没有归结到街题系列是因为我觉得这个问题应该不是被常问道的一个，所以归结到技术中。
本文是参考[插件化入门篇-如何启动一个未注册过的Activity](https://www.jianshu.com/p/4fc77fbac938)，只是加上自己的了解，原理还有代码都是[插件化入门篇-如何启动一个未注册过的Activity](https://www.jianshu.com/p/4fc77fbac938)他里面的，
1. 我把 attachBaseContext()里面的代码提取出来一个新的方法，命名为relaceActivity(),经过证实，  relaceActivity()方法其他方法，例如onCreat()或者onStart()中也可以，我之前一直以为必须得放到attachBaseContext()中呢
2. ` String stubPackage = "com.jerey.activityplugin";` 这是硬编码，我改了，通过getPackageName()得到报名，
3. 一直不理解为啥要用反射Handler中的mCallback变量，后来终于明白了， 看源码，mCallback.handleMessage()比Handler的handleMessage()方法要优先执行，而在handleMessage()里面是进行 handleLaunchActivity(r, null)的，所以可以在mCallback.handleMessage（）里面做一些太子换狸猫的工作，把TmpActivity换成我们真正要启动的Activity，文中mCallback.handleMessage() 返回了true，但是在mCallback.handleMessage()中主动调用了mH.handleMessage(msg);其实完全没必须要，直接把mCallback.handleMessage() 返回false，这样既做了太子换狸猫的工作，有能继续执行mH.handleMessage(msg)方法，

```java
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

```

```java
private void replaceActivity() {
		try {
			//欺骗AMN，将要启动的Activity替换成我们占坑用的Activity
			Class<?> activityManagerNativeClass = Class.forName("android.app.ActivityManagerNative");//class android.app.ActivityManagerNative
			//getDeclaredField()获取一个类本身的所有字段，getField()只能获取类以及父类的public字段
			Field gDefaultField = activityManagerNativeClass.getDeclaredField("gDefault");//private static final android.util.Singleton android.app.ActivityManagerNative.gDefault
			gDefaultField.setAccessible(true);//在反射中访问私有便利
			Object gDefault = gDefaultField.get(null);//得到 android.app.ActivityManagerNative对象 相当于执行 ActivityManagerNative.getDefault()

			Class<?> singleton = Class.forName("android.util.Singleton");//class android.util.Singleton
			Field mInstanceField = singleton.getDeclaredField("mInstance");//private java.lang.Object android.util.Singleton.mInstance
			mInstanceField.setAccessible(true);//启用和禁用访问安全检查的开关,并不是为true就能访问为false就不能访问
			//从AMN的gDefault对象中原始的IActivityManager对象
			final Object rawIActivityManager = mInstanceField.get(gDefault);//class android.app.ActivityManagerProxy

			// 创建一个这个对象的代理对象, 然后替换这个字段, 让我们的代理对象帮忙干活
			Class<?> iActivityManagerInterface = Class.forName("android.app.IActivityManager");//interface android.app.IActivityManager
			Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class[]{iActivityManagerInterface}, new InvocationHandler() {
				@Override
				public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
					if ("startActivity".equals(method.getName())) {
						Intent raw;
						int index = 0;
						for (int i = 0; i < args.length; i++) {
							if (args[i] instanceof Intent) {
								index = i;
								break;
							}
						}

						raw = (Intent) args[index];
						//替身用的包名
						String packageName = getPackageName();

						ComponentName componentName = new ComponentName(packageName, TmpActivity.class.getName());

						Intent newIntent = new Intent();
						newIntent.setComponent(componentName);
						newIntent.putExtra(EXTRA_TARGET_INTENT, raw);//把原始的启动目标封装到intent中

						args[index] = newIntent;
						//这样就相当于启动了TmpActivity，因为TmpActivity在AndroidManifest.xml文件中注册过，所以可以正常启动
						return method.invoke(rawIActivityManager, args);//相当于执行 ActivityManagerNative.getDefault().startActivity()
					}
					return method.invoke(rawIActivityManager, args);
				}
			});
			mInstanceField.set(gDefault, proxy);//proxy :android.app.ActivityManagerProxy@4f63dda


			/**
			 * 欺骗ActivityThread
			 */
			Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");//class android.app.ActivityThread
			Field currentActivityThreadField = activityThreadClass.getDeclaredField("sCurrentActivityThread");//private static volatile android.app.ActivityThread android.app.ActivityThread.sCurrentActivityThread
			currentActivityThreadField.setAccessible(true);
			Object currentActivityThread = currentActivityThreadField.get(null);//class android.app.ActivityThread
			Field mHField = activityThreadClass.getDeclaredField("mH");//final android.app.ActivityThread$H android.app.ActivityThread.mH
			mHField.setAccessible(true);
			final Handler mH = (Handler) mHField.get(currentActivityThread);//Handler (android.app.ActivityThread$H) {13dec9c}

			//mCallback 之所以使用,是因为Handler消息机制，如果有mCallback不为null的话，那么就会先执行mCallback的handleMessage()方法，
			// 而如果mCallback的handlMessage()返回true的时候，就不会再执行Hander的handleMessage(msg)了，
			// 但是这里面做了一个处理，就是在mCallback的handlMessage()返回true之前，主动调用了Hander的handleMessage()方法，这样就又执行了mH的handlerMessage()方法，从而启动我们设置好的Activity

			Field mCallBackField = Handler.class.getDeclaredField("mCallback");//final android.os.Handler$Callback android.os.Handler.mCallback
			mCallBackField.setAccessible(true);
			mCallBackField.set(mH, new Handler.Callback() {
				@Override
				public boolean handleMessage(Message msg) {
					if (msg.what == 100) {//LAUNCH_ACTIVITY = 100
						Object obj = msg.obj;//其实就是ActivityClientRecord 对象
						try {
							//得到ActivityClientRecord 中的intent变量
							Field intent = obj.getClass().getDeclaredField("intent");
							intent.setAccessible(true);

							Intent raw = (Intent) intent.get(obj);//得到ActivityClientRecord 对象中的intent变量
							Intent target = raw.getParcelableExtra(EXTRA_TARGET_INTENT);
							raw.setComponent(target.getComponent());//从新设置要启动的Activity组件

						} catch (NoSuchFieldException e) {
							e.printStackTrace();
						} catch (IllegalAccessException e) {
							e.printStackTrace();
						}
//						mH.handleMessage(msg);
					}
					return false;//返回false,这样就可以继续执行mH.handleMessage(msg) 了，在mCallback中只是进行偷梁换柱的工作
				}
			});
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

```

---
搬运地址：    
[插件化入门篇-如何启动一个未注册过的Activity](https://www.jianshu.com/p/4fc77fbac938)
