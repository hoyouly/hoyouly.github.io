---
layout: post
title: 工作填坑系列---Android 高版本中AlarmManager，PendingIntent的坑
category: 工作填坑
tags: AlarmManager PendingIntent
---
* content
{:toc}

## 前提
这两天要做一个需求，简单来讲，就是定时的往服务器上传数据，原本以为是一个很简单的需求，选择合适的定时器，然后传递数据，执行网络操作，结果发现遇到了各种坑

## 定时器选择
Android上能做定时器的有好几种方式，大概有以下几种，
1. Timer
2. AlarmManager
3. Handler
4. Thread   

这几种方式的优劣可以参考  [Android实现定时器的几种方法](https://blog.csdn.net/u011315960/article/details/52121717) 里面写的很详细。   
最终我选择了 AlarmManager方式，原因不解释

## AlarmManager
使用方式，这个就简单了。可是谁知道，越是简单的东西，往往坑越多，先看我们通用的实现定时器的方式吧。

### 1. 得到AlarmManager对象
这个简单，
```java
AlarmManager mAlarmManager = (AlarmManager) context.getSystemService(Context.ALARM_SERVICE);
```
### 2. 选择要开启的组件
是一个Activity，还是Service，或者Broadcast，
因为要执行网络操作，耗时的，肯定在子线程中，那最明智的选择就是IntentService
因为这个IntentService 要处理相应的上传操作，我又不想创建新的网络操作对象，就直接写了一个内部类，结果一个坑就在这里了，后面会将解释
```java
public class TimerActivity extends AppCompatActivity {
    private static final String TAG = "hoyouly";

    private RetrofitUtils mRetrofitUtils;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    public class UploadService extends IntentService {

        public UploadService() {
            super("UploadService");
        }

        @Override
        protected void onHandleIntent(@Nullable Intent intent) {
            Log.d(TAG, "onHandleIntent: ");
        }
    }
}
```
然后通过Intent 启动这个Service
```java
mIntent = new Intent(context, TimerActivity.UploadService.class);
```

### 3. 创建相应的PendingIntent
这个说是简单，主要是这几个参数的意思，不懂的自己Google吧。
```java
mPendingIntent = PendingIntent.getService(context, 0, mIntent, PendingIntent.FLAG_UPDATE_CURRENT);
```

### 4. 设置定时器
好久不用mAlarmManager了，第一次直接使用的 set方法，后来发现这个只能使用一次，不能重复，要使用setRepeating()方法才行。坑又来了，后面会说道这个坑。
```java
mAlarmManager.setRepeating(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(),TIME_INTERVAL, mPendingIntent);
```
思路理清了，开始写代码吧。
## 写代码

1. 我把这一部分都封装起来，成了一个AlarmManagerWrapper类

```java
public class AlarmManagerWrapper {
    private static AlarmManagerWrapper mInstance;

    //循环上报的间隔，先定10秒，方便自测。
    private static final long TIME_INTERVAL = 10 * 1000;

    private PendingIntent mPendingIntent;
    private AlarmManager mAlarmManager;
    private final Intent mIntent;

    private AlarmManagerWrapper(Context context) {
        mAlarmManager = (AlarmManager) context.getSystemService(Context.ALARM_SERVICE);
        mIntent = new Intent(context, TimerActivity.UploadService.class);
        mPendingIntent = PendingIntent.getService(context, 0, mIntent, PendingIntent.FLAG_UPDATE_CURRENT);
    }

    public static AlarmManagerWrapper getInstance(Context context) {
        if (mInstance == null) {
            synchronized (AlarmManagerWrapper.class) {
                if (mInstance == null) {
                    mInstance = new AlarmManagerWrapper(context);
                }
            }
        }
        return mInstance;
    }

    /**
     * 取消定时
     */
    public void cancelAlarm() {
        mAlarmManager.cancel(mPendingIntent);
    }

    /**
     * 开启定时
     */
    public void startAlarm() {
        mAlarmManager.setRepeating(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(),TIME_INTERVAL, mPendingIntent);
    }
}
```
2. 在Activity中设置了两个点击事件，开始和结束    

```java
public class TimerActivity extends AppCompatActivity {
    private static final String TAG = "hoyouly";
    private AlarmManagerWrapper mWrapper;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mNetWork = NetWork.getInstance(this);
        mWrapper = AlarmManagerWrapper.getInstance(this);
    }

    public void cancel(View view) {
        mWrapper.cancelAlarm();
    }

    public void start(View view) {
        mWrapper.startAlarm();
    }

    public  class UploadService extends IntentService {

        public UploadService() {
            super("UploadService");
        }

        @Override
        protected void onHandleIntent(@Nullable Intent intent) {
            Log.d(TAG, "onHandleIntent: ");
            //TODO 上传数据
        }
    }
}
```
3. 在AndroidManifest.xml中注册该Service，这点不能忘记。

```java
<service android:name=".TimerActivity$UploadService"/>
```
满心欢喜的以为这样就行了，可是谁知道，运行后，点击开始按钮，然后崩了，瞬间心情就不好了，但是没办法，咱的工作就是写bug然后改bug的啊。好吧
`adb logcat -b crash` 查看崩溃日志吧,然后就看到了

```java
java.lang.InstantiationException: java.lang.Class<***.TimerActivity$UploadService> has no zero argument constructor
```
第一个坑出现了。
### 坑一：内部类的组件必须得是static

这是毛线啊，我明明设置无参构造函数了啊。不懂简单，Google吧。于是就有了大神的解释，简单的来说就是内部Service需要设置成静态的，原因参考大神的解释   [No empty constructor when create a service](https://stackoverflow.com/questions/11859403/no-empty-constructor-when-create-a-service)。
好吧，那就设置成静态的吧，其实我是不太想设置成静态的，因为如果设置成静态的，那么我前面的请求网络的变量也的设置成静态的，可是这样好像不太好，但是又没有其他的好处理的办法。   
注：没有直接在这个内部类中创建一个新的网络请求参数变量，是因为我们业务逻辑中，这个网络请求变量创建需要很多参数，太麻烦，就想直接使用外部类中的那个变量。

改好后的代码就如下了
```java

public class TimerActivity extends AppCompatActivity {
    private static final String TAG = "hoyouly";
    private static NetWork mNetWork;
    private AlarmManagerWrapper mWrapper;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mNetWork = NetWork.getInstance(this);
        mWrapper = AlarmManagerWrapper.getInstance(this);
    }

    public void cancel(View view) {
        mWrapper.cancelAlarm();
    }

    public void start(View view) {
        mWrapper.startAlarm();
    }

    public static class UploadService extends IntentService {

        public UploadService() {
            super("UploadService");
        }

        @Override
        protected void onHandleIntent(@Nullable Intent intent) {
            Log.d(TAG, "onHandleIntent: ");
            //TODO 上传数据
        }
    }
}
```
`注意内部类 加上了static 关键字，`   
log 是打印出来了，
```java
11-07 18:54:02.892 13581 13613 D hoyouly : onHandleIntent:
11-07 18:55:32.467 13581 13660 D hoyouly : onHandleIntent:
11-07 18:56:04.088 13581 13678 D hoyouly : onHandleIntent:
```
但是感觉不对啊，我设置的10秒请求，可是这间隔也太长了吧，一分钟半才执行一次，这是个毛线问题啊。第二个坑来了
### 坑二：Android 6.0 为了性能优化修改AlarmManager的定时API
继续Google吧。原来在Android 6.0 后，Google为了 对低电耗模式和应用待机模式进行针对性优化，改API了，需要使用setExactAndAllowWhileIdle()这个API定时发送才行，具体原因查看 [关于使用 AlarmManager 的注意事项](https://juejin.im/entry/588628e8128fe10065eb62a9)。   
按照这上面的重新修改后的代码如下，AlarmManagerWrapper和UploadService都需要改
```java
public class AlarmManagerWrapper {
    private static AlarmManagerWrapper mInstance;

    //循环上报的间隔，先定12秒，方便自测。
    private static final long TIME_INTERVAL = 10 * 1000;

    private PendingIntent mPendingIntent;
    private AlarmManager mAlarmManager;
    private final Intent mIntent;

    private AlarmManagerWrapper(Context context) {
        mAlarmManager = (AlarmManager) context.getSystemService(Context.ALARM_SERVICE);
        mIntent = new Intent(context, TimerActivity.UploadService.class);
        mPendingIntent = PendingIntent.getService(context, 0, mIntent, PendingIntent.FLAG_UPDATE_CURRENT);
    }

    public static AlarmManagerWrapper getInstance(Context context) {
        if (mInstance == null) {
            synchronized (AlarmManagerWrapper.class) {
                if (mInstance == null) {
                    mInstance = new AlarmManagerWrapper(context);
                }
            }
        }
        return mInstance;
    }
    /**
     * 取消定时
     */
    public void cancelAlarm() {
        mAlarmManager.cancel(mPendingIntent);
    }

    /**
     * 开启定时
     */
    public void startAlarm() {
        mAlarmManager.setExactAndAllowWhileIdle(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), mPendingIntent);
    }

    /**
     * 由于Android 6.0  对低电耗模式和应用待机模式进行针对性优，所以需要再接受到执行定时任务的时候，再次开启，保证在低电耗模式下的也能正常执行
     */
    public void startAgain() {
        mAlarmManager.setExact(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime() + TIME_INTERVAL, mPendingIntent);
    }
}

public static class UploadService extends IntentService {

       public UploadService() {
           super("UploadService");
       }

       @Override
       protected void onHandleIntent(@Nullable Intent intent) {
           Log.d(TAG, "onHandleIntent: ");
           AlarmManagerWrapper.getInstance(getApplicationContext()).startAgain();
           //TODO 上传数据
       }
   }
```

这样log 看起来就正常了，每10秒打印一次。
```java
1-07 19:02:36.930 14109 14138 D hoyouly : onHandleIntent:
11-07 19:02:46.938 14109 14145 D hoyouly : onHandleIntent:
11-07 19:02:56.945 14109 14154 D hoyouly : onHandleIntent:
11-07 19:03:06.955 14109 14161 D hoyouly : onHandleIntent:
11-07 19:03:16.967 14109 14167 D hoyouly : onHandleIntent:
11-07 19:03:26.975 14109 14172 D hoyouly : onHandleIntent:
```
可我还想再每次上传数据的时候传递一些参数过去啊，
## 通过PendingIntent 传递参数
这个应该简单了吧，通过Intent，然后putExtra()，不管是基本数据类型，还是Parcelable类型的，Serializable类型的，都可以，因为业务需要传递一个Parcelable类型实体对象比较合适。那就传递吧，简单。
1. 在startAlarm()中接受一个实现 Parcelable类型的对象（Trip.java），然后放到Intent中    

```java
public void startAlarm(Trip data) {
     mIntent = new Intent(mContext, TimerActivity.UploadService.class);
     mIntent.putExtra("trip",data);
     mPendingIntent = PendingIntent.getService(mContext, 0, mIntent, PendingIntent.FLAG_UPDATE_CURRENT);
     mAlarmManager.setExactAndAllowWhileIdle(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), mPendingIntent);
 }
```

2. 在调用该方法的地方创建一个Trip对象，传递过去

```java
public void start(View view) {
        Trip trip = new Trip(1, "hello");
        mWrapper.startAlarm(trip);
    }
```
3. 在UpdateService中接受这个对象

```java
@Override
protected void onHandleIntent(@Nullable Intent intent) {
   Trip trip = intent.getParcelableExtra("trip");
   Log.d(TAG, "onHandleIntent: " + trip);
   AlarmManagerWrapper.getInstance(getApplicationContextstartAgain();
    //TODO  上传数据
}
```
4. 打印这个对象，打印结果如下：

```java
11-07 22:44:07.974  8596  8664 D hoyouly : onHandleIntent: null
11-07 22:44:17.980  8596  8668 D hoyouly : onHandleIntent: null
11-07 22:44:27.985  8596  8675 D hoyouly : onHandleIntent: null
11-07 22:44:37.993  8596  8681 D hoyouly : onHandleIntent: null
11-07 22:44:47.997  8596  8683 D hoyouly : onHandleIntent: null
11-07 22:44:58.002  8596  8686 D hoyouly : onHandleIntent: null
11-07 22:45:08.009  8596  8688 D hoyouly : onHandleIntent: null
11-07 22:45:18.014  8596  8692 D hoyouly : onHandleIntent: null
11-07 22:45:28.019  8596  8693 D hoyouly : onHandleIntent: null
11-07 22:45:38.025  8596  8697 D hoyouly : onHandleIntent: null
11-07 22:45:48.031  8596  8698 D hoyouly : onHandleIntent: null
11-07 22:45:58.036  8596  8710 D hoyouly : onHandleIntent: null
11-07 22:46:08.040  8596  8712 D hoyouly : onHandleIntent: null
11-07 22:46:18.045  8596  8720 D hoyouly : onHandleIntent: null
11-07 22:46:28.056  8596  8723 D hoyouly : onHandleIntent: null
```
怎么一直是null呢，不应该啊，断点调试，发现在创建PendingIntent的时候，那个mIntent对象里面是有这个trip的啊，第三个坑出现了。

### 坑三：Android 7.0 后通过PendingIntent传递Parcelable类型数据为null

继续Google，PendingIntent 参数为null，网上说了一大堆，和什么创建PendingIntent的时候的requestCode 或者flags有关，可是按照他们说的做了，还是为null。最后无意间发现了，原来是一个bug ,可以参照  [Android 7.0 pendingIntent bug(AlarmManager通过PendingIntent传递数据（跨进程数据传递](https://blog.csdn.net/m190607070/article/details/78492887) 上面的解释，按照上面的方法做，不传递Parcelable类型的对象，而是把对象转换成String。
```java
public void startAlarm(String data) {
       mIntent = new Intent(mContext, TimerActivity.UploadService.class);
       mIntent.putExtra("trip",data);
       mPendingIntent = PendingIntent.getService(mContext, 0, mIntent, PendingIntent.FLAG_UPDATE_CURRENT);
       mAlarmManager.setExactAndAllowWhileIdle(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), mPendingIntent);
   }
```
再传递，结果正常了

```java
11-07 22:55:18.480  9151  9184 D hoyouly : onHandleIntent: {"time":1,"name":"hello"}
11-07 22:55:28.485  9151  9186 D hoyouly : onHandleIntent: {"time":1,"name":"hello"}
11-07 22:55:38.491  9151  9189 D hoyouly : onHandleIntent: {"time":1,"name":"hello"}
11-07 22:55:48.494  9151  9192 D hoyouly : onHandleIntent: {"time":1,"name":"hello"}
11-07 22:55:58.498  9151  9198 D hoyouly : onHandleIntent: {"time":1,"name":"hello"}
11-07 22:56:08.502  9151  9200 D hoyouly : onHandleIntent: {"time":1,"name":"hello"}
11-07 22:56:18.507  9151  9203 D hoyouly : onHandleIntent: {"time":1,"name":"hello"}
11-07 22:56:28.524  9151  9205 D hoyouly : onHandleIntent: {"time":1,"name":"hello"}
```
目前为止，坑终于平晚了，主要还是Android 新版本的一些问题，原本熟悉的正常的代码到了新版本，尤其是Android 6.0 以后，就会遇到各种坑。不过总算解决了，唯一美中不足的是，要把Service变成静态内部类，这样他使用的所有外部类的变量都得成静态的，可是我又没办法把Service变成单独的类，如果这样，需要创建的变量会更麻烦，所以只能这样了。

---
搬运地址：  
 [No empty constructor when create a service](https://stackoverflow.com/questions/11859403/no-empty-constructor-when-create-a-service)    
[关于使用 AlarmManager 的注意事项](https://juejin.im/entry/588628e8128fe10065eb62a9)   
[Android 7.0 pendingIntent bug(AlarmManager通过PendingIntent传递数据（跨进程数据传递](https://blog.csdn.net/m190607070/article/details/78492887)
