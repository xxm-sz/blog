## 前言
以前对下面的问题，我的态度是，不报错就是没问题，报错就用快捷键，根据Android Studio提示修复问题，从来不去问个为什么？现在代码洁癖症越来越严重的我，忍不住想看高清什么东西在搞鬼。

认真看完本文，一定可以学到最新的知识。就算看不下去，也要点个赞收藏，绝对不亏。本文并不是吐槽Lint的不好，而是在学习Lint过程碰到问题，心态是奔溃的，以及解决每个问题带来的喜感。

不知道大家有没有注意项目中黄色代码块的提示，如下图所示：
![](https://user-gold-cdn.xitu.io/2019/7/18/16c054df73e921b9?w=1750&h=754&f=jpeg&s=172409)
或者红色标记的代码(并没有任何错误)，如下图所示：

![](https://user-gold-cdn.xitu.io/2019/7/18/16c054e281bbdb36?w=2016&h=162&f=jpeg&s=108125)
上文黄色的提醒和红色警告，都是来自Android Studio内置的Lint工具检查我们的代码后而作出的动作。
通过配置Lint,也可以消除上面的提醒。例如，我开发系统APK,根本不需要考虑用户是否授权。
那么Lint是什么呢？
## Lint
> Android Studio 提供一个名为Lint的静态代码扫描工具，可以发现并纠正代码结构中的质量问题，而无需实际执行该应用，也不必编写测试用例。
> Lint 工具可检查您的 Android 项目源文件是否包含潜在错误，以及在正确性、安全性、性能、易用性、便利性和国际化方面是否需要优化改进。

也就是说，通过Lint工具，我们可以写出更高质量的代码和代码潜在的问题，妈妈再也不用担心我的同事用中文命名了。也可以通过定制Lint相关配置，提高开发效率。

### Lint禁止检查
由于Android Studio内置了Lint工具，好像不需要我们干嘛。可是呀，我有强迫症，看着上面的黄色块，超级不爽的。所以我们得了解如何配置Lint，让它为我们服务，而不是为Google服务。

本文开始的红色错误可以通过注解来消除（一般建议是根据提示进行修正，除非明白自己在做什么），可以在类或该代码所在的方法添加`@SuppressLint`。

![](https://user-gold-cdn.xitu.io/2019/7/18/16c054e9611cb00a?w=2124&h=458&f=jpeg&s=170114)
上图中是禁止Lint检查特定的问题检查，如果要禁止该Java文件所有的Lint问题，可以在类前添加如下注解:`@SuppressLint(all)`。
对XMl文件的禁止，则可以采用如下形式：
1. 在lint.xml声明命名空间
```
namespace xmlns:tools="http://schemas.android.com/tools"
```
2. 在布局中使用：
```
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    tools:ignore="UnusedResources" >

    <TextView
        android:text="@string/auto_update_prompt" />
</LinearLayout>
```
父容器声明了ignore属性，那么子视图会继承该属性。例如上文LinearLayout中声明了禁止Lint检查LinearLayout的UnusedResources问题，TextView自然也禁止检查该问题。禁止检查多个问题，问题之间用逗号隔开；禁止检查所有问题则使用`all`关键字。
```
tools:ignore="all"
```
我们也可以通过配置项目的Gradle文件来禁止检查。

例如禁止Lint检查项目AndroidManifest.xml文件的GoogleAppIndexingWarning问题。在项目对应组件工程的Gradle文件添加如下配置,这样就不会有黄色提醒了。
```
defaultConfig{
    lintOptions {
        disable 'GoogleAppIndexingWarning'
    }
}
```
那么，可以禁止lint工具检查什么问题？
### 配置Lint
在上文中通过注解和在xml使用属性来禁止Lint工具检查相关问题,其实已经是对Lint的配置了。Lint将多个问题归为一个issue(规则)，例如下图右边的的六大规则。

![](https://user-gold-cdn.xitu.io/2019/7/18/16c054effc474eee?w=1242&h=610&f=jpeg&s=86329)
上图是Lint工具的工作流程，下面了解相关概念。
**App Source Files**
源文件包含组成 Android 项目的文件，包括 Java 和 XML 文件、图标和 ProGuard 配置文件等。
**lint.xml 文件**
此配置文件可用于指定您希望排除的任何 Lint 检查以及自定义问题严重级别。
**lint Tool**
我们可以通过Android Studio 对 Android 项目运行此静态代码扫描工具。也可以手动运行。Lint 工具检查可能影响 Android 应用质量和性能的代码结构问题。
**Lint 检查结果**
我们可以在控制台（命令行运行）或 Android Studio 的 Inspection Results 窗口中查看 Lint 检查结果。

通过Lint工具的工作流程了解到，可以在lint.xml文件配置一些信息。**一般新建项目都是没有lint.xml文件的，在项目的根目录创建lint.xml文件**。格式如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<lint>
        <!-- list of issues to configure -->
</lint>
```
那么有哪些Issues（规则）呢？

在Android主要有如下六大类：
* Security 安全性。在AndroidManifest.xml中没有配置相关权限等。
* Usability 易用性。重复图标；上文开始黄色警告也属于该规则等。
* Performance 性能。内存泄漏，xml结构冗余等。
* Correctness 正确性。超版本调用API，设置不正确的属性值等。
* Accessibility 无障碍。单词拼写错误等。
* Internationalization国际化。字符串缺少翻译等。

其他更多Issues,可以通将命令行切换到../Android/sdk/tools/bin目录下，然后输入`lint --list`。例如在Mac下：
`cd /Users/gitcode8/Library/Android/sdk/tools/bin`输入`./lint --list`
结果如下：

![](https://user-gold-cdn.xitu.io/2019/7/18/16c054f4dae85220?w=1414&h=938&f=jpeg&s=383050)

例如官网提供的参考例子:
```
<?xml version="1.0" encoding="UTF-8"?>
<lint>
    <!-- 忽略整个工程目录下指定问题的检查 -->
    <issue id="IconMissingDensityFolder" severity="ignore" />

    <!-- 忽略对指定文件指定问题的检查 -->
    <issue id="ObsoleteLayoutParam">
        <ignore path="res/layout/activation.xml" />
        <ignore path="res/layout-xlarge/activation.xml" />
    </issue>

    <!-- 更改检查问题归属的严重性  -->
    <issue id="HardcodedText" severity="error" />
</lint>
```
学习Lint工具仅仅是为了安抚我的强迫症？不，还不知道Lint真正用来干嘛呢？
### 检查项目质量
不好容易开发了个APP,准备开始上班摸鱼了。还让代码自查？那就通过Lint来看看代码质量如何吧。
#### 1、Android Studio 检查
1. 通过Android Studio 的菜单栏Analyze选项下拉选择第一个选项Inspect Code.

![](https://user-gold-cdn.xitu.io/2019/7/18/16c054f8ad55eef7?w=748&h=336&f=jpeg&s=88867)
2、在弹出框根据自己需要选择lint工具的检查范围，这里选择整个项目。检查时间也是根据项目大小来定的。

![](https://user-gold-cdn.xitu.io/2019/7/18/16c054fa4b68bb50?w=1232&h=726&f=jpeg&s=136813)
3、等待一段时间后，会列出检查结果。从下图看到，不仅会检查Android存在的问题，也会检查Java等其他问题。通过单击问题，可以从右边提示框看到问题发生的地方和相关建议。

![](https://user-gold-cdn.xitu.io/2019/7/18/16c054fcd6a6956c?w=2628&h=880&f=jpeg&s=230150)
到这里，就开始对项目修修补补吧。

## 自定义规则
为什么要自定义呢？已有规则不符合自己或团队开发需求，或者觉得Lint存在一些缺陷。在网上大多数文章千篇一律，都是通过将Log打印来举例，看着都好累哦。由于没有相关官方文档和第三方教程（可能由于lint的api更新太快，没人愿意做这种吃力不讨好的工作），也这就只有这样了。本文通过自定义命名规范规则来讲解整个过程。
### Lint中重点的API
先学习相关api，可以快速理解一些概念，可以粗略看过，下结实践再回来看。
#### 1、Issue
Issue如上文所说，表示lint 工具检查的一个规则，一个规则包含若干问题。常在Detector中创建。下文是创建一个Issue的例子。
```
   private  static final Issue ISSUE = Issue.create("NamingConventionWarning",
            "命名规范错误",
            "使用驼峰命名法，方法命名开头小写，类大写字母开头",
            Category.USABILITY,
            5,
            Severity.WARNING,
            new Implementation(NamingConventionDetecor.class,
                    EnumSet.of(Scope.JAVA_FILE)));
```
* **第一个参数id** 唯一的id,简要表面当前提示的问题。
* **第二个参数briefDescription** 简单描述当前问题
* **第三个参数explanation** 详细解释当前问题和修复建议
* **第四个参数category** 问题类别，例如上文讲到的Security、Usability等等。
* **第五个参数priority** 优先级，从1到10，10最重要
* **第六个参数Severity** 严重程度：FATAL（奔溃）, ERROR（错误）, WARNING（警告）,INFORMATIONAL（信息性）,IGNORE（可忽略）
* **第七个参数Implementation** Issue和哪个Detector绑定，以及声明检查的范围。Scope有如下选择范围：
RESOURCE_FILE（资源文件),BINARY_RESOURCE_FILE（二进制资源文件）,RESOURCE_FOLDER（资源文件夹）,ALL_RESOURCE_FILES（所有资源文件）,JAVA_FILE（Java文件）, ALL_JAVA_FILES（所有Java文件）,CLASS_FILE（class文件）, ALL_CLASS_FILES（所有class文件）,MANIFEST（配置清单文件）, PROGUARD_FILE（混淆文件）,JAVA_LIBRARIES（Java库）, GRADLE_FILE（Gradle文件）,PROPERTY_FILE(属性文件),TEST_SOURCES（测试资源）,OTHER(其他);

这样就能很清楚的定义一个规则，上文只定义了检查命名规范的规则。

#### 2、IssueRegistry 

用于注册要检查的Issue(规则)，只有注册了Issue,该Issue才能被使用。例如注册上文的命名规范规则。
```
public class Register extends IssueRegistry {
    @NotNull
    @Override
    public List<Issue> getIssues() {
        return Arrays.asList(NamingConventionDetector.ISSUE);
    }
}
```

#### 4、Detector

查找指定的Issue，一个Issue对应一个Detector。自定义Lint 规则的过程也就是重写Detector类相关方法的过程。具体看下小结实践。

#### 5、Scanner

扫描并发现代码中的Issue,Detector需要实现Scaner,可以继承一个到多个。
* UastScanner 扫描Java文件和Kotlin文件
* ClassScanner 扫描Class文件
* XmlScanner 扫描Xml文件
* ResourceFolderScanner 扫描资源文件夹
* BinaryResourceScanner 扫描二进制资源文件
* OtherFileScanner 扫描其他文件
* GradleScanner 扫描Gradle脚本

旧版本的JavaScanner、JavaPsiScanner随着版本的更新已经被UastScanner替代了。

### 自定义Lint规则实践
通过实现命名规范Issue来熟悉和运用上小节相关的api。自定义规则需要在Java工程中创建，这里通过Android Studio来创建一个Java Library。

步骤：`File->New->New Mudle->Java Library`

这里Library Name为lib。

定义类NamingConventionDetector,并继承自Detector。因为这里是检测Java文件类名和方法是否符合规则，所以实现Detector.UastScanner接口。
```
public class NamingConventionDetector 
    extends Detector 
    implements Detector.UastScanner {
}
```
在NamingConventionDetector类内定义上文的Issue：
```
public class NamingConventionDetector 
    extends Detector 
    implements Detector.UastScanner {
    
    public static final Issue ISSUE = Issue.create("NamingConventionWarning",
        "命名规范错误",
        "使用驼峰命名法，方法命名开头小写",
        Category.USABILITY,
        5,
        Severity.WARNING,
        new Implementation(NamingConventionDetector.class,
            EnumSet.of(Scope.JAVA_FILE)));
}
```
重写Detector的createUastHandler方法，实现我们自己的处理类。
```
public class NamingConventionDetector extends Detector implements Detector.UastScanner {
    //定义命名规范规则
    public static final Issue ISSUE = Issue.create("NamingConventionWarning",
            "命名规范错误",
            "使用驼峰命名法，方法命名开头小写",
            Category.USABILITY,
            5,
            Severity.WARNING,
            new Implementation(NamingConventionDetector.class,
                    EnumSet.of(Scope.JAVA_FILE)));

    //返回我们所有感兴趣的类，即返回的类都被会检查
    @Nullable
    @Override
    public List<Class<? extends UElement>> getApplicableUastTypes() {
        return Collections.<Class<? extends UElement>>singletonList(UClass.class);
    }

    //重写该方法，创建自己的处理器
    @Nullable
    @Override
    public UElementHandler createUastHandler(@NotNull final JavaContext context) {
        return new UElementHandler() {
            @Override
            public void visitClass(@NotNull UClass node) {
                node.accept(new NamingConventionVisitor(context, node));
            }
        };
    }
    //定义一个继承自AbstractUastVisitor的访问器，用来处理感兴趣的问题
    public static class NamingConventionVisitor extends AbstractUastVisitor {

        JavaContext context;

        UClass uClass;

        public NamingConventionVisitor(JavaContext context, UClass uClass) {
            this.context = context;
            this.uClass = uClass;
        }

        @Override
        public boolean visitClass(@org.jetbrains.annotations.NotNull UClass node) {
            //获取当前类名
            char beginChar = node.getName().charAt(0);
            int code = beginChar;
            //如果类名不是大写字母，则触碰Issue，lint工具提示问题
            if (97 < code && code < 122) {
                context.report(ISSUE,context.getNameLocation(node),
                        "the  name of class must start with uppercase:" + node.getName());
                //返回true表示触碰规则，lint提示该问题；false则不触碰
                return true;
            }

            return super.visitClass(node);
        }

        @Override
        public boolean visitMethod(@NotNull UMethod node) {
            //当前方法不是构造方法
            if (!node.isConstructor()) {
            char beginChar = node.getName().charAt(0);
            int code = beginChar;
                //当前方法首字母是大写字母，则报Issue
                if (65 < code && code < 90) {
                    context.report(ISSUE, context.getLocation(node),
                            "the method must start with lowercase:" + node.getName());
                    //返回true表示触碰规则，lint提示该问题；false则不触碰
                    return true;
                }
            }
            return super.visitMethod(node);

        }

    }
}
```
上文NamingConventionDetector类，已经是全部代码，只检查类名和方法名是否符合驼峰命名法，可以根据具体需求,重写抽象类AbstractUastVisitor的visitXXX方法。

如果处理特定的方法或者其他，也可以使用默认的处理器。重写Scanner相关方法。例如：
```
 @Override
public List<String> getApplicableMethodNames() {
    return Arrays.asList("e","v");
}
```
表示e(),v()方法会被检测到，并调用visitMethod()方法，实现自己的逻辑。
```
    @Override
    public void visitMethod JavaContext context,  JavaElementVisitor visitor,  PsiMethodCallExpression call, PsiMethod method) {
        //todo something
        super.visitMethod(context, visitor, call, method);
    }
```
接下来就是注册自定义的Issue:
```
public class Register extends IssueRegistry {
    @NotNull
    @Override
    public List<Issue> getIssues() {
        return Arrays.asList(NamingConventionDetector.ISSUE);
    }
}
```
在lib项目的build.gradle文件添加相关代码：
```
apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.tools.lint:lint-api:26.4.2'
    implementation 'com.android.tools.lint:lint-checks:26.4.2'

}
//添加如下代码
jar {
    manifest {
        attributes 'Lint-Registry': 'com.gitcode.lib.Register'
    }
}

sourceCompatibility = "7"
targetCompatibility = "7"
```
到这里就自定义Lint自定义规则就搞定了，接着是使用和确定规则是否正确。
### 使用自定Lint规则
使用自定义Lint规则有两种形式：jar包和AAR文件。
#### jar形式使用
在Android Studio的Terminal输入下面命令：
```
./gradlew lib:assemble
```
看到`BUILD SUCCESSFUL `则表示生成jar包成功，可以在下面路径找到：
```
lib->build->libs
```
如图：

![](https://user-gold-cdn.xitu.io/2019/7/23/16c1a8e3ca1670a4?w=854&h=254&f=png&s=51153)
将lib.jar拷贝下面目录：
```
~/.android/lint/
```
如果lint文件夹不存在，则创建。通过命令行输入lint --list。滑到最后可以看到配置的规则，如图：

![](https://user-gold-cdn.xitu.io/2019/7/23/16c1a914fe512ccf?w=1144&h=242&f=png&s=157254)
重启Android Studio,让规则生效。
检测到方法大写，不符合命名规范，报导该问题。
![](https://user-gold-cdn.xitu.io/2019/7/23/16c1a92eb29bdf1f?w=1208&h=216&f=png&s=26383)
类名不符合规范：

![](https://user-gold-cdn.xitu.io/2019/7/23/16c1a93de9a99458?w=1150&h=236&f=png&s=32058)
从上文可以看到，放在目录下的jar包对所有工程都是有效的。如果要针对单个工程，那么就需要需要AAR形式了。
#### AAR形式
在同个工程新建一个Android Library，名为lintLibrary，修改相关配置。
##### 1、修改Java工程的依赖
修改自定义lint规则的Java库的build.gradle（这里是上文的Java lib库），注意到要将implementation改为compileOnly。
```
apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    //将implementation改为compileOnly，不然报错
    compileOnly 'com.android.tools.lint:lint-api:26.4.2'
    compileOnly 'com.android.tools.lint:lint-checks:26.4.2'

}

jar {
    manifest {
        attributes 'Lint-Registry-v2': 'com.gitcode.lib.Register'
    }
}

sourceCompatibility = "7"
targetCompatibility = "7"
```
##### 2、修改Android Library依赖
Android Library主要用来输出AAR文件，要注意到Android Studio新特性的变更（在这里踩了大坑）。
```
dependencies {
    ......
    
    lintPublish project(':lib')
}
```
在Android Studio 3.4+，`lintChecks project(':lib')`：lint检查只在当前工程生效，也就是Android Library,并不会打包到AAR文件中。` lintPublish project(':lib')`才会将lint检查包含AAR文件中。
##### 3、输出AAR文件
此时跟输出普通的AAR文件没什么区别，但为了手把手教会第一个自定义Issue，我写！

步骤：
```
菜单栏：View->Tool Windows->Gradle
```
此时Android Studio在右边会打开如下窗口：

![](https://user-gold-cdn.xitu.io/2019/7/23/16c1dbf3addc9422?w=826&h=1012&f=png&s=121172)
根据上图操作，双击assemble,稍等一会，在控制台看`BUILD SUCCESSFUL `,则可在下面目录找到AAR文件。
```
lintLibrary->build->outputs->aar
```
这一小节的步骤也可以通过命令行执行。
##### 4、使用AAR文件

有本地依赖或者上传远程仓库，这里只介绍本地依赖。将上小结生成的AAR文件拷贝在app的libs文件夹。并配置app组件的build.gradle
```
repositories {
    flatDir {
        dirs 'libs'
    }
}
dependencies {
    implementation (name:'lintlibrary-release', ext:'aar')
}
```
到这里，就能使用自定义的lint规则了，效果和上面使用jar包是一致的。如果不生效，重启Android Studio看看。
## 采坑记
### 1、Found more than one jar in the 'lintChecks' configuration. Only one file is supported
这是因为在输出AAR文件中，参考其他人的文章。没有将Java Library的依赖改为`compileOnly`。而且Android Library中使用`lintChecks`。
#### 2、输出AAR文件没有生效
不知道为什么，[Linkedin的参考文章没有生效](https://engineering.linkedin.com/android/writing-custom-lint-checks-gradle),可能是Android Studio版本的问题。

另外使用lintChecks输出AAR不生效，Android Studio 3.4+新特性变更，采用lintPublish。

## 总结
花了好长好长的时间写本文，差点就放弃了。因为自己Android Studio看不了lint的源码，只能从网上找，网上又找不到最新的doc。过滤太多雷同文章，差点想哭，一些最新的文章也跟不上相关技术的更新。。。

但是一切都值得，因为能帮助到想学习Android Studio lint工具的同学，一起向往美好的生活。


<center>

[GitHub](https://github.com/GitCode8/GitCode)

**点个赞行不**

</center>

写此文找到的一些具有参考意义的文章：

[Android 官方指导](https://developer.android.google.cn/studio/write/lint)

[Linkedin 指导](https://engineering.linkedin.com/android/writing-custom-lint-checks-gradle)

[美团->Android自定义Lint实践](https://tech.meituan.com/2016/03/21/android-custom-lint.html)

[lint-custom-rules](http://tools.android.com/tips/lint-custom-rules)

另外：本文没有demo，demo的代码已经贴在文章里了。








