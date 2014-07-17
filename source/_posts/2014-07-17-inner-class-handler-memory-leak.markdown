---
layout: post
title: "Android中Activity是如何导致内存泄漏的：Handler 与 Inner Calass(内部类)"
date: 2014-07-17 21:13:15 +0800
comments: true
categories: 
---
首先思考下面的代码：

```java
public class SampleActivity extends Activity {

  private final Handler mLeakyHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ... 
    }
  }
}
```

虽然不是很明显，但是这段代码的确会导致严重的内存泄漏，Android会给出如下的警告:

>在Android中，Handler类需要定义为static，否则可能会导致内存泄漏。

但是哪里会导致泄漏，泄漏是如何产生的呢？让我门首先阅读下面的文档：


当Android应用程序首次启动，系统会为程序主线程创建一个Looper对象，Looper实现了一个消息队列，循环处理Message。程序中所有重要的事件（如调用Activity的生命周期方法，按钮点击事件等）都是在Message对象中，这些Message对象被添加到了Looper的消息队列里面并且一个一个依次执行。主线程的Looper存在于应用程序的整个生命周期。


当一个Handler在主线程实例化后，它就被关联到了Looper的消息队列中，Looper在处理消息时为了能够调用Hnadler的```handeMessage(Message)```方法，所以推送到消息队列的Message会持有一个Handler的引用。

>在Java中，非静态内部类和匿名类会持有一个外部类的引用，而静态的内部类则不会。

因此，内存泄漏到底发生在哪呢？虽然比较隐藏，但是思考下面的代码：

```java
public class SampleActivity extends Activity {
 
  private final Handler mLeakyHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ...
    }
  }
 
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
 
    // Post a message and delay its execution for 10 minutes.
    mLeakyHandler.postDelayed(new Runnable() {
      @Override
      public void run() { }
    }, 60 * 10 * 1000);
 
    // Go back to the previous Activity.
    finish();
  }
}
```

当Activity销毁时，这个延迟执行的消息在真正执行之前会在主线程的消息队列中存在10分钟，这个消息持有Handler的引用，而Handler持有它的外部类（SampleActivity）的引用。这个引用会一直存在直到消息被处理，这会阻止垃圾回收器回收activity上下文对象，以及程序所有资源。代码15行中匿名的Runnable对象同样也有这样的问题。**非静态匿名类对象会持有其外部类对象的引用，所以会导致泄漏。**


为了解决这个问题，可以定义Handler在一个新的文件中或者使用静态内部类。静态内部类不会持有外部类的引用，所以activity不会被泄漏。如果在Handler中需要调用外部Activity的方法，可以让Handler持有Activity的弱引用。要解决实例化Runnable时导致的内存泄漏，可以指定runnable对象为```static```（因为静态的匿名对象不会持有外部类的引用）

```java
public class SampleActivity extends Activity {

  /**
   * Instances of static inner classes do not hold an implicit
   * reference to their outer class.
   */
  private static class MyHandler extends Handler {
    private final WeakReference<SampleActivity> mActivity;

    public MyHandler(SampleActivity activity) {
      mActivity = new WeakReference<SampleActivity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
      SampleActivity activity = mActivity.get();
      if (activity != null) {
        // ...
      }
    }
  }

  private final MyHandler mHandler = new MyHandler(this);

  /**
   * Instances of anonymous classes do not hold an implicit
   * reference to their outer class when they are "static".
   */
  private static final Runnable sRunnable = new Runnable() {
      @Override
      public void run() { }
  };

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Post a message and delay its execution for 10 minutes.
    mHandler.postDelayed(sRunnable, 60 * 10 * 1000);
    
    // Go back to the previous Activity.
    finish();
  }
}
```

虽然静态内部类和非静态内部类的区别很小，但是作为Aandroid开发者是需要了解的。在Activity中，如果内部类对象的生命周期长于其外部类对象，那么避免使用非静态内部类，否则使用静态内部类，并让其持有外部类对象的弱引用。

有任何问题欢迎留言！

原文：[How to Leak a Context: Handlers & Inner Classes](http://www.androiddesignpatterns.com/2013/01/inner-class-handler-memory-leak.html)