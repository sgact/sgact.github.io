---
layout:     post
title:      Android 系统源码之应用(四) 主动探测 卡顿与ANR
date:       2019-03-24
author:     sg
catalog: true
tags:
    - Android
    - 源码
---、


### Android 系统源码之应用(四) 主动探测 卡顿与ANR

​	今天要分析的代码意外的让人熟悉呢，就是 Looper啦。想想也是的，一旦 hander 处理一个消息花费了好长时间，那不就是卡顿了吗？

##### 一、Solution1

```java
  public static void loop() {
      final Looper me = myLooper();
      ...
      for (;;) {
          Message msg = queue.next(); // might block
					...
          // This must be in a local variable, in case a UI event sets the logger
          Printer logging = me.mLogging;
          if (logging != null) {
              logging.println(">>>>> Dispatching to " + msg.target + " " +
                      msg.callback + ": " + msg.what);
          }

          msg.target.dispatchMessage(msg);

          if (logging != null) {
              logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
          }
					...
      }
  }
```

​	说到 Looper 的源码，那基本上就是这一段了，可以看到msg.target.dispatchMessage(msg); 的上下两端由logging.println 所包围。msg.target 就是 处理 msg 的对象也就是 Handler,这句话会一直顺序执行到 handleMessage 方法，因此，我们考虑 通过logging.println 来获取两执行 handleMessage 的大致耗时。

​	为了保证logging != null，我们要手动的给他设置一个同时在内部对Dispatching to与Finished to进行匹配与计时，如果超过你的阈值的话就说明卡顿了：

```java
Looper.getMainLooper().setMessageLogging(new Printer() {
            @Override
            public void println(String s) {
                
            }
        });
```

​	在卡顿处我们可以获取调用栈，但是这和 OOM 的问题是一样的，即发生卡顿的代码只是压死骆驼的最后一根稻草，并不一定时真正有问题的代码，因此我们可以高频的采集多个堆栈然后进行分析，也可以上传到后台，考虑到数据量不算小，应该对堆栈进行去重。

##### 二、solution2

​	ANR 的发生就是因为主线程没有响应用户的输入，基于这一点我们同样可以判断，这里偷懒一下，把我以前写的一个测试工具的部分代码直接贴过来

```java
static class MainThreadHandler extends Handler {
        public static long lastTime = -1L;
        public MainThreadHandler(Looper looper) {
            super(looper);
        }
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            lastTime = (long) msg.obj;
        }
    }

    private MainThreadHandler hMainThread;
    private AnrListenThread anrListenThread;

    class AnrListenThread extends HandlerThread{
        private boolean enable = true;
        private Application target;
        public void setEnable(boolean enable) {
            this.enable = enable;
        }
        public AnrListenThread(String name, Application target) {
            super(name);
            this.target = target;
        }
        @Override
        public void run() {
            while (enable && !Thread.currentThread().isInterrupted()){
                try {
                    Message m = Message.obtain();
                    long sendTime = System.currentTimeMillis();
                    m.obj = sendTime;
                    hMainThread.sendMessage(m);
                    Thread.sleep(1000);
                    if (sendTime - MainThreadHandler.lastTime > 990 && MainThreadHandler.lastTime > 0){
                        sendANR(target);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private void listenAnr(Application target){
        hMainThread = new MainThreadHandler(target.getMainLooper());
        anrListenThread = new AnrListenThread("anr_listen", target);
        anrListenThread.start();
    }

    private void sendANR(Application target) {
        anrListenThread.setEnable(false);
        anrListenThread.interrupt();
        RestartApp.go(target, "anr", CMD_ANR_HANDLE);
    }
```

​	首先创建一个线程，这个线程负责往一个主线程的 Handler 中发送消息，并记录发送消息的时间，主线程的 handler 在收到消息之后会获取当前的时间，如果这两个时间差距过大的话就证明卡顿发生了，其实和上面的原理差不多。

