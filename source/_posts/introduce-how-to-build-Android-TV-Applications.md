---
title: 利用Leanback实现安卓TV应用教程一
date: 2018-08-06 21:32:25
tags:
- Android
- Leanback
---

# 前言

12年实习去了一家机顶盒公司，时隔5年后再次进入智能机顶盒研发的公司，也许这就是一种宿命吧！记得12年安卓还是起步阶段，当时能够在机顶盒上面研发应用没有什么参考资料，就当成不能触摸的手机开发而已；现在就不同了，安卓已经非常的成熟。Google为安卓TV提供了专门的SDK，针对TV应用开发者提供了Leanback库。

但是国内对于Leanback库的介绍和资料还是挺少的，其实我个人觉得也很正常了。

- Leanback上手很麻烦；
- Leanback很难定制化；

但是不管咋样，自己毕竟是做TV开发的，Leanback还是要掌握的，从今天起开始给给大家来个如何利用Leanback开发TV应用的系列教程，希望大家喜欢！

<!-- more -->

# 正题

UI及操作是手机应用和TV应用最大的不同之一，比如我们是通过遥控器远程操控的，而不是通过屏幕触摸。为了实现这些要求，Android开源项目Leanback应运而生，它能够让开发者很轻松的开发满足TV开发的各种要求的应用。

谁适合阅读这篇文章：

- 没有安卓TV开发经验的安卓开发者；
- 初学者

因为eclipse于2015年之后便不会在支持，Android Studio将被用来作为AndroidTV开发的IDE开发工具，因此我们下面的内容都是利用Android Studio进行介绍的。好了，让我们开始吧！

## 新建新的TV应用项目

打开Android Studio，选择File->New->New Project新建项目，如下图：

![create_project](create_project.png)

| 名称                   | 含义                   |
| ---------------------- | ---------------------- |
| Company domain         | 公司域名               |
| Project location       | 项目保存地址           |
| Package name           | 应用包名               |
| Include C++ support    | 是否包含C++支持        |
| Include Kotlin support | 是否包含Kotlin语言支持 |
| Application name       | 应用名称               |

填入以上信息后，点击Next进入下一步。如下图：

![target_android_devices](target_android_devices.png)

默认是选中“Phone and Tablet”的，因为是TV开发，所以我们选中TV就好了，然后Next，如下图：

![create_default_activity](create_default_activity.png)

默认是选中“Android TV Activity”的，但是如果选择这个选项的话会生成很多文件，对于初学者来说上手很难，因此我们这里选择“Add No Activity”，然后点击Finish，这样项目就新建好了。

## 创建Activity

项目新建完成后，因为我们选择的是“Add No Activity”，所以我们的项目默认是没有Activity，也就是没有界面的。这个时候就需要我们自己添加Activity了。

右键选中我们的包名，比如“com.example.tvdemo”，然后选择“New”->Activity->Empty Activity，选中“Launcher Activity”选项，将它作为我们应用的启动页。

![empty_activity.png](empty_activity.png)

![create_empty_activity.png](create_empty_activity.png)

*注意：默认情况下MainActivity是继承Activity的，这里我们要改成FragmentActivity，因为等下要用到Fragment。*

## 创建Fragment

右键包名，New->Java Class->Name:MainFragment。MainFragment需要继承BrowseSupportFragment。

![create_main_fragment.png](create_main_fragment.png)

现在，我们有了MAinActivity和MAinFragment，怎么将两者关联在一起呢？其实只需要我们修改activity_main文件就好了，具体如下：

```
<?xml version="1.0" encoding="utf-8"?>
<fragment xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:name="site.yhaojun.tvdemo.fragment.MainFragment"
    android:id="@+id/main_fragment"
    android:layout_height="match_parent"
    tools:context=".MainActivity"/>
```

这样的话MainFragment就与Activity进行关联了。

**问：为什么MainFragment需要继承BroseSupportFragment呢？**

因为BroseSupportFragment是由Leanback库提供的类，专门用来进行TV应用开发的库，封装了很多适用于TV的功能和接口。

运行项目，效果如下：

![first_show_result.png](first_show_result.png)

正如上图所展示的，BroseSupportFragment是由HeaderFragment和RowsFragment组成的，RowsFragment是左边区域，HeaderFragment是右边区域，后续我们要介绍如何设计Header和Row。

## 添加标题

在MainFragment中新增setupUIElements方法，用来设置应用程序相关信息。

```
 @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        Log.i(TAG, "onActivityCreated");
        super.onActivityCreated(savedInstanceState);

        setupUIElements();
        
    }

    private void setupUIElements() {
        // setBadgeDrawable(getActivity().getResources().getDrawable(R.drawable.videos_by_google_banner));
        setTitle("Hello Android TV!"); // Badge, when set, takes precedent
        // over title
        setHeadersState(HEADERS_ENABLED);
        setHeadersTransitionOnBackEnabled(true);

        // set fastLane (or headers) background color
        setBrandColor(getResources().getColor(R.color.fastlane_background));
        // set search icon color
        setSearchAffordanceColor(getResources().getColor(R.color.search_opaque));
    }
```

我们做了两个操作：

1. 设置应用的标题或图标
2. 设置左边栏的背景色

其中这里定义了两个颜色fastlane_background和search_opaque，那如何增加颜色呢？首先在values目录下新建colors.xml文件，然后定义两个颜色值。定义如下：

```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="fastlane_background">#0096e6</color>
    <color name="search_opaque">#ffaa3f</color>
</resources>
```

运行看下效果吧！见下图：

![second_result.png](second_result.png)

![third_result.png](third_result.png)

| 方法             | 含义     |
| ---------------- | -------- |
| setBadgeDrawable | 设置图标 |
| setTitle         | 设置标题 |

*注意：设置图标和设置标题只能存在一个。*

上面虽然设置了“搜索”安卓的颜色，但是没有显示该按钮，后来我尝试添加搜索按钮的点击事件，发现按钮就出现了。具体代码如下：

```
    private void setupUIElements() {
         setBadgeDrawable(getActivity().getResources().getDrawable(R.drawable.app_icon_quantum));
//        setTitle("Hello Android TV!"); // Badge, when set, takes precedent
        // over title
        setHeadersState(HEADERS_ENABLED);
        setHeadersTransitionOnBackEnabled(true);

        // set fastLane (or headers) background color
        setBrandColor(getResources().getColor(R.color.fastlane_background));
        // set search icon color
        setSearchAffordanceColor(getResources().getColor(R.color.search_opaque));
        setOnSearchClickedListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Toast.makeText(getActivity(), "search", Toast.LENGTH_SHORT).show();
            }
        });
    }
```

效果如下：

![forth_result.png](forth_result.png)

## 修改AndroidManifest

我用的是模拟器，所以主页是TV Launcher，我发现在主页上并没有显示我的应用启动图标，主要是因为我们还需要修改AndroidManifest.xml文件。

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.corochann.androidtvapptutorial" >


    <!-- TV app need to declare touchscreen not required -->
    <uses-feature
        android:name="android.hardware.touchscreen"
        android:required="false" />

    <!--
     true:  your app runs on only TV
     false: your app runs on phone and TV -->
    <uses-feature
        android:name="android.software.leanback"
        android:required="true" />


    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/Theme.Leanback" >
        <activity
            android:name=".MainActivity"
            android:icon="@drawable/app_icon_your_company"
            android:label="@string/app_name"
            android:logo="@drawable/app_icon_your_company" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

前面两个uses-feature表示该应用不要求触摸屏且只能运行在TV上；对于启动页MainActivity我们也定义了logo和icon，这样的话它和我们的应用（application中的icon）图标可以不一致。在设置的应用列表中显示的是application的ic_launcher，而桌面上的图标显示的是app_icon_your_company；要在主页显示我们的logo，需要添加LEANBACK_LUANCHER。

```
 <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
```

最终效果：

![fifth_result.png](fifth_result.png)

![sixth_result.png](sixth_result.png)

今天就介绍到这里吧，都是些入门知识，后面会继续介绍leanback其他功能的使用。

**这里我有个疑问就是TvDemo的logo为什么不是铺满整个Item呢？如果有人知道，麻烦给我留言，谢谢！！**