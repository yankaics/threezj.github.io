title: onServiceConnected方法没有回调的解决方案
date: 2015-09-25 15:13:06
tags: ["Android"]
---
这两天在做一个关于截取电话的service服务。当我想让activity和service进行信息交换的时候，发现onServiceConnected()这个方法一直无法调用。

google了好久终于解决了。下面把这个解决方案，总结如下。

## 注册Service

首先要确保在AdnroidManifest.xml文件正确注册service
注意 建议把名称换成全名，包名也写进去，如下所示。stackoverflow里确实有人就是这样解决了。 
```xml
<service android:name="three.com.phoneservice.PhoneServer" />
```

## 返回实例化的Binder

必须要在onBind()方法中返回一个实例化的Binder。
```java
private final MyBinder myBinder=new MyBinder();

@Nullable
@Override
public IBinder onBind(Intent intent) {
    return myBinder;
}

class MyBinder extends Binder {

    public void getDb(TextView mytextView,Db mdb,Context mcontext){
        textView=mytextView;
        db=mdb;
        context=mcontext;
    }
}
```
## 别在主线程中做费时的操作

这是需要特别注意的一点。我就是遇到了这个坑。因为activity和service是同一个线程！如果你在主线程中阻塞等待，那么service自然会崩溃。

所以最好像这样写。
```java
private ServiceConnection serviceConnection=new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        myBinder= (PhoneServer.MyBinder) service;
        myBinder.getDb(textView, db,MainActivity.this);
        new Thread(new Runnable() {
            @Override
            public void run() {
                Utility.createPersonInfo(db);
            }
        }).start();
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {

    }
};
```
