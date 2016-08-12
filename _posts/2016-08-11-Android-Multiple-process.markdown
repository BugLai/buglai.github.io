---
layout: post
title: Android 多进程浅析
tags: [android] 
categories: [Android]
---

# Android多进程介绍

    Android在默认情况下所有组件都是运行在同一个进程中的，当然也允许同一个应用运行在多个进程中，只不过在日常开发中基本用不到所以对于这方面的知识有点陌生。一般为了提高应用的性能和运行效率，像uc，facebook等业务复杂的大型app一般都会采用多进程的方式。那么接下来就介绍一下Android多进程的相关知识，在什么情况下可以使用多进程以及需要注意的事项
    
    一个进程都有独立的资源和内存空间，其他进程不能任意访问当前进程的内存和资源，并且Android系统对于进程的内存分配也有限制，早期的一些版本可能是16MB，现在不同的设备有不同的大小，在进程的内存占用超过这个限制还会报OOM异常，Android在API11,可以通过给AndroidManifest.xml的Application标签增加“android:largeHeap=”true“” 来请求分配更多内存

{% highlight ruby %}
  <application
        android:name="com.internal.system.App"
        android:allowBackup="true"
        android:persistent="true"
        android:largeHeap="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme" >
{% endhighlight %}
   
   不过这样做也不能很好解决问题，当API小于11,或者应用需要的内存比largeHeap更大，系统总的内存不变的情况下，每个应用都申请更多的内存，会影响到设备的运行效率，为了解决应用内存问题，Android又引入了多进程的概念，下面简单概括一下多进程

## 1.多进程用途
      一般把组件设置成多进程目的是能够获得更多的空间处理资源，比如你可以把负责音乐播放的服务，负责视频播放的Activity，负责后台检测推送或者更新的服务等放在一个独立的进程中进行，这样就可以在系统杀死你的主进程是，另一个进程可能还存在着，这里举几个例子，比如Facebook新开命名为com.facebook.katana:videoplayer的进程负责app视频播放的模块，UC会开多一个com.UCMobile.intl:push进程负责推送模块等

## 2.开启多进程模式
      Android提供了一个属性android:process，我们可以在四大组件activity，service，content provider和broadcast receiver中添加这个属性来指定组件该运行在那个进程中，如下所示，我们指定了一个service和一个receiver运行在process这个进程中，这样当启动这个两个组件的时候，它们就会单独的运行在一个进程中，不会占用主进程的内存,当然除了Android提供这种正规方式，其实我们也可以自己通过JNI在native层去fork一个新的进程，不过这种情况比较特殊，之前做进程守护的时候就用过，在native层fork一个子进程进行轮询，这里就不做过多讨论。下面两个列举了设置多进程属性时的两种方式，第一种前面带“:”，它的完整进程名就是“packagename：process”，而第二种命名方式不会附加包名信息，进程名直接是”com.process“。
      其实进程名以”：“开头的进程属于当前应用的私有进程，其他应用的组件不能和它跑在同一个进程中，而后一种进程命名属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中，Android为每个应用都分配一个唯一的UID，具有相同UID的应用才能共享数据，假如两个应用通过shareUID跑在同一个进程中，需要两个应用的shareUID和签名相同。在这种条件下，两者可以相互访问data目录的私有数据，两者看起来像是一个应用的两部分。

{% highlight ruby %}
        <receiver android:name="com.rescue.Singular" android:process=":process"/>
        <service android:name="com.tribute.Vision" android:process="com.process"/>
{% endhighlight %}

 

## 3.多进程的运行机制
      对于不同的进程，有时候虽然载入的class文件名相同，但是他们其实加载的是不同的内存地址空间，所以获取到的相关变量是不同的，这就是进程间内存的不可见性，如下代码所示：
 
 

{% highlight ruby %}

public class MainActivity extends Activity {
   
    final static String TAG = "MultiProcessTest";
    static boolean iscreated=false;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        iscreated=true;
        Log.i(TAG,"on create() in MainActivity "+iscreated);
        startService(new Intent(this,MainService.class));
    }
  }

   public class MainService extends Service{
    String TAG = "MultiProcessTest";
    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(TAG,"MainService is onCreate");
    }
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.i(TAG,"MainActivity is created"+MainActivity.iscreated);
        return START_STICKY;
    }
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
  }

     <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
     </activity>

     <service android:name=".MainService"
              android:process=":MultiProcess"/>
{% endhighlight %}


   运行上面代码，假如在service中不添加process属性，那么我们打印出来的结果是：
   
   
{% highlight ruby %}

08-11 17:20:28.244 7361-7361/com.android.multiprocess I/MultiProcessTest: on create() in MainActivity :true
08-11 17:20:28.280 7361-7361/com.android.multiprocess I/MultiProcessTest: MainService is onCreate
08-11 17:20:28.280 7361-7361/com.android.multiprocess I/MultiProcessTest: MainActivity is created:true

{% endhighlight %}


   假如不在service中添加process属性，那么我们打印出来的结果是：
   
{% highlight ruby %}

08-11 17:18:38.819 5215-5215/com.android.multiprocess I/MultiProcessTest: on create() in MainActivity :true
08-11 17:18:38.993 5286-5286/com.android.multiprocess:MultiProcess I/MultiProcessTest: MainService is onCreate
08-11 17:18:39.006 5286-5286/com.android.multiprocess:MultiProcess I/MultiProcessTest: MainActivity is created:false
{% endhighlight %}

   从上面运行过程中看，我们在启动MainActivity（com.android.multiprocess进程）中的onCreate方法中设置一个布尔变量为true，顺便启动一个添加了process属性的MainService服务（com.android.multiprocess:MultiProcess进程），并在服务启动起来后打印MainActivity的这个变量，发现变量还是为默认值，在不设置process属性时却是改变了，这是因为系统为每个进程都分配了一个独立的虚拟机，不同虚拟机的内存分配上有不同的地址空间，这就导致在不同的虚拟机中访问同一个类的对象会产生对份副本


## 4.多进程的创建顺序
   
   上面我们已经开始使用了多进程，那么对于Android系统来说，是怎么创建一个进程的呢，在一个应用多进程下是如何创建进程的，下面我们通过代码来查看一下，我们在Manifest声明两个Service，其中一个在进程名为ServiceA中，那么现在我们在Application中启动这两个service，通过输出log，我们看到应用首先实例化一个PID为18939的进程，当执行启动ServiceA时，由于ServiceA定义为一个新的进程，系统会首先初始化该进程，此时ServiceA处于线程阻塞中，这样直接跳过ServiceA执行了ServiceB，新的进程实例化出来之后，在Application的Oncreate中获取到PID为19113的进程，这时ServiceA会执行两次，一次是进程PID为18939的，一次是进程PID19113的，然后进程PID19113再执行ServiceB一次，结论就是Android在遇到需要放入进程的组件时，会先创建此进程，此时该组件是线程阻塞的，直到新进程创建完。通过分析也可以知道进程的创建是会影响Application的实例。

{% highlight ruby %}

    <service android:name=".ServiceA" android:process=":ServiceA"/>
    <service android:name=".ServiceB"/>

    public class App extends Application  {
    final static String TAG = "MultiProcessTest";
    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(TAG,"App onCreate pid:"+android.os.Process.myPid());
        startService(new Intent(this,ServiceA.class));
        startService(new Intent(this,ServiceB.class));
    }
  }

08-11 19:34:54.281: I/MultiProcessTest(18939): App onCreate pid:18939
08-11 19:34:54.354: I/MultiProcessTest(18939): ServiceB is onStart
08-11 19:34:54.404: I/MultiProcessTest(19113): App onCreate pid:19113
08-11 19:34:54.409: I/MultiProcessTest(19113): ServiceA is onStart
08-11 19:34:54.410: I/MultiProcessTest(19113): ServiceA is onStart
08-11 19:34:54.412: I/MultiProcessTest(18939): ServiceB is onStart

{% endhighlight %}

 
 
## 5.多进程Application注意事项
    
  通过上面的分析我们知道Application会根据进程的多少进行多次实例化，假如我们把很多初始化的工作放在Application中进行，这时就要特别主要进行进程间的区分，哪部分逻辑执行在主进程，哪部分逻辑执行在其他的进程，这样才不会出现一些不必要的重复执行和不可预知的bug，那么怎么进行区分，通过下面两种方式都能通过传入当前的pid来取得进程名进行区别，默认是进程名是包名，其他的进程名为包名在process属性的value，其中第一种获取方式是通过API正常的获取，一般比较耗时，第二种是直接读取系统文件会比较快，两种可以互相补充，像微信是用第一种获取，加入获取不到再用第二种获取、
  
{% highlight ruby %}
    if (getProcessName(android.os.Process.myPid()).equals(context.getPackageName())) {
     //判断进程名，保证只有主进程运行,主进程初始化逻辑

     }else if(getProcessName(android.os.Process.myPid()).equals(“com.android.multiprocess:ServiceA”)){
     //假如在com.android.multiprocess:ServiceA进程中，进行相应的操作

     }


   private static String getProcessName(Context context, int pid) {
        ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        List<ActivityManager.RunningAppProcessInfo> runningApps = am.getRunningAppProcesses();
        if (runningApps == null) {
            return null;
        }
        for (ActivityManager.RunningAppProcessInfo procInfo : runningApps) {
            if (procInfo.pid == pid) {
                return procInfo.processName;
            }
        }
        return null;
    }


    private static String getProcessName(int pid) {
        try {
            File file = new File("/proc/" + pid + "/" + "cmdline");
            BufferedReader mBufferedReader = new BufferedReader(new FileReader(file));
            String processName = mBufferedReader.readLine().trim();
            mBufferedReader.close();
            return processName;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
{% endhighlight %}

    

## 6.多进程造成的问题

    通过上面的一些分析以及查到的一些资料显示，多进程会造成几个方面的问题，其中SharePreferences在多进程读写下会导致一定概率的数据丢失，因为可能出现并发的情况。
    (1)静态成员和单例模式失效 
    (2)线程同步机制失效 
    (3)SharePreferences的可靠性下降
    (4)Application多次创建


## 7.跨进程通信

         跨进程通信其实是有很方法实现的，比如通过Demo测试在Intent中传递数据给另一个进程的组件，是可以实现的。在确保不会出现并发情况下使用SharePrefrences也可以，还能基于
Binder的Messenger和AIDL以及Socket等。这里举一个最简单的例子测试一下，我们通过Intent中附加extras从主进程中MainActivity组件传递信息给另一个进程的MainService，通过下面
输出的日志可以看出通过Intent是可以进行进程间通信的。

{% highlight ruby %}

        Intent intent = new Intent(this,MainService.class);
        intent.putExtra(TAG,"I AM FROM MainActivity");
        startService(intent); Intent intent = new Intent(this,MainService.class);
        intent.putExtra(TAG,"I AM FROM MainActivity");
        startService(intent);

      
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        String value = intent.getStringExtra(TAG);
        Log.i(TAG,"MainService is onStartCommand:"+value);
        return START_STICKY;
    }


08-11 22:02:05.522 19681-19681/com.android.multiprocess:mainService I/MultiProcessTest: MainService is onStartCommand:I AM FROM MainActivity
{% endhighlight %}

  




  




    













