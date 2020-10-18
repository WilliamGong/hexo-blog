---
title: “O泡果奶”app解析1 apk文件逆向
date: 2020-10-15T09:47:30+08:00
toc: true
cover: /images/cover-obubbleapp.jpg
thumbnail: /images/cover-obubbleapp.jpg
categories: 
    - Android
tags: 
    - Android
    - 逆向工程
    - 杂谈
---
当比对完hash后，接下来就是对整个apk进行逆向了。    
首先我们对“一份礼物.apk”进行逆向
# 需要的工具
+ [Jadx](https://github.com/skylot/jadx)
# 分析apk文件结构
apk本质上是一个加了签名和元数据的压缩包，用普通的解压工具解压即可得到内部的文件。    
内部的文件结构如下所示：
```
.
├── AndroidManifest.xml
├── META-INF
├── assets
├── classes.dex
├── com
├── lib
├── lua
└── res
```
再看看/assets下的文件：
```
.
├── icon.png
├── init.lua
├── layout.lua
├── main.lua
└── mc.mp3
```
`mc.mp3`就是O泡果奶的广告音频。

# 查看app信息
我们打AndroidManifest.xml，查看apk包名等信息。    
AndroidManifest.xml文件如下:
```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.lc.nb" android:versionCode="9" android:versionName="凉城fork by Keven">
    <uses-sdk android:minSdkVersion="21" android:targetSdkVersion="21"/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <uses-permission android:name=""/>
    <application android:label="一份礼物" android:icon="@drawable/icon" android:name="com.androlua.LuaApplication" android:persistent="true" android:largeHeap="true" android:resizeableActivity="true" android:supportsPictureInPicture="true">
        <meta-data android:name="android.max_aspect" android:value="4"/>
        <activity android:theme="@style/Theme.Holo.Light.NoActionBar" android:label="插件9.0" android:name="com.androlua.Main" android:screenOrientation="user" android:configChanges="keyboardHidden|orientation|screenSize">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <action android:name="android.intent.action.EDIT"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="file"/>
                <data android:host="*"/>
                <data android:pathPattern=".*\\.alp"/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <action android:name="android.intent.action.EDIT"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="content"/>
                <data android:host="*"/>
                <data android:pathPattern=".*\\.alp"/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <action android:name="android.intent.action.EDIT"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="file"/>
                <data android:mimeType="application/*"/>
                <data android:mimeType="audio/*"/>
                <data android:mimeType="video/*"/>
                <data android:mimeType="text/*"/>
                <data android:mimeType="*/*"/>
                <data android:host="*"/>
                <data android:pathPattern=".*\\.alp"/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <action android:name="android.intent.action.EDIT"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="content"/>
                <data android:host="*"/>
                <data android:mimeType="application/*"/>
                <data android:mimeType="audio/*"/>
                <data android:mimeType="video/*"/>
                <data android:mimeType="text/*"/>
                <data android:mimeType="*/*"/>
                <data android:pathPattern=".*\\.alp"/>
            </intent-filter>
        </activity>
        <activity android:theme="@style/Theme.Translucent.NoTitleBar" android:name="com.tencent.connect.common.AssistActivity" android:screenOrientation="behind" android:configChanges="keyboardHidden|orientation"/>
        <activity android:name="com.tencent.tauth.AuthActivity" android:launchMode="singleTask" android:noHistory="true">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="tencent222222"/>
            </intent-filter>
        </activity>
        <activity android:theme="@style/Theme.Holo.Light.NoActionBar" android:label="插件9.0" android:name="com.androlua.LuaActivity" android:configChanges="keyboardHidden|orientation|screenSize">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme=""/>
                <data android:host="com.andlua.ly"/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <action android:name="android.intent.action.EDIT"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="file"/>
                <data android:host="*"/>
                <data android:pathPattern=""/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <action android:name="android.intent.action.EDIT"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="content"/>
                <data android:host="*"/>
                <data android:pathPattern=""/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="file"/>
                <data android:mimeType="text/*"/>
                <data android:host="*"/>
                <data android:pathPattern=""/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <category android:name="android.intent.category.BROWSABLE"/>
                <data android:scheme="content"/>
                <data android:mimeType="text/*"/>
                <data android:host="*"/>
                <data android:pathPattern=""/>
            </intent-filter>
        </activity>
        <activity android:theme="@style/Theme.Holo.Light.NoActionBar" android:label="一份礼物" android:name="com.androlua.LuaActivityX" android:excludeFromRecents="false" android:screenOrientation="portrait" android:configChanges="keyboardHidden|orientation|screenSize" android:documentLaunchMode="intoExisting"/>
        <activity android:theme="@style/Theme.NoDisplay" android:label="一份礼物" android:name="com.androlua.Welcome" android:screenOrientation="portrait">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <activity android:theme="@style/Theme.Translucent.NoTitleBar" android:name="com.branch.www.screencapture.ScreenCaptureActivity"/>
        <service android:name="com.androlua.LuaService" android:enabled="true"/>
        <service android:label="一份礼物" android:name="com.androlua.LuaAccessibilityService" android:permission="" android:enabled="true" android:exported="true">
            <meta-data android:name="android.accessibilityservice" android:resource="@xml/accessibility_service_config"/>
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService"/>
                <category android:name="android.accessibilityservice.category.FEEDBACK_AUDIBLE"/>
                <category android:name="android.accessibilityservice.category.FEEDBACK_HAPTIC"/>
                <category android:name="android.accessibilityservice.category.FEEDBACK_SPOKEN"/>
            </intent-filter>
        </service>
        <provider android:name="android.content.FileProvider" android:exported="false" android:authorities="com.lc.nb" android:grantUriPermissions="true">
            <meta-data android:name="android.support.FILE_PROVIDER_PATHS" android:resource="@anim/abc_fade_out"/>
        </provider>
    </application>
</manifest>
```
可以看到作者把ID留在了versionName里，心够大的。     
以及可以看到申请的权限只有储存权限，这样看来估计没有窃取信息等行为了。      
不过

    package="com.lc.nb"
凉城NB！（个鬼）

# 使用Jadx对整个apk进行逆向
下载好Jadx后，用它打开这个apk![jadx](jadx.jpg)    
看，一键反编译，自动反混淆！    
文件里有很多第三方包，看上去很可疑。

# 定位入口文件
注意AndroidManifest.xml中的这一个标签：

    <activity 
    android:theme="@style/Theme.Holo.Light.NoActionBar"
    android:label="插件9.0" 
    android:name="com.androlua.Main"
    android:screenOrientation="user"
    android:configChanges="keyboardHidden|orientation|screenSize">
可见入口文件在`com.android.Main`中。
那么打开看看：![Jadx-main](jadx-main.jpg)
确定了，这是AndroidLua应用，而`/asset`中的lua脚本才是本体。    
OK，先到这里，下次我们重点解析作为本体的lua文件。