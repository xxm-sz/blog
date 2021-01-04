
硬件加速指的是利用CPU和GPU各自的特性，将绘制工作一分为二，CPU负责复杂的逻辑运算，利用底层软件代码，将CPU不擅长的图形计算转换成GPU专用指令，由GPU完成,从而提高绘制速度。
## 开启硬件加速
在分析Android的绘制流程中，定位到ViewRootImpl类的`draw`函数，会发现在这里会有绘制的两个分支，一个走的是硬件绘制` mAttachInfo.mThreadedRenderer.draw`函数，一个是软件绘制`drawSoftware`函数。而它们的判断走向条件是`mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()`为true走硬件绘制，false软件绘制。定位到`mThreadedRenderer`初始化的地方`enableHardwareAcceleration`函数；

1、当application处于兼容模式，不允许硬件加速。什么是兼容模式？
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab66a973f2934f9fb628e50cd98c74f0~tplv-k3u1fbpfcp-watermark.image)

2、请求开启硬件加速，即设置了`FLAG_HARDWARE_ACCELERATED`属性。请求了硬件加速，不保证一定开启了硬件加速，还需要根据后面的情况。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ed3ebb92dd24d309c7269df20003cbc~tplv-k3u1fbpfcp-watermark.image)

如何设置该属性，以请求硬件加速呢？

应用级别，在AndroidManifest.xml文件，application节点添加下面属性。
```
//开启
<application android:hardwareAccelerated="true" ...>
//关闭
<application android:hardwareAccelerated="false" ...>
```
Activity 级别，在AndroidManifest.xml文件，activity节点添加下面属性。
```
//开启加速
<activity android:hardwareAccelerated="true" />
//关闭加速
<activity android:hardwareAccelerated="false" />
```
窗口级别，在Activity或Dialog的`onCreate`方法中，`setContentView`函数之前，调用下面代码开启硬件加速。
```
window.setFlags(WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED, 
	WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED)
```
通过代码开启的窗口级别硬件加速是没办法停止的。而View级别可以停用硬件加速，但无法开启加速。
```
view.setLayerType(View.LAYER_TYPE_SOFTWARE, null)
```

3、在请求了硬件加速之后，需要判断当前设备是否支持硬件加速。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f00a0702135a4008a0fb05f76b9be3fa~tplv-k3u1fbpfcp-watermark.image)

设备是真机，则支持是硬件加速；`SurfaceFlinger`服务开启失败，不支持硬件加速；若是虚拟机，其是OpenGL ES 2.0则支持硬件加速。

## 硬件加速绘制流程
硬件加速绘制流程只是[Android的绘制流程](https://juejin.cn/post/6913743020244336653)
的一个分支。在ViewRootImpl的`draw`函数，会判断是否开启硬件加速走不同的分支。
开启硬件加速函数调用流程：
```
ViewRootImpl.performDraw->ViewRootImpl.draw->ThreadedRender.draw->
ThreadedRender.updateRootDisplayList->ThreadedRender.updateViewTreeDisplayList->
View.updateDisplayListIfDirty->View.draw->....
```

在View的draw流程执行结束后，会通过`ThreadedRender.syncAndDrawFrame`函数将`DisplayList`交由GPU绘制到屏幕上。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5dd6de9390d74150839f2f7dfdee32ce~tplv-k3u1fbpfcp-watermark.image)




【参考文章】

[Android硬件加速原理与实现简介](https://tech.meituan.com/2017/01/19/hardware-accelerate.html)

[硬件加速](https://developer.android.com/guide/topics/graphics/hardware-accel?hl=zh-cn#top_of_page)
