## 前言
本文讲的是Settings相关开发经常用到的地方，主要有WIFI、蓝牙、系统或应用升级、音量调节、亮度调节、多语言切换等。在不同Android版本也会进行适配和踩坑提示，本文更多是一个指导性作用，扩展一下宽度，对Android不同的知识点有个大概的了解，在需要使用的时候，知道如何下手。
## WIFI设置
WIFI设置，主要是获取WIFI列表，判断WIFI强度，连接WIFI等...

这部分内容非常繁琐，每次我代码写到这里都好累，后续把它成AAR包，大家一起用.......
1. 声明权限

WIFI需要声明的权限比较多：
```
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
```
Android 6.0以上需要动态申请权限：
```
 requestPermissions(new String[]{Manifest.permission.ACCESS_FINE_LOCATION}, REQUEST_PERMISSION_CODE);
```
本文对于大多数权限只是告知同学们需要注意相关权限，处理逻辑还是需要同学们根据自己业务需求进行处理。毕竟文章只是指导性作用，而不是手把手教你实现需求。

2. 判断WIFI状态：
只有打开了WIFI，我们才可能进行下一步操作：
```
wifiManager = (WifiManager) getActivity().getSystemService(Context.WIFI_SERVICE);
isWifiEnabled = wifiManager.isWifiEnabled();
```
如果WIFI处于关闭状态，可以自己用代码打开或者提醒用户打开WIFI,这里通过代码打开：
```
//设置true为打开，false为关闭，操作结果
wifiManager.setWifiEnabled(true);
```
WIFI的打开和关闭结果，我们需要通过广播来监听：
```
IntentFilter intentFilter = new IntentFilter();   
intentFilter.addAction(WifiManager.WIFI_STATE_CHANGED_ACTION);
getActivity().registerReceiver(wifiReceiver, intentFilter);
```
3. 扫描WIFI

如果WIFI处于打开状态，可以直接调用`wifiManager.startScan()`开始扫描WIIF列表，或者在收到WIFI打开广播后扫描。`startScan()`在Android 9.0标记为过时，但官网还没有给出合适的接口调用。可以通过`wifiManager.getScanResults()`函数来获取扫描结果，但需要在获取扫描广播完成后。
```
IntentFilter intentFilter = new IntentFilter();   
intentFilter.addAction(WifiManager.WIFI_STATE_CHANGED_ACTION);
//添加此行
intentFilter.addAction(WifiManager.SCAN_RESULTS_AVAILABLE_ACTION);
getActivity().registerReceiver(wifiReceiver, intentFilter);

wifiReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {       
        if (intent.getAction().equals(WifiManager.SCAN_RESULTS_AVAILABLE_ACTION)) {
            list=wifiManger.getScanResults();
        } else if (intent.getAction().equals(WifiManager.WIFI_STATE_CHANGED_ACTION)) {
            if (wifiManager.getWifiState() == WifiManager.WIFI_STATE_ENABLED) {
                wifiManager.startScan();
            }
        }
    }
}
```
`getScanResults()`函数返回的持有`ScanResult`类型的集合，一般情况下，我们不会直接拿`ScanResult`来展示数据，而是封装自己的`Bean`。一个`ScanResult`元素代表着一个WIFI热点,WIFI热点可能存在重复，所以记得去重。下面是`ScanResult`的一些常用属性：
* SSID：WIFI名称
* BSSID:WIFI的MAC地址
* capabilities：用来描述WIFI的加密方式，身份验证，密码管理，我们也常通过该属性来判断是否加密。
* level：接收信号强度RSSI

如果展示信号强度，一般不直接通过level来显示，而是通过`WifiManager#calculateSignalLevel(int rssi, int numLevels)`计算结果来显示，`numLevels`表示将信号强度分为几层。

如果只是简单判断是否加密，可以通过下面判断：
```
if (scanResult.capabilities.trim().equals("") ||
            scanResult.capabilities.equals("[ESS]")) {
    //未加密，不需要密码
 }
```
4. 获取已保存过的WIFI
在扫描结束后，可以通过下面代码获得已保存过的代码：
```
List<WifiConfiguration> configurations = wifiManager.getConfiguredNetworks();
```
5. 连接WIFI
连接WIFI分很多种类型，有密码没有密码，不同的加密方式，是否已经保存过的。

**已经保存过的WIFI**

也就是要连接的WIFI存在第4步获取到的configurations集合中。
```
WifiConfiguration configuration=configurations.get(0);
wifiManager.enableNetwork(configuration.networkId, true)
```
连接状态我们可以通过`enableNetwork()`函数的返回值来处理，或者监听广播`WifiManager.NETWORK_STATE_CHANGED_ACTION`。

**未保存过的WIFI**

对于连接没有密码的WIFI，我们通过下面代码创建`WifiConfiguration`对象：
```
private WifiConfiguration createWifiConfigurationNoPassword(String wifiName) {
    WifiConfiguration config = new WifiConfiguration();
    config.allowedAuthAlgorithms.clear();
    config.allowedGroupCiphers.clear();
    config.allowedKeyManagement.clear();
    config.allowedPairwiseCiphers.clear();
    config.allowedProtocols.clear();
    config.SSID = "\"" + wifiName + "\"";
    config.wepKeys[0] = "";
    config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.NONE);
    config.wepTxKeyIndex = 0;
    return config;
}
```
有密码的可以通过下面代码创建`WifiConfiguration`对象：
```
public static WifiConfiguration createWifiConfigurationNeedPassword(WifiInfo wifiInfo) {
    WifiConfiguration config = new WifiConfiguration();
    config.allowedAuthAlgorithms.clear();
    config.allowedGroupCiphers.clear();
    config.allowedKeyManagement.clear();
    config.allowedPairwiseCiphers.clear();
    config.allowedProtocols.clear();
    config.SSID = "\"" + wifiInfo.getWifiName() + "\"";
    String capabilities = wifiInfo.getCapabilities();

    if (capabilities.toLowerCase().contains("wep")) // encryption wep
    {
        config.hiddenSSID = true;
        config.wepKeys[0] = "\"" + wifiInfo.getPassword() + "\"";
        config.allowedAuthAlgorithms.set(WifiConfiguration.AuthAlgorithm.SHARED);
        config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.CCMP);
        config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.TKIP);
        config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.WEP40);
        config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.WEP104);
        config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.NONE);
        config.wepTxKeyIndex = 0;
    }
    if (capabilities.toLowerCase().contains("wpa")) //encryption wpa
    {
        config.preSharedKey = "\"" + wifiInfo.getPassword() + "\"";
        config.hiddenSSID = true;
        config.allowedAuthAlgorithms.set(WifiConfiguration.AuthAlgorithm.OPEN);
        config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.TKIP);
        config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.WPA_PSK);
        config.allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.TKIP);
        config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.CCMP);
        config.allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.CCMP);
        config.status = WifiConfiguration.Status.ENABLED;
    }
    return config;
}
```
那如何知道WIFI是否需要密码呢？就是再第3步扫描WIFI列表时，`ScanResult`的`capabilities`属性。这个过程也是需要用户输入密码的，不然密码从何而来。这里的`WifiInfo`是对`ScanResult`的进一步封装而已。

最后就是连接WIFI了。
```
 int netId = wifiManager.addNetwork(createWifiConfigurationNeedPassword(wifiInfo));
 wifiManager.enableNetwork(wcgID, true)
```


## 经典蓝牙和低功耗蓝牙
之前相关文章，这里不再重复了...

[经典蓝牙](https://juejin.im/post/5cfdb788f265da1bb67a0c40)

[低功耗蓝牙](https://juejin.im/post/5d0876926fb9a07efa091a9d)
## 系统或应用升级
在Android开发，经常需要进行OTA升级，或者更新版本。
### 软件下载
如果在Android较低版本建议使用DownloadManger系统类来下载，`DownloadManger`自动支持断点续传等，操作较简单。前提是服务器支持断点续传。在Android 9.0版本，因为禁止明文HTTP传输,会报`Cleartext HTTP traffic to xxx not permitted”`问题，尽管配置了`cleartextTrafficPermitted`，大多数机器也是无效，需要谷歌更新补丁。
**使用Android 9.0以上建议自己实现下载或者第三方库**，如：
```
implementation 'com.yaoxiaowen:download:1.4.1'
```
那么，如何使用`DownloadManger`来下载呢？
```
DownloadManager manager = (DownloadManager) getSystemService(Context.DOWNLOAD_SERVICE);

DownloadManager.Request request = new DownloadManager.Request(Uri.parse(strURL));
//如果不存在下载路径，则建立该目录
Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS).mkdir() ;
//设置文件的存储路径，还可以通过其他方法设置不同目录
request.setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS,"ota.zip");
//设置在通知通知栏显示，还可以进一步设置标题，内容等
request.setNotificationVisibility(View.VISIBLE);
//指定在WiFi网络环境下载
request.setAllowedNetworkTypes(DownloadManager.Request.NETWORK_WIFI);

long netId = manager.enqueue(request);
```
可以通过`DownloadManager.Request`来配置一些信息，例如上文代码设置在通知栏显示和指定下载位置。

通过`enqueue()`函数将我们的请求发送给`DownloadManger`的队列，返回值`netId`可以用来在后续查询下载进度等相关信息。

下载完成之后，会发出广播通知，为此，我们需要监听广播：
```
BroadcastReceiver downloadReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        // TODO: download complete 
    }
};

IntentFilter intentFilter = new IntentFilter();
intentFilter.addAction(DownloadManager.ACTION_DOWNLOAD_COMPLETE);
registerReceiver(downloadReceiver, intentFilter);
```
在收到下载完成广播之后去升级系统。如果想在UI上显示进度条等信息，那么就需要查询下载进度了。
```
DownloadManager.Query query = new DownloadManager.Query();

Cursor cursor = manager.query(query.setFilterById(netId));

if(cursor!=null&&cursor.moveToFirst()){
    //下载的文件到本地的目录 
    String path=cursor.getString(cursor.getColumnIndex(DownloadManager.COLUMN_LOCAL_URI));
    //已经下载大小
    int downloadSize=cursor.getInt(cursor.getColumnIndex(DownloadManager.COLUMN_BYTES_DOWNLOADED_SO_FAR));
    //升级包大小
    int totalSize=cursor.getInt(cursor.getColumnIndex(DownloadManager.COLUMN_TOTAL_SIZE_BYTES));
    //下载状态
    int status=cursor.getInt(cursor.getColumnIndex(DownloadManager.COLUMN_STATUS));
}
```
通过`Cursor`来查询下载的相关状态，例如文件大小，已下载大小等等相关信息。如果要持续更新进度条，需要自己用Handler或者Timer等定时查询下载状态了。

记得声明网络权限和配置HTTP明文传输（Android 9.0上哦）

### 系统升级
系统升级只需要调用下面方法，`PACKAGE_SAVE_PATH`是我们的升级包路径。
```
 RecoverySystem.installPackage(getApplication(), new File(PACKAGE_SAVE_PATH));
```
在`AndroidManifest.xml`声明权限：
```
 <uses-permission android:name="android.permission.RECOVERY" />
 //在application标签增加下面属性，适配HTTP明文
  android:usesCleartextTraffic="true"
```

### 应用升级
应用升级主要是调用安装助手，但是下载APK后，要适配不同版本。

在Android 6.0以上，如果通过DownloadManager下载的，要通过查询接口获得下载路径的位置。
```
String path=cursor.getString(cursor.getColumnIndex(DownloadManager.COLUMN_LOCAL_URI));
```
在Android 7.0以上，需要配置文件访问权限。

1. 在`res`目录下的`xml`文件夹新建`share_paths.xml`文件。如果没有xml文件夹则新建，`share_paths.xml`命名随意，其内容为：
```
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-path
        name="external"
        path="" />
    <external-files-path
        name="Download"
        path="" />
</paths>
```
2. 在`AndroidManifest.xml`声明`provider`。
```
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="packgeName.fileProvider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/share_paths" />
</provider>
```
3. 在需要路径的地方通过下面代码获取：
```
uri = FileProvider.getUriForFile(context,
            "packageNam.fileProvider",
            new File(context.getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS), "xxx.apk"));
```
在Android 8.0以上，配置允许安装位置应用来源权限。
1. 在`AndroidManifest.xml`配置权限。
```
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
```
2. 检测是否允许安装未知来源应用。如果用户不允许则要跳转授权列表，允许的话安装应用。
```
//判断是否授权安装未知来源应用
context.getPackageManager().canRequestPackageInstalls();
//未授权的话要动态申请授权
ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.REQUEST_INSTALL_PACKAGES}, 1);
//用户拒绝授权的话，可以跳转到授权界面
Intent intent = new Intent(Settings.ACTION_MANAGE_UNKNOWN_APP_SOURCES, Uri.parse("package:" + getPackageName()));
        startActivityForResult(intent, 1);
```
在Android 9.0，如果下载APK使用HTTP,配置允许HTTP明文即可。与上面配置DownloadManger是一致的。
到此，就可以正常安装APK了。
```
Intent intentInstall = new Intent();
intentInstall.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intentInstall.setAction(Intent.ACTION_VIEW);
intentInstall.setDataAndType(uri, "application/vnd.android.package-archive");
intentInstall.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
context.startActivity(intentInstall);
```
## 音量设置
虽然平常我们对外说是音量设置，但还是有挺多种类型，例如音乐，通知等的声音。在Android中，设置音量相关的是AudioManger类，通过该类对象可以读取和设置相关属性。调节音量通常有两种操作：UI界面调节和物理按键上下调节。
### 声音类型
了解Audio的类型，才明确自己要设置的音量是什么类型的。有以下类型。
* AudioManger.STREAM_VOICE_CALL：通话
* STREAM_SYSTEM：系统
* STREAM_RING：手机铃声
* STREAM_ALARM：闹钟
* STREAM_NOTIFICATION：状态栏通知
* STREAM_DTMF：双音多频，个人理解应该拨号音
* STREAM_ACCESSIBILITY:提示
* STREAM_BLUETOOTH_SCO：通过蓝牙拨打电话的音量
* STREAM_TTS：扬声器
* STREAM_SYSTEM_ENFORCED：在某些国家用于识别强制系统声音，例如日本相机拍照声音

后面三个类型是隐藏的。
### UI界面设置
通过界面来设置音量大小，主要通过AudioManger相关api来设置。
```
audioManager = (AudioManager) getActivity().getSystemService(Context.AUDIO_SERVICE);
//获取媒体类型最大音量的值
maxVolume = audioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC);
//获取当前音量值
curVolume = audioManager.getStreamVolume(AudioManager.STREAM_MUSIC);
//直接设置媒体类型音量大小
audioManager.setStreamVolume(AudioManager.STREAM_MUSIC, volume, 0);
```
在`setStreamVolume（int streamType, int index, int flags）`方法中：
* flags:声音类型
* index:声音的大小，范围在0-100之间，100音量最大
* flags:主要有：`AudioManager.FLAG_PLAY_SOUND（调节音量是播放声音）`;`AudioManager.FLAG_SHOW_UI (调节时显示音量条)`;`0(什么也做)`。其他的可以自己参考一下文档。

调节音量还有另外一个api:
```
 audioManager.adjustStreamVolume(AudioManager.STREAM_MUSIC,AudioManager.ADJUST_LOWER,0);
 //原型
audioManager.adjustStreamVolume(int streamType, int direction, int flags) 
```
在原型函数主要看第二个参数，主要有三个值：
* AudioManager.ADJUST_LOWER：降低音量，类似按音量-键
* AudioManager.ADJUST_RAISE：调高音量，类似按音量+键
* AudioManager.ADJUST_SAME：保持不变，展示音量

**设置静音**
判断指定类型是否静音：
```
audioManager.isStreamMute(AudioManager.STREAM_MUSIC);
```
设置静音，在Android 6.0前：
```
audioManager.setStreamMute(int streamType, boolean state)
```
将state设为false为关闭静音，ture开启静音。获取AudioManger对象建议采用Application的上下文，避免踩坑。

Android 6.0之后：
```
audioManager.adjustStreamVolume(int streamType, int direction, int flags)
```
第二个参数的值可设为：`AudioManager.ADJUST_MUTE（开启静音）、AudioManager.ADJUST_UNMUTE（关闭静音）`。
### 物理按键设置音量
这个好像没什么好说，应用层能做的就是监听音量变化，更新UI界面。音量变化广播`android.media.VOLUME_CHANGED_ACTION`对应用层是隐藏的，所以需要以字符串来声明。
```
public static final String ACTION_VOLUME_CHANGED="android.media.VOLUME_CHANGED_ACTION";

volumeReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {                curVolume=audioManager.getStreamVolume(AudioManager.STREAM_MUSIC);
        Log.d(TAG,"curVolume:"+curVolume);
    }
};

IntentFilter intentFilter = new IntentFilter();   
intentFilter.addAction(ACTION_VOLUME_CHANGED);
getActivity().registerReceiver(volumeReceiver, intentFilter);
```
其他关于音量的操作建议亲自操刀。

## 多语言设置（国际化）
随着市场的全球化，app也要进行多语言适配，而不仅仅是支持中文。由于大多数App属于第三方应用,不属于系统应用。这里讲解app自身多语言切换。
### 1、建立资源文件
根据市场的需求，建立不同的语言资源文件。例如这里支持中文和英文。
![](https://user-gold-cdn.xitu.io/2019/7/27/16c3299386051fa4?w=227&h=202&f=png&s=9145)
### 2、获取系统语言
一般新安装app，最好的步骤就是跟随系统语言，在用户指定不同语言后，后续启动才加载指定语言。Android 7.1.1 之前和之后的获取方式不同：
```
Resources resources = getActivity().getResources();
Configuration configuration = resources.getConfiguration();
Locale locale = Locale.getDefault();
locale.getLanguage()
```
获得Locale对象之后，可以获取语言、国家等信息。
### 3、设置系统语言
Android 7.1.1之前和之后的设置方式并不相同，Android 7.1.1 为多语言用户提供增强的支持，让他们可以在设置中选择多个语言区域。

**Android 7.1.1之前**
```
configuration.setLocale(Locale.ENGLISH);
resources.updateConfiguration(configuration, resources.getDisplayMetrics());
```
然后重启相关Activity即可，语言对应就切换了。

**Android 7.1.1之后**
```
 configuration.setLocale(Locale.CHINESE);
 context.createConfigurationContext(configuration);
```
上面的`context`必须是Activity的context，而不能是Application，所以需要所有Activity继承一个基类BaseActivity，并重写attachBaseContext（）方法,这样就不用每个单独的Activity中重写并重写attachBaseContext()方法。
```
// BaseActivity
@Override
protected void attachBaseContext(Context newBase) {
    super.attachBaseContext(LanguageUtil.getLanguageContext(newBase));
}

// LanguageUtil
public static Context getLanguageContext(Context context) {
    SharedPreferences sharedPreferences = context.getSharedPreferences(file, Context.MODE_PRIVATE);

    String language = sharedPreferences.getString(key, "");

    if (language.length() == 0) { 
        //如果用户没有指定语言，返回默认的context
        return context;
    } else {
    //用户指定语言，返回新的context
     Configuration configuration = context.getResources().getConfiguration();
    if (language.equals(ENGLISH)) {
            configuration.setLocale(Locale.ENGLISH);
        } else {
            configuration.setLocale(Locale.CHINESE);
        }
        return context.createConfigurationContext(configuration);
    }
}
```
指定语言之后，同样需要重启Activity，切换语言才会生效。
## 亮度调节
Android系统默认带有两种调节模式：自动模式和手动模式。
如下图，打开为设置自动模式，关闭为手动模式。

![](https://user-gold-cdn.xitu.io/2019/7/27/16c325c55d3f1f26?w=292&h=133&f=png&s=16699)
### 1、声明权限
在AndroidManifest.xml文件声明如下权限：
```
<uses-permission android:name="android.permission.WRITE_SETTINGS"/>
```
按照Android Studio Lint 工具的提示，该权限仅对系统app授权。但在我的Nokia X7不声明该权限都可以正常调节亮度。如果需要，在AndroidManifest.xml根元素添加如下属性，然后用对应平台签名。
```
 android:sharedUserId="android.uid.system"
```
### 2、设置亮度模式
在app中需要将亮度模式设置为手动模式，才能调节亮度。
* 手动模式：`Settings.System.SCREEN_BRIGHTNESS_MODE_MANUAL`
* 自动模式：`Settings.System.SCREEN_BRIGHTNESS_MODE_AUTOMATIC`

示例代码：
```
public void setScreenManualMode() {
    ContentResolver contentResolver = getActivity().getContentResolver();
    try {
    //获取当前系统亮度模式
    int mode = Settings.System.getInt(contentResolver,
    Settings.System.SCREEN_BRIGHTNESS_MODE);
    //如果当前模式是自动模式，则设为手动模式
    if (mode == Settings.System.SCREEN_BRIGHTNESS_MODE_AUTOMATIC) {
        Settings.System.putInt(contentResolver, Settings.System.SCREEN_BRIGHTNESS_MODE,
                Settings.System.SCREEN_BRIGHTNESS_MODE_MANUAL); }
        } catch (Settings.SettingNotFoundException e) {
            e.printStackTrace();
        }
    }
```
### 3、当前亮度
屏幕亮度的值为0到255,值255为最亮亮度。
```
private int getScreenBrightness() {
    ContentResolver contentResolver = getActivity().getContentResolver();

    return Settings.System.getInt(contentResolver,
            Settings.System.SCREEN_BRIGHTNESS, 0);
}
```
### 4、设置屏幕亮度
设置屏幕亮度，也就是对手机所有的界面都生效。
```
private void setScreenBrightness(int value) {
    ContentResolver contentResolver = getActivity().getContentResolver();
    
    Settings.System.putInt(contentResolver,
            Settings.System.SCREEN_BRIGHTNESS, value);
}
```
### 5、设置当前界面亮度
有时并不想设置所有界面，只想设置单前界面的亮度。例如看视频追剧的时候。
```
private void setCurrentWindowBrightness(int brightness) {
    Window window = getActivity().getWindow();
    WindowManager.LayoutParams lp = window.getAttributes();
    lp.screenBrightness = brightness / 255.0f;
    window.setAttributes(lp);
}
```