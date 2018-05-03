---
title: tinker热修复gradle接入
date: 2018-04-26 17:48:16
---
本篇文章已授权微信公众号 guolin_blog （郭霖）独家发布

今天研究了一天的热修复，热修复，简单的来讲就是在不需要发包的情况下，修改你线上应用的bug，接入使用后对于我这种小白来说还是很神奇的，同时也考虑了一下，要不要接入我们的项目中，这样就不用因为一个小BUG而去再次发包了，不过，就算要接入项目中，也还有很多坑需要踩，tinker有俩种接入方式，一种命令行接入，一种是gradle接入，本篇只讲gradle接入，下篇我在补充命令行，主要用于自己做个记录，把踩得坑和感想写下来.
首先，基本的配置
在peoject的build中配置如下
```
dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        classpath "com.tencent.tinker:tinker-patch-gradle-plugin:${TINKER_VERSION}"
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
```
TINKER_VERSION需要在gradle.properties中进行配置
```
TINKER_VERSION=1.7.7
```
<!--more-->
这里这样写的目的是把资源统一管理，接下来是配置app的build，官方文档推荐我们使用这个
https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/build.gradle
于是我全部copy了过来，不过里面的几个地方还是需要修改的
首先先贴上全部的
我会在里面写上注释

```
apply plugin: 'com.android.application'



dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.1.1'
    //可选，用于生成application类
    compile("com.tencent.tinker:tinker-android-lib:${TINKER_VERSION}") { changing = true }
    //tinker的核心库
    provided("com.tencent.tinker:tinker-android-anno:${TINKER_VERSION}") { changing = true }
    compile 'com.android.support:multidex:1.0.1'
    //use to test multiDex
    //    compile group: 'com.google.guava', name: 'guava', version: '19.0'
    //    compile "org.scala-lang:scala-library:2.11.7"
    //use for local maven test
    //    compile("com.tencent.tinker:tinker-android-loader:${TINKER_VERSION}") { changing = true }
    //    compile("com.tencent.tinker:aosp-dexutils:${TINKER_VERSION}") { changing = true }
    //    compile("com.tencent.tinker:bsdiff-util:${TINKER_VERSION}") { changing = true }
    //    compile("com.tencent.tinker:tinker-commons:${TINKER_VERSION}") { changing = true }
}

//废弃，因为我们吧tinker_id写死了，这里的作用是把git的提交记录用作tinker_id的版本
def gitSha() {
    try {
        String gitRev = 'git rev-parse --short HEAD'.execute(null, project.rootDir).text.trim()
        if (gitRev == null) {
            throw new GradleException("can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId'")
        }
        return gitRev
    } catch (Exception e) {
        throw new GradleException("can't get git rev, you should add git to system path or just input test value, such as 'testTinkerId'")
    }
}

//配置java版本
def javaVersion = JavaVersion.VERSION_1_7

android {
    compileSdkVersion 23
    buildToolsVersion "23.0.2"

    compileOptions {
        sourceCompatibility javaVersion
        targetCompatibility javaVersion
    }
    //recommend
    dexOptions {
        jumboMode = true
    }
    //配置自己的签名文件，签名文件的生成和导入可以去百度，本篇不讲解
    signingConfigs {
        release {
            try {
                keyAlias 'china'
                keyPassword '123456'
                storeFile file('D:/work/release.jks')
                storePassword '123456'
            } catch (ex) {
                throw new InvalidUserDataException(ex.toString())
            }

        }
        debug {
            storeFile file('D:/work/debug.jks')
            keyAlias 'china'
            keyPassword '123456'
            storePassword '123456'
        }
    }

    defaultConfig {
        applicationId "tinker.sample.android"
        minSdkVersion 10
        targetSdkVersion 22
        versionCode 1
        versionName "1.0.0"
        /**
         * you can use multiDex and install it in your ApplicationLifeCycle implement
         */
        multiDexEnabled true
        /**
         * buildConfig can change during patch!
         * we can use the newly value when patch
         */
        buildConfigField "String", "MESSAGE", "\"I am the base apk\""
//        buildConfigField "String", "MESSAGE", "\"I am the patch apk\""
        /**
         * client version would update with patch
         * so we can get the newly git version easily!
         */
        buildConfigField "String", "TINKER_ID", "\"1.0\""
        buildConfigField "String", "PLATFORM",  "\"all\""
    }

//    aaptOptions{
//        cruncherEnabled false
//    }

//    //use to test flavors support
//    productFlavors {
//        flavor1 {
//            applicationId 'tinker.sample.android.flavor1'
//        }
//
//        flavor2 {
//            applicationId 'tinker.sample.android.flavor2'
//        }
//    }

    buildTypes {
        release {
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    sourceSets {
        main {
            jniLibs.srcDirs = ['libs']
        }
    }
}

def bakPath = file("${buildDir}/bakApk/")

/**
 * you can use assembleRelease to build you base apk
 * use tinkerPatchRelease -POLD_APK=  -PAPPLY_MAPPING=  -PAPPLY_RESOURCE= to build patch
 * add apk from the build/bakApk
 */
//老版本的文件所在的位置，大家也可以动态配置，不用每次都在这里修改
ext {
    //for some reason, you may want to ignore tinkerBuild, such as instant run debug build?
    tinkerEnabled = true

    //for normal build
    //old apk file to build patch apk
    tinkerOldApkPath = "${bakPath}/app-release-0313-16-16-16.apk"
    //proguard mapping file to build patch apk
    tinkerApplyMappingPath = "${bakPath}/app-release-0313-16-16-mapping.txt"
    //resource R.txt to build patch apk, must input if there is resource changed
    tinkerApplyResourcePath = "${bakPath}/app-release-0313-16-16-R.txt"

    //only use for build all flavor, if not, just ignore this field
    tinkerBuildFlavorDirectory = "${bakPath}/app-1018-17-32-47"
}


def getOldApkPath() {
    return hasProperty("OLD_APK") ? OLD_APK : ext.tinkerOldApkPath
}

def getApplyMappingPath() {
    return hasProperty("APPLY_MAPPING") ? APPLY_MAPPING : ext.tinkerApplyMappingPath
}

def getApplyResourceMappingPath() {
    return hasProperty("APPLY_RESOURCE") ? APPLY_RESOURCE : ext.tinkerApplyResourcePath
}
//废弃
/*def getTinkerIdValue() {
    return hasProperty("TINKER_ID") ? TINKER_ID : gitSha()
}*/

def buildWithTinker() {
    return hasProperty("TINKER_ENABLE") ? TINKER_ENABLE : ext.tinkerEnabled
}

def getTinkerBuildFlavorDirectory() {
    return ext.tinkerBuildFlavorDirectory
}

if (buildWithTinker()) {
    apply plugin: 'com.tencent.tinker.patch'

    tinkerPatch {
        /**
         * necessary，default 'null'
         * the old apk path, use to diff with the new apk to build
         * add apk from the build/bakApk
         */
        oldApk = getOldApkPath()
        /**
         * optional，default 'false'
         * there are some cases we may get some warnings
         * if ignoreWarning is true, we would just assert the patch process
         * case 1: minSdkVersion is below 14, but you are using dexMode with raw.
         *         it must be crash when load.
         * case 2: newly added Android Component in AndroidManifest.xml,
         *         it must be crash when load.
         * case 3: loader classes in dex.loader{} are not keep in the main dex,
         *         it must be let tinker not work.
         * case 4: loader classes in dex.loader{} changes,
         *         loader classes is ues to load patch dex. it is useless to change them.
         *         it won't crash, but these changes can't effect. you may ignore it
         * case 5: resources.arsc has changed, but we don't use applyResourceMapping to build
         */
        ignoreWarning = true

        /**
         * optional，default 'true'
         * whether sign the patch file
         * if not, you must do yourself. otherwise it can't check success during the patch loading
         * we will use the sign config with your build type
         */
        useSign = true

        /**
         * optional，default 'true'
         * whether use tinker to build
         */
        tinkerEnable = buildWithTinker()

        /**
         * Warning, applyMapping will affect the normal android build!
         */
        buildConfig {
            /**
             * optional，default 'null'
             * if we use tinkerPatch to build the patch apk, you'd better to apply the old
             * apk mapping file if minifyEnabled is enable!
             * Warning:
             * you must be careful that it will affect the normal assemble build!
             */
            applyMapping = getApplyMappingPath()
            /**
             * optional，default 'null'
             * It is nice to keep the resource id from R.txt file to reduce java changes
             */
            applyResourceMapping = getApplyResourceMappingPath()

            /**
             * necessary，default 'null'
             * because we don't want to check the base apk with md5 in the runtime(it is slow)
             * tinkerId is use to identify the unique base apk when the patch is tried to apply.
             * we can use git rev, svn rev or simply versionCode.
             * we will gen the tinkerId in your manifest automatic
             */
            tinkerId = "1.0"/*getTinkerIdValue()*/

            /**
             * if keepDexApply is true, class in which dex refer to the old apk.
             * open this can reduce the dex diff file size.
             */
            keepDexApply = false
        }

        dex {
            /**
             * optional，default 'jar'
             * only can be 'raw' or 'jar'. for raw, we would keep its original format
             * for jar, we would repack dexes with zip format.
             * if you want to support below 14, you must use jar
             * or you want to save rom or check quicker, you can use raw mode also
             */
            dexMode = "jar"

            /**
             * necessary，default '[]'
             * what dexes in apk are expected to deal with tinkerPatch
             * it support * or ? pattern.
             */
            pattern = ["classes*.dex",
                       "assets/secondary-dex-?.jar"]
            /**
             * necessary，default '[]'
             * Warning, it is very very important, loader classes can't change with patch.
             * thus, they will be removed from patch dexes.
             * you must put the following class into main dex.
             * Simply, you should add your own application {@code tinker.sample.android.SampleApplication}
             * own tinkerLoader, and the classes you use in them
             *
             */
            loader = [
                    //use sample, let BaseBuildInfo unchangeable with tinker
                    "tinker.sample.android.app.BaseBuildInfo"
            ]
        }

        lib {
            /**
             * optional，default '[]'
             * what library in apk are expected to deal with tinkerPatch
             * it support * or ? pattern.
             * for library in assets, we would just recover them in the patch directory
             * you can get them in TinkerLoadResult with Tinker
             */
            pattern = ["lib/*/*.so"]
        }

        res {
            /**
             * optional，default '[]'
             * what resource in apk are expected to deal with tinkerPatch
             * it support * or ? pattern.
             * you must include all your resources in apk here,
             * otherwise, they won't repack in the new apk resources.
             */
            pattern = ["res/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]

            /**
             * optional，default '[]'
             * the resource file exclude patterns, ignore add, delete or modify resource change
             * it support * or ? pattern.
             * Warning, we can only use for files no relative with resources.arsc
             */
            ignoreChange = ["assets/sample_meta.txt"]

            /**
             * default 100kb
             * for modify resource, if it is larger than 'largeModSize'
             * we would like to use bsdiff algorithm to reduce patch file size
             */
            largeModSize = 100
        }

        packageConfig {
            /**
             * optional，default 'TINKER_ID, TINKER_ID_VALUE' 'NEW_TINKER_ID, NEW_TINKER_ID_VALUE'
             * package meta file gen. path is assets/package_meta.txt in patch file
             * you can use securityCheck.getPackageProperties() in your ownPackageCheck method
             * or TinkerLoadResult.getPackageConfigByName
             * we will get the TINKER_ID from the old apk manifest for you automatic,
             * other config files (such as patchMessage below)is not necessary
             */
            configField("patchMessage", "tinker is sample to use")
            /**
             * just a sample case, you can use such as sdkVersion, brand, channel...
             * you can parse it in the SamplePatchListener.
             * Then you can use patch conditional!
             */
            configField("platform", "all")
            /**
             * patch version via packageConfig
             */
            configField("patchVersion", "1.0")
        }
        //or you can add config filed outside, or get meta value from old apk
        //project.tinkerPatch.packageConfig.configField("test1", project.tinkerPatch.packageConfig.getMetaDataFromOldApk("Test"))
        //project.tinkerPatch.packageConfig.configField("test2", "sample")

        /**
         * if you don't use zipArtifact or path, we just use 7za to try
         */
        sevenZip {
            /**
             * optional，default '7za'
             * the 7zip artifact path, it will use the right 7za with your platform
             */
            zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
            /**
             * optional，default '7za'
             * you can specify the 7za path yourself, it will overwrite the zipArtifact value
             */
//        path = "/usr/local/bin/7za"
        }
    }

    List<String> flavors = new ArrayList<>();
    project.android.productFlavors.each {flavor ->
        flavors.add(flavor.name)
    }
    boolean hasFlavors = flavors.size() > 0
    /**
     * bak apk and mapping
     */
    android.applicationVariants.all { variant ->
        /**
         * task type, you want to bak
         */
        def taskName = variant.name
        def date = new Date().format("MMdd-HH-mm-ss")
        tasks.all {
            if ("assemble${taskName.capitalize()}".equalsIgnoreCase(it.name)) {

                it.doLast {
                    copy {
                        def fileNamePrefix = "${project.name}-${variant.baseName}"
                        def newFileNamePrefix = hasFlavors ? "${fileNamePrefix}" : "${fileNamePrefix}-${date}"

                        def destPath = hasFlavors ? file("${bakPath}/${project.name}-${date}/${variant.flavorName}") : bakPath
                        from variant.outputs.outputFile
                        into destPath
                        rename { String fileName ->
                            fileName.replace("${fileNamePrefix}.apk", "${newFileNamePrefix}.apk")
                        }

                        from "${buildDir}/outputs/mapping/${variant.dirName}/mapping.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("mapping.txt", "${newFileNamePrefix}-mapping.txt")
                        }

                        from "${buildDir}/intermediates/symbols/${variant.dirName}/R.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("R.txt", "${newFileNamePrefix}-R.txt")
                        }
                    }
                }
            }
        }
    }
    project.afterEvaluate {
        //sample use for build all flavor for one time
        if (hasFlavors) {
            task(tinkerPatchAllFlavorRelease) {
                group = 'tinker'
                def originOldPath = getTinkerBuildFlavorDirectory()
                for (String flavor : flavors) {
                    def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Release")
                    dependsOn tinkerTask
                    def preAssembleTask = tasks.getByName("process${flavor.capitalize()}ReleaseManifest")
                    preAssembleTask.doFirst {
                        String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 15)
                        project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release.apk"
                        project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-mapping.txt"
                        project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-release-R.txt"

                    }

                }
            }

            task(tinkerPatchAllFlavorDebug) {
                group = 'tinker'
                def originOldPath = getTinkerBuildFlavorDirectory()
                for (String flavor : flavors) {
                    def tinkerTask = tasks.getByName("tinkerPatch${flavor.capitalize()}Debug")
                    dependsOn tinkerTask
                    def preAssembleTask = tasks.getByName("process${flavor.capitalize()}DebugManifest")
                    preAssembleTask.doFirst {
                        String flavorName = preAssembleTask.name.substring(7, 8).toLowerCase() + preAssembleTask.name.substring(8, preAssembleTask.name.length() - 13)
                        project.tinkerPatch.oldApk = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug.apk"
                        project.tinkerPatch.buildConfig.applyMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-mapping.txt"
                        project.tinkerPatch.buildConfig.applyResourceMapping = "${originOldPath}/${flavorName}/${project.name}-${flavorName}-debug-R.txt"
                    }

                }
            }
        }
    }
}

```
主要修改了下面几个地方
```
  buildConfigField "String", "TINKER_ID", "\"${getTinkerIdValue()}\""
```
官方文档默认是将git提交的记录作为think_id记录下来，这里我不需要，所以我改成下图
```
  buildConfigField "String", "TINKER_ID", "\"1.0\""
```
我直接写死，需要git提交记录作为tinker_id的也可以按照官方文档推荐的写
同时，这些地方也可以相应的替换掉
```
//废弃
/*def getTinkerIdValue() {
    return hasProperty("TINKER_ID") ? TINKER_ID : gitSha()
}*/

 tinkerId = "1.0"/*getTinkerIdValue()*/
```
接下来修改自己的签名文件,我这里配置的文件路径是自己电脑上面的，这个位置大家需要修改为自己的签名文件路径，也可以按照我的去生成一个签名文件，我相信这个还是很简单的
```
   //配置自己的签名文件，签名文件的生成和导入可以去百度，本篇不讲解
    signingConfigs {
        release {
            try {
                keyAlias 'china'
                keyPassword '123456'
                storeFile file('D:/work/release.jks')
                storePassword '123456'
            } catch (ex) {
                throw new InvalidUserDataException(ex.toString())
            }

        }
        debug {
            storeFile file('D:/work/debug.jks')
            keyAlias 'china'
            keyPassword '123456'
            storePassword '123456'
        }
    }
```
```
   buildTypes {
        release {
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
```
官网文档没有设置debug的 proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'也就是说没有应用混淆文件，这样是不会生成mapping文件的，所以这里我也加上
```
 android.applicationVariants.all { variant ->
        /**
         * task type, you want to bak
         */
        def taskName = variant.name
        def date = new Date().format("MMdd-HH-mm-ss")
        tasks.all {
            if ("assemble${taskName.capitalize()}".equalsIgnoreCase(it.name)) {

                it.doLast {
                    copy {
                        def fileNamePrefix = "${project.name}-${variant.baseName}"
                        def newFileNamePrefix = hasFlavors ? "${fileNamePrefix}" : "${fileNamePrefix}-${date}"

                        def destPath = hasFlavors ? file("${bakPath}/${project.name}-${date}/${variant.flavorName}") : bakPath
                        from variant.outputs.outputFile
                        into destPath
                        rename { String fileName ->
                            fileName.replace("${fileNamePrefix}.apk", "${newFileNamePrefix}.apk")
                        }

                        from "${buildDir}/outputs/mapping/${variant.dirName}/mapping.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("mapping.txt", "${newFileNamePrefix}-mapping.txt")
                        }

                        from "${buildDir}/intermediates/symbols/${variant.dirName}/R.txt"
                        into destPath
                        rename { String fileName ->
                            fileName.replace("R.txt", "${newFileNamePrefix}-R.txt")
                        }
                    }
                }
            }
        }
```
这下面主要是设置生成的文件所在的目录是什么

配置完上面之后，build就配置好了，接下来配置application
先上代码
```
package com.anlaiye.swt.gradletest;

import android.app.Application;
import android.content.Context;
import android.content.Intent;
import android.support.multidex.MultiDex;

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
        MultiDex.install(base);
        TinkerInstaller.install(this);

    }

    @Override
    public void onCreate() {
        super.onCreate();
    }
}
```
这个application官网有提供的，也可以copy我的，ApplicationLike 并不是一个application
真正的application是@DefaultLifeCycle(application = ".SimpleTinkerInApplication",这个
所以在androidmanifest中配置applica的name
```
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
    </application>

    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />

```
同时配置读取sd卡的权限
接下来测试
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.anlaiye.swt.gradletest.MainActivity">

    <TextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="111"/>


    <Button
        android:layout_below="@+id/tv"
        android:onClick="loadPath"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="更新"/>
</RelativeLayout>

```
第一次我设置textview的值是111
然后设置按钮调用onclick
```
package com.anlaiye.swt.gradletest;

import android.os.Bundle;
import android.os.Environment;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Toast;

import com.tencent.tinker.lib.tinker.TinkerInstaller;

import java.io.File;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

        //加载补丁
        public void loadPath(View view) {
            String path = Environment.getExternalStorageDirectory().getAbsolutePath()+"/patch_signed_7zip.apk";
            File file = new File(path);
            if (file.exists()){
                Toast.makeText(this, "补丁已经存在", Toast.LENGTH_SHORT).show();
                TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), path);
            }else {
                Toast.makeText(this, "补丁已经不存在", Toast.LENGTH_SHORT).show();
            }
        }

}

```

然后我运行APP
会生成如下的目录结构
![这里写图片描述](http://img.blog.csdn.net/20170313165038089?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
同时界面效果
![这里写图片描述](http://img.blog.csdn.net/20170313165151211?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
接下来做如下修改
```

//老版本的文件所在的位置，大家也可以动态配置，不用每次都在这里修改
ext {
    //for some reason, you may want to ignore tinkerBuild, such as instant run debug build?
    tinkerEnabled = true

    //for normal build
    //old apk file to build patch apk
    tinkerOldApkPath = "${bakPath}/app-release-0313-16-49-55.apk"
    //proguard mapping file to build patch apk
    tinkerApplyMappingPath = "${bakPath}/app-release-0313-16-49-55-mapping.txt"
    //resource R.txt to build patch apk, must input if there is resource changed
    tinkerApplyResourcePath = "${bakPath}/app-release-0313-16-49-55-R.txt"

    //only use for build all flavor, if not, just ignore this field
    tinkerBuildFlavorDirectory = "${bakPath}/app-1018-17-32-47"
}
```
注意这里是release版本，不能debug版本,release版本混着来，比如oldapk是debug版本，新的apk是release版本，这样是不行的，统一用release版本

这里我将对应路径改为bakapk中第一次运行生成的apk文件名字，比如第一次生成的apk文件名为app-release-0313-16-49-55.apk,所以我把tinkerOldApkPath 的路径后面的名字修改为对应的app-release-0313-16-49-55.apk，mapping文件和R文件路径也要对应修改，最后一项tinkerBuildFlavorDirectory 可以忽略，我一直觉得这里太死板，应该像设置tinker_id那样对这里进行动态的配置，不用每次都来修改

接下来我把textview改成2222，用来和第一次作区别

然后点击
![这里写图片描述](http://img.blog.csdn.net/20170313165446790?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

生成目录如下

![这里写图片描述](http://img.blog.csdn.net/20170313165553369?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 注意箭头所指的文件
 我们在onclick中执行的文件名就是这个，这个文件就是我们需要的最终的文件
 然后将这个文件导入SD卡的最外层目录，点击更新按钮
 点击后如下
 ![这里写图片描述](http://img.blog.csdn.net/20171012105544027?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
 因为模拟器不好导入patch的apk所以直接截图了
成功
本篇只是简单的演示了整个流程，整理了一下容易踩得坑，可以优化的地方还有很多，大家可以在这个基础上自行发挥。

总结一下自己碰到的坑，
1,首先没有在gradle.properties中配置tinker_id，会提示错误tinker_id not set!!!
2,debug没有配置  proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
导入生成的apk一直没有mapping文件，找了半天才发现
最后说下热修复的大概流程
最终我们需要的是一个patch.apk文件，这个文件是通过老apk和新apk对比生成的,
具体怎么对比和生成，是tinker控制的，所以我们需要导入他的依赖包
新APK和老APK的mapping必须是一致的，所以我们需要把老APK的mapping保存起来，方便新APK与他对比。
这样就能得到最终的patch，然后调用sdk的TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), path);
这样就实现了热修复的目的

下篇讲解命令行接入
本篇下载地址：
csdn下载地址
http://download.csdn.net/download/qq_15527709/10017530
考虑到现在csdn改版下载代码都需要分了，所以我重新上传了一份到了github
https://github.com/BigSweet/TinkerGradleDemo
