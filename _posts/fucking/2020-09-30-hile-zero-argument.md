---
layout: post
title:  卧槽系列 - Hilt遇到的 InstantiationException *** has no zero argument constructor
category: 卧槽系列
tags: Hilt  
---
<!-- * content -->
<!-- {:toc} -->
看到很多人都学习Jetpack，我也想着充充电，学习过程中知道了Hilt 这个库,觉得有点意思，就想着玩一把，刚开始以为很简单，不就是一个依赖注入，简单。

因为前段时间根据[学习Android Jetpack? 实战和教程这里全都有！](https://www.jianshu.com/p/f32c8939338d)，这个把里面的demo敲了一遍，想着就直接在这上面改吧。

教程是根据[Jetpack 新成员 Hilt 实践（一）启程过坑记](https://juejin.im/post/6844904198803292173?utm_source=gold_browser_extension#heading-14)
这个上面写的，博主写的很详细，而且里面写了好几个遇到的坑。想着有人已经踩过坑了，我就直接一路小跑，飞奔到Hilt怀抱即可。

直接改造LoginModel，三下五除二就搞定，运行看看效果吧。

然后悲剧了。崩溃了，一直提示我这个异常
```
java.lang.RuntimeException: Cannot create an instance of class com.hoyouly.jetpackdemo.viewmodel.LoginModel
	at androidx.lifecycle.ViewModelProvider$NewInstanceFactory.create(ViewModelProvider.java:221)
	at androidx.lifecycle.ViewModelProvider$AndroidViewModelFactory.create(ViewModelProvider.java:278)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.java:187)
	at androidx.lifecycle.ViewModelProvider.get(ViewModelProvider.java:150)
	at androidx.lifecycle.ViewModelLazy.getValue(ViewModelProvider.kt:54)
	at androidx.lifecycle.ViewModelLazy.getValue(ViewModelProvider.kt:41)
	at com.hoyouly.jetpackdemo.ui.fragment.login.LoginFragment.getLoginModel(Unknown Source:7)
	at com.hoyouly.jetpackdemo.ui.fragment.login.LoginFragment.onCreateView(LoginFragment.kt:41)
	at androidx.fragment.app.Fragment.performCreateView(Fragment.java:2698)
	at androidx.fragment.app.FragmentStateManager.createView(FragmentStateManager.java:310)
	at androidx.fragment.app.FragmentManager.moveToState(FragmentManager.java:1185)
	at androidx.fragment.app.FragmentManager.addAddedFragments(FragmentManager.java:2222)
	at androidx.fragment.app.FragmentManager.executeOpsTogether(FragmentManager.java:1995)
	at androidx.fragment.app.FragmentManager.removeRedundantOperationsAndExecute(FragmentManager.java:1951)
	at androidx.fragment.app.FragmentManager.execPendingActions(FragmentManager.java:1847)
	at androidx.fragment.app.FragmentManager$4.run(FragmentManager.java:413)
	at android.os.Handler.handleCallback(Handler.java:790)
	at android.os.Handler.dispatchMessage(Handler.java:99)
	at android.os.Looper.loop(Looper.java:164)
	at android.app.ActivityThread.main(ActivityThread.java:6523)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:857)
Caused by: java.lang.InstantiationException: java.lang.Class<com.hoyouly.jetpackdemo.viewmodel.LoginModel> has no zero argument constructor
	at java.lang.Class.newInstance(Native Method)
	at androidx.lifecycle.ViewModelProvider$NewInstanceFactory.create(ViewModelProvider.java:219)
	... 22 more
```
我开始思考人生，不应该啊，我是按照教程写的，并且人家项目[PokemonGo](https://github.com/hi-dhl/PokemonGo) 中用的好好的啊。

先看看代码吧
```kotlin
class LoginModel @ViewModelInject constructor(private val respository: UserRepository) : ViewModel() {
...
}

class UserRepository private constructor(private val userDao: UserDao) {
...
}

@Module
@InstallIn(ApplicationComponent::class)
class RepositoryModule {

    @Singleton
    @Provides
    fun providerUserRepository(userDao: UserDao): UserRepository {
        return UserRepository.getInstance(userDao)
    }
...
}  

@Module
@InstallIn(ApplicationComponent::class)
object RoomModule {
    @Singleton
    @Provides
    fun providerAppDataBase(application: Application): AppDataBase {
        return Room.databaseBuilder(application, AppDataBase::class.java, "jetpackdemo.db")
            .addCallback(object : RoomDatabase.Callback() {
                override fun onCreate(db: SupportSQLiteDatabase) {
                    super.onCreate(db)
                    val request = OneTimeWorkRequestBuilder<ShoeWorker>().build()
                    WorkManager.getInstance(application).enqueue(request)
                }
            }).build()
    }

    @Singleton
    @Provides
    fun providerUserDao(dataBase: AppDataBase): UserDao {
        return dataBase.userDao()
    }
}  

```
然后在Fragment中使用这个model

```kotlin
@AndroidEntryPoint
class LoginFragment : Fragment() {
    private val loginModel: LoginModel by viewModels()
    ...
}

@AndroidEntryPoint
class LoginActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)
    }
}

@HiltAndroidApp
open class BaseApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        context = this
    }

    companion object {
        lateinit var context: Context
    }
}
```
该添加的注解我都添加了，不应该出问题啊。可是它就是出现了。我一直在说不应该啊，不应该啊。可事与愿违啊。

而且让人很纠结，不知道从哪里下手查找原因。

因为本身Hilt是第一次玩，而kotlin，jetpack,也是刚认识几天，一大堆新玩意儿，MVVM架构更导致调试起来麻烦，但是遇见问题总的解决吧。

MVVM不好断点调试，就一点点打印log调试吧。这里面一共就用了UserDao,UserRepository,先看看这两个对象能不能正常的实例化吧。
```kotlin
@AndroidEntryPoint
class LoginFragment : Fragment() {
  @Inject
  lateinit var userRepository: UserRepository

  @Inject
  lateinit var userDao: UserDao

  override fun onCreateView(
          inflater: LayoutInflater,
          container: ViewGroup?,
          savedInstanceState: Bundle?
      ): View? {
          val binding = FragmentLoginBinding.inflate(inflater, container, false)
          GlobalScope.launch {
              userDao.getAllUsers()
              userRepository.getAllUsers()
          }
          binding.model = loginModel
          binding.activity = activity
          binding.isEnable = isEnable
          this.dataBinding = binding
          loginModel.lifecycleOwner=viewLifecycleOwner
          return binding.root
      }
  ...
}

```
在onCreatView中直接调用者两个变量，发现是正常的，这说明已经实例化的，可为啥LoginModel就不行了呢，然后我有看了异常，发现了 `has no zero argument constructor`,没有无参构造函数，难道和构造函数有关，我就改了一把，把LoginMode改成了无参构造函数。如下
```kotlin
class LoginModel @ViewModelInject constructor() : ViewModel() {
...
}
```
这种形式LoginModel是可以正常实例化的，这就奇怪了，构造函数里面传递的 UserRepository 可以正常实例化，为啥组合在一起就不行了。

[PokemonGo](https://github.com/hi-dhl/PokemonGo) 这个项目中就是这么写的啊，能正常运行啊？？挠头。到底是哪里出了问题。

遇到这种问题，最直接的办法就是
1. 精简代码
我就开始精简 PokemonGo 中的代码，只留下了MainViewModel 相关的代码，剩下大概六七个类吧。也能正常运行的。
2. copy 代码
把PokemonGo 中的代码copy到我的项目中，发现也出问题，那说明不是代码的问题，会是什么呢，

既然不是代码的问题，那就是依赖库的原因呗，然后就开始精简  PokemonGo 中依赖库。移调所有已经用不着的三方库，什么okhttp,cardview,retrofit 之类的库，剩下必须留下的六个。还真发现了一点问题。有些依赖库的版本比我使用的要高一些，会和那个依赖库有关呢？

一个一个试试吧，皇天不负有心人啊，终于找到原因了。

竟然是`androidx.fragment:fragment-ktx 版本的问题。`
我的fragment-ktx 版本是1.1.0-alpha09，而 PokemonGo 中的版本是1.3.0-alpha06 。

因为 [学习Android Jetpack? 实战和教程这里全都有！](https://www.jianshu.com/p/f32c8939338d)这个教程写的比较早，可能当时的fragment-ktx最高就是 1.1.0吧，而 PokemonGo 项目写的晚，版本已经升级到了1.0.3。

一个小小的版本问题，导致我纠结了好几天。不过还好，总算解决了，继续我的Hilt学习之路吧。






---
搬运地址：    
[Jetpack 新成员 Hilt 实践（一）启程过坑记](https://juejin.im/post/6844904198803292173?utm_source=gold_browser_extension#heading-14)

[学习Android Jetpack? 实战和教程这里全都有！](https://www.jianshu.com/p/f32c8939338d)
