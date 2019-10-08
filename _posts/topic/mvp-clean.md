# 分析google开源项目android-architecture-todo-mvp-clean 的流程


# 删除Task为例


## UseCaseHandler
是一个单例对象，维护了一个UseCaseThreadPoolScheduler 对象，UseCaseThreadPoolScheduler中其实定义了ThreadPoolExecutor来执行异步任务

```java
public static UseCaseHandler getInstance() {
        if (INSTANCE == null) {//维护了一个UsdCaseThreadPoolScheduler实例对象，在其中定义了ThreadPoolExecutor来执行异步任务。
            //任务的结果使用Handler post的方法来切换到主线程
            INSTANCE = new UseCaseHandler(new UseCaseThreadPoolScheduler());
        }
        return INSTANCE;
    }
```

在TaskDetailFragment 中点击删除Task的时候，执行mPresenter.deleteTask();
这个mPresenter就是TaskDetailPresenter，在初始化TaskDetailPresenter 的时候，把Fragment对象传递过去，同时还传递进去UseCaseHandler对象，并且传递过去几个UseCase

```java
public TaskDetailPresenter(@NonNull UseCaseHandler useCaseHandler, @Nullable String taskId, @NonNull TaskDetailContract.View taskDetailView, @NonNull GetTask getTask, @NonNull CompleteTask
           completeTask, @NonNull ActivateTask activateTask, @NonNull DeleteTask deleteTask) {
       mTaskId = taskId;
       mUseCaseHandler = checkNotNull(useCaseHandler, "useCaseHandler cannot be null!");
       mTaskDetailView = checkNotNull(taskDetailView, "taskDetailView cannot be null!");
       mGetTask = checkNotNull(getTask, "getTask cannot be null!");
       mCompleteTask = checkNotNull(completeTask, "completeTask cannot be null!");
       mActivateTask = checkNotNull(activateTask, "activateTask cannot be null!");
       mDeleteTask = checkNotNull(deleteTask, "deleteTask cannot be null!");
       mTaskDetailView.setPresenter(this);
   }
```
其中GetTask CompleteTask ActivateTask  DeleteTask 都是继承自UseCase

接下来我们看看deleteTask()方法
```java
mUseCaseHandler.execute(mDeleteTask, new DeleteTask.RequestValues(mTaskId), new UseCase.UseCaseCallback<DeleteTask.ResponseValue>() {
           @Override
           public void onSuccess(DeleteTask.ResponseValue response) {
               mTaskDetailView.showTaskDeleted();
           }

           @Override
           public void onError() {
               // Show error, log, etc.
           }
       });

```
mUseCaseHandler 和mDeleteTask 是初始化的时候传递过来的，至于后面的两个参数

RequestValues 可以理解为传递的时候需要的参数，例如要删除task，必须要知道他对应的mTaskId
UseCaseCallback 就可以理解为 执行后的回调，是成功了，还是失败了啊

然后我们继续看UseCaseHandler.execute()方法
```java
public <T extends UseCase.RequestValues, R extends UseCase.ResponseValue> void execute(final UseCase<T, R> useCase, T values, UseCase.UseCaseCallback<R> callback) {

       useCase.setRequestValues(values);
       useCase.setUseCaseCallback(new UiCallbackWrapper(callback, this));

       // 网络请求可能在不同的线程中处理，因此请确保Espresso知道应用程序正忙，直到处理响应为止。
       //应用程序很忙，直到另行通知
       EspressoIdlingResource.increment();

       mUseCaseScheduler.execute(new Runnable() {
           @Override
           public void run() {

               //UseCaseHandler 中其实还是调用 UseCase 的 run() 方法，而 UseCase 的 executeUseCase() 是抽象方法，最终逻辑是由各个Task子类实现的，比如 DeleteTask：
               useCase.run();
              // 这个回调可以被调用两次，一次用于缓存，一次用于从服务器API加载数据，所以我们在减量之前检查，否则抛出“计数器已经损坏！”的异常
               if (!EspressoIdlingResource.getIdlingResource().isIdleNow()) {
                   EspressoIdlingResource.decrement(); // Set app as idle.
               }
           }
       });
   }

```
这里进行了二次封装，
1. 把传递过来的参数设置到对应的useCase中
2. 设置接口回调 ，这里注意一下UiCallbackWrapper()，虽然它也是一实现了UseCase.UseCaseCallback，但是却还接受一个UseCase.UseCaseCallback 对象，这个就有点意思了，其实看里面的代码就好理解了，
```java
private static final class UiCallbackWrapper<V extends UseCase.ResponseValue> implements UseCase.UseCaseCallback<V> {
       private final UseCase.UseCaseCallback<V> mCallback;
       private final UseCaseHandler mUseCaseHandler;

       public UiCallbackWrapper(UseCase.UseCaseCallback<V> callback, UseCaseHandler useCaseHandler) {
           mCallback = callback;
           mUseCaseHandler = useCaseHandler;
       }

       @Override
       public void onSuccess(V response) {
           mUseCaseHandler.notifyResponse(response, mCallback);
       }

       @Override
       public void onError() {
           mUseCaseHandler.notifyError(mCallback);
       }
   }
```
3. UseCaseScheduler.execute()执行，UseCaseScheduler里面就是一个ThreadPoolExecutor，其实这个就是开启一个线程，执行耗时操作，
4. 要执行那种耗时操作呢，就是useCase.run(），而run()中又直接执行了executeUseCase（）方法，这个executeUseCase 又是一个抽象的方法，所以呢，传递过来的useCase肯定会实现executeUseCase（）的，所以就会执行传递过来的case的executeUseCase()方法，因为我们传递过来的DeleteTask，所以执行到了

```java
   protected void executeUseCase(final RequestValues values) {
       mTasksRepository.deleteTask(values.getTaskId());
       getUseCaseCallback().onSuccess(new ResponseValue());
   }

```
首先这个方法是在线程里面做的，第一行代码就是根据传递过来的id删除相应的Task,
第二行代码有点意思，getUseCaseCallback() 就是在UseCaseHandler.execute()中设置的useCase.setUseCaseCallback(new UiCallbackWrapper(callback, this)); 设置进去的，也就是UiCallbackWrapper对象，而他的onSuccess()里面执行了mUseCaseHandler.notifyResponse(response, mCallback);
然后又到了UseCaseHandler.notifyResponse()方法中,
```java
public <V extends UseCase.ResponseValue> void notifyResponse(final V response, final UseCase.UseCaseCallback<V> useCaseCallback) {
       mUseCaseScheduler.notifyResponse(response, useCaseCallback);
   }
```
useCaseCallback 这个才是我们刚开始调用的时候设置进去的，然后执行到了UseCaseScheduler。java中的notifyResponse
```java
//UseCaseScheduler。java
@Override
public <V extends UseCase.ResponseValue> void notifyResponse(final V response, final UseCase.UseCaseCallback<V> useCaseCallback) {
    // response使用Handler来处理结果回调
    mHandler.post(new Runnable() {
        @Override
        public void run() {
            useCaseCallback.onSuccess(response);
        }
    });
}
```
看到了熟悉的地方，通过Handler把处理结果回调。最后就执行到了   mTaskDetailView.showTaskDeleted(); 方法


接下来就看

https://www.jianshu.com/p/7ae3095f2cb5   
https://www.jianshu.com/p/e0258ce7d392
