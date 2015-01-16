---
layout: post
title: “Android APK 加壳原理浅析”
tags: [android]
categories: [Android]
---

最近在研究android加壳,查了一些资料,经过理论上的认识和实践,初步实现apk加壳,以下作个总结:
准备工具: apk(随便找个可以反编译的) 反编译工具(apktool...) sdk(开发环境的sdk)  eclipse/android studio

原理:解压apk取出dex加密，自定义Application类,在进入应用时获取到控制权进行dex解密并动态加载到内存，加载完由原来的apk接管控制权.（原apk里面包含动态加载以及静态so/动态so的功能也能够成功加壳）

测试apk：免费红利/试货

关键字：动态加载 

缺陷：目前运行apk完全没有问题，但是出现图片模糊，字体大小改变的情况，分辨率很模糊，希望读者一起找原因
<img src="/image/shell_one.png" />
<img src="/image/shell_two.png" />

流程：
1.反编译apk获取res资源文件和Manifest.xml配置文件，新建工程把res资源文件和Manifest.xml
2.apk其实就是一个压缩包，解压取出classes.dex文件复制到sdk/build-tools/xxx路径下，比如sdk/build-tools/android-4.4.2，里面有自带的dx工具
执行命令：bash dx --dex --output=classdex.jar classes.dex，会将classes.dex压缩成classdex.jar（本质上还是dex文件，利用这个命令可以把真正的jar包转换为dex文件形式进行动态加载)，复制classdex.jar到assets目录
3.解压的apk中加入有libs/lib文件，里面有so文件，请复制到新建工程
4查看修改manifest文件中的application,注意请不要改动其他地方，避免新的apk加载出错，下面试货的配置文件

修改前：
{% highlight ruby %}
  <application android:theme="@style/AppTheme"
           android:hardwareAccelerated="true"
         android:label="@string/app_name" 
        android:icon="@drawable/ic_launcher"
         android:name="com.android.shihuo.MyApplication" 
         android:allowBackup="true">
     .......
   </application>
{% endhighlight %}


修改后：
{% highlight ruby %}
  <application android:theme="@style/AppTheme"
           android:hardwareAccelerated="true"
         android:label="@string/app_name" 
        android:icon="@drawable/ic_launcher"
         android:name="com.shell.ProxyApplication" 
         android:allowBackup="true">
             <meta-data android:name="APPLICATION_CLASS_NAME" android:value="com.android.shihuo.MyApplication"/>
     .......
   </application>
{% endhighlight %}

上面由于试货有自己的Application，所以需要配置如下标签，假如apk中的application没有自定义application则不用下面的标签
{% highlight ruby %}
<meta-data android:name="APPLICATION_CLASS_NAME"android:value="com.android.shihuo.MyApplication"/>
{% endhighlight %}


5定义代理的Application，由于应用进入时首先执行ProxyApplication，要获取到最高控制权，由于优先级问题（优先级BaseContext>contentprovider>applicationcontext）,所以在BaseContext里面可以利用ConfigApp.configLoader(base)方法进行classdex.jar的动态加载;然后在ProxyApplication中onCreate方法用ConfigApp.configApplication(this)判断原apk是否有<meta-data android:name="APPLICATION_CLASS_NAME" android:value=""/>，有的话将代理
权交个原来的Application负责（example："com.android.shihuo.MyApplication"），至此整个流程跑完
{% highlight ruby %}
public class ProxyApplication extends Application {
	
	protected void attachBaseContext(Context base) {
		super.attachBaseContext(base);	
             //理论上可以对classdex.jar进行加密，然后在这里进行解密后再执行下面方法把解密的dex load进内存（还没实践）	
		ConfigApp.configLoader(base);	
	}


	@Override
	public void onCreate() {
		Application app = ConfigApp.configApplication(this);
		if (app==null) {
			return;
		}else{
			 app.onCreate();  
		}    
	}

}
{% endhighlight %}



此方法是把assets文件夹下的classdex.jar load进内存，测试红利的时候没有问题，在测试试货时崩溃，原因是试货里面有so文件，导致没有加载进去，后来在DexClassLoader类中找到一个传so路径的参数，传入后可以正常调用so
{% highlight ruby %}
public static void configLoader(Context base){
		File file = new File("/data/data/" + base.getPackageName() + "/.cache/");
		if (!file.exists()) {
			file.mkdirs();
		}
		Util.copyJarFile(base);
		try {
			Object packageInfo = RefInvoke.getField(base, "mPackageInfo");
			CustomClassLoader dLoader = new CustomClassLoader("/data/data/"+ base.getPackageName() + "/.cache/classdex.jar", 
					"/data/data/"+ base.getPackageName() + "/.cache",
					"/data/data/"+ base.getPackageName() + "/lib/", 
					base.getClassLoader());
			RefInvoke.setFieldOjbect("android.app.LoadedApk", "mClassLoader",packageInfo, dLoader);
		} catch (NoSuchFieldException e) {
			e.printStackTrace();
		}
	}
{% endhighlight %}

此方法用来将控制权交还给原来的apk的application进行运作
{% highlight ruby %}

public static Application configApplication(ProxyApplication proxyApplication){				// 如果源应用配置有Appliction对象，则替换为源应用Applicaiton，以便不影响源程序逻辑。          String appClassName = null;          try {              ApplicationInfo ai = proxyApplication.getPackageManager().getApplicationInfo(proxyApplication.getPackageName(),  PackageManager.GET_META_DATA);              Bundle bundle = ai.metaData;              if (bundle != null  && bundle.containsKey("APPLICATION_CLASS_NAME")) {                  appClassName = bundle.getString("APPLICATION_CLASS_NAME");              } else {                  return null;              }          } catch (NameNotFoundException e) {              e.printStackTrace();          }                 Object currentActivityThread = RefInvoke.invokeStaticMethod(  "android.app.ActivityThread", "currentActivityThread",new Class[] {}, new Object[] {});          Object mBoundApplication = RefInvoke.getFieldOjbect(  "android.app.ActivityThread", currentActivityThread,  "mBoundApplication");          Object loadedApkInfo = RefInvoke.getFieldOjbect("android.app.ActivityThread$AppBindData", mBoundApplication, "info");          RefInvoke.setFieldOjbect("android.app.LoadedApk", "mApplication", loadedApkInfo, null);          Object oldApplication = RefInvoke.getFieldOjbect("android.app.ActivityThread", currentActivityThread,"mInitialApplication");          ArrayList<Application> mAllApplications = (ArrayList<Application>) RefInvoke.getFieldOjbect("android.app.ActivityThread",currentActivityThread, "mAllApplications");          mAllApplications.remove(oldApplication);          ApplicationInfo appinfo_In_LoadedApk = (ApplicationInfo) RefInvoke.getFieldOjbect("android.app.LoadedApk", loadedApkInfo,"mApplicationInfo");          ApplicationInfo appinfo_In_AppBindData = (ApplicationInfo) RefInvoke.getFieldOjbect("android.app.ActivityThread$AppBindData",mBoundApplication, "appInfo");          appinfo_In_LoadedApk.className = appClassName;          appinfo_In_AppBindData.className = appClassName;          Application app = (Application) RefInvoke.invokeMethod( "android.app.LoadedApk", "makeApplication", loadedApkInfo, new Class[] { boolean.class, Instrumentation.class },  new Object[] { false, null });          RefInvoke.setFieldOjbect("android.app.ActivityThread",  "mInitialApplication", currentActivityThread, app);  		return app;	}



{% endhighlight %}


总结：目前已经能够顺利执行整个流程，对加壳后的apk进行反编译后查看源码，原apk的smali代码只有寥寥的几个R文件，完全没有了smali代码，但是加壳后的apk失真太严重，整个画面模糊，目前还不清楚原因。
测试工程的链接地址：













