# 转场动画

当从 Activity A 切换到  Activity B 的时候，Activity 布局的内容会按照预先定义好的动画来执行过渡动画。
在 android.transition 包中，已经有三种现成的动画可以用: Explode，Slide 和 Fade。所有这些过渡都会跟踪
Activity 布局中可见的目标 Views，驱动这些 Views 按照过渡的规则产生响应的动画效果。



转场中发生了什么:

1. Activity A 启动 Activity B
2. Transition Framework 发现 **A** 中定义了`Exit Transition (slide)` 然后就会对它的所有可见的View使用这个过渡动画.
3. Transition Framework 发现 **B** 中定义了`Enter Transition (fade)` 然后机会对它所有可见的Views使用这个过渡动画.
4. On Back Pressed(按返回键) Transition Framework 会执行把 Enter and Exit 过渡动画反过来执行(但是如果定义了 returnTransition 和 reenterTransition，那么就会执行这些定义的动画)，



## xml创建

定义在 xml中， 目录是 res/transition

```
<fade xmlns:android="http://schemas.android.com/apk/res/"
    android:duration="1000"/>
    
<slide xmlns:android="http://schemas.android.com/apk/res/"
    android:duration="1000"/>
 
```

使用 TransitionInflater 来实例化它们

```
private void setupWindowSlideAnimations() {
    Slide slide =      TransitionInflater.from(this).inflateTransition(R.transition.activity_slide);
    getWindow().setExitTransition(slide);
}

private void setupWindowFadeAnimations() {
    Fade fade = TransitionInflater.from(this).inflateTransition(R.transition.activity_fade);
    getWindow().setEnterTransition(fade);
}
```

### 代码创建

```
private void setupWindowFadeAnimations() {
    Fade fade = new Fade();
    fade.setDuration(1000);
    getWindow().setEnterTransition(fade);
}

private void setupWindowSlideAnimations() {
    Slide slide = new Slide();
    slide.setDuration(1000);
    getWindow().setExitTransition(slide);
}
```

### 启动

> 要使这些过渡动画生效，我们需要调用 startActivity(intent，bundle) 方法来启动 Activity。bundle 需要通过 ActivityOptionsCompat.makeSceneTransitionAnimation().toBundle() 的方式来生成



### 允许过渡效果的重叠

另外,在界面切换的时候，动画是一起进行的。如果想实现分开
A进入B时,B中设置getWindow().setAllowEnterTransitionOverlap(false);退出时，A中设置 getWindow().setAllowReturnTransitionOverlap(false);默认为true。

也可以去排除一些view的动画效果

```
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android">
<slide android:duration="1000"
      android:slideEdge="right">
    <targets >
        <!--表示除了状态栏-->
        <target android:excludeId="@android:id/statusBarBackground"/>
        <!--表示只针对状态栏-->
 <!--<target android:targetId="@android:id/statusBarBackground"/>-->  </targets>
</slide>
</transitionSet>
```



# Activity 之间共享元素(Share Elements)

这里的思想就是通过动画的形式将两个不同布局中的两个不同的View联系起来。
然后 Transition framework 就会在用户从一个View切换到另一个View的时候给用户展现一些必要的动画。
但你要记住:发生动画的View并不是从一个布局中移动到另一个布局。他们是两个独立的View。为了使共享元素动画生效，你需要给共享元素的两个View设置**相同**的`android:transitionName`属性值。不过他们的id和其他属性可以不同。

```
<TextView
  android:text="withShared"
  android:transitionName="shared_text_"
 style="@style/MaterialAnimations.TextAppearance.Title.Inverse"
  android:layout_width="wrap_content"
  android:layout_height="wrap_content" />


<de.hdodenhof.circleimageview.CircleImageView
  android:id="@+id/icon_gg"
  android:layout_centerInParent="true"
  android:src="@mipmap/xkl"
  android:transitionName="shared_image_"
  android:layout_width="150dp"
  android:layout_height="150dp" />
```



最后，使用 `ActivityOptions.makeSceneTransitionAnimation()` 方法指定要共享元素的 View 和
`android:transitionName` 属性的值

```
ActivityOptionsCompat activityOptionsCompat = ActivityOptionsCompat.makeSceneTransitionAnimation(this
  ,new Pair<View, String>(shared_image,"shared_image_")
        ,new Pair<View, String>(shared_text,"shared_text_"));
startActivity(intent,activityOptionsCompat.toBundle());
```



### fragment 之间实现 Shared elements 过渡动画

只需要在使用 FragmentTransaction 启动 fragment 的时候，需要同时带上共享元素过渡动画的相关信息

```
Slide slideTransition = new Slide(Gravity.RIGHT);
slideTransition.setDuration(1000);
sharedElementFragment2.setEnterTransition(slideTransition);

// setSharedElementEnterTransition可以对共享的view设置转换
ChangeBounds changeBoundsTransition = TransitionInflater.from(this).inflateTransition(R.transition.change_bounds);
fragmentB.setSharedElementEnterTransition(changeBoundsTransition);

getFragmentManager().beginTransaction()
        .replace(R.id.content, fragmentB)
        .addSharedElement(blueView, getString(R.string.blue_name))
        .commit();
```

共享元素转换也可以添加监听

```
sharedTransition.addListener(new Transition.TransitionListener() {
            @Override
            public void onTransitionEnd(Transition transition) {
                
            }
});
```



### 延时共享元素

有的时候又有这种情况，比如要展示一个网络图片，在网络图片获取到之前，这个共享元素的动画效果没啥作用。怎么的也得等我图片获取完成之后在开始共享元素的动画效果吧，这个时候延时元素动画就派上大用场了。 `postponeEnterTransition()`函数用于延时动画，`startPostponedEnterTransition()`函数用于开始延时的共享动画。那咱们就可以这么干了，在Activity进入的时候先调用`postponeEnterTransition()`延时动画，在网络图片获取完成之后在调用`startPostponedEnterTransition()`开始动画

```
postponeEnterTransition();
Picasso.with(this)
                .load(R.drawable.k)
                .into(imageView, new Callback() {
                    @Override
                    public void onSuccess() {
                        scheduleStartPostponedTransition(imageView);
                    }

                    @Override
                    public void onError() {

                    }
                });




private void scheduleStartPostponedTransition(final View sharedElement) {
        sharedElement.getViewTreeObserver().addOnPreDrawListener(
                new ViewTreeObserver.OnPreDrawListener() {
                    @Override
                    public boolean onPreDraw() {
                        sharedElement.getViewTreeObserver().removeOnPreDrawListener(this);
                        startPostponedEnterTransition();//完成之后在调用
                        return true;
                    }
                });
    }
```

### 更新共享元素对应关系

有这种情况，比如我们第一个界面是一个列表(RecyclerView)每个item都是一个图片，点击进入另一个页面详情页面，详情页面呢有是`ViewPager`的形式。可以左右滑动。咱们有的时候就想，就算详情界面滑动到了其他照片，在返回到第一个页面的时候也想要有共享元素动画的效果。这个时候就得更新下共享元素的对应关系了。怎么更新呢?关键是看SharedElementCallback类的`onMapSharedElements()`函数，这个函数是用来**装载共享元素****的。比如有这么个情况，还是上面的例子A界面跳转到B界面。那么A界面在B返回的时候要更新下、B界面在返回之前要更新下。所以给A界面设置**setExitSharedElementCallback(SharedElementCallback)**;给B界面设置**setEnterSharedElementCallback(SharedElementCallback)**