上篇写了gradle导入的方式，这篇讲解命令行接入的方式
首先还是导入包,同时修改下签名的配置
很简单 直接放出源码
```
apply plugin: 'com.android.application'

android {
    signingConfigs {
        release {
            keyAlias 'china'
            keyPassword '123456'
            storeFile file('D:/work/release.jks')
            storePassword '123456'
        }
        debug {
            keyAlias 'china'
            keyPassword '123456'
            storeFile file('D:/work/debug.jks')
            storePassword '123456'
        }
    }
    compileSdkVersion 25
    buildToolsVersion "25.0.1"
    defaultConfig {
        applicationId "com.anlaiye.swt.tinkertest"
        minSdkVersion 14
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            debuggable true
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }

        debug {
            debuggable true
            minifyEnabled true
            signingConfig signingConfigs.debug
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.0.1'
    testCompile 'junit:junit:4.12'
    provided 'com.tencent.tinker:tinker-android-anno:1.7.7'
    //tinker的核心库
    compile 'com.tencent.tinker:tinker-android-lib:1.7.7'
}

```

编写application类
```
package com.anlaiye.swt.tinkertest;

import android.app.Application;
import android.content.Context;
import android.content.Intent;
import com.tencent.tinker.anno.DefaultLifeCycle;
import com.tencent.tinker.lib.tinker.TinkerInstaller;
import com.tencent.tinker.loader.app.ApplicationLike;
import com.tencent.tinker.loader.shareutil.ShareConstants;

@DefaultLifeCycle(application = ".SimpleTinkerInApplication",
        flags = ShareConstants.TINKER_ENABLE_ALL,
        loadVerifyFlag =true)
public class SimpleTinkerInApplicationLike extends ApplicationLike {
    public SimpleTinkerInApplicationLike(Application application, int tinkerFlags, boolean tinkerLoadVerifyFlag, long applicationStartElapsedTime, long applicationStartMillisTime, Intent tinkerResultIntent) {
        super(application, tinkerFlags, tinkerLoadVerifyFlag, applicationStartElapsedTime, applicationStartMillisTime, tinkerResultIntent);
    }

    @Override
    public void onBaseContextAttached(Context base) {
        super.onBaseContextAttached(base);
    }

    @Override
    public void onCreate() {
        super.onCreate();
        TinkerInstaller.install(this);
    }
}
```

在manifest中标明同时 加上权限

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.anlaiye.swt.tinkertest">

    <application
        android:name=".SimpleTinkerInApplication"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>

                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <meta-data
            android:name="TINKER_ID"
            android:value="id_171717" />
    </application>

    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />


</manifest>
```
手动设置一下tinker_id
接下来开始测试
修改xml文件
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.anlaiye.swt.test.MainActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="第一次"/>

    <Button
        android:onClick="loadPatch"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="更新"/>
</LinearLayout>

```
mainactivity
```
package com.anlaiye.swt.test;

import android.os.Environment;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;

import com.tencent.tinker.lib.tinker.TinkerInstaller;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }


    public void loadPatch(View view) {
        TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(),
                Environment.getExternalStorageDirectory().getAbsolutePath() + "/patch_signed.apk");
        Log.d("swt", Environment.getExternalStorageDirectory().getAbsolutePath());

    }
}

```

接下来下载命令行工具
https://github.com/Tencent/tinker/tree/master/tinker-build/tinker-patch-cli/tool_output
全部copy带同一个文件夹，
然后是生成jar文件，点开build.gradle可以看到
```
// copy the jar to work directory
task buildTinkerSdk(type: Copy, dependsOn: [build, jar]) {
    group = "tinker"
    from('build/libs') {
        include '*.jar'
        exclude '*-javadoc.jar'
        exclude '*-sources.jar'
    }
    from('./tool_output') {
        include '*.*'
    }
    into(rootProject.file("buildSdk/build"))
}
```
这里有一段生成jar的脚本
在androidStudio中Terminal 里面切换到tinker-patch-cli目录下，然后执行gradle buildTinkerSdk命令，即可生成Jar包。
然后拿到了所有的工具了
开始打包
首先运行项目，拿到old.apk和mapping文件
将mapping 文件copy到proguard-rules.pro同目录，然后在proguard中输入
```
-applymapping mapping.txt
```

这是为了确保下次生成的apk和第一次生成的用的是同一个mapping
修改textview，随便修改点代码，不要运行，直接build出apk拿到新的apk
然后把old.apk new.apk放到前面下载的工具的同一个目录
进入tinker_config.xml文件
修改<loader value="com.anlaiye.swt.gradletest.SimpleTinkerInApplication"/>
为你自己的报名加application

接下来cmd进入命令行，进入所在的目录
输入
```
java -jar tinker-patch-cli-1.7.7.jar -old old.apk -new new.apk -config tinker_config.xml -out output
```
在目录下会多出一个output文件夹，里面有patch文件
![这里写图片描述](http://img.blog.csdn.net/20170314095458161?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
整体的目录结构如下
拿出patch导入你的手机sd卡最外层目录，点击APP里面的更新按钮，textview变化为你第二次修改的
成功
代码地址
http://download.csdn.net/detail/qq_15527709/9780203