<center>Handler使用教程</center>
---
本文将从以下5个方面进行讲解：
> 1.定义     
> 2.作用    
> 3.工作原理     
> 4.使用场景    
> 5.总结          

# 定义
一个轻量的线程通信类

# 作用
实现不同线程之间的通信(常用于更新UI)   
 
# 工作原理
Handler发送消息(Message)到队列(MessageQueue,先进先出)，接着looper循环取出消息再交给Handler处理。

引用网络上的一张图片表示它们的关系：

![handler_looper_messagequeque](https://raw.githubusercontent.com/xyhrun/BlogRes/master/handler/handler_message_looper.png)
 

看到这里读者也许会有些疑问, 在这里我做出一些回答：
* 1.在哪个线程处理消息？
  > **handler内部的Looper在哪个线程创建，就在哪个线程处理消息。**
* 2.取出Message后如何发送给handler, 也就是Message和handler是如何关联的?
  > **当Handler发送Message时，会将其引用传给Message。在Looper取出Message后，通过Message拿到Handler引用来完成处理消息操作。**
  
关于它们三者之间的深入解读，日后我会写一篇文章。本篇主要介绍Handler使用方法

# 使用场景
* 1.线程间通信
  日常开发中，由于Android主线程无法做耗时操作（超过5秒引起ANR）以及子线程无法更改UI，所以我们需要在子线程做耗时任务，然后把结果传递到主线程更新UI。由于涉及到线程通信且这种场景极其多，于是Android封装了Handler方便我们使用。

  > 需求：查询天气信息并显示
  步骤：
1.在主线程和子线程分别创建Handler
2.用户点击查询按钮后 主线程通知子线程查询天气信息
3.子线程查询完天气信息后将结果传递到主线程并更新UI

  下面是代码：

  资源文件(strings.xml)

```
<resources>
    <string name="app_name">salen</string>
    <string name="check_weather">查询天气</string>
    <string name="is_checking_weather">正在查询中...</string>
    <string name="check_weather_success">查询成功</string>
</resources>
```
  
     
布局如下(activity_ui_handler.xml)：
  
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_centerInParent="true"
        android:id="@+id/tv_weather"
        android:layout_marginBottom="30dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="24sp"
        android:textColor="#11cd6e"/>

    <Button
        android:id="@+id/btn_check_weather"
        android:layout_below="@+id/tv_weather"
        android:layout_marginTop="30dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:text="@string/check_weather" />

</RelativeLayout>

```
  
功能代码如下：
  
```
package xyz.minelife.handler;

import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.os.Message;
import android.support.annotation.NonNull;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;

import xyz.minelife.salen.R;

/**
 * description:
 * created by salen
 * on 2019/6/29
 * blog: https://www.minelife.xyz
 */

public class HandlerActivity extends AppCompatActivity {

    public static final int MSG_CHECK_WEATHER_START = 0x100;
    public static final int MSG_CHECK_WEATHER_FINISH = 0x200;
    private static final String TAG = "HandlerActivity==";
    private Button mCheckWeatherBtn;
    private Handler mWorkHandler;
    private TextView mWeatherTv;

    private Handler mUIHandler = new Handler() {
        @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
            if (msg.what == MSG_CHECK_WEATHER_FINISH) {
                if (msg.obj instanceof String) {
                    updateWeatherInfo((String) msg.obj);
                }
            }
        }
    };

    private Handler.Callback mWorkCallBack = new Handler.Callback() {

        @Override
        public boolean handleMessage(Message msg) {
            Log.d(TAG, "handleMessage thread: " + Thread.currentThread().getName());
            if (msg.what == MSG_CHECK_WEATHER_START) {
                checkWeather();
            }
            return false;
        }

        private void checkWeather() {
            try {
                /* 模拟耗时工作 */
                Thread.sleep(1200);
                String weatherInfo = "城市：北京\n天气：多云\n温度：23~35度";

                /* 通知主线程更新天气信息 */  
                Message.obtain(mUIHandler, MSG_CHECK_WEATHER_FINISH, weatherInfo).sendToTarget();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        initView();
        initListener();
    }

    private void initView() {
        mCheckWeatherBtn = findViewById(R.id.btn_check_weather);
        mWeatherTv = findViewById(R.id.tv_weather);
    }

    private void initListener() {
        mCheckWeatherBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                checkTheWeather();
            }
        });
    }

    private void initData() {
        Thread WorkThread = new Thread() {
            @Override
            public void run() {
                super.run();
                Log.d(TAG, " work thread: " + getName());
                // 创建Looper,不调用它将会报错：Can't create handler inside thread that has not called Looper.prepare()  
                Looper.prepare();
                mWorkHandler = new Handler(mWorkCallBack);
//              等同于上面代码   
//              mWorkHandler = new Handler(Looper.myLooper(), mWorkCallBack);    
                Looper.loop();
            }
        };
        WorkThread.start();
    }

    private void checkTheWeather() {
        Log.d(TAG, "checkTheWeather");
        // 通知工作线程 查询天气信息
        Message.obtain(mWorkHandler, MSG_CHECK_WEATHER_START).sendToTarget();
        mCheckWeatherBtn.setText(R.string.is_checking_weather);
        mCheckWeatherBtn.setEnabled(false);
    }

    private void updateWeatherInfo(String weatherInfo) {
        Log.d(TAG, "updateWeatherInfo");
        Toast.makeText(this, getText(R.string.check_weather_success), Toast.LENGTH_SHORT).show();
        mWeatherTv.setText(weatherInfo);
        mCheckWeatherBtn.setEnabled(true);
        mCheckWeatherBtn.setText(R.string.check_weather);
    }

    @Override
    protected void onResume() {
        super.onResume();
        initData();
        mCheckWeatherBtn.setEnabled(true);
        mCheckWeatherBtn.setText(R.string.check_weather);
    }

}

```
  
效果如下：

![gif](https://raw.githubusercontent.com/xyhrun/BlogRes/master/handler/check_weather.gif)

日志如下：
 
```
06-30 17:08:36.673 3800-3856/xyz.minelife.salen D/HandlerActivity==: work thread: Thread-129
06-30 17:08:40.965 3800-3800/xyz.minelife.salen D/HandlerActivity==: checkTheWeather
06-30 17:08:40.965 3800-3856/xyz.minelife.salen D/HandlerActivity==: handleMessage thread: Thread-129
06-30 17:08:42.168 3800-3800/xyz.minelife.salen D/HandlerActivity==: updateWeatherInfo
06-30 17:08:45.384 3800-3800/xyz.minelife.salen D/HandlerActivity==: checkTheWeather
06-30 17:08:45.386 3800-3856/xyz.minelife.salen D/HandlerActivity==: handleMessage thread: Thread-129
06-30 17:08:46.587 3800-3800/xyz.minelife.salen D/HandlerActivity==: updateWeatherInfo

```
通过日志我们可以看到 创建Looper的线程和Handler处理消息的线程是同一个，都是Thread-129.
为什么它们在同一个线程？Handler和Looper之间有怎样的关系？
其实Handler内部维护了Looper的实例。当Handler发送消息到队列后，Looper则调用Looper.loop()方法无限循环取出队列里的消息，取出消息后执行msg.target.dispatchMessage(msg)方法。这里的target就是Handler，最终hander处理了这个消息。 在本代码中Looper是在子线程中创建的，loop()也运行其中，所以消息也是在该线程处理的。那么子线程的这个Looper是如何被赋值给Handler内部的looper呢？
1.在构造Handler的时候我们可以传入Looper
![构造Looper](https://raw.githubusercontent.com/xyhrun/BlogRes/master/handler/handler_transmit_looper.png )

![构造Looper](https://raw.githubusercontent.com/xyhrun/BlogRes/master/handler/handler_transmit_looper1.png )

2.在创建Handler的线程调用Looper.prepare()方法在该线程创建Looper的实例，然后Handler内部的looper.myLooper()拿到Looper。
![创建Looper](https://raw.githubusercontent.com/xyhrun/BlogRes/master/handler/handler_create_looper.png "关系图")

![返回Looper](https://raw.githubusercontent.com/xyhrun/BlogRes/master/handler/return_looper.png )

* 2.定时/延迟发送消息
  日常开发中，我们经常会遇到每隔一定时间发送一个请求这种需求。我们可以通过Handler来完成定时功能。
  > 需求：在上面需求的基础上我们再扩展一个功能，每隔5秒查询一次天气信息
  步骤： 
 
   1.子线程查询天气，查询完并返回结果给主线程
   2.主线程更新完界面信息，然后延迟5秒查询天气
   
   然后循环执行1、2步骤 实现定时功能。

  主要在updateWeatherInfo()增加以下代码：
```
 public static final int MSG_DELAY_TIME = 5 * 1000;
 private void updateWeatherInfo(String weatherInfo) {
        Log.d(TAG, "updateWeatherInfo");
        Toast.makeText(this, getText(R.string.check_weather_success), Toast.LENGTH_SHORT).show();
        mWeatherTv.setText(weatherInfo);
        mCheckWeatherBtn.setEnabled(true);
        mCheckWeatherBtn.setText(R.string.check_weather);

        // 延迟5秒发送消息
        mUIHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                checkTheWeather();
            }
        }, MSG_DELAY_TIME);

    }
```
 mUIHandler.postDelayed(r, time);方法有2个参数，一个是Runnable，一个是延迟的时间。 handler把消息发送到队列中，在延迟一定时间后，取出消息，然后执行消息(runnable)里的代码。
 
代码运行效果如下：

![gif](https://raw.githubusercontent.com/xyhrun/BlogRes/master/handler/post_delay.gif)

以下是日志：

![handler_post_delay](https://raw.githubusercontent.com/xyhrun/BlogRes/test/handler/handler_post_delay.png )

我们可以看到 主线程是每隔5秒都会发起 查询天气的请求。
另外我们发现了一个问题，当activity调用stop()方法后 仍然会发起请求。这不仅会影响用户体验，同时会造成内存泄漏。
所以我们需要在stop()里关闭消息循环
```
    @Override
    protected void onStop() {
        super.onStop();
        if (mWorkHandler.getLooper() != null) {
            // 移除队列内的所有消息
            mWorkHandler.getLooper().quit();
        }
        Log.d(TAG, "onStop()");
    }
```
最终效果如下：

![gif](https://raw.githubusercontent.com/xyhrun/BlogRes/master/handler/final.gif)
 


# 总结
本文对handler的使用方法做了简单的介绍，它主要用于线程切换和定时/延迟发送消息。由于本人技术有限，如有错误之处，欢迎读者指出批评。 另外我会写一篇文章来分析Handler、Looper、MessageQueue之间的关系，欢迎大家观看支持！