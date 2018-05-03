---
title: 一行代码操作数据库GreenDao
date: 2018-04-26 17:48:16
---
上篇文章写了[GreenDao的基本使用](http://blog.csdn.net/qq_15527709/article/details/62223308)，后来考虑了一下，发现还有封装的余地。
所以我把初始化数据库的操作写到了基类中，这样就更加方便了,
首先看下上篇的userdao要想得到dao，需要以下的几个基本步骤
```
 mHelper = new DaoMaster.DevOpenHelper(this, “swt-db”, null);
        db = mHelper.getWritableDatabase();
        mDaoMaster = new DaoMaster(db);
        mDaoSession = mDaoMaster.newSession();
        dao=mDaoSession.getUserDao();

```

首先找到这里面的俩个变量"swt-db"，userdao，一个是数据库名字，一个是user表的操作对象，所以可以这样改
<!--more-->
```
    mHelper = new DaoMaster.DevOpenHelper(this, getDbName(), null);
        db = mHelper.getWritableDatabase();
        mDaoMaster = new DaoMaster(db);
        mDaoSession = mDaoMaster.newSession();
        getDao();

     protected abstract void getDao();

    protected abstract String getDbName();
```
将需要动态改变的对外暴露一个方法，让子类去实现它，所以我上个列子的子类是这样的
```
    @Override
    protected void getDao() {
        dao = mDaoSession.getUserDao();
    }

    @Override
    protected String getDbName() {
        return "swt-db";
    }
```

需要什么数据库名字和对象我就返回过去就好啦，然后突然我想到一个问题，mDaoSession是不是能得到所有的表的对象呢？我急忙看下了DaoSession的源码，我发现

```
 public UserDao getUserDao() {
        return userDao;
    }
```
它有这个方法，于是我测试了一下，我新建了一个people表，make了一下项目然后我发现在DaoSession的代码中出现了这个
``` 
  public PeoPleDao getPeoPleDao() {
        return peoPleDao;
    }
```

很好，people表的控制对象也是生成在这里面的，那就可以通过mDaoSession得到所有的表的对象，这样是OK的。

于是我们得到了表的dao对象，于是可以开心的操作数据库啦
```
 switch (v.getId()) {
            case R.id.add:
               if (getEtName() != null && EtAge.getText().toString() != null && getSex() != null) {
                    User insertUser = new User(null, getEtName(), getAge(), getSex());
                    dao.insert(insertUser);
                }
                break;
            case R.id.delete:
                dao.deleteByKey(getId());
                break;
            case R.id.update:
                User UpdateUser = new User(null, getEtName(), getAge(), getSex());
                dao.update(UpdateUser);
                break;
            case R.id.query:
                dao.loadAll();
                //动态展示的 所以暂时不需要这个东西
                break;
```

我在网上看到很多的人都对这个进行过简单的封装，不过都是封装在了application中，我觉得这样不是特别好，首先在真正的开发中，并不是每个模块都是需要去操作数据库，所以我觉得将它封装在一个baseactivity或者fragment中，作为一个数据库的activity基类，需要用到的时候就继承它，或者把他集成进去，application是APP每次启动都会去运行的，不应该放太多的初始化操作，就像有很多的项目中，支付类的SDK初始化都是采用懒加载的。

接下来贴出主要的代码，文末有代码下载地址
```
package com.anlaiye.swt.greendaodemo;

import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.text.TextUtils;
import android.view.View;
import android.widget.EditText;

import com.anlaiye.swt.greendaodemo.base.BaseActivity;
import com.anye.greendao.gen.UserDao;

import java.util.ArrayList;
import java.util.List;

public class MainActivity extends BaseActivity implements View.OnClickListener {
    //    private Button Add, Delect, Update, Query;
    private EditText EtId, EtName, EtAge, EtSex;
    private RecyclerView mRecyclerView;
    private List<User> mUserList = new ArrayList<>();
    private RvAdapter adapter;
    private UserDao dao;

    @Override
    protected void initview() {
        findViewById(R.id.add).setOnClickListener(this);
        findViewById(R.id.delete).setOnClickListener(this);
        findViewById(R.id.update).setOnClickListener(this);
        findViewById(R.id.query).setOnClickListener(this);
        EtId = (EditText) findViewById(R.id.et_id);
        EtName = (EditText) findViewById(R.id.et_name);
        EtAge = (EditText) findViewById(R.id.et_age);
        EtSex = (EditText) findViewById(R.id.et_sex);
        mRecyclerView = (RecyclerView) findViewById(R.id.rv_result);
        initRv();

    }

    @Override
    protected int getLayoutId() {
        return R.layout.activity_main;
    }

    @Override
    protected void getDao() {
        dao = mDaoSession.getUserDao();
    }

    @Override
    protected String getDbName() {
        return "swt-db";
    }

    public void initRv() {
        mUserList = dao.loadAll();
        adapter = new RvAdapter(this, mUserList, R.layout.rv_item);
        mRecyclerView.setAdapter(adapter);
        mRecyclerView.setLayoutManager(new LinearLayoutManager(this));
    }


    public Long getId() {
        Long id = Long.parseLong(EtId.getText().toString());
        return id;
    }

    public String getEtName() {
        String name = EtName.getText().toString();
        return name;
    }

    public int getAge() {
        if (EtAge.getText().toString() != null && !TextUtils.isEmpty(EtAge.getText().toString())) {
            int age = Integer.parseInt(EtAge.getText().toString());
            return age;
        }
        return 0;
    }

    public String getSex() {
        String sex = EtSex.getText().toString();
        return sex;
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.add:
                if (getEtName() != null && EtAge.getText().toString() != null && getSex() != null) {
                    User insertUser = new User(null, getEtName(), getAge(), getSex());
                    dao.insert(insertUser);
                }
                break;
            case R.id.delete:
                dao.deleteByKey(getId());
                break;
            case R.id.update:
                User UpdateUser = new User(null, getEtName(), getAge(), getSex());
                dao.update(UpdateUser);
                break;
            case R.id.query:
                dao.loadAll();
                //动态展示的 所以暂时不需要这个东西
                break;
        }
        initRv();
        resetET();
    }

    private void resetET() {
        EtAge.setText("");
        EtName.setText("");
        EtSex.setText("");
        EtId.setText("");
    }
}

```


基类
```
package com.anlaiye.swt.greendaodemo.base;

import android.database.sqlite.SQLiteDatabase;
import android.os.Bundle;
import android.support.annotation.Nullable;
import android.support.v7.app.AppCompatActivity;

import com.anye.greendao.gen.DaoMaster;
import com.anye.greendao.gen.DaoSession;

/**
 * 介绍：这里写介绍
 * 作者：sweet
 * 邮箱：sunwentao@imcoming.cn
 * 时间: 2017/3/20
 */

public abstract class BaseActivity extends AppCompatActivity {
     DaoMaster.DevOpenHelper mHelper;
     SQLiteDatabase db;
     DaoMaster mDaoMaster;
    public DaoSession mDaoSession;


    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getLayoutId());
        openDb();
        initview();
    }

    protected abstract void initview();

    protected abstract int getLayoutId();


    private void openDb() {
        mHelper = new DaoMaster.DevOpenHelper(this, getDbName(), null);
        db = mHelper.getWritableDatabase();
        mDaoMaster = new DaoMaster(db);
        mDaoSession = mDaoMaster.newSession();
        getDao();
    }

    protected abstract void getDao();

    protected abstract String getDbName();

}

```


这里只简单的写了个实现，可以作为入门用，大神可以无视我。。。。入门人员可以继续扩展。
代码下载地址http://download.csdn.net/detail/qq_15527709/9786940