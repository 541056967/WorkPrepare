# Android面经
## 1. Android的四大组件
* Activity： 用户与程序进行交互的界面
* Service：后台长时间耗时的服务
* Broadcast Receiver： 接收一个或多个Intent作触发事件
* Content Provider： 提供的第三方应用程序的访问方案

## 2. 常用的布局
* FrameLayout
* LinearLayout
* RelativeLayout
* AbsoluteLayout
* TableLayout

## 3. Activity的生命周期
> onCreate()、onStart()、onReStart()、onResume()、onPause()、onStop()、onDestory()；
* 可见生命周期：从onStart()直到系统调用onStop()
* 前台生命周期：从onResume()直到系统调用onPause()

## 4. activity在屏幕旋转时的生命周期
> 不设置Activity的android:configChanges时，切屏会重新调用各个生命周期，切横屏时会执行一次，切竖屏时会执行两次；设置Activity的android:configChanges="orientation"时，切屏还是会重新调用各个生命周期，切横、竖屏时只会执行一次；设置Activity的android:configChanges="orientation|keyboardHidden"时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged方法

## 5. Service的生命周期图
![service生命周期](https://www.runoob.com/wp-content/uploads/2015/08/11165797.jpg)

## 6. 使用Service的方式
> AndroidManifest.xml完成Service注册
* StartService()启动Service
  * 首次启动会创建一个Service实例,依次调用onCreate()和onStartCommand()方法,此时Service 进入运行状态,如果再次调用StartService启动Service,将不会再创建新的Service对象, 系统会直接复用前面创建的Service对象,调用它的onStartCommand()方法！
  * 但这样的Service与它的调用者无必然的联系,就是说当调用者结束了自己的生命周期, 但是只要不调用stopService,那么Service还是会继续运行的!
  * 无论启动了多少次Service,只需调用一次StopService即可停掉Service
* BindService()启动Service
* 启动Service后绑定Service
  * 如果Service已经由某个客户端通过StartService()启动,接下来由其他客户端 再调用bindService(）绑定到该Service后调用unbindService()解除绑定最后在 调用bindService()绑定到Service的话,此时所触发的生命周期方法如下:onCreate( )->onStartCommand( )->onBind( )->onUnbind( )->onRebind( ) 
  * 使用bindService来绑定一个启动的Service,注意是已经启动的Service!!! 系统只是将Service的内部IBinder对象传递给Activity,并不会将Service的生命周期 与Activity绑定,因此调用unBindService( )方法取消绑定时,Service也不会被销毁！

## 7. ANR是什么？怎么避免？
> Application Not Responding.
* 活动管理器和窗口管理器这两个系统服务负责监视应用程序的响应，当用户操作的在5s内应用程序没能做出反应，BroadcastReceiver在10秒内没有执行完毕，就会出现应用程序无响应对话框，这既是ANR。
* 避免方法：Activity应该在它的关键生命周期方法（如onCreate()和onResume()）里尽可能少的去做创建操作。潜在的耗时操作，例如网络或数据库操作，或者高耗时的计算如改变位图尺寸，应该在子线程里（或者异步方式）来完成。主线程应该为子线程提供一个Handler，以便完成时能够提交给主线程。

## 8. Handler
> Android为了线程安全，并不允许我们在UI线程外操作UI；很多时候我们做界面刷新都需要通过Handler来通知UI组件更新
> Handler的执行流程图

![执行流程](https://www.runoob.com/wp-content/uploads/2015/07/25345060.jpg)
### 主线程
```java
   final Handler myHandler = new Handler()  
    {  
      @Override  
      //重写handleMessage方法,根据msg中what的值判断是否执行后续操作  
      public void handleMessage(Message msg) {  
        if(msg.what == 0x123)  
           {  
            imgchange.setImageResource(imgids[imgstart++ % 8]);  
           }  
        }  
    }; 

    myHandler.sendEmptyMessage(0x123);  
```

### 子线程
```java
  Looper.prepare();  
  //实现Handler
  Looper.loop();

```

### post与sendMassage的区别
> handler.post和handler.sendMessage本质上是没有区别的，都是发送一个消息到消息队列中，而且消息队列和handler都是依赖于同一个线程的。
```java
 /*post方法解决UI更新，直接在runnable里面完成更新操作，这个任务会被添加到handler所在线程的消息队列中，即主线程的消息队列中*/
 handler_post.post(new Runnable() {
          @Override
          public void run() {
            tv_up.setText(new_str);
          }
        });
```

### HandlerThread和Thread的区别
> HandlerThread继承于Thread，所以它本质就是个Thread。与普通Thread的差别就在于，然后在内部直接实现了Looper的实现，这是Handler消息机制必不可少的。有了自己的looper,可以让我们在自己的线程中分发和处理消息。如果不用HandlerThread的话，需要手动去调用Looper.prepare()和Looper.loop()这些方法。

## 9. Activity 的四种启动模式对比
* standard:默认的启动模式，每启动一个Activity就会在栈顶创建一个新的实例
* singleTop：该模式会判断要启动的Activity实例是否位于栈顶，如果位于栈顶直接复用，否则创建新的实例。如果Activity并未处于栈顶位置，则可能还会创建多个实例
* singleTask：使Activity在整个应用程序中只有一个实例。每次启动Activity时系统首先检查栈中是否存在当前Activity实例，如果存在则直接复用，并把当前Activity之上所有实例全部出栈
* singleInstance：在整个系统里只有一个实例

## 10. onNewIntent()的调用时机
> 在该Activity的实例已经存在于Task和Back stack中(或者通俗的说可以通过按返回键返回到该Activity )时,当使用intent来再次启动该Activity的时候,如果此次启动不创建该Activity的新实例,则系统会调用原有实例的onNewIntent()方法来处理此intent.

## 11. 广播的两种启动方式
### 动态注册
* Context.registerReceiver()方法来注册；
* 自定义一个BroadcastReceiver，在onReceive()方法中完成广播要处理的事务
* 指定IntentFilter，添加不同的Action即可，一定要调用unregisterReceiver让广播取消注册
### 静态注册
* AndroidManifest.xml中通过<receiver> 

## 12. Worker
### WorkManager


## 13. Context
![context](https://upload-images.jianshu.io/upload_images/1187237-1b4c0cd31fd0193f.png?imageMogr2/auto-orient/strip|imageView2/2/w/628/format/webp)
> 与有关应用程序环境的全局信息的接口。 这是一个抽象类，其实现由Android系统提供。 它允许访问特定于应用程序的资源和类，以及向上调用应用程序级操作，例如启动活动、广播和接收意图等。

## 14. EventBus
> 一种用于Android的事件发布-订阅机制，简化了各个组件之间通信的复杂度，避免由于使用广播通信带来的不便

### 四种线程模型
* Posting：默认的，事件处理函数的线程跟发布的在同一个线程
* MAIN：
* BackGround：
* ASYNC

### 发布订阅流程
* 创建一个事件类型，可以是int，可以是String，可以是自定义的
* 在需要订阅的模块中，注册eventBus
```java
    @Override
    protected void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }

    @Override
    protected void onStop() {
        super.onStop();

    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);
    } 
```        
* 注册后创建一个方法来接受消息
```java
@Subscribe(threadMode = ThreadMode.MAIN)
    public void onReceiveMsg(EventMessage message) {
        Log.e(TAG, "onReceiveMsg: " + message.toString());
    }

```
* 在需要发送事件的地方调用post发送
```java
    EventMessage msg = new EventMessage(1,"Hello MainActivity");
    EventBus.getDefault().post(msg);
```

## 15. 使用SQLite
* 创建xxHelper类，继承SQLiteOpenHelper
  * `onCreat()`创建数据库，第一次建表时使用
  * `onUpgrade()`更新数据库表结构，数据库版本发生变化的时候回调，必须传入一个version参数
* 创建xxOperator
  * 获得之前的xxHelper的实例，   SQLiteDatabase db = xxHelper.getWritableDatabase()/getReadableDatabase();
  * 更新可以用contentValues

## 16. Maven 和 Gradle 的区别
> Gradle 的优势：依赖管理、多模块构建、Maven 基于 XML 配置繁琐，阅读性差，Gradle 基于 Groovy，简化了构建代码的行数，易于阅读
* 依赖管理方面：Gradle 支持依赖动态版本管理，解决依赖冲突机制更明确
* 多模块构建方面：Gradle 使用 allprojects 和 subprojects 来定义里面的配置是应用于所有项目还是子项目，更加灵活
* 构建周期方面：Gradle 本身与项目构建周期是解耦的，可以灵活的增删 task