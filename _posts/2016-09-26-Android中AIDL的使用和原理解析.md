---
layout:     post
title:      Android中AIDL的使用和原理解析
subtitle:   
date:       2016-09-26
author:     XiaoZhui Wang
header-img: img/post-bg-1.jpg
catalog: true
tags:
    - Android
---

写这篇文章之前，首先要感谢一下任玉刚大哥写了《Android开发艺术探索》这本书。这篇文章其实就是对书中讲解AIDL的那个小节的一个简单的总结。

### Android进程间通信方式
Android实现进程间通信的方式有很多种，比如通过Intent来传递数据，共享文件，SharedPreferences，基于Binder的Messager和AIDL，以及socket等。Binder是Android中最有特色的进程间通信方式。

### Binder
Binder主要用于进程间通信，是Android中的一个类，实现了IBinder接口。从Android应用层来说，Binder是服务端与客户端通信的媒介，当bindService一个服务的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个对象客户端就可以获取服务端数据或向服务端传递数据，这里的服务包括普通服务和包含了AIDL的服务。

在Android开发中，Binder主要用在Service中，包括AIDL、Messager。普通Service中的Binder不涉及到进程间通信，所以较为简单。Messager底层其实就是AIDL，这里选择AIDL来学习Binder的工作机制。

### AIDL在Service中的使用方法

##### 服务端
* 定义AIDL接口，将暴露给客户端的接口在这个AIDL文件中声明
 
```java
// IBookManager.aidl
package com.example.android.aidl;

interface IBookManager{

    void setName(String name);
    String getName();
}
```
> 系统会自动根据AIDL文件生成用于通信的java类。建议将AIDL文件及所有AIDL用到的类和文件放在同一个包中，这样做的好处是，当客户端在另一个工程时，我们可以直接整个包复制到客户端工程。

* 编写Service，在Service实现AIDL接口

```java
//MyService.java
package com.example.android.service;

public class MyService extends Service {

    private String name;
    @Override
    public void onCreate() {
        super.onCreate();
        
    }
    @Override
    public IBinder onBind(Intent arg0) {
        
        return mBinder;
    }
 
    private final IBookManager.Stub mBinder = new IBookManager.Stub() {
        void setName(String name){
            this.name = name;
        }
        String getName(){
            return name;
        }
    };
 
}
```


然后在清单文件中注册Service，这里假设action为：

```java
com.example.android.action.bookmanager
```

##### 客户端
* 拷贝AIDL文件
将AIDL文件及所有AIDL用到的类和文件拷贝到客户端工程中（如果服务端跟客户端没在一个工程的话），注意客户端AIDL的包结构必须和服务端保持一致。

* 绑定远程服务。绑定成功后将服务端返回的Binder对象转化成AIDL接口所属类型，接着就可以调用AIDL中的方法。

```java
//MainActivity.java
package com.example.android.activiy;

public class MainActivity extends Activity {

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    
        String action = "com.example.android.action.bookmanager";
        Intent intent = new Intent(action);
        //Android 5.0后不能使用隐式Intent启动服务，所以这里需要指定包名
        intent.setPackage("com.example.android");
        bindService(intent, conn, Context.BIND_AUTO_CREATE);
    }
     
    ServiceConnection conn = new ServiceConnection() {
         
        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
         
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            //将服务端返回的Binder对象转化成AIDL接口所属类型
            IBookManager mBookManager = IBookManager.Stub.asInterface(service);
            try {
               String name = mBookManager.getName();
               mBookManager.setName("Android开发艺术探索");
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    };
         
    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(conn);
    }
}
```

### AIDL 源码解析
上面简单介绍了一下AIDL在Service中的使用，接着就通过AIDL接口对应的java源码来分析一下AIDL是如何通过Binder实现进程间通信的。
```java
//IBookManager.java
package com.example.android.aidl;

/**
 * 系统根据aidl文件自动生成的java文件
 */
public interface IBookManager extends android.os.IInterface {

    // 用于标识setName()、getName()方法的id
    static final int TRANSACTION_setName = 
                              (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_getName =     
                              (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);

    // aidl文件接口中声明的方法
    public void setName(java.lang.String name)
            throws android.os.RemoteException;
    public java.lang.String getName() throws android.os.RemoteException;

    /** Stub 是IPC通信实现类，Stub即是Binder对象，也是IBookManager对象 */
    public static abstract class Stub extends android.os.Binder implements
            com.example.xutils3demo.aidl.IBookManager {

        // Binder唯一标识，一般用当前Binder的类名表示
        private static final java.lang.String DESCRIPTOR = 
                                    "com.example.xutils3demo.aidl.IBookManager";

        /** Construct the stub at attach it to the interface. */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an
         * com.example.xutils3demo.aidl.IBookManager interface, generating a
         * proxy if needed.
         * 客户端应调用此方法将服务端返回的Binder对象转换成AIDL接口所属的类
         * 型，接着就可以调用AIDL中的方法了。
         * 如果不跨进程通信直接返回IBookManager对象本身，否则返回IBookManager对象的代理
         */
        public static com.example.xutils3demo.aidl.IBookManager asInterface(
                android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null)
                     && (iin instanceof com.example.xutils3demo.aidl.IBookManager))) {
                // 不跨进程，返回IBookManager对象本身
                return ((com.example.xutils3demo.aidl.IBookManager) iin);
            }
            // 跨进程，返回IBookManager对象的代理Proxy对象
            return new com.example.xutils3demo.aidl.IBookManager.Stub.Proxy(obj);
        }

        // Binder唯一标识，一般用当前Binder的类名表示
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        /**
         * 方法执行在服务端，由客户端通过RPC(远程过程调用)进行调用， 通过这
         * 个方法，最终使得服务端Binder对象中对应的方法被调用。
         *
         * 服务端可以根据code值，确定客户端请求的目标方法是什么。 接着从data
         * 中取出目标方法所需的参数，然后执行目标方法。目标方法执行完后，就    
         * 向reply中写入返回值，返回给客户端。
         * 注意：如果此方法返回false的话，那么客户端的请求会失败， 因此可以利
         * 用这个特性来做权限验证。使得符合权限的客户端才能 调用服务端方法。
         */
        @Override
        public boolean onTransact(int code, android.os.Parcel data,
                android.os.Parcel reply, int flags)
                throws android.os.RemoteException {
            switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_setName: {
                data.enforceInterface(DESCRIPTOR);
                java.lang.String _arg0;
                // 从data中取出目标方法所需参数
                _arg0 = data.readString();
                // 这里使得服务端Binder中的方法执行
                this.setName(_arg0);
                reply.writeNoException();
                return true;
            }
            case TRANSACTION_getName: {
                data.enforceInterface(DESCRIPTOR);
                // 这里使得服务端Binder中的方法执行
                java.lang.String _result = this.getName();
                reply.writeNoException();
                reply.writeString(_result);
                return true;
            }
            }
            return super.onTransact(code, data, reply, flags);
        }

        /**
         * Binder代理，类中方法执行在客户端，这个类才是真正的RPC。
         *
         * RPC过程：当客户端调用Proxy对象的setName()方法时，会通 过    
         * transact()发起RPC请求，并将当前线程挂起。接着服务端的 
         * onTransact()会被调用，从而使得服务端的Binder中的对应的
         * setName()方法被调用，直到RPC过程返回后，当前线程才继续执行，并
         * 从_reply 中取出RPC过程的返回结果，然后返 给setName()方法(如果有
         * 返回值的话)。getName()方法同理。
         */
        private static class Proxy implements
                com.example.xutils3demo.aidl.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            @Override
            public void setName(java.lang.String name)
                    throws android.os.RemoteException {
                // _data 输入型Parcel对象，用于携带客户端传递给服务端的参数
                android.os.Parcel _data = android.os.Parcel.obtain();
                // _reply 输出型Parcel对象，用于携带从服务端返回给客户端的返回结果
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    //将要传递给服务端的参数写入_data中
                    _data.writeString(name);
                    // 调用Binder对象的transact()方法发送RPC请求，并将当前挂起
                    mRemote.transact(Stub.TRANSACTION_setName, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public java.lang.String getName() throws android.os.RemoteException {
                
                android.os.Parcel _data = android.os.Parcel.obtain();    
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getName, _data, _reply, 0);
                    _reply.readException();
                    // 从_reply中读取服务端返回给客户端的返回结果
                    _result = _reply.readString();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }
    }

}

```
小结：客户端绑定服务端后，服务端会返回一个Binder对象。客户端拿着这个Binder对象调用AIDL接口中的方法实际上是一次远程调用请求过程（RPC）。原理是这样的，当客户端用Binder调用AIDL接口中的方法时，实际上是调用Stub.Proxy类的的同名代理方法（其实在调用`Stub.asInterface()`时，就将服务端返回的Binder转换成了Stub.Proxy对象，Proxy对象也是一个实现了AIDAL接口的对象），该代理方法会调用`Binder.transact(Stub.TRANSACTION_methodName, _data, _reply, 0)`（Stub.TRANSACTION_methodName要调用的服务端AIDL接口中方法的标识，其实是就是与代理方法同名的那个方法，_data携带要发送给服务端的数据，_reply携带服务器返回给客户端的数据），发起一次远程调用请求，然后客户端挂起，此时服务端AIDL中的`onTransact(）`会被调用，在该方法中会调用到`Stub.TRANSACTION_methodName`标识的方法，并将_data传递该方法，将方法的返回结果写入_reply中，然后RPC过程返回，客户端代码继续往下执行，客户端读取_reply拿到返回结果。

### AIDL支持的参数类型
在AIDL文件中，并不是支持所有的数据类型，只有某些数据类型才能使用。AIDL支持的数据类型有：

 - 基本数据类型（int、long、char、boolean、double等）
 - String 和 CharSequence
 - List：只支持ArrayList，里面的元素都必须能够被AIDL支持
 - Map：只支持HashMap，里面的每个元素都必须能够被AIDL支持，包括key和value
 - Parcelable：所有实现了Parcelable 的对象
 - AIDL：所有AIDL接口本身也可以在AIDL文件中使用

### AIDL文件定义注意事项
 - Parcelable对象在AIDL中使用前必须新建一个和它同名的AIDL文件，并在其中声明该类为Parcelable类型。
 - 在AIDL中，使用自定义的Parcelable对象和AIDL接口必须显示import进来，不管他们是否和当前AIDL位于同一个包中。这是AIDL的规范。
 -     AIDL接口中只支持方法，不支持静态常量，这一点区别于传统接口。
 - AIDL中除基本数据类型外，其他类型的参数必须标上方向：in、out和inout。in表示输入型参数，out表示输出型参数，inout表示输入输出型参数。

以下是书中提到的更多关于AIDL使用的内容，这里就不一一讲解了。

### 使用AIDL在服务端注册与解注册监听器

### 远程服务断开重连

### 使用AIDL做权限验证