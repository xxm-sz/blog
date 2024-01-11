`Android`中，`Intent`匹配规则分为两种：显式和隐式。显式则是直接通过指定目标组件`Activity`、`Service`、`Broadcast`、`ContentProvider`。隐式则是通过`Category`、`Action`和`Data`进行规则匹配。

### 一、显式匹配

显示匹配，指的直接指定要启动目标组件的包名和组件类，这样系统在执行就不用通过规则去匹配合适的目标，而是直接去启动目标。

例如下面代码启动包名为`cn.njdbl.test`的应用的`cn.njdbl.test.MainActivity`

```
val intent=Intent()
val componentName=ComponentName("cn.njdbl.test","cn.njdbl.test.MainActivity")
intent.component = componentName
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
startActivity(intent)
```

### 二、隐式匹配

隐式匹配过程中，如果有多个组件匹配符合规则，则弹出选择框，让用户选择。例如多开应用微信时，跳转微信会提示选择哪个微信。

#### 1、Category
`Intent`中的`category`会自带`android.intent.category.DEFAULT`,所以`IntentFilter`(后续简称`filter`)中`category`必须指定该`category`才能匹配通过。如果`filter`已经带`MAIN`或者`LAUNCHER`，可以不需要指定`DEFAULT`。

`intent`中无论指定多少个`category`，`filter`必须需要对应的`category`可以匹配通过。

例如下面`Intent`与下面`intent-filter`匹配通过,这样就可以打开`TestActivity`。

```kotlin
val intent = Intent()
intent.action = "cn.njdbl.test"
intent.addCategory("cn.njdbl.category.test")
intent.addCategory("cn.njdbl.category.test2")
startActivity(intent)
```

`intent-filter`中的`category`只能多于`intent`，不能少于`intent`指定的`category`。

```xml
<activity
    android:name=".TestActivity"
    android:exported="true">
    <intent-filter>
        <!--action需要指定和匹配成功才可以-->
        <action android:name="cn.njdbl.test" />
        <!--intent设置的category需要在intent-filter匹配才可以通过,
            也就是filter中只能多不能少-->
        <category android:name="cn.njdbl.category.test" />
        <category android:name="cn.njdbl.category.test2" />
        <category android:name="cn.njdbl.category.test3" />
        <!--如果没有该category,则匹配失败，intent默认携带DEFAULT-->
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

**前提条件：**必须指定了`Action`，并且`Action`也匹配通过。

#### 2、Action

`intent`和`intent-filter`必须都指定`Action`。且`intent`无论指定了多少个`Action`，`filter`都需要存在才可以匹配成功，这一点和`category`的匹配规则一致，只能多，不能少。

例如：

```kotlin
val intent = Intent()
//指定了两个action，则intent-filter需要存在这两个action才能通过
intent.action = "cn.njdbl.test"
intent.action = "cn.njdbl.test1"
startActivity(intent)
```

下面`intent-filter`可以匹配通过，打开`TestActivity`。

```xml
<activity
    android:name=".TestActivity"
    android:exported="true">
    <intent-filter>
         <!--intent指定的两个action匹配到了，通过。如果没有匹配到，或者只有一个，则失败-->
        <action android:name="cn.njdbl.test" />
        <action android:name="cn.njdbl.test1" />
        <!--intent没有指定该action，但不影响-->
        <action android:name="cn.njdbl.test2" />
        <!--必须指定该category-->
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

#### 3、Data

我们知道`Data`是`URI`类型，其数据格式为`scheme://host:port/path`，在实际测试过程，只有`intent`和`filter`至少同时指定了`scheme`和`host`,才能达到匹配效果。如果`filter`指定了`port`，那么`intent`也需要指定。也就说在`Data`的匹配是以`filter`为准，如果`filter`指定了，而`intent`没有指定或者不一致，则无法匹配该目标组件。

由此我们可以看出，`Action`和`Category`的匹配规则和`Data`是相反的。`Action`和`Category`中，`intent`指定的`action`和`category`是`filter`的子集，而`Data`中，`filter`指定的属性则是`intent`的子集。

### 三、Intent源码分析

我们以`startActivity`，传递`Intent`为例，分析`Intent`匹配流程。

![image-20231225135830078](https://cdn.jsdelivr.net/gh/xxm-sz/dio/imgimage-20231225135830078.png)

####  `ActivityTaskSupervisor.resolveIntent`

`ActivityTaskSupervisor`类的`resolveIntent`函数，主要处理几个特殊的`Flags`，然后交由`PackgeManagerService`的内部处理类进行解析。这里传递进来的`resolvedType`，也就是`MINE type,`此时是`null`。`Activity`的`Intent`的解析不存在`MINE type`情况。

```java
ResolveInfo resolveIntent(Intent intent, String resolvedType, int userId, int flags,
        int filterCallingUid) {
    try {
        
        int modifiedFlags = flags
                | PackageManager.MATCH_DEFAULT_ONLY | ActivityManagerService.STOCK_PM_FLAGS;
        if (intent.isWebIntent()//action=ACTION_VIEW,scheme=http
                    || (intent.getFlags() & Intent.FLAG_ACTIVITY_MATCH_EXTERNAL) != 0) {
            modifiedFlags |= PackageManager.MATCH_INSTANT;
        }
        int privateResolveFlags  = 0;
        if (intent.isWebIntent()//不是默认浏览器才启动
                    && (intent.getFlags() & Intent.FLAG_ACTIVITY_REQUIRE_NON_BROWSER) != 0) {
            privateResolveFlags |= PackageManagerInternal.RESOLVE_NON_BROWSER_ONLY;
        }
        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_REQUIRE_DEFAULT) != 0) {//不匹配解析
            privateResolveFlags |= PackageManagerInternal.RESOLVE_NON_RESOLVER_ONLY;
        }

        final long token = Binder.clearCallingIdentity();
        try {
            //调用PackageManagerInternalImpl
            return mService.getPackageManagerInternalLocked().resolveIntent(
                    intent, resolvedType, modifiedFlags, privateResolveFlags, userId, true,
                    filterCallingUid);
        } finally {
            Binder.restoreCallingIdentity(token);
        }
    } finally {
        Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
    }
}
```

`mService.getPackageManagerInternalLocked()`函数返回了`PackageManagerInternal`类型的`mPmInternal`对象，而`mPmInternal`首次是通过` LocalServices.getService`函数获取的。而`LocalServices`是`SystemServer`进程模仿`ServiceManager`能力，在进程内部提供一个服务中心，然后`WMS`、`PMS`、`AMS`等服务向其注册，达到进程内不同线程中的服务可以互相调用，而不需要使用`Binder`机制。

根据`LocalServices`的原理，那么`PackageManagerInternal.class`是一个`key`,抽象类`PackageManagerInternal`是用于包管理的内部类，所以我们猜测其应该是在`PackageManagerService`添加到`LocalServices`中的（为什么是猜测，因为在`SourceInsight`没有关联到实现类。事实证明也是）。

```java
PackageManagerInternal getPackageManagerInternalLocked() {
    if (mPmInternal == null) {
        mPmInternal = LocalServices.getService(PackageManagerInternal.class);
    }
    return mPmInternal;
}
```

`packageManagerService`构造函数内向`LocalServices`添加`PackageManagerInternal`对象。

```java
//PackageManagerService构造函数内向LocalServices添加PackageManagerInternal
mPmInternal = new PackageManagerInternalImpl();
LocalServices.addService(PackageManagerInternal.class, mPmInternal);
```

`PackageManagerInternalImpl`类`resolveIntent`函数。

```java
public ResolveInfo resolveIntent(Intent intent, String resolvedType,
        int flags, int privateResolveFlags, int userId, boolean resolveForStart,
        int filterCallingUid) {
    return resolveIntentInternal(
            intent, resolvedType, flags, privateResolveFlags, userId, resolveForStart,
            filterCallingUid);
}
    
private ResolveInfo resolveIntentInternal(Intent intent, String resolvedType, int flags,
    @PrivateResolveFlags int privateResolveFlags, int userId, boolean resolveForStart,
    int filterCallingUid) {
    try {
        if (!mUserManager.exists(userId)) return null;
        final int callingUid = Binder.getCallingUid();
        flags = updateFlagsForResolve(flags, userId, filterCallingUid, resolveForStart,
                isImplicitImageCaptureIntentAndNotSetByDpcLocked(intent, userId, resolvedType,
                        flags));
        enforceCrossUserPermission(callingUid, userId, false /*requireFullPermission*/,
                false /*checkShell*/, "resolve intent");

		//调用queryIntentActivitiesInternal进行解析
        final List<ResolveInfo> query = queryIntentActivitiesInternal(intent, resolvedType,
                flags, privateResolveFlags, filterCallingUid, userId, resolveForStart,
                true /*allowDynamicSplits*/);


        final boolean queryMayBeFiltered =
                UserHandle.getAppId(filterCallingUid) >= Process.FIRST_APPLICATION_UID
                        && !resolveForStart;

        final ResolveInfo bestChoice =
                chooseBestActivity(
                        intent, resolvedType, flags, privateResolveFlags, query, userId,
                        queryMayBeFiltered);
        final boolean nonBrowserOnly =
                (privateResolveFlags & PackageManagerInternal.RESOLVE_NON_BROWSER_ONLY) != 0;
        if (nonBrowserOnly && bestChoice != null && bestChoice.handleAllWebDataURI) {
            return null;
        }
        return bestChoice;
    } finally {
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
}
```

![image-20231225144614420](https://cdn.jsdelivr.net/gh/xxm-sz/dio/imgimage-20231225144614420.png)

`queryIntentActivitiesInternal`函数内部又调用了`ComputerEngine`对象的`queryIntentActivitiesInternal`函数。

`queryIntentActivitiesInternal`函数主要逻辑是判断当前有没有设置`ComponentName`,如果有，则通过`getActivityInfo`函数获取解析结果，没有通过`queryIntentActivitiesInternalBody`函数。

```java
public final @NonNull List<ResolveInfo> queryIntentActivitiesInternal(Intent intent,
        String resolvedType, int flags, @PrivateResolveFlags int privateResolveFlags,
        int filterCallingUid, int userId, boolean resolveForStart,
        boolean allowDynamicSplits) {
    if (!mUserManager.exists(userId)) return Collections.emptyList();
    final String instantAppPkgName = getInstantAppPackageName(filterCallingUid);
    enforceCrossUserPermission(Binder.getCallingUid(), userId,
            false /* requireFullPermission */, false /* checkShell */,
            "query intent activities");
    final String pkgName = intent.getPackage();
    ComponentName comp = intent.getComponent();
    if (comp == null) {
        if (intent.getSelector() != null) {
            intent = intent.getSelector();
            comp = intent.getComponent();
        }
    }

    flags = updateFlagsForResolve(flags, userId, filterCallingUid, resolveForStart,
            comp != null || pkgName != null /*onlyExposedExplicitly*/,
            isImplicitImageCaptureIntentAndNotSetByDpcLocked(intent, userId, resolvedType,
                    flags));
    if (comp != null) {//设置ComponentName
        final List<ResolveInfo> list = new ArrayList<>(1);
        //获取Activity信息
        final ActivityInfo ai = getActivityInfo(comp, flags, userId);
        if (ai != null) {
            ......
            //一些标志位的判断，得出下面两个if变量的值
            if (!blockInstantResolution && !blockNormalResolution) {
                final ResolveInfo ri = new ResolveInfo();
                ri.activityInfo = ai;
                list.add(ri);
            }
        }

        List<ResolveInfo> result = applyPostResolutionFilter(
                list, instantAppPkgName, allowDynamicSplits, filterCallingUid,
                resolveForStart,
                userId, intent);
        return result;
    }
	//没有设置ComponentName,调用queryIntentActivitiesInternalBody
    QueryIntentActivitiesResult lockedResult =
            queryIntentActivitiesInternalBody(
                intent, resolvedType, flags, filterCallingUid, userId, resolveForStart,
                allowDynamicSplits, pkgName, instantAppPkgName);
    if (lockedResult.answer != null) {
        return lockedResult.answer;
    }

    if (lockedResult.addInstant) {
        String callingPkgName = getInstantAppPackageName(filterCallingUid);
        boolean isRequesterInstantApp = isInstantApp(callingPkgName, userId);
        lockedResult.result = maybeAddInstantAppInstaller(lockedResult.result, intent,
                resolvedType, flags, userId, resolveForStart, isRequesterInstantApp);
    }
    if (lockedResult.sortResult) {
        Collections.sort(lockedResult.result, RESOLVE_PRIORITY_SORTER);
    }
    return applyPostResolutionFilter(
            lockedResult.result, instantAppPkgName, allowDynamicSplits, filterCallingUid,
            resolveForStart, userId, intent);
}
```

`queryIntentActivitiesInternalBody`函数主要是判断`intent`有没有设置`pkgName`,然后在不同的地方获取解析结果，但最终还是调用了`ComponentResolver.queryActivities`函数进行解析，返回`ResolveInfo`对象。

```java
public @NonNull QueryIntentActivitiesResult queryIntentActivitiesInternalBody(
        Intent intent, String resolvedType, int flags, int filterCallingUid, int userId,
        boolean resolveForStart, boolean allowDynamicSplits, String pkgName,
        String instantAppPkgName) {
    // reader
    boolean sortResult = false;
    boolean addInstant = false;
    List<ResolveInfo> result = null;
     //没有设置pkgName，代表没有给Intent指定在哪个包（进程空间），意味着在所有的进程进行查找IntentFilter
    if (pkgName == null) {
        //正在匹配的IntentFilter
        //通过userId在settings中获取CrossProfileIntentResolver，然后queryIntent
        List<CrossProfileIntentFilter> matchingFilters =
                getMatchingCrossProfileIntentFilters(intent, resolvedType, userId);
        //检测是否跳过当前用户检测，Filter设置了SKIP_CURRENT_PROFILE，调用了ComponentResolver.queryActivities
        ResolveInfo skipProfileInfo  = querySkipCurrentProfileIntents(matchingFilters,
                intent, resolvedType, flags, userId);
        if (skipProfileInfo != null) {
            List<ResolveInfo> xpResult = new ArrayList<>(1);
            xpResult.add(skipProfileInfo);
            return new QueryIntentActivitiesResult(
                    applyPostResolutionFilter(
                            filterIfNotSystemUser(xpResult, userId), instantAppPkgName,
                            allowDynamicSplits, filterCallingUid, resolveForStart, userId,
                            intent));
        }

        //检查本进程是否有匹配结果，调用ComponentResolver.queryActivities解析Intent
        result = filterIfNotSystemUser(mComponentResolver.queryActivities(
                intent, resolvedType, flags, userId), userId);
        addInstant = isInstantAppResolutionAllowed(intent, result, userId,
                false /*skipPackageCheck*/, flags);
        //检查其他进程是否有结果
        boolean hasNonNegativePriorityResult = hasNonNegativePriority(result);
        CrossProfileDomainInfo specificXpInfo = queryCrossProfileIntents(
                matchingFilters, intent, resolvedType, flags, userId,
                hasNonNegativePriorityResult);
        if (intent.hasWebURI()) {
            CrossProfileDomainInfo generalXpInfo = null;
            final UserInfo parent = getProfileParent(userId);
            if (parent != null) {
                generalXpInfo = getCrossProfileDomainPreferredLpr(intent, resolvedType,
                        flags, userId, parent.id);
            }
            CrossProfileDomainInfo prioritizedXpInfo =
                    generalXpInfo != null ? generalXpInfo : specificXpInfo;

            if (!addInstant) {
                if (result.isEmpty() && prioritizedXpInfo != null) {
                    result.add(prioritizedXpInfo.resolveInfo);
                    return new QueryIntentActivitiesResult(
                            applyPostResolutionFilter(result, instantAppPkgName,
                                    allowDynamicSplits, filterCallingUid, resolveForStart,
                                    userId, intent));
                } else if (result.size() <= 1 && prioritizedXpInfo == null) {
                    return new QueryIntentActivitiesResult(
                            applyPostResolutionFilter(result, instantAppPkgName,
                                    allowDynamicSplits, filterCallingUid, resolveForStart,
                                    userId, intent));
                }
            }
            result = filterCandidatesWithDomainPreferredActivitiesLPr(
                    intent, flags, result, prioritizedXpInfo, userId);
            sortResult = true;
        } else {
            //其他进程匹配的目标添加到结果中
            if (specificXpInfo != null) {
                result.add(specificXpInfo.resolveInfo);
                sortResult = true;
            }
        }
    } else {//intent设置pkgName
        final PackageSetting setting =
                getPackageSettingInternal(pkgName, Process.SYSTEM_UID);
        result = null;
        if (setting != null && setting.pkg != null && (resolveForStart
                || !shouldFilterApplicationLocked(setting, filterCallingUid, userId))) {
            //调用ComponentResolver查询
            result = filterIfNotSystemUser(mComponentResolver.queryActivities(
                    intent, resolvedType, flags, setting.pkg.getActivities(), userId),
                    userId);
        }
        if (result == null || result.size() == 0) {
            // the caller wants to resolve for a particular package; however, there
            // were no installed results, so, try to find an ephemeral result
            addInstant = isInstantAppResolutionAllowed(intent, null /*result*/, userId,
                    true /*skipPackageCheck*/, flags);
            if (result == null) {
                result = new ArrayList<>();
            }
        }
    }
    return new QueryIntentActivitiesResult(sortResult, addInstant, result);
}
```

`ComponentResolver`类`queryActivities`函数内部调用`ActivityIntentResolver`对象解析`Activity`的`Intent`。同时也提供其他类似函数，用于解析`Service`、`Receiver`、`Provider`的`Intent`。到这里，Intent后续的解析逻辑应该是一致的，公共的解析由`IntentResolver`处理，匹配规则由`IntentFilter`提供，一些特定的解析由`IntentResolver`的子类`ActivityIntentResolver`、`ProviderIntentResolver`、`ReceiverIntentResolver`、`ServiceIntentResolver`处理。

如果`intent`存在`resolvedType`，则查询所有适配该`resolvedType`的`IntentFilter`，包括`b/*`,`b/s`,`*/*`,`*/s`情况的MINE type,后两种属于同一种情况。`resolvedType`进行`/`拆解，然后根据上面三种情况，将所有`filter MINE type`符合的拆解到F泛型`firstTypeCut`、`secondTypeCut`、`thirdTypeCut`,它们具体类型由子类决定。

如果`Intent`的`scheme`不为`null`，需要查询设置了`scheme`的`filter`，保存到`schemeCut`。如果`Intent`没有设置`Data`,那么`firstTypeCut`会被设置为`filter`的`action`。

最后调用`buildResolveList`函数再进一步处理。

```java
//如果要看到intent的调试resolveList信息，可以设置Intent.FLAG_DEBUG_LOG_RESOLUTION，下面会把打印相关逻辑移除
//activity传递进来的resolvedType为null
public List<R> queryIntent(Intent intent, String resolvedType, boolean defaultOnly,
        int userId) {
    String scheme = intent.getScheme();

    ArrayList<R> finalList = new ArrayList<R>();

    F[] firstTypeCut = null;
    F[] secondTypeCut = null;
    F[] thirdTypeCut = null;
    F[] schemeCut = null;
    //如果intent携带MINE type,查找匹配的IntentFilter(*/*,*/s,b/s,b/*)
    if (resolvedType != null) {//Intent携带MineType
        int slashpos = resolvedType.indexOf('/');//分割线
        if (slashpos > 0) {//存在baseType/subType
            final String baseType = resolvedType.substring(0, slashpos);
            if (!baseType.equals("*")) {//获取filter是(b/s,b/*)
                if (resolvedType.length() != slashpos+2
                        || resolvedType.charAt(slashpos+1) != '*') {//具体b/s情况
                    //mTypeToFilter存放所有的MineType;这里获取b/s情况
                    firstTypeCut = mTypeToFilter.get(resolvedType);
                    //mWildTypeToFilter存放所有 subType是*;具体(包括*)/*;这里获取filter是b/*情况
                    secondTypeCut = mWildTypeToFilter.get(baseType);
                } else { //	baseType/*情况 (b/*)
                   	//获取filter 只有basetype的情况
                    firstTypeCut = mBaseTypeToFilter.get(baseType);
                 	//获取filter 是baseType/*情况
                    secondTypeCut = mWildTypeToFilter.get(baseType);
                }
                // filter是*/*情况
                thirdTypeCut = mWildTypeToFilter.get("*");
            } else if (intent.getAction() != null) {//(*/*,*/s情况，Action！=null)
                //Intent适配所有MINE type,所以IntentFilter需要有type,且还有Action
                firstTypeCut = mTypedActionToFilter.get(intent.getAction());
            }
        }
    }

    //Data含有URI
    if (scheme != null) {
        schemeCut = mSchemeToFilter.get(scheme);
    }

    // 没有设置Data（Intent没有Data的情况）
    if (resolvedType == null && scheme == null && intent.getAction() != null) {
        firstTypeCut = mActionToFilter.get(intent.getAction());//Action
    }
	//categories
    FastImmutableArraySet<String> categories = getFastIntentCategories(intent);
    if (firstTypeCut != null) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                scheme, firstTypeCut, finalList, userId);
    }
    if (secondTypeCut != null) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                scheme, secondTypeCut, finalList, userId);
    }
    if (thirdTypeCut != null) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                scheme, thirdTypeCut, finalList, userId);
    }
    if (schemeCut != null) {
        buildResolveList(intent, categories, debug, defaultOnly, resolvedType,
                scheme, schemeCut, finalList, userId);
    }
    //过滤掉一些匹配结果
    filterResults(finalList);
    //排序匹配结果
    sortResults(finalList);

    return finalList;
}
```

`buildResolveList`函数分别对所有的`IntentFilter`进行匹配`action`、`category`、`data`，并将结果写到入参`dest`。

```java

// defaultOnly true表示intent的categroy默认携带CATEGORY_DEFAULT，因此filter也需要携带才匹配成功
//debug打印信息，下面会把打印相关逻辑移除
//filter Pair<ParsedActivity, ParsedIntentInfo>\Pair<ParsedProvider, ParsedIntentInfo>
private void buildResolveList(Intent intent, FastImmutableArraySet<String> categories,
        boolean debug, boolean defaultOnly, String resolvedType, String scheme,
        F[] src, List<R> dest, int userId) {
    final String action = intent.getAction();
    final Uri data = intent.getData();
    final String packageName = intent.getPackage();

    final boolean excludingStopped = intent.isExcludingStopped();

    final int N = src != null ? src.length : 0;
    boolean hasNonDefaults = false;
    int i;
    F filter;
    for (i=0; i<N && (filter=src[i]) != null; i++) {
        int match;

        if (excludingStopped && isFilterStopped(filter, userId)) {
            continue;
        }

        //filter设置包名过滤,判断packageName与filter的ParsedActivity的packageName是否一致(filter.first)
        //具体看子类
        if (packageName != null && !isPackageForFilter(packageName, filter)) {
            continue;
        }
		//filter.second，具体看子类
        IntentFilter intentFilter = getIntentFilter(filter);

        // activity判断dest存在，则return false
        if (!allowFilterResult(filter, dest)) {
            continue;
        }
 		//intenFilter的匹配函数
        match = intentFilter.match(action, resolvedType, scheme, data, categories, TAG);
        if (match >= 0) {//大于零表示intentFilter匹配成功。（filter没有设置DEFAULT,且Intent默认情况，则匹配失败）
            if (!defaultOnly || intentFilter.hasCategory(Intent.CATEGORY_DEFAULT)) {
                final R oneResult = newResult(filter, match, userId);
               
                if (oneResult != null) {
                    dest.add(oneResult);//添加到结果
                }
            } else {
                hasNonDefaults = true;
            }
        } else {
            ...
        }
    }
}
```

### 四、`IntentFilter`与`Intent`的匹配

`match`函数从上到下，对`Action`、`Data`、`Category`依次进行匹配判断，其中一个匹配失败，那么该`IntentFilter`与给定的`Intent`匹配失败，无法命中该`IntentFilter`的目标组件。

```java
public final int match(String action, String type, String scheme,
        Uri data, Set<String> categories, String logTag, boolean supportWildcards,
        @Nullable Collection<String> ignoreActions) {
    //action的匹配规则
    if (action != null && !matchAction(action, supportWildcards, ignoreActions)) {
        return NO_MATCH_ACTION;
    }
	//data的匹配规则，内含URI和MINE type的匹配规则
    int dataMatch = matchData(type, scheme, data, supportWildcards);
    if (dataMatch < 0) {
        return dataMatch;
    }
	//category的匹配规则
    String categoryMismatch = matchCategories(categories);
    if (categoryMismatch != null) {
        return NO_MATCH_CATEGORY;
    }
    return dataMatch;
}
```

#### 1、Action的匹配

* 支持通配符且`intent`的`action`为*，`ignoreActions`为`null`，`filter`存在`action`,则匹配通过。这种应该属于特殊情况。
* `ignoreActions`不为`null`,且存在`intent`的`action`，则匹配失败。
* `intent`没有设置`action`或者`filter`没有设置与`intent`相同的`action`（`intent`的`action`不为`intentFilter`的子集），则匹配失败。

```java
//wildcardSupported默认情况下是false,ignoreActions为null。具体需要看具体intent。
private boolean matchAction(String action, boolean wildcardSupported,
        @Nullable Collection<String> ignoreActions) {
    //支持通配符，且Intent的action是*，ignoreActions为空，filter有action，则匹配通过
    if (wildcardSupported && WILDCARD.equals(action)) {
        if (ignoreActions == null) {
            return !mActions.isEmpty();
        }
        for (int i = mActions.size() - 1; i >= 0; i--) {
            if (!ignoreActions.contains(mActions.get(i))) {
                return true;
            }
        }
        return false;
    }
    if (ignoreActions != null && ignoreActions.contains(action)) {
        return false;
    }
    return hasAction(action);
}
//如果Intent没有设置action，或者filter没有包含该action，匹配失败
public final boolean hasAction(String action) {
    return action != null && mActions.contains(action);
}
```

#### 2、Data的匹配

我们知道`Data`的类型是`URI`,那么其格式：`scheme://authority/path`,其中`authority`的格式是`host:port`,也就是说，最终`Data`的最终表现格式是`scheme://host:port/path`。因此`matchData`函数需要针对`Data`的格式分别对`sheme`、`authority`、`path`进行匹配判断。 而对`authority`又要分`host`和`port`判断，只有`authority`匹配成功情况，才会对`path`进行匹配判断。最后，我们知道`Data`匹配过程中，是支持`MINE type`类型的，所以也需要对`MINE type`进行判断。同时，也要考虑通配符的情况匹配。整个过程，如果只要有一个匹配失败，`intent`将匹配不到该`filter`指定的目标组件。

```java
public final int matchData(String type, String scheme, Uri data) {
    return matchData(type, scheme, data, false);
}

private int matchData(String type, String scheme, Uri data, boolean wildcardSupported) {
    final boolean wildcardWithMimegroups = wildcardSupported && countMimeGroups() != 0;
    final List<String> types = mDataTypes;
    final ArrayList<String> schemes = mDataSchemes;

    int match = MATCH_CATEGORY_EMPTY;
    //不支持泛型下，filter没有types和schemes,intent也没有设置，则匹配通过
    if (!wildcardWithMimegroups && types == null && schemes == null) {
        return ((type == null && data == null)
            ? (MATCH_CATEGORY_EMPTY+MATCH_ADJUSTMENT_NORMAL) : NO_MATCH_DATA);
    }

    if (schemes != null) {
        //filter包含intent的scheme或者filter是支持*且intent是*
        if (schemes.contains(scheme != null ? scheme : "")
                || wildcardSupported && WILDCARD.equals(scheme)) {
            match = MATCH_CATEGORY_SCHEME;
        } else {
            return NO_MATCH_DATA;
        }
        //scheme特殊处理
        final ArrayList<PatternMatcher> schemeSpecificParts = mDataSchemeSpecificParts;
        if (schemeSpecificParts != null && data != null) {
            match = hasDataSchemeSpecificPart(data.getSchemeSpecificPart(), wildcardSupported)
                    ? MATCH_CATEGORY_SCHEME_SPECIFIC_PART : NO_MATCH_DATA;
        }
        if (match != MATCH_CATEGORY_SCHEME_SPECIFIC_PART) {
        final ArrayList<AuthorityEntry> authorities = mDataAuthorities;
            //先判断authoriteis是否为空
            if (authorities != null) {
                int authMatch = matchDataAuthority(data, wildcardSupported);
                if (authMatch >= 0) {
                    final ArrayList<PatternMatcher> paths = mDataPaths;
                    if (paths == null) {//filter没有设置path,匹配通过
                        match = authMatch;
                    } else if (hasDataPath(data.getPath(), wildcardSupported)) {//path匹配
                        match = MATCH_CATEGORY_PATH;
                    } else {
                        return NO_MATCH_DATA;
                    }
                } else {
                    return NO_MATCH_DATA;
                }
            }
        }
        // If neither an ssp nor an authority matched, we're done.
        if (match == NO_MATCH_DATA) {
            return NO_MATCH_DATA;
        }
    } else {
        // 不支持自定义的scheme?
        if (scheme != null && !"".equals(scheme)
                && !"content".equals(scheme)
                && !"file".equals(scheme)
                && !(wildcardSupported && WILDCARD.equals(scheme))) {
            return NO_MATCH_DATA;
        }
    }

    if (wildcardWithMimegroups) {//通配符？
        return MATCH_CATEGORY_TYPE;
    } else if (types != null) {//filters设置了type
        if (findMimeType(type)) {
            match = MATCH_CATEGORY_TYPE;
        } else {
            return NO_MATCH_TYPE;
        }
    } else {//filters没有设置type,但intent设置了type
        if (type != null) {
            return NO_MATCH_TYPE;
        }
    }

    return match + MATCH_ADJUSTMENT_NORMAL;
}
```

##### 1). Scheme的匹配

`scheme`的匹配比较简单，判断`filter`中`scheme`是否包含当前`intent`的`scheme`。或者当前`filter`支持通配符，`intent`也是设置了通配符。

##### 2). Authority的匹配

`IntentFilter`把`Authority`相关信息封装到了`AuthorityEntry`类，然后通过其成员函数`match`来进行匹配判断。我们知道`authority`的格式如：`host:port`。所以`match`函数判断内容是通配符、`host`、`port`。

* `filter`存在`authority`,`intent`不存在，匹配失败。
* `Filter`和`intent`是否都支持通配符，`host`是否相同，`port`是否相同。其中一个为否，则匹配失败。

```java
public final int matchDataAuthority(Uri data, boolean wildcardSupported) {
    //这里只考虑data==null情况，外层有mDataAuthorities！=null判断
    if (data == null || mDataAuthorities == null) {
        return NO_MATCH_DATA;
    }
    final int numDataAuthorities = mDataAuthorities.size();
    for (int i = 0; i < numDataAuthorities; i++) {
        final AuthorityEntry ae = mDataAuthorities.get(i);
        int match = ae.match(data, wildcardSupported);
        if (match >= 0) {
            return match;
        }
    }
    return NO_MATCH_DATA;
}
//AuthorityEntry.match函数
public boolean match(AuthorityEntry other) {
    //都支持通配符
    if (mWild != other.mWild) {
        return false;
    }
    //host相同
    if (!mHost.equals(other.mHost)) {
        return false;
    }
    //port相同
    if (mPort != other.mPort) {
        return false;
    }
    return true;
}
```

##### 3). Path的匹配

如果`Filter`的`Authority`匹配通过，且`Data`携带`path`,则需要进行匹配。`path`信息被封装到了`PatternMatcher`类，调用它的`match`函数匹配。

* `Filter`没有设置，但`Intent`有，匹配通过。

* `filter`和`Intent`都支持通配符，匹配通过。
* `filter`与`Intent`的`path`一致，匹配通过。

```java
private boolean hasDataPath(String data, boolean wildcardSupported) {
    if (mDataPaths == null) {
        return false;
    }
    if (wildcardSupported && WILDCARD_PATH.equals(data)) {
        return true;
    }
    final int numDataPaths = mDataPaths.size();
    for (int i = 0; i < numDataPaths; i++) {
		//按字面理解PatternMatcher的匹配类型是literal，具体看看
        final PatternMatcher pe = mDataPaths.get(i);
        if (pe.match(data)) {
            return true;
        }
    }
    return false;
}
//PatternMatcher
public boolean match(String str) {
    return matchPattern(str, mPattern, mParsedPattern, mType);
}
//pattern此时PATTERN_LITERAL
static boolean matchPattern(String match, String pattern, int[] parsedPattern, int type) {
    if (match == null) return false;
    if (type == PATTERN_LITERAL) {
        return pattern.equals(match);
    } if (type == PATTERN_PREFIX) {
        return match.startsWith(pattern);
    } else if (type == PATTERN_SIMPLE_GLOB) {
        return matchGlobPattern(pattern, match);
    } else if (type == PATTERN_ADVANCED_GLOB) {
        return matchAdvancedPattern(parsedPattern, match);
    } else if (type == PATTERN_SUFFIX) {
        return match.endsWith(pattern);
    }
    return false;
}
```

##### 4). MINE type的匹配

将`MINE type`具体类型表示为`baseType/subType`。例如`image/jpeg`,则`baseType`是`image`,`subType`是`jpeg`。而在`Android`的`MINE type`存在这么几种情况：`*/*`,`*`,`xxx/*`,`xxx/xxx`。其中`xxx`表示具体的类型，`*`表示通配符。

总共有以下几种情况：

* 外层判断函数得知：`filter`没有设置`type`，`intent`设置了，匹配失败。

* `filter`设置了`type`,而`intent`没有设置，则匹配失败。
* `filter`设置的`type`与`intent`相同，则匹配成功。
* `filter`的`type`为`*/*`,或(`intent`的`type`为`*/*`，且`filter`有`type`)，则匹配成功。
* `intent`属于`xxx/*`,或`filter`是`xxx/*`，判断是否有`baseType`相同，有则匹配成功。

```java
//这里入参type是intent的，而filter则是在mDataTypes
private final boolean findMimeType(String type) {
    final ArrayList<String> t = mDataTypes;
	//filter存在type,而intent没有
    if (type == null) {
        return false;
    }
	//filter存在intent的type
    if (t.contains(type)) {
        return true;
    }

    //intent 匹配所有filter
    final int typeLength = type.length();
    if (typeLength == 3 && type.equals("*/*")) {
        return !t.isEmpty();
    }

    //filter匹配所有intent
    if (hasPartialTypes() && t.contains("*")) {
        return true;
    }

    final int slashpos = type.indexOf('/');
    if (slashpos > 0) {
		//filter在`xxx/*`情况，hasPartialTypes返回true，且mDataTypes缓存的是xxx
        //判断type的baseType是否一致
        if (hasPartialTypes() && t.contains(type.substring(0, slashpos))) {
            return true;
        }
		//type是`xxx/*`情况，查找filter是否存在baseType是xxx情况
        if (typeLength == slashpos+2 && type.charAt(slashpos+1) == '*') {
            final int numTypes = t.size();
            for (int i = 0; i < numTypes; i++) {
                final String v = t.get(i);
                if (type.regionMatches(0, v, 0, slashpos+1)) {
                    return true;
                }
            }
        }
    }

    return false;
}
```

#### 3、Category的匹配

* `intent`没有设置`category`，匹配通过。
* `filter`没有设置`category`,且`intent`也没有设置，匹配通过。
* `intent`设置的`category`只要有一个没有在`filter`中被设置，则匹配失败。`intent`的`category`需要为`filter`的子集。

```java
//return null表示成功
public final String matchCategories(Set<String> categories) {
    if (categories == null) {
        return null;
    }
	//intent
    Iterator<String> it = categories.iterator();
	//filters
    if (mCategories == null) {
        return it.hasNext() ? it.next() : null;
    }
	//intent的category为filter的子集，则匹配成功
    while (it.hasNext()) {
        final String category = it.next();
        if (!mCategories.contains(category)) {
            return category;
        }
    }

    return null;
}
```

### 五、MINE type的推断

`Android 12`对`MINE type`的解析发生在`Instrumentation`类的`execStartActivity`函数，调用ATS的`startActivity`函数时候。

```
Intent.resolveTypeIfNeeded(who.getContentResolver())
```

这里的`who`是`Context`，实现类为`ContextImpl`,其`getContentResolver`函数返回了`ApplicationContentResolver`对象。

```java
public @Nullable String resolveTypeIfNeeded(@NonNull ContentResolver resolver) {
    if (mComponent != null) {
        return mType;
    }
    return resolveType(resolver);
}

public @Nullable String resolveType(@NonNull ContentResolver resolver) {
    if (mType != null) {
        return mType;
    }
    if (mData != null) {
        if ("content".equals(mData.getScheme())) {//Content类型的scheme才可以进行推断
            return resolver.getType(mData);
        }
    }
    return null;
}
```

`MINE type`的推断必须基于`scheme`是`content`的情况,然后调用`ContentResolver`的`getType`函数,然后进一步查找`ContentProvider`,也就是说，我们在自定义`ContentProvider`时，实现`getType`函数返回`String`对象，将作为`MINE type`参与到`Intent`的解析中。`MINE type`的解析最终还是开发者来实现。

```java
public final @Nullable String getType(@NonNull Uri url) {
    try {
        if (mWrapped != null) return mWrapped.getType(url);
    } catch (RemoteException e) {
        return null;
    }

    //在已存在的Provider中查找
    IContentProvider provider = acquireExistingProvider(url);
    if (provider != null) {
        try {
            final StringResultListener resultListener = new StringResultListener();
            provider.getTypeAsync(url, new RemoteCallback(resultListener));
            resultListener.waitForResult(CONTENT_PROVIDER_TIMEOUT_MILLIS);
            if (resultListener.exception != null) {
                throw resultListener.exception;
            }
            return resultListener.result;
        } catch (RemoteException e) {
            return null;
        } catch (java.lang.Exception e) {
            return null;
        } finally {
            releaseProvider(provider);
        }
    }

    if (!SCHEME_CONTENT.equals(url.getScheme())) {
        return null;
    }

    try {
        final StringResultListener resultListener = new StringResultListener();
       	//在AMS进行查找
        ActivityManager.getService().getProviderMimeTypeAsync(
                ContentProvider.getUriWithoutUserId(url),
                resolveUserId(url),
                new RemoteCallback(resultListener));
        resultListener.waitForResult(REMOTE_CONTENT_PROVIDER_TIMEOUT_MILLIS);
        if (resultListener.exception != null) {
            throw resultListener.exception;
        }
        return resultListener.result;
    } catch (RemoteException e) {
        return null;
    } catch (java.lang.Exception e) {
        return null;
    }
}
```

