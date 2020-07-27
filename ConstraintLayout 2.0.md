
一年前写：[ConstraintLayout,看完一篇真的就够了么？](https://juejin.im/post/5d12c4146fb9a07ea33c24b7) 文章的时候说过，任何技术都会有时限性，只有不断的学习，不断的更新自我，才不会outer。 

目前2.0只是新增了一些新功能和新玩法，对1.x版本无取代之意，所以1.x版本还是得学习的。[好文推荐](https://juejin.im/post/5d12c4146fb9a07ea33c24b7) 2.0版本新增的内容在实践开发也是非常实用的，建议可以上车了。

**由于无知与惰性，让我们感觉摸到了技术的天花板**

**对你有用，帮忙点赞~**

基于本文发表，ConstraintLayout版本已经更新到`2.0.0-beta8`,所以添加依赖的姿势：

`AndroidX:`
```
  implementation 'androidx.constraintlayout:constraintlayout:2.0.0-beta8'
```
支持库：
```
 implementation 'com.android.support.constraint:constraint-layout:2.0.0-beta8'
```

版本说明：

`alpha`:内部测试版,bug多多；

`beta`:公开测试版本，bug少点，支持尝鲜；

`rc`:候选版本，功能不增，和发布版本一致，修修小bug；

`stable`:稳定版本，你尽管有，bug能找到是福气。
### 一、Flow流式虚拟布局(alpha 5增加)

正常情况下，列表显示一般采用ListView或者RecyclerView来实现，但其子Item布局是非常呆板的。想象一下，如果一部作品的详情页结束打上一堆标签，样式如下，该怎么实现？

![](https://user-gold-cdn.xitu.io/2020/7/25/17385442c45e457e?w=361&h=111&f=png&s=16067)
这种布局采用`Flow`来实现特别的简单和方便，而且通过`flow_wrapMode`属性可以设置不同对齐方式。

下面布局代码简单示例：`Flow`布局通过`constraint_referenced_ids`属性将要约束的`View`的`id`进行关联，这里简单关联`A`到`G`的`TextView`,由于`TextView`没有设置约束条件，所以Android Studio 4.0 会报红，给`ConstraintLayout `布局添加`tools:ignore="MissingConstraints"`忽略报红提示。

`Flow`布局的`flow_horizontalGap`属性表示水平之间两个View的水平间隔，`flow_verticalGap`则是垂直方向间隔。
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:ignore="MissingConstraints"//忽略Android Studio 4.0报红提示
    tools:context=".MainActivity">

    <androidx.constraintlayout.helper.widget.Flow
        android:id="@+id/flow"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:constraint_referenced_ids="tvA,tvB,tvC,tvD,tvE,tvF,tvG"
        app:flow_horizontalGap="30dp" //View水平间隔
        app:flow_verticalGap="30dp" //垂直间隔
        app:flow_wrapMode="none"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/tvA"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="A"
        android:textColor="#ffffff"
        android:textSize="16sp"
         />

    <TextView
        android:id="@+id/tvB"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="B"
        android:textColor="#ffffff"
        android:textSize="16sp"
      />

    <TextView
        android:id="@+id/tvC"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="C"
        android:textColor="#ffffff"
        android:textSize="16sp"
       />

    <TextView
        android:id="@+id/tvD"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="D"
        android:textColor="#ffffff"
        android:textSize="16sp"
      />

    <TextView
        android:id="@+id/tvE"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="E"
        android:textColor="#ffffff"
        android:textSize="16sp"
        />

    <TextView
        android:id="@+id/tvF"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="F"
        android:textColor="#ffffff"
        android:textSize="16sp"
      />

    <TextView
        android:id="@+id/tvG"
        android:layout_width="50dp"
        android:layout_height="50dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="G"
        android:textColor="#ffffff"
        android:textSize="16sp"
        />

</androidx.constraintlayout.widget.ConstraintLayout>
```
`flow_wrapMode`属性一共有三种值，在上面的布局的基础上，更换一下不同的值，看一下效果：

**none值**：所有引用的View形成一条链，水平居中，超出屏幕两侧的`View`不可见。


![](https://user-gold-cdn.xitu.io/2020/7/25/173864826dd3f605?w=381&h=90&f=png&s=3446)

**chian值**：所引用的`View`形成一条链，超出部分会自动换行，同行的`View`会平分宽度。

![](https://user-gold-cdn.xitu.io/2020/7/25/173864b35b4b685d?w=381&h=144&f=png&s=5264)


**aligned值**：所引用的`View`形成一条链，但`View`会在同行同列。

![](https://user-gold-cdn.xitu.io/2020/7/25/173864f48e0a9ac2?w=380&h=159&f=png&s=5071)

即然是一条链，那么可以通过链的样式进行约束。
#### 链约束

当`flow_wrapMode`属性为`aligned`和`chian`属性时，通过链进行约束。[ConstraintLayout,看完一篇真的就够了么？](https://juejin.im/post/5d12c4146fb9a07ea33c24b7) 此文有谈到链约束（`Chain`）。

给`Flow`布局添加以下属性进行不同`chain`约束:
* `flow_firstHorizontalStyle`  约束第一条水平链，当有多条链（多行）时，只约束第一条链（第一行），其他链（其他行）不约束；
* `flow_lastHorizontalStyle` 约束最后一条水平链，当有多条链（多行）时，只约束最后一条链（最后一行），其他链（其他行）不约束；
* `flow_horizontalStyle`  约束所有水平链；
* `flow_firstVerticalStyle` 同水平约束；
* `flow_lastVerticalStyle` 同水平约束；
* `flow_verticalStyle` 约束所有垂直链；

以上属性，取值有：`spread`、`spread_inside`、`packed`

**效果：**

**`spread值：`**

![](https://user-gold-cdn.xitu.io/2020/7/26/1738907cc6fa7f8d?w=383&h=149&f=png&s=5696)
代码：
```
    <androidx.constraintlayout.helper.widget.Flow
        android:id="@+id/flow"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="50dp"
        app:constraint_referenced_ids="tvA,tvB,tvC,tvD,tvE,tvF,tvG"
        app:flow_maxElementsWrap="4"
        app:flow_horizontalGap="30dp"
        app:flow_verticalGap="30dp"
        app:flow_wrapMode="chain"
        app:flow_horizontalStyle="spread"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

```
***
**`spread_inside值：`**

![](https://user-gold-cdn.xitu.io/2020/7/26/1738905dc50444a8?w=383&h=146&f=png&s=5227)
代码：
```
  <androidx.constraintlayout.helper.widget.Flow
        android:id="@+id/flow"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="50dp"
        app:constraint_referenced_ids="tvA,tvB,tvC,tvD,tvE,tvF,tvG"
        app:flow_maxElementsWrap="4"
        app:flow_horizontalGap="30dp"
        app:flow_verticalGap="30dp"
        app:flow_wrapMode="chain"
        app:flow_horizontalStyle="spread_inside"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```
***
**`packed值：`**

![](https://user-gold-cdn.xitu.io/2020/7/26/1738909a82b512cb?w=381&h=144&f=png&s=5183)
代码：
```
    <androidx.constraintlayout.helper.widget.Flow
        android:id="@+id/flow"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="50dp"
        app:constraint_referenced_ids="tvA,tvB,tvC,tvD,tvE,tvF,tvG"
        app:flow_maxElementsWrap="4"
        app:flow_horizontalGap="30dp"
        app:flow_verticalGap="30dp"
        app:flow_wrapMode="chain"
        app:flow_horizontalStyle="packed"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

```
其他效果大家在实践可以尝试看看效果，建议**点赞**收藏本文，在使用不会可以翻阅一下，效率事半功倍，免得重新浪费时间谷歌搜索。
***
### 对齐
上文XML布局中，所有TextView的宽高是一致的，所以看着整整齐齐，当宽高不一致时，可以进行对齐处理。个人试了一下`app:flow_wrapMode="aligned"`下的对齐，没啥效果，估计有默认值了吧。看看`flow_wrapMode`属性为`none`和`chain`情况吧。

给`Flow`布局添加以下属性进行不同`Align`约束:

* `flow_verticalAlign` 垂直方向对齐,取值有：`top`、`bottom`、`center`、`baseline`;
* `flow_horizontalAlign` 水平方向对齐,取值有：`start`、`end`、`center`;

对齐方向一般与链的方向相反才可生效，例如垂直链样式，一般对齐`View`的左右边和中间。

简单举个例子：垂直方向顶部对齐。

**效果图：**

![](https://user-gold-cdn.xitu.io/2020/7/26/173891ad7c95153e?w=385&h=132&f=png&s=5590)
可以看到`E`和`G`、`F`顶部对齐。

代码：
```
    <androidx.constraintlayout.helper.widget.Flow
        android:id="@+id/flow"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="50dp"
        app:constraint_referenced_ids="tvA,tvB,tvC,tvD,tvE,tvF,tvG"
        app:flow_maxElementsWrap="4"
        app:flow_horizontalGap="30dp"
        app:flow_verticalGap="30dp"
        app:flow_wrapMode="chain"
        app:flow_verticalAlign="top"
        app:layout_constraintHorizontal_chainStyle="spread"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```
简单的理解`aligned`和`chian`是`none`的定制版，通过添加不同的属性定制而成。由于`Flow`是虚拟布局，简单理解就是约束助手，它并不会增加布局层级，却可以像正常的布局一样使用。

**其他属性**

上文的XML的布局没有设置Flow对View的组织方式（水平or 垂直），可以通过`orientation`属性来设置水平`horizontal`和垂直`vertical`方向，例如改为垂直方向。

![](https://user-gold-cdn.xitu.io/2020/7/27/1738e1e8a187f05f?w=378&h=605&f=png&s=12567)
***
当`flow_wrapMode`属性为`aligned`和`chian`时，通过`flow_maxElementsWrap`属性控制每行最大的子`View`数量。例如：`flow_maxElementsWrap=3`。

![](https://user-gold-cdn.xitu.io/2020/7/25/173866aa230a00d6?w=382&h=203&f=png&s=4851)
***
当`flow_wrapMode`属性为`none`时，A和G被挡住了，看不到。

![](https://user-gold-cdn.xitu.io/2020/7/27/1738e2397fc8c655?w=382&h=86&f=png&s=4009)
要A或者G可见，通过设置`flow_horizontalBias`属性，取值在`0-1`之间。前提条件是`flow_horizontalStyle`属性为`packed`才会生效。
```
   <androidx.constraintlayout.helper.widget.Flow
        android:id="@+id/flow"
        android:orientation="horizontal"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:constraint_referenced_ids="tvA,tvB,tvC,tvD,tvE,tvF,tvG"
        app:flow_horizontalGap="30dp"
        app:flow_verticalGap="30dp"
        app:flow_wrapMode="none"
        app:flow_horizontalStyle="packed"
        app:flow_horizontalBias="0"
        android:layout_marginTop="10dp"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

```
**效果图：**
![](https://user-gold-cdn.xitu.io/2020/7/27/1738e2744e9bd356?w=386&h=70&f=png&s=3618)
设置`flow_horizontalBias=1`那么G就可以看到了。该属性还有其他类似ChainStyle的属性w玩法，具体可以实践体验。当然，也可以在`flow_wrapMode`属性为其他值生效。

通过不同的属性可以搭配很多不同的效果，再加上MotionLayout动画，那就更炫酷了。

### 二、Layer 层布局
Layer也是一个约束助手`ConstraintHelper`，相对Flow比较简单，常用来增加背景，或者共同动画。由于`ConstraintHelper`本身继承自`View`，跟我们自己通过View在`ConstraintLayout`布局中给多个View添加共同背景没什么区别，只是更方便而已。

**1、添加背景**

给`ImageView`和`TextView`添加个共同背景：

**效果：**

![](https://user-gold-cdn.xitu.io/2020/7/27/1738e4288d69819c?w=182&h=158&f=png&s=7058)

**代码：**
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"

    android:layout_height="match_parent">


    <androidx.constraintlayout.helper.widget.Layer
        android:id="@+id/layer"
        android:layout_marginTop="50dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="@color/colorPrimary"
        app:constraint_referenced_ids="ivImage,tvName"
        app:layout_constraintLeft_toLeftOf="@id/ivImage"
        app:layout_constraintRight_toRightOf="parent"
        android:padding="10dp"
        app:layout_constraintTop_toTopOf="parent"
        tools:ignore="MissingConstraints" />


    <ImageView
        android:id="@+id/ivImage"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="10dp"
        android:src="@mipmap/ic_launcher_round"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toBottomOf="@id/layer" />

    <TextView
        android:id="@+id/tvName"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="新小梦"
        android:textColor="#FFFFFF"
        android:paddingTop="5dp"
        app:layout_constraintLeft_toLeftOf="@id/ivImage"
        app:layout_constraintRight_toRightOf="@id/ivImage"
        app:layout_constraintTop_toBottomOf="@id/ivImage" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
**2、共同动画**

通过属性动画给ImageView和TextView添加通过动画效果。

**效果：**

![](https://user-gold-cdn.xitu.io/2020/7/27/1738e4f8f76ca1fc?w=212&h=170&f=gif&s=183294)
**代码：**
```
val animator = ValueAnimator.ofFloat( 0f, 360f)
animator.repeatMode=ValueAnimator.RESTART
animator.duration=2000
animator.interpolator=LinearInterpolator()
animator.repeatCount=ValueAnimator.INFINITE
animator.addUpdateListener {
    layer.rotation=  it.animatedValue as Float
}
layer.setOnClickListener {
   animator.start()
}
```
对属性动画模糊的同学可以看看：[Android属性动画,看完这篇够用了吧](https://juejin.im/post/5ef8399e5188252e415f40d8)

支持：旋转、位移、缩放动画。透明效果试了一下，是针对自身的，而不是约束的View。
### 三、自定义ConstraintHelper
`Flow`和`Layer`都是`ConstraintHelper`的子类，当两者不满足需求时，可以通过继承`ConstraintHelper`来实现想要的约束效果。

在某乎APP有这么个类似的动画广告：


![](https://user-gold-cdn.xitu.io/2020/7/27/1738e6da61c210aa?w=351&h=158&f=gif&s=81294)
那么通过自定义ConstraintHelper来实现就非常简单：
```
class AdHelper :
    ConstraintHelper {

    constructor(context: Context?) : super(context)

    constructor(context: Context?,attributeSet: AttributeSet):super(context,attributeSet)

    constructor(context: Context?,attributeSet: AttributeSet,defStyleAttr: Int):super(context,attributeSet,defStyleAttr)


    override fun updatePostLayout(container: ConstraintLayout?) {
        super.updatePostLayout(container)
        val views = getViews(container)
        views.forEach {
            val anim = ViewAnimationUtils.createCircularReveal(it, 0, 0, 0f, it.width.toFloat())
            anim.duration = 5000
            anim.start()
        }
    }

}
```
布局引用`AdHleper`：
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.example.constraint.AdHelper
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:constraint_referenced_ids="ivLogo"
        app:layout_constraintLeft_toLeftOf="@id/ivLogo"
        app:layout_constraintRight_toRightOf="@id/ivLogo"
        app:layout_constraintTop_toTopOf="@id/ivLogo" />

    <ImageView
        android:id="@+id/ivLogo"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="20dp"
        android:adjustViewBounds="true"
        android:scaleType="fitXY"
        android:src="@mipmap/ic_logo"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
### 四、ImageFilterButton
圆角图片，圆形图片怎么实现？自定义View?通过`ImageFilterButton`,一个属性就搞定；`ImageFilterButto`能做的还有更多。

看看如何实现圆角或圆形图片:

**原图：**

![](https://user-gold-cdn.xitu.io/2020/7/27/1738eb640d862345?w=131&h=132&f=png&s=4265)
将`roundPercent`属性设置为`1`，取值在`0-1`，由正方形向圆形过渡。
```
    <androidx.constraintlayout.utils.widget.ImageFilterButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="100dp"
        app:roundPercent="1"
        android:src="@mipmap/ic_launcher"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```
**效果：**

![](https://user-gold-cdn.xitu.io/2020/7/27/1738eb598fae81cd?w=193&h=167&f=png&s=5759)
也可以通过设置`round`属性来实现：
```
    <androidx.constraintlayout.utils.widget.ImageFilterButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="100dp"
        android:src="@mipmap/ic_launcher"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:round="50dp" />
```
***
**其他属性：**

`altSrc`和`src`属性是一样的概念，`altSrc`提供的资源将会和`src`提供的资源通过`crossfade`属性形成交叉淡化效果。默认情况下,`crossfade=0`，`altSrc`所引用的资源不可见,取值在`0-1`。
例如：
```
    <androidx.constraintlayout.utils.widget.ImageFilterButton
        android:id="@+id/ivImage"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="100dp"
        android:src="@mipmap/ic_launcher"
        app:altSrc="@mipmap/ic_sun"
        app:crossfade="0.5"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:round="50dp" />
```
crossfade=0.5时，效果：

![](https://user-gold-cdn.xitu.io/2020/7/27/1738ec620410585f?w=143&h=124&f=png&s=7436)
crossfade=1时，效果：

![](https://user-gold-cdn.xitu.io/2020/7/27/1738ec748a420ec1?w=141&h=126&f=png&s=6392)
***
接下来几个属性是对图片进行调节：

`warmth`色温：`1=neutral自然, 2=warm暖色, 0.5=cold冷色`

![](https://user-gold-cdn.xitu.io/2020/7/27/1738ed7880cc6383?w=304&h=171&f=png&s=14387)
`brightness`亮度：`0 = black暗色, 1 = original原始, 2 = twice as bright两倍亮度`；这个效果不好贴图，大家自行验证；

`saturation`饱和度：`0 = grayscale灰色, 1 = original原始, 2 = hyper saturated超饱和`；

![](https://user-gold-cdn.xitu.io/2020/7/27/1738edb9ef446400?w=339&h=199&f=png&s=22414)
`contrast`对比：`1 = unchanged原始, 0 = gray暗淡, 2 = high contrast高对比`;

![](https://user-gold-cdn.xitu.io/2020/7/27/1738ee14859d5bc0?w=339&h=199&f=png&s=18831)

上面属性的取值都是0、1、2，不过大家可以取其他值，效果也是不一样的。
最后一个属性`overlay`,表示不知道怎么用，看不到没效果，大家看看评论跟我说声？

![](https://user-gold-cdn.xitu.io/2020/7/27/1738ee64a68b92c8?w=839&h=70&f=png&s=10429)
### 五、ImageFilterView
`ImageFilterView`与`ImageFilterButton`的属性一模一样，只是它两继承的父类不一样，一些操作也就不一样。`ImageFilterButton`继承自`AppCompatImageButton`,也就是`ImageButtion`。而`ImageFilterView`继承自`ImageView `。

![](https://user-gold-cdn.xitu.io/2020/7/27/1738ee9bd310529c?w=858&h=599&f=png&s=84433)
### 六、MockView
还记得你家项目经理给你的UI原型图么？想不想回敬一下项目经理，是时候了~
![](https://user-gold-cdn.xitu.io/2020/7/27/1738eef74f180c46?w=329&h=671&f=png&s=25058)
`MockView`能简单的帮助构建UI界面，通过对角线形成的矩形+标签。例如：
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.constraintlayout.utils.widget.MockView
        android:id="@+id/first"
        android:layout_width="100dp"
        android:layout_height="100dp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <androidx.constraintlayout.utils.widget.MockView
        android:id="@+id/second"
        android:layout_width="100dp"
        android:layout_height="100dp"
        app:layout_constraintLeft_toRightOf="@id/first"
        app:layout_constraintTop_toBottomOf="@id/first" />
    
</androidx.constraintlayout.widget.ConstraintLayout>
```
**效果：**

![](https://user-gold-cdn.xitu.io/2020/7/27/1738ef385aa17261?w=386&h=286&f=png&s=13402)
中间黑色显示的是`MockView`的`id`。通过`MockView`可以很好的构建一些UI思路。
### 七、MotionLayout
MitionLayou主要是用来实现动作动画，可以参考我的另一篇文章：[Android MotionLayout动画：续写ConstraintLayout新篇章](https://juejin.im/post/5f0e9eea6fb9a07e7e0444e3)
### 八、边距问题的补充
有`ConstraintLayout`实践经验的朋友应该知道`margin`设置负值在`ConstraintLayout`是没有效果的。例如下面布局：TextView B的`layout_marginLeft`和`layout_marginTop`属性是不会生效的。
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/vA"
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:layout_marginLeft="30dp"
        android:layout_marginTop="30dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="A"
        android:textColor="#FFFFFF"
        android:textSize="20sp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/vB"
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:layout_marginLeft="-30dp"
        android:layout_marginTop="-30dp"
        android:background="@color/colorAccent"
        android:gravity="center"
        android:text="B"
        android:textColor="#FFFFFF"
        android:textSize="20sp"
        app:layout_constraintLeft_toRightOf="@id/vA"
        app:layout_constraintTop_toBottomOf="@id/vA" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
**效果：**
![](https://user-gold-cdn.xitu.io/2020/7/27/1738f06726a6dff8?w=384&h=225&f=png&s=4204)
可以通过轻量级的`Space`来间接实现这种效果。
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/vA"
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:layout_marginLeft="30dp"
        android:layout_marginTop="30dp"
        android:background="@color/colorPrimary"
        android:gravity="center"
        android:text="A"
        android:textColor="#FFFFFF"
        android:textSize="20sp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Space
        android:id="@+id/space"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_marginRight="30dp"
        android:layout_marginBottom="30dp"
        app:layout_constraintBottom_toBottomOf="@id/vA"
        app:layout_constraintRight_toRightOf="@id/vA" />

    <TextView
        android:id="@+id/vB"
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:layout_marginLeft="-30dp"
        android:layout_marginTop="-30dp"
        android:background="@color/colorAccent"
        android:gravity="center"
        android:text="B"
        android:textColor="#FFFFFF"
        android:textSize="20sp"
        app:layout_constraintLeft_toRightOf="@id/space"
        app:layout_constraintTop_toBottomOf="@id/space" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
**效果：**
![](https://user-gold-cdn.xitu.io/2020/7/27/1738f0bdb4f18d5a?w=378&h=211&f=png&s=3960)


2.0还增加了`ConstraintProperties`类用于通过api(代码)更新`ConstraintLayout`子视图；其他一些可以参考官方文档，估计也差不多了。

**参考：**

[官方英文文档](https://developer.android.com/reference/androidx/constraintlayout/classes)

能读到末尾的小伙伴都是很棒，耐力很好。如果本文对你有用，帮忙**点个赞**，推荐好文。


最后的最后，个人能力有限，有理解错误或者错误的地方，希望大家帮忙纠正，非常感谢。
