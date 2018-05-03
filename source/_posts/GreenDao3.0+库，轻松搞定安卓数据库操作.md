---
title: GreenDao3.0+库，轻松搞定安卓数据库操作
date: 2018-04-26 17:48:16
---
GreenDao是安卓的数据库操作库，它实现了ORM，ORM称为：对象关系映射
我把它简单的理解为用对象来操作数据库
比如我需要创建一个User名字的数据库表，我可以直接定义一个User类，里面加上俩个参数
age,name,导入GreenDao库后，我只需要在User类上面加上一个@Entity注解
像这样
```
@Entity
public class User {
    @Id(autoincrement = true)
    private Long id;
    private String name;
    private int age;
```
然后make一下项目，User表就生成了，同时表中带有age和name的属性，
再比如我们需要往表中插入一个数据，那么只需要
```
User user= new User("sweet","20");
dao.insert(user);
```
<!--more-->
这样就实现了数据的插入操作，怎么样是不是很简单？
这里有个dao对象是怎么来的呢？
好了，正式进入正文，首先回顾下我们以前使用sqlite操作数据库的时候是怎么样的
首先：
```
package com.demo.swt.mystudyappshop.Wight;

import android.content.Context;
import android.database.sqlite.SQLiteDatabase;
import android.database.sqlite.SQLiteOpenHelper;

/**
 * 介绍：这里写介绍
 * 作者：sweet
 * 邮箱：sunwentao@imcoming.cn
 * 时间: 2017/2/10
 */

public class DatabaseHelper extends SQLiteOpenHelper {

    private static final int VERSION = 1;

    //在SQLiteOepnHelper的子类当中，必须有该构造函数
    public DatabaseHelper(Context context, String name, SQLiteDatabase.CursorFactory factory,
                          int version) {
        //必须通过super调用父类当中的构造函数
        super(context, name, factory, version);
    }

    public DatabaseHelper(Context context, String name) {
        this(context, name, VERSION);
    }

    public DatabaseHelper(Context context, String name, int version) {
        this(context, name, null, version);
    }

    //该函数是在第一次创建数据库的时候执行,实际上是在第一次得到SQLiteDatabse对象的时候，才会调用这个方法
    @Override
    public void onCreate(SQLiteDatabase db) {
        System.out.println("create a Database");
        //execSQL函数用于执行SQL语句
        db.execSQL("create table user(id int,name varchar(20))");
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        System.out.println("update a Database");
    }
}

```
我们需要创建一个helper类继承SQLiteOpenHelper 同时在里面定义数据库的版本号，同时还得自己写数据库语句
db.execSQL("create table user(id int,name varchar(20))");
这里有个db是SQLiteDatabase 的对象，
接下来创建好helper类之后
创建数据库
```
 //创建一个DatabaseHelper对象
            if (dbHelper == null) {
                dbHelper = new DatabaseHelper(SqliteDataBaseActivity.this, "test_mars_db", 3);
                //只有调用了DatabaseHelper对象的getReadableDatabase()方法，或者是getWritableDatabase()方法之后，才会创建，或打开一个数据库
                db = dbHelper.getReadableDatabase();
                db.getVersion();
            } else {
                dataview.setText("你已经创建过数据库");
            }
```
插入数据
```
 if (dbHelper == null) {
                dataview.setText("请先更新数据库");
            } else {
                //生成ContentValues对象
                ContentValues values = new ContentValues();
                //想该对象当中插入键值对，其中键是列名，值是希望插入到这一列的值，值必须和数据库当中的数据类型一致
                values.put("id", 1);
                values.put("name", "zhangsan");
                //调用insert方法，就可以将数据插入到数据库当中
                db.insert("user", null, values);
            }
```
大概就实列这俩个就可以看出，sqlite操作数据库远远没有我们使用greenDao库操作方便，
说了这么多，也对比了sqlite和greendao，分析了greendao的好处

这里还得说一句，本篇使用的是3.0+的GreenDao，3.0后的库和3.0之前的有很大的不用，3.0后的greendao库不需要添加外部依赖，用java代码生成辅助类
直接用注解的方式生成辅助类，并在app的build 中能指定生成的文件的目录
接下来正式接入
首先在project的build中加入classpath
```
  dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        //add
        classpath 'org.greenrobot:greendao-gradle-plugin:3.2.1'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
```
接下来配置app的build
顶部的
apply下面加入这一句
```
apply plugin: 'org.greenrobot.greendao'
```

dependencies下加入
```
  compile 'org.greenrobot:greendao:3.2.0'
    compile 'org.greenrobot:greendao-generator:3.0.0'
```
最外层配置
```
//指定生成的文件目录
greendao {
    schemaVersion 1
    daoPackage 'com.anye.greendao.gen'
    targetGenDir 'src/main/java'
}
```
全部代码如下
新增加的我会加入//add
```
apply plugin: 'com.android.application'
//add
apply plugin: 'org.greenrobot.greendao'
android {
    compileSdkVersion 25
    buildToolsVersion "25.0.1"
    defaultConfig {
        applicationId "com.anlaiye.swt.greendaodemo"
        minSdkVersion 14
        targetSdkVersion 24
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

}
//add
greendao {
    schemaVersion 1
    daoPackage 'com.anye.greendao.gen'
    targetGenDir 'src/main/java'
}



dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.0.1'
    testCompile 'junit:junit:4.12'
    //add
    compile 'org.greenrobot:greendao:3.2.0'
    compile 'org.greenrobot:greendao-generator:3.0.0'
    compile 'com.android.support:recyclerview-v7:25.0.0'

}

```

到这里基本的配置就完成了
接下来创建一个user类
```
@Entity
public class User {
//表示id会自增长
    @Id(autoincrement = true)
    private Long id;
    private String name;
    private int age;
    private String sex;
    }
```
因为3.0后都是使用注解的方式，所以如果大家对别的注解像深入了解的话可以去官网看下
创建好后点击build-make
会在我们指定的目录下生成特定的文件
![这里写图片描述](http://img.blog.csdn.net/20170315141446085?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
同时在User类中会自动生成get set方法 
```
package com.anlaiye.swt.greendaodemo;

import org.greenrobot.greendao.annotation.Entity;
import org.greenrobot.greendao.annotation.Id;
import org.greenrobot.greendao.annotation.Generated;

/**
 * 介绍：这里写介绍
 * 作者：sweet
 * 邮箱：sunwentao@imcoming.cn
 * 时间: 2017/3/9
 */
@Entity
public class User {
    @Id(autoincrement = true)
    private Long id;
    private String name;
    private int age;
    private String sex;
    @Generated(hash = 689493095)
    public User(Long id, String name, int age, String sex) {
        this.id = id;
        this.name = name;
        this.age = age;
        this.sex = sex;
    }
    @Generated(hash = 586692638)
    public User() {
    }
    public Long getId() {
        return this.id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getName() {
        return this.name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return this.age;
    }
    public void setAge(int age) {
        this.age = age;
    }
    public String getSex() {
        return this.sex;
    }
    public void setSex(String sex) {
        this.sex = sex;
    }

  
}

```

接下来进行测试
在前面有个dao对象，现在来看下他是怎么生成的
```
   private DaoMaster.DevOpenHelper mHelper;
    private SQLiteDatabase db;
    private DaoMaster mDaoMaster;
    private DaoSession mDaoSession;
       private UserDao dao;
  private void openDb() {
          //创建swt_db的数据库
        mHelper = new DaoMaster.DevOpenHelper(this, "swt-db", null);
        //拿到可读可写的对象
        db = mHelper.getWritableDatabase();
        //拿到DaoMaster对象
        mDaoMaster = new DaoMaster(db);
        //拿到daosession对象
        mDaoSession = mDaoMaster.newSession();
        //创建了user表所以会生成usedao的实例
        dao = mDaoSession.getUserDao();
    }
```

上面的操作是固定的，也就是说我们可以把打开数据库的这一系列操作，封装起来，比如放到aplication中
或者使用一个util类封装，
好了，拿到了user表的dao对象，接下来就可以正式对user表进行操作了
我写了一个简陋的ui来形象的展示
首先看下我写的DEMO界面效果
![这里写图片描述](http://img.blog.csdn.net/20170315142216901?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


我定义了3个edittext拿到输入的值作为user表的属性
首先是插入操作
```
//这里不用设置id，id设置了自增加，会自动生成
 User insertUser = new User(null, getEtName(), getAge(), getSex());
                    dao.insert(insertUser);
```

删除操作
```
//根据id删除内容
  dao.deleteByKey(getId());
```
更新操作
```
//根据id更新指定的数据
   User UpdateUser = new User(getid(), getEtName(), getAge(), getSex());
                dao.update(UpdateUser);
```
 查询操作
```
//查询出所有的数据
  dao.loadAll();
```
增删改查基本上都是一俩句话就完成了
真的是灰常简单
接下来看下正常的效果
![这里写图片描述](http://img.blog.csdn.net/20170315142925717?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTU1Mjc3MDk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
这里只是简单的使用recyclerview实现做个演示
代码下载地址
https://github.com/swtandyz/GreenDaoDemo
随手start thanks


 
 

