本文基于Android 12进行广播流程的分析，主要从四个方面：广播的注册、解注册、处理、结束四方面进行分析，会比较全面、按个人理解对广播进行解析。但个人能力有限，可能存在部分理解错误，但绝对是一篇理解Android 广播流程的好文。

在AMS中持有集合用于存储所有的广播，应用程序可以从向其注册和解注册广播。当应用发送广播时，AMS检查相关权限和特殊的`Intent`。然后再根据对应`IntentFilter`匹配到一个或多个`Receiver`，在应用进程回调其`onReceive`函数。

## 广播的注册

广播的注册分动态注册和静态注册两种，静态注册是指将`BroadcastReceiver`和`IntentFilter`写在配置清单里，Android系统在解析包信息的时候，会将其添加到`PackageManagerService`的`mComponentResolver`中，后续处理广播会从中查找匹配的接收者`BroadcastReceiver`。而动态注册，指的是在程序运行后，通过代码进行注册，下面分析的是动态注册的内容。

我们常在`Activity`或`Service`、甚至在`Application`中调用`registerReceiver`函数来动态注册广播，该函数其实来自他们的父类`ContextWrapper`中。`ContextWrapper`是`Context`的子类，我们会在介绍`Context`的文章介绍它们的关系。

```java
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    return mBase.registerReceiver(receiver, filter);
}
```

这里`Context`类型的`mBase`，在`Activity`的创建过程会被赋值为`ContextImpl`实例。

```java

public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    return registerReceiver(receiver, filter, null, null);
}
    
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
    String broadcastPermission, Handler scheduler) {
	return registerReceiverInternal(receiver, getUserId(),
        	filter, broadcastPermission, scheduler, getOuterContext(), 0);
}
```

经过`registerReceiver`重载函数，接着调用`registerReceiverInternal`函数。

##### `ContextImpl.registerReceiverInternal`

`registerReceiverInternal`函数,执行到注释1处，说明当前进程还存活着的，通过`LoadedApk`对象`mPackageInfo`的`getReceiverDispatcher`函数,根据`context`从`mReceivers`容器中获`ArrayMap`对象`map`，再以`receiver`为`Key`在`map`查询对应的`ReceiverDispatcher`对象。也就是说，一个`Context`对象,可能是`Activity`、`Application`、`Service`，会注册一个或多个广播接收者`BroadcastReceiver`,而一个广播接收者对应一个`ReceiverDispatcher`。后续广播的处理会通过`ReceiverDispatcher`找到对应的`BroadcastReceiver`，将`Intent`传递给它处理。

如果查询不到`ReceiverDispatcher`对象，说明之前没有添加过，则创建`ReceiverDispatcher`对象（内部会创建`InnerReceiver`对象）,按照前面获取的逻辑，进行逆操作，最终将`receiver`添加到`mReceivers`映射中。如果已经存在，则更新`Context`和`Handler`对象。

![image-20240105214017012](https://cdn.jsdelivr.net/gh/xxm-sz/dio/img/202401052140074.png)

有了`ReceiverDispatcher`对象后，通过其`getIIntentReceiver`函数获得`InnerReceiver`对象。`InnerReceiver`继承自`IIntentReceiver.Stub`,说明`InnerReceiver`会在进程之间传递。

注释2，相对于注释1，少传递了`Instrumentation`对象，我们知道`Instrumentation`使用来监视系统与应用程序之间的交互的，注释2处，由于应用程序未启动完毕，所以不需要。

注释3调用了AMS的`registerReceiverWithFeature`函数。这时候离开了应用所在的进程。总结的说，这里缓存我们注册的`BroadcastReceiver`对象,将相关信息封装成`IIntentReceiver`对象，传递给了`AMS`。

```java
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context, int flags) {
    IIntentReceiver rd = null;
    if (receiver != null) {
        //应用已启动
        if (mPackageInfo != null && context != null) {
            if (scheduler == null) {
                //主线程的H对象，用于接收广播用于保持receivcer广播顺序到达问题
                scheduler = mMainThread.getHandler();
            }
            //注释1：已注册，则获取旧的对象，否则新创建
            rd = mPackageInfo.getReceiverDispatcher(
                receiver, context, scheduler,
                mMainThread.getInstrumentation(), true);
        } else {
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            //注释2
            rd = new LoadedApk.ReceiverDispatcher(
                    receiver, context, scheduler, null, true).getIIntentReceiver();
        }
    }
    try {
        //注释3,返回值是第一个匹配IntentFilter的黏性广播的Intent或者null
        final Intent intent = ActivityManager.getService().registerReceiverWithFeature(
                mMainThread.getApplicationThread(), mBasePackageName, getAttributionTag(), rd,
                filter, broadcastPermission, userId, flags);
        if (intent != null) {
            intent.setExtrasClassLoader(getClassLoader());
            intent.prepareToEnterProcess();
        }
        return intent;
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

##### `AMS.registerReceiverWithFeature`

我们知道，一个广播接收者（`BroadcastReceiver`）可能会多个匹配条件（`IntentFilter`）。`AMS`的`registerReceiverWithFeature`函数的主要功能是处理这层关系和存储它们。我们将涉及的几个重要成员用下图的关系表达出来。

![image-20240105201502308](https://cdn.jsdelivr.net/gh/xxm-sz/dio/img/202401052015377.png)

* `mRegisteredReceivers`：AMS的成员变量，类型为`HashMap<IBinder, ReceiverList>`,`Hash Key`是`IIntentReceiver`的`IBinder`对象，也就是客户端`BroadcastReceiver`在客户端的代表。而`Value`就是`ReceiverList`对象。
* `ReceiverList`：继承自`ArrayList<BroadcastFilter>`,用于存储`BroadcastFilter`对象,表示一个`BroadReceiver`所对应的多个`IntenerFilter`的关系。
* `BroadcastFilter`: 内部封装了`IntentFilter`,可以表示为`IntentFilter`在系统层的代表。

* `mReceiverResolver`：存储`BroadcastFilter`,表示当前所有的`IntentFilter`对象。后续发送广播时，如果广播没有`component`，需要通过它区解析匹配的接收者。

`registerReceiverWithFeature`函数首先根据`receiver.asBinder()`在`mRegisteredReceivers`映射中查找是否有历史的`ReceiverList`对象。如果没有历史记录，则创建`ReceiverList`对象，并添加到`mRegisteredReceivers`中和`receiver`所在进程的`ProcessRecord`的`mReceivers`中。**一个应用程序允许创建最多的`ReceiverList`对象数量是1000个**。

接着会将相关信息，主要是`IntentFilter`,封装成`BroadcastFilter`对象。第一次会将该`BroadcastFilter`对象添加到对应的`ReceiverList`对象。也会添加到`mReceiverResolver`中。

`registerReceiverWithFeature`函数的另一个功能就是对黏性广播的处理。我们知道广播分三种类型：无序广播、有序广播、黏性广播。黏性广播的处理时机在此处，其他两个类型的处理可以看后文**广播的发送**小节。

先从AMS的`mStickyBroadcasts`数组中找出发给当前用户的`Intent`数组`stickyIntents`，包括发给当前用户和所有用户的黏性`Intent`。然后再在`stickyIntents`数组中找出与`IntentFilter`匹配的所有`Intent`，存储到`allSticky`中。然后根据`allSticky`数组，创建出每个`Intent`对应的`BroadcastRecord`（一个`BroadcastRecord`代表着一个广播），并将它们的`sticky`属性赋值为`true`，表示是黏性广播。然后加入到并行广播队列中，调用队列的 `scheduleBroadcastsLocked`的函数执行广播的处理流程。总结的说，就是找出与当前`IntentFilter`匹配的黏性广播，交给其`receiver`处理。

到这里，广播的注册就完成了。

![image-20240105225704698](https://cdn.jsdelivr.net/gh/xxm-sz/dio/img/202401052257761.png)

## 广播的解注册

有了对广播注册流程的了解，那么对广播的解注册流程理解，会容易很多。也就是在注册过程中，在`LoadedApk`对象加入到`mReceiver`中，现在就要将其移除，并重置相关属性。在AMS中，注册时是加入`mRegisteredReceivers`和`ReceiverList`，以及`mReceiverResolver`,那么解注册就是从它们移除。

回到`ContextWrapper`的`unregisterReceiver`函数。

```java
//ContextWrapper
public void unregisterReceiver(BroadcastReceiver receiver) {
    mBase.unregisterReceiver(receiver);
}

//ContextImpl
public void unregisterReceiver(BroadcastReceiver receiver) {
    if (mPackageInfo != null) {
        //LoadedApk
        IIntentReceiver rd = mPackageInfo.forgetReceiverDispatcher(
                getOuterContext(), receiver);
        try {
            //AMS
            ActivityManager.getService().unregisterReceiver(rd);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    } else {
        throw new RuntimeException("Not supported in system context");
    }
}
```

`LoadedApk`类`forgetReceiverDispatcher`函数的处理逻辑是和注册时`getReceiverDispatcher`函数反着来。根据context对象在`mReceivers`映射中获取当前`context`的所有`BroadcastReceiver`对应的`ReceiverDispatcher`对象。这是得到的是一个`ArrayMap`对象`map`,再在该`map`中根据`BroadcastReceiver`对象获取对应的`ReceiverDispatcher`。假如存在的话，依次从`map`、`mReceiver`(没有其他`receiver`情况下)中移除。如果不存在`map`对象中,还要从已解注册列表`mUnregisteredReceivers`中查看是否存在，如果存在，报下异常。

接着调用AMS的`unregisterReceiver`函数。AMS的解注册稍微复杂一点。如果此时正有有序广播在处理，那么要调用广播队列的`finishReceiverLocked`函数，如果返回结果为`true`,则调用队列的`processNextBroadcastLocked`将广播传递给下一个`receiver`。然后将`ReceiverList`对象从`mRegisteredReceivers`中移除，`BroadcastFilter`对象从`mReceiverResolver`中移除。

```java
public void unregisterReceiver(IIntentReceiver receiver) {
    final long origId = Binder.clearCallingIdentity();
    try {
        boolean doTrim = false;

        synchronized(this) {
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl != null) {
                //当前正在处理的有序广播，可以参考=》广播的结束小节
                final BroadcastRecord r = rl.curBroadcast;
                if (r != null && r == r.queue.getMatchingOrderedReceiver(r)) {
                    final boolean doNext = r.queue.finishReceiverLocked(
                            r, r.resultCode, r.resultData, r.resultExtras,
                            r.resultAbort, false);
                    if (doNext) {
                        doTrim = true;
                        //参考=》广播队列对广播的处理小节
                        r.queue.processNextBroadcastLocked(/* frommsg */ false,
                                /* skipOomAdj */ true);
                    }
                }
				//ProcessRecord移除
                if (rl.app != null) {
                    rl.app.mReceivers.removeReceiver(rl);
                }
                //从mRegisteredReceivers移除
                //从mReceiverResolver移除
                removeReceiverLocked(rl);
                if (rl.linkedToDeath) {
                        rl.linkedToDeath = false;
                        rl.receiver.asBinder().unlinkToDeath(rl, 0);
                    }
                }

                //清理进程
                if (doTrim) {
                    trimApplicationsLocked(false, OomAdjuster.OOM_ADJ_REASON_FINISH_RECEIVER);
                    return;
                }

    } finally {
        Binder.restoreCallingIden
```

## 广播的处理

`ContextWrapper`的`sendBroadcast`函数,调用了`ContextImpl`类`sendBroadcast`函数，进而调用了`AMS`的`broadcastIntentWithFeature`函数。

```java
public void sendBroadcast(Intent intent) {
    mBase.sendBroadcast(intent);
}

public void sendBroadcast(Intent intent) {
    warnIfCallingFromSystemProcess();
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    try {
        intent.prepareToLeaveProcess(this);
        //AMS
        ActivityManager.getService().broadcastIntentWithFeature(mMainThread.getApplicationThread(), getAttributionTag(), intent, resolvedType,null, Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false,false, getUserId());
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

### 1、AMS对广播的处理

调用`AMS`的`broadcastIntentWithFeature`函数。

`verifyBroadcastLocked`函数，对`Intent`的合法性进行检查，下面三种情况都会抛出异常。

* 携带文件描述符；
* 系统未启动完毕，当前Intent发送给所有的接收者。
* 开机升级广播。

如果`Intent`携带`Intent.FLAG_RECEIVER_FROM_SHELL`的`Flags`,即该广播时从Shell发送出来，且调用者不是系统进程或者`Shell`进程的`UID`，则需要将该Flags移除,定义为普通的广播。

```java
public final int broadcastIntentWithFeature(IApplicationThread caller, String callingFeatureId,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle resultExtras,
        String[] requiredPermissions, String[] excludedPermissions, int appOp, Bundle bOptions,
        boolean serialized, boolean sticky, int userId) {
   
    synchronized(this) {
   	   //对广播Intent合法性检查
       intent = verifyBroadcastLocked(intent);

        final ProcessRecord callerApp = getRecordForAppLOSP(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();

        final long origId = Binder.clearCallingIdentity();
        try {
            //下一个调用,会在调用重载函数
            return broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null, callingFeatureId,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, excludedPermissions, appOp, bOptions, serialized,
                    sticky, callingPid, callingUid, callingUid, callingPid, userId);
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
}
```

接着调用了`broadcastIntentLocked`函数。根据该函数内部处理逻辑顺序，大概出下面主要功能。

* Instant应用禁止发送广播给Instant应用接收者，即快应用之间禁止通过广播交互。
* 系统未启动完成（非升级过程），禁止通过广播启动新的进程。
* 广播发给特定用户，用户或者父用户没有运行，则不发送广播。

* 广播发送时，设置`options`数据，需要根据options情况，检查白名单、受限目标、后台启动Activity等权限进行检查。

* 检查广播`Action`，判断是否受保护的广播的`Action`，即声明在`framework/base/core/res/AndroidMenifest.xml`文件内的`protected-broadcast`,这些广播只能由系统应用发出，例如息屏`android.intent.action.SCREEN_OFF`。非系统应用发送这些广播会报异常。同时非系统应用在发送`AppWidgetManager.ACTION_APPWIDGET_CONFIGURE`或`AppWidgetManager.ACTION_APPWIDGET_UPDATE`广播时，只能发给自己，这些是操作自己的桌面小组件。
* 如果当前广播`Action`属于系统的隐式广播一种，例如`ACTION_LOCALE_CHANGED`,那么添加上`Intent.FLAG_RECEIVER_INCLUDE_BACKGROUND`标志，表示广播允许被静态注册的接收者收到。

* 如果广播`Action`属于系统的一些特殊广播，需要进行一些特别的处理,例如时间变化`ACTION_TIME_CHANGED`、`ACTION_TIMEZONE_CHANGED`;应用相关的变化`ACTION_PACKAGE_ADDED`、`ACTION_PACKAGE_DATA_CLEARED`、`ACTION_PACKAGE_CHANGED`等等很多。 

以上的逻辑基本是针对系统的一些机制进行特殊逻辑处理，而接下来就是普通广播进行处理，而广播又三种类型，分为黏性、无序、有序广播。

* 黏性广播
  * 需要检测发送者是否声明`BROADCAST_STICKY`权限，且该广播不能携带其他权限检查。

  * 不能指定具体的目标组件。即`Intent`不能设置`setComponent`函数。

  * 非全局的黏性广播不能与已存在全局的黏性广播相同（冲突）。

  * 黏性广播的处理：

    从`mStickyBroadcasts`获取当前用户所有的黏性广播`stickies`，我们知道，广播注册时会从`mStickyBroadcasts`中取出黏性广播发给广播队列进行处理。根据当前黏性广播的`action`在`stickies`映射中查询，查看是否有相同`action`的黏性广播（`list`集合）,如果有且`IntentFilter`相同，则替代旧的，不相同则添加到`list`中。也就是说黏性广播发送只会添加到黏性广播容器中，在广播接收者注册时候才被处理。

接下来，与当前广播Intent匹配的广播接收者，静态注册会被收集到`receivers`集合中（`FLAG_RECEIVER_REGISTERED_ONLY`没有设置），动态注册的会被收集到`registerReceiver`容器。

* 无序广播

  如果当前广播属于普通，无序的，且`registerReceiver`集合有收集到匹配的接收者。

  1. 如果广播发送者是系统应用，需要调用`checkBroadcastFromSystem`函数检测一下广播的action，做一些警告。
  2. 根据`Intent`的`flags`获取当前广播属于前台广播队列或后台广播队列。
  3. 广播相关信息封装成`BroadcastRecord`对象,**添加到广播队列的并行集合中`mParallelBroadcasts`**，表示接收者之间可以同时处理该广播。
  4. 调用广播队列`BroadcastQueue`的``scheduleBroadcastsLocked`函数。

那么接下来，还有静态注册的广播接收者（假如有的话）和有序广播没有处理。在处理完无序广播之后，`registerReceiver`会被清空。根据所有接收者（`receivers`和`registerReceiver`）的优先权合并到同一个同一个队列中，优先权高的放在前面。这里面有几种情况：

* 如果广播是无序广播，那么不会有合并到同一个队列的动作，因为有判断条件。这时候，接下来处理的是无序广播被静态注册接收者处理的部分。
* 如果广播是有序广播，那么不会执行无序广播的处理逻辑，`receivers`和`registerReceiver`被合并在一起。然后就是下面的有序广播处理逻辑。

* 有序广播

  1. 如果广播发送者是系统应用，需要调用`checkBroadcastFromSystem`函数检测一下广播的action，做一些警告。
  2. 根据`Intent`的`flags`获取当前广播属于前台广播队列或后台广播队列。
  3. 广播相关信息封装成`BroadcastRecord`对象,**添加到广播队列的顺序集合中`mOrderedBroadcasts`**。
  4. 调用广播队列`BroadcastQueue`的``scheduleBroadcastsLocked`函数。

也就是说AMS的`broadcastIntentLocked`函数主要处理一些特殊的`Action`。然后根据广播类型：黏性、无序、有序，添加到队列的不同集合中，然后调用广播队列`BroadcastQueue`的`scheduleBroadcastsLocked`函数对它们进一步处理。

而广播队列分三种类型：前台（优先级最高）、后台（普通优先级）、长广播（耗时很长，例如开机广播）

**特殊Flags处理**：

* 默认会给广播`Intent`对象添加上`Intent.FLAG_EXCLUDE_STOPPED_PACKAGES`，表示广播不发送给处于停止状态（没有在运行）的`BroadcastReceiver`。意味着正常情况下，如果我们程序被退出，将无法接收任何广播。
* `FLAG_RECEIVER_REGISTERED_ONLY`表示广播只能被动态注册的接收者接收。

###  2、广播队列对广播的处理

`scheduleBroadcastsLocked`函数调用`mHandler`对象发送一个`BROADCAST_INTENT_MSG`消息。

```java
//BroadcastQueue xxm
public void scheduleBroadcastsLocked() {
    if (mBroadcastsScheduled) {//避免重复执行
        return;
    }
    //BroadcastHandler
    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
    mBroadcastsScheduled = true;
}
```

`BroadcastHandler`的`handleMessage`函数对`BROADCAST_INTENT_MSG`消息的处理，直接调用了`processNextBroadcast`函数。

```java
#BroadcastHandler xxm
public void handleMessage(Message msg) {
    switch (msg.what) {
        case BROADCAST_INTENT_MSG: {
            processNextBroadcast(true);//执行该函数
        } break;
        case BROADCAST_TIMEOUT_MSG: {
            synchronized (mService) {
                broadcastTimeoutLocked(true);
            }
        } break;
    }
}

private void processNextBroadcast(boolean fromMsg) {
    synchronized (mService) {
        processNextBroadcastLocked(fromMsg, false);
    }
}
```

`processNextBroadcastLocked`函数第一步是处理无序广播，即将`mParallelBroadcasts`列表中的广播发送给它们的接收者进行处理，这里调用了`deliverToRegisteredReceiverLocked`函数。

```java
//Broadcast对象r来自mParallelBroadcasts集合，N即为receivers长度
for (int i=0; i<N; i++) {
    Object target = r.receivers.get(i);
    deliverToRegisteredReceiverLocked(r,
            (BroadcastFilter) target, false, i);
}
```

第二步是对有序广播进行处理：

1. 根据`mPendingBroadcast==null`来判断当前是否正在等待接收者进程完成启动来处理广播。`mPendingBroadcast`会在后续进程后被赋值。如果`mPendingBroadcast`不为`null`，且进程活着，则继续等待,也就是说，当进程启动完毕之后，处理完广播后，在某个地方会重置`mPendingBroadcast`，并走到此处。如果进程死亡，则指向下一个接收者，重置`mPendingBroadcast`。

2. 获取下一个广播,`getNextBroadcastLocked`函数获取。

   优先闹钟广播，接着到期广播，最后有序广播。如果没有获取到广播，调整odj和回收一些资源，那么执行到此处就结束了。

   ```java
   final long now = SystemClock.uptimeMillis();
   //优先获取闹钟广播，接着到期广播，最后有序广播
   r = mDispatcher.getNextBroadcastLocked(now);//获取下一个BroadcastRecord
   if (r == null) {//没有广播需要处理
       ...
       return;
   }
   ```

3. 丢弃早期的超时广播，即当前时间已经超过了广播约定的处理时间`2 * mConstants.TIMEOUT * numReceivers`，那么就结束强行结束该广播。

   ```java
   //当前时间超过了广播约定的处理时间，结束该广播
   int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
   if (mService.mProcessesReady && !r.timeoutExempt && r.dispatchTime > 0) {
       if ((numReceivers > 0) &&
               (now > r.dispatchTime + (2 * mConstants.TIMEOUT * numReceivers))) {
           //最终调用finishReceiverLocked，参考广播的结束
           broadcastTimeoutLocked(false); // forcibly finish this broadcast
           forceReceive = true;
           r.state = BroadcastRecord.IDLE;
       }
   }
   ```

4. 如果广播`BraodcastRecord`对象没有接收者（`receivers ==null`），或接收者都已经处理完毕或跳过（`nextReceiver >= numReceivers`），或者广播被放弃`resultAbort`、或因为超时被强制丢弃`forceReceive`，都判断为广播在当前`receiver`处理结束,调用`performReceiveLocked`函数。也就意味着一个**有序广播处理结束**。

   ```java
   //广播的默认状态是IDLE，也就是位对广播处理
   if (r.state != BroadcastRecord.IDLE) {
       return;
   }
   
   //四个条件来判断广播在所有的receiver中处理完毕
   if (r.receivers == null || r.nextReceiver >= numReceivers
           || r.resultAbort || forceReceive) {
       // Send the final result if requested
       if (r.resultTo != null) {
           boolean sendResult = true;
   
           // splitToken是用于处理延期广播的一个特殊计数
           if (r.splitToken != 0) {
               int newCount = mSplitRefcounts.get(r.splitToken) - 1;
               if (newCount == 0) {
                   mSplitRefcounts.delete(r.splitToken);
               } else {
                   sendResult = false;
                   mSplitRefcounts.put(r.splitToken, newCount);
               }
           }
           if (sendResult) {
               if (r.callerApp != null) {
                   mService.mOomAdjuster.mCachedAppOptimizer.unfreezeTemporarily(
                           r.callerApp);
               }
               try {
                   //广播下个处理的地方
                   performReceiveLocked(r.callerApp, r.resultTo,
                           new Intent(r.intent), r.resultCode,
                           r.resultData, r.resultExtras, false, false, r.userId);
                   r.resultTo = null;
               } catch (RemoteException e) {
                   r.resultTo = null;
               }
           }
       }
   	//将广播添加到历史消息
       cancelBroadcastTimeoutLocked();
       addBroadcastToHistoryLocked(r);
       if (r.intent.getComponent() == null && r.intent.getPackage() == null
               && (r.intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
           mService.addBroadcastStatLocked(r.intent.getAction(), r.callerPackage,
                   r.manifestCount, r.manifestSkipCount, r.finishTime-r.dispatchTime);
       }
       mDispatcher.retireBroadcastLocked(r);
       r = null;
       looped = true;
       continue;
   }
   ```

   如果当前需要将结果传递给调用者，那么需要调用`performReceiveLocked`函数。

5. `Broadcast`对象`deferred`属性为`true`时，说明当前广播是从延期广播数组获取的，不需要再进行延期判断。为`false`时，判断当前广播后续接收者的进程对广播是否需要延期处理（如果之前处理广播所花费的时间超过约定时间，该进程会被记录）

   ```java
   //在被加入延期广播的时候为true，默认为false
   if (!r.deferred) {
       //广播下一个接收者的进程uid
       final int receiverUid = r.getReceiverUid(r.receivers.get(r.nextReceiver));
       //根据uid在mDeferredBroadcasts列表获取是否有延期的广播
       if (mDispatcher.isDeferringLocked(receiverUid)) {
           BroadcastRecord defer;
           if (r.nextReceiver + 1 == numReceivers) {//只有一个接收者，不需要设计splitToken
               defer = r;
               mDispatcher.retireBroadcastLocked(r);
           } else {//多个接收者
               //splitRecipientsLocked函数会检查当前广播剩下的receiver,获取于当前receiver相同进程的receiver列表，新建broadcast对象，把receiver列表加到该对象中，也就是这里的defer
               defer = r.splitRecipientsLocked(receiverUid, r.nextReceiver);
               // Track completion refcount as well if relevant
               if (r.resultTo != null) {
                   int token = r.splitToken;
                   if (token == 0) {
                       // 第一次给广播设计splitToken，本质了i++。叠加mSplitRefcounts计数
                       r.splitToken = defer.splitToken = nextSplitTokenLocked();
                       mSplitRefcounts.put(r.splitToken, 2);
                   } else {
                       //后续只需要增加mSplitRefcounts计数
                       final int curCount = mSplitRefcounts.get(token);
                       mSplitRefcounts.put(token, curCount + 1);
                   }
               }
           }
           //将广播添加到延期队列中,这里deferred会被设置为true;
           mDispatcher.addDeferredBroadcast(receiverUid, defer);
           r = null;
           looped = true;
           continue;
       }
   }
   ```

   第2到第4，是在一个`do-while`循环，退出条件是`r!=null`,即找到下个要处理广播对象。在第2，是从闹钟广播队列、延期广播队列、顺序广播队列获取即将要执行的广播，如果没有广播，会直接跳出本函数；第4是当前广播处理结束，执行结束流程；第5是广播转入到延期广播队列中。经过前面的步骤，能获取到广播对象，则进入下面第6。

6. 获取`broadcast`下个接收者`receiver`。广播的`receiverTime`和`dispatchTime`在此被记录，会影响广播处理时间的计算，可能会导致receiver的进程加入延期队列，被特殊对待。
   
   1. 是`BroadcastFilter`对象，说明是动态注册的接收者，直接调用`deliverToRegisteredReceiverLocked`,与无序广播处理逻辑一致。
   2. 是`ResolveInfo`对象。说明是静态注册的，创建目标组件对象`ComponentName`。
   
7. 在第6步情况下，`receiver`是`ResolveInfo`对象，需要进行毕竟繁琐复杂的逻辑处理。
   1. 进行权限检查，这部分内容很多，感兴趣的朋友可以自己翻阅看看，主要是对系统的一些机制，广播发送者、接收者的合法权限检查。
   2. 进程已启动，执行`processCurBroadcastLocked`函数。该函数处理调到后面第3小节。
   3. 进程未启动，执行`AMS.startProcessLocked`启动进程，如果启动进程失败，需要结束接收者（参考广播的结束），执行下个广播处理（即处理本小结所有内容）。`mPendingBroadcast`被设置为当前广播对象，于开头等待广播所在进程启动完毕相照应。

回到第一步的无序广播处理中,调用`deliverToRegisteredReceiverLocked`函数主要是对权限进行处理,与前面的权限检查大致相同,然后调用了`performReceiveLocked`函数，与有序广播在第4对结束处理逻辑一致。

在`performReceiveLocked`函数中，如果`receiver`所在的进程已经启动，直接调用` app.thread.scheduleRegisteredReceiver`异步方式调用,这样保证广播`Intent`回到`receiver`所在进程进行处理。如果`app`未启动，则以同步方式调用` receiver.performReceive`。

```java
//BroadcastQueue xxm
void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
        Intent intent, int resultCode, String data, Bundle extras,
        boolean ordered, boolean sticky, int sendingUser)
        throws RemoteException {
    //异步方式调用receiver所在线程scheduleRegisteredReceiver函数
    if (app != null) {
        if (app.thread != null) {
           	...
            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
            	data, extras, ordered, sticky, sendingUser, app.getReportedProcState());
            ...
    } else {
            //同步方式
        	receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
    }
}
```

异步方式，我们定位到`ApplicationThread`的`scheduleRegisteredReceiver`函数。

```java
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
        int resultCode, String dataStr, Bundle extras, boolean ordered,
        boolean sticky, int sendingUser, int processState) throws RemoteException {
    updateProcessState(processState, false);
    receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
            sticky, sendingUser);
}       
```

所以，无论异步方式还是同步方式调用，最终还是回调`LoadedApk.ReceiverDispatcher.InnerReceiver`类的`performReceive`函数。

```java
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    final LoadedApk.ReceiverDispatcher rd;
    if (intent == null) {
        rd = null;
    } else {
        rd = mDispatcher.get();
    }
    if (rd != null) {
        //调用performReceive
        rd.performReceive(intent, resultCode, data, extras,
                ordered, sticky, sendingUser);
    } else {//Intent为null，finish掉receiver，逻辑与解注册unregisterReceiver函数相似
        IActivityManager mgr = ActivityManager.getService();
        try {
            if (extras != null) {
                extras.setAllowFds(false);
            }
            mgr.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

`ReceiverDispatcher`的`performReceive`函数

```java
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    final Args args = new Args(intent, resultCode, data, extras, ordered,
            sticky, sendingUser);
    ...
 	//分析2
 	if (intent == null || !mActivityThread.post(args.getRunnable())) {
        if (mRegistered && ordered) {//顺序广播
            IActivityManager mgr = ActivityManager.getService();
            args.sendFinished(mgr);//通知结束
        }
    }
}
```

`ReceiverDispatcher.performReceive`函数中将相关数据封装成`Args`对象,`Args`继承自`PendingResult`，代表了广播的处理结果。注意分析1，`if`语句，`Args`的`getRunnable`函数返回了`Runnable`对象，也就说这里执行了`Runnable`对象的`run`函数。这时切换到应用进程的主线程。

`Args.getRunnable`函数调用了`BroadcastReceiver`的`onReceive`函数，即我们的业务逻辑。接着就是调用`finish`函数，执行广播的结束操作，这部分可以参考后文**广播的结束**。

```java
public final Runnable getRunnable() {
    return () -> {
        //自定义的BroadcastReceiver对象
        final BroadcastReceiver receiver = mReceiver;
        final boolean ordered = mOrdered;
        final IActivityManager mgr = ActivityManager.getService();
        final Intent intent = mCurIntent;

        mCurIntent = null;
        mDispatched = true;
        mRunCalled = true;
        //广播异常、接收者异常、丢弃
        if (receiver == null || intent == null || mForgotten) {
            if (mRegistered && ordered) {
                sendFinished(mgr);//有序广播的回调
            }
            return;
        }

        try {
            ClassLoader cl = mReceiver.getClass().getClassLoader();
            intent.setExtrasClassLoader(cl);
intent.prepareToEnterProcess(ActivityThread.isProtectedBroadcast(intent,
                    mContext.getAttributionSource());
            setExtrasClassLoader(cl);
            //设置了处理结果
            receiver.setPendingResult(this);
            //回调onReceive函数
            receiver.onReceive(mContext, intent);
        } catch (Exception e) {
            //异常，有序广播通知结束
            if (mRegistered && ordered) {
                sendFinished(mgr);
            }
             ...
        }
        //onReceiver函数前设置了
        if (receiver.getPendingResult() != null) {
            finish();
        }
    };
}
```

### 3、`processCurBroadcastLocked`

在第2步中，有序广播在对`Receiver`进行处理的时候，如果进程已经启动，会调用`processCurBroadcastLocked`函数。该函数进又调用了`ApplicationThread`的`scheduleReceiver`函数。
```java
  private final void processCurBroadcastLocked(BroadcastRecord r,
            ProcessRecord app) throws RemoteException {

    final IApplicationThread thread = app.getThread();
	...
    r.receiver = thread.asBinder();
    r.curApp = app;
    final ProcessReceiverRecord prr = app.mReceivers;
    prr.addCurReceiver(r);
	...
    // Tell the application to launch this receiver.
    r.intent.setComponent(r.curComponent);

    boolean started = false;
    try {
		...
        thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,
                mService.compatibilityInfoForPackage(r.curReceiver.applicationInfo),
                r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId,
                app.mState.getReportedProcState());
        started = true;
    } finally {
        if (!started) {
            r.receiver = null;
            r.curApp = null;
            prr.removeCurReceiver(r);
        }
    }
}
```

`ApplicationThread`的`scheduleReceiver`函数将相关信息封装成了`ReceiverData`类型的对象。`ReceiverData`继承自`PendingResult`，也代表了广播在当前`BroadcastReceiver`的处理结果,广播结束会调用它的`finish`函数。后面再看看finish函数做了什么。接着给`ActivityThread`的`H`对象发`RECEIVER`消息。

```java
//这里的info是通过Resolver解析出来的，存储了BroadcastReceiver信息
//sync顺序与无序==同步与异步
public final void scheduleReceiver(Intent intent, ActivityInfo info,
        CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
        boolean sync, int sendingUser, int processState) {
    updateProcessState(processState, false);
    ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
            sync, false, mAppThread.asBinder(), sendingUser);
    r.info = info;
    r.compatInfo = compatInfo;
    sendMessage(H.RECEIVER, r);
}
```
`H.handleMessage`的`RECEIVER`分支调用了`ActivityThread`的`handleReceiver`函数，也就说，现在切回到应用的主线程。函数中通过`packageInfo.getAppFactory().instantiateReceiver`函数通类加载机制创建了`BroadcastReceiver`对象，也就是我们自定义的广播接收者，接着回调了`onReceive`函数,将`Intent`对象传递给我们，广播处理流程也到此结束了。

```java

 private void handleReceiver(ReceiverData data) {
	...
    String component = data.intent.getComponent().getClassName();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);

    IActivityManager mgr = ActivityManager.getService();

    Application app;
    BroadcastReceiver receiver;
    ContextImpl context;
    try {
        app = packageInfo.makeApplication(false, mInstrumentation);
        context = (ContextImpl) app.getBaseContext();
        if (data.info.splitName != null) {
            context = (ContextImpl) context.createContextForSplit(data.info.splitName);
        }
        if (data.info.attributionTags != null && data.info.attributionTags.length > 0) {
            final String attributionTag = data.info.attributionTags[0];
            context = (ContextImpl) context.createAttributionContext(attributionTag);
        }
        //类加载机制所必须的信息
        java.lang.ClassLoader cl = context.getClassLoader();
        data.intent.setExtrasClassLoader(cl);
        data.intent.prepareToEnterProcess(
                isProtectedComponent(data.info) || isProtectedBroadcast(data.intent),
                context.getAttributionSource());
        data.setExtrasClassLoader(cl);
        //通过类加载机制创建我们的BroadcastReceiver对象,packageInfo.getAppFactory返回AppComponentFactory对象
        receiver = packageInfo.getAppFactory()
                .instantiateReceiver(cl, data.info.name, data.intent);
    } catch (Exception e) {
        data.sendFinished(mgr);
    }

    try {
        //线程内的全局单例，应该在某个地方会读取判断
        sCurrentBroadcastIntent.set(data.intent);
        //设置处理结果
        receiver.setPendingResult(data);
        //回调我们自定义的onReceive方法。
        receiver.onReceive(context.getReceiverRestrictedContext(),
                data.intent);
    } catch (Exception e) {
       
        data.sendFinished(mgr);
       
    } finally {
        sCurrentBroadcastIntent.set(null);
    }
	//前面默认设置了data，所以默认情况接收处理，如果不想广播结束，再次调用setResult
    if (receiver.getPendingResult() != null) {
        data.finish();
    }
}
//AppComponentFactory
public @NonNull BroadcastReceiver instantiateReceiver(@NonNull ClassLoader cl,
        @NonNull String className, @Nullable Intent intent)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (BroadcastReceiver) cl.loadClass(className).newInstance();
}
```

![image-20240108210255101](https://cdn.jsdelivr.net/gh/xxm-sz/dio/img/202401082102151.png)

## 广播的结束

按照前面的分析，广播Intent传递给接收者的`onReceive`函数之后，都会调用`PendingRsult`的`finish`函数来结束当前广播。

如果程序当前工作队列还有要执行的工作，则添加一个执行`sendFinished`函数任务到队列，排队等待执行。否则就立即执行`sendFinished`函数。

```java
//PendingRsult
public final void finish() {
    if (mType == TYPE_COMPONENT) {//ReceiverData创建时，被设置为TYPE_COMPONENT,Args创建时TYPE_REGISTERED
        final IActivityManager mgr = ActivityManager.getService();
        //QueuedWork用于跟踪应用全局的工作是否结束
        if (QueuedWork.hasPendingWork()) {
            QueuedWork.queue(new Runnable() {
                @Override public void run() {
                    sendFinished(mgr);
                }
            }, false);
        } else {
            sendFinished(mgr);
        }
    } else if (mOrderedHint && mType != TYPE_UNREGISTERED) {//mOrderedHint为true表示顺序广播，TYPE_UNREGISTERED表示当前结注册
        final IActivityManager mgr = ActivityManager.getService();
        sendFinished(mgr);
    }
}
```

如果当前广播已经执行过结束流程，那么`mFinished`设置为`true`，再次执行就会抛出异常。接着根据广播的类型是有序还是无序，传递不同参数给`ActivityManagerService`的`finishReceiver`函数，通知AMS结束广播。

```java
public void sendFinished(IActivityManager am) {
    synchronized (this) {
        if (mFinished) {
            throw new IllegalStateException("Broadcast already finished");
        }
        mFinished = true;

        try {
            if (mResultExtras != null) {
                mResultExtras.setAllowFds(false);
            }
            if (mOrderedHint) {//有序
                am.finishReceiver(mToken, mResultCode, mResultData, mResultExtras,
                        mAbortBroadcast, mFlags);
            } else {//无序
                am.finishReceiver(mToken, 0, null, null, false, mFlags);
            }
        } catch (RemoteException ex) {
        }
    }
}
```

在广播的解注册，我们分析过AMS的`unregisterReceiver`函数，与这里的`finishReceiver`函数类似。根据广播`Intent`的`flags`查询广播所在队列，然后再广播队列中查出当前广播`BroadcastRecord`对象，调用其`finishReceiverLocked`函数，结束掉当前广播，如果还有其他接收者在等待，调用`processNextBroadcastLocked`函数，执行下一个广播

```java
public void finishReceiver(IBinder who, int resultCode, String resultData,
        Bundle resultExtras, boolean resultAbort, int flags) {
    
    // Refuse possible leaked file descriptors
    if (resultExtras != null && resultExtras.hasFileDescriptors()) {
        throw new IllegalArgumentException("File descriptors passed in Bundle");
    }

    final long origId = Binder.clearCallingIdentity();
    try {
        boolean doNext = false;
        BroadcastRecord r;
        BroadcastQueue queue;

        synchronized(this) {
            //根据Intent的Flags获取广播队列
            if (isOnOffloadQueue(flags)) {
                queue = mOffloadBroadcastQueue;
            } else {
                queue = (flags & Intent.FLAG_RECEIVER_FOREGROUND) != 0
                        ? mFgBroadcastQueue : mBgBroadcastQueue;
            }
			//获取当前正在处理的广播，也就是现在接收者对应的广播
            r = queue.getMatchingOrderedReceiver(who);
            if (r != null) {
                doNext = r.queue.finishReceiverLocked(r, resultCode,
                    resultData, resultExtras, resultAbort, true);
            }
            if (doNext) {//如果有，执行下一个广播。参考=>广播队列对广播的处理小节
                r.queue.processNextBroadcastLocked(/*fromMsg=*/ false, /*skipOomAdj=*/ true);
            }
            // updateOomAdjLocked() will be done here
            trimApplicationsLocked(false, OomAdjuster.OOM_ADJ_REASON_FINISH_RECEIVER);
        }

    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```

`finishReceiverLocked`函数主要是对`Broadcast`和`BroadcastReceiver`做一些清场工作，同时根据本次广播处理时间在核心系统应用做一些优化策略。同时也有可能需要等待下一个程序的`Receiver`来执行当前广播，也就是说广播只是在当前程序的接收者处理完，可能还有下个程序也要处理。

```java
public boolean finishReceiverLocked(BroadcastRecord r, int resultCode,
        String resultData, Bundle resultExtras, boolean resultAbort, boolean waitForServices) {
    final int state = r.state;
    final ActivityInfo receiver = r.curReceiver;
    final long finishTime = SystemClock.uptimeMillis();
    final long elapsed = finishTime - r.receiverTime;
    r.state = BroadcastRecord.IDLE;
   
    if (r.allowBackgroundActivityStarts && r.curApp != null) {
        if (elapsed > mConstants.ALLOW_BG_ACTIVITY_START_TIMEOUT) {
            //执行时间超过允许后台Activity启动时间，则没有启动，移除token即可
            r.curApp.removeAllowBackgroundActivityStartsToken(r);
        } else {
            //赋予更多的时间用来启动后台Activity
            postActivityStartTokenRemoval(r.curApp, r);
        }
    }
    //记录执行时间
    if (r.nextReceiver > 0) {
        r.duration[r.nextReceiver - 1] = elapsed;
    }

    // 广播在当前程序执行缓慢，非核心系统app下次处理广播时启动延迟策略
    //timeoutExempt为true表示不受超时影响，默认为false
    if (!r.timeoutExempt) {
        if (r.curApp != null
                && mConstants.SLOW_TIME > 0 && elapsed > mConstants.SLOW_TIME) {
            if (!UserHandle.isCore(r.curApp.uid)) {
                //将进程添加到缓慢列表中
                mDispatcher.startDeferring(r.curApp.uid);
            }
        }
    } 
	//当前广播的清场工作
    r.receiver = null;
    r.intent.setComponent(null);
    if (r.curApp != null && r.curApp.mReceivers.hasCurReceiver(r)) {
        r.curApp.mReceivers.removeCurReceiver(r);
        mService.enqueueOomAdjTargetLocked(r.curApp);
    }
    if (r.curFilter != null) {
        r.curFilter.receiverList.curBroadcast = null;
    }
    r.curFilter = null;
    r.curReceiver = null;
    r.curApp = null;
    mPendingBroadcast = null;

    r.resultCode = resultCode;
    r.resultData = resultData;
    r.resultExtras = resultExtras;
    if ( && (r.intent.getFlags()&Intent.FLAG_RECEIVER_NO_ABORT) == 0) {
        r.resultAbort = resultAbort;
    } else {
        r.resultAbort = false;
    }

    // 其他接收者正在等待当前接收者处理完
    if (waitForServices && r.curComponent != null && r.queue.mDelayBehindServices
            && r.queue.mDispatcher.getActiveBroadcastLocked() == r) {
        ActivityInfo nextReceiver;
        if (r.nextReceiver < r.receivers.size()) {
            Object obj = r.receivers.get(r.nextReceiver);
            nextReceiver = (obj instanceof ActivityInfo) ? (ActivityInfo)obj : null;
        } else {
            nextReceiver = null;
        }
        //相同进程的receiver不会执行到此处
        if (receiver == null || nextReceiver == null
                || receiver.applicationInfo.uid != nextReceiver.applicationInfo.uid
                || !receiver.processName.equals(nextReceiver.processName)) {
            //有后台服务，表示当前正在切换不同的程序的receiver来处理当前broadcast
            if (mService.mServices.hasBackgroundServicesLocked(r.userId)) {
                r.state = BroadcastRecord.WAITING_SERVICES;
                return false;
            }
        }
    }

    r.curComponent = null;

    // We will process the next receiver right now if this is finishing
    // an app receiver (which is always asynchronous) or after we have
    // come back from calling a receiver.
    return state == BroadcastRecord.APP_RECEIVE
            || state == BroadcastRecord.CALL_DONE_RECEIVE;
}
```

![image-20240109194936281](https://cdn.jsdelivr.net/gh/xxm-sz/dio/img/202401091949353.png)

## 总结

广播会发送到广播队列中不同集合。其中广播队列有三种类型，分别对应优先级从高到底：前台、后台、长广播队列类型。而广播又分三种：黏性广播、无序广播、有序广播。无序广播和有序广播的处理主要要发送广播的时候，而黏性广播则在广播接收者注册时候被处理。

一个应用程序允许注册最大的广播接收者是1000个。广播之间的传递也要经历各种权限检查，所以广播不适合在应用间用于频繁的交互。
