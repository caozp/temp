# ViewPager2

**viewpager2 内部实现原理是使用recycleview加LinearLayoutManager实现竖直滚动, 其实可以理解为对recyclerview的二次封装**

```
 private void initialize(Context context, AttributeSet attrs) {
        mRecyclerView = new RecyclerView(context) {
            @Override
            public CharSequence getAccessibilityClassName() {
                return "androidx.viewpager.widget.ViewPager";
            }

            @Override
            public void onInitializeAccessibilityEvent(@NonNull AccessibilityEvent event) {
                super.onInitializeAccessibilityEvent(event);
                event.setFromIndex(mCurrentItem);
                event.setToIndex(mCurrentItem);
            }
        };
        mRecyclerView.setId(ViewCompat.generateViewId());

        mLayoutManager = new LinearLayoutManager(context);
        mRecyclerView.setLayoutManager(mLayoutManager);
        setOrientation(context, attrs);

        mRecyclerView.setLayoutParams(
                new ViewGroup.LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT));
        .....
        attachViewToParent(mRecyclerView, 0, mRecyclerView.getLayoutParams());
    }
```



ViewPager2位于androidx包下

```
dependencies {
    implementation "androidx.viewpager2:viewpager2:1.0.0"
}
```



### 变化

* recyclerview的adapter 替代了recyclerview的adapter。

* FragmentStateAdapter 替代了 FragmentPagerAdapter

* 可以设置方向

  > viewPager2.orientation = ViewPager2.ORIENTATION_VERTICAL

* 滑动监听ViewPager.OnPageChangeListener  -> ViewPager2.OnPageChangeCallback()

* 禁止用户滑动  **isUserInputEnabled**

* ViewPager2新增了一个fakeDragBy的方法。通过这个方法可以来模拟拖拽。在使用fakeDragBy前需要先beginFakeDrag方法来开启模拟拖拽。fakeDragBy会返回一个boolean值，true表示有fake drag正在执行，而返回false表示当前没有fake drag在执行。我们通过代码来尝试下：

  ```
  fun fakeDragBy(view: View) {
          viewPager2.beginFakeDrag()
          if (viewPager2.fakeDragBy(-310f))
              viewPager2.endFakeDrag()
      }
      
   注意到是fakeDragBy接受一个float的参数，当参数值为正数时表示向前一个页面滑动，当值为负数时表示向下一个页面滑动
  ```



### PageTransformer

相比ViewPager，ViewPager2的Transformer功能有了很大的扩展。ViewPager2不仅可以通过PageTransformer不仅可以用来设置pageMarge，还可以同时添加多个PageTransformer。接下来我们就来认识下ViewPager2的PageTransformer吧

###### setPageMarge

在第一章中我们提到了ViewPager2移除了setPageMargin方法，那么怎么为ViewPager2设置页面间距呢？其实在ViewPager2中为我们提供了MarginPageTransformer，我们可以通过ViewPager的setPageTransformer方法来设置页面间距

```
viewPager2.setPageTransformer(MarginPageTransformer(100)
```

###### 页面动画

为ViewPager2设置了页面间距后如果还想设置页面动画的Transformer怎么办呢？这时候就该CompositePageTransformer出场了。从名字上也可以看出来它是一个组合的PageTransformer。没错，CompositePageTransformer实现了PageTransformer接口，同时在其内部维护了一个List集合，我们可以将多个PageTransformer添加到CompositePageTransformer中

```text
	val compositePageTransformer = CompositePageTransformer()
    compositePageTransformer.addTransformer(ScaleInTransformer())
    compositePageTransformer.addTransformer(MarginPageTransformer(100)
    viewPager2.setPageTransformer(compositePageTransformer)
```

ViewPager2的PageTransformer和ViewPager的PageTransformer实现方式一模一样



###### 一屏多页效果

ViewPager2的一屏多页实现跟ViewPager并无多大差别。需要把ViewPager2父容器的clipChildren设置为false，同时ViewPager2的clipChildren也设置为false并且将ViewPager2的offscreenPageLimit 设置为2以及为ViewPager2设置marge即可

```
        container.clipChildren = false
        viewpager.clipChildren = false
        viewpager.offscreenPageLimit = 2
        val params = viewpager.layoutParams as ViewGroup.MarginLayoutParams
        params.leftMargin = resources.getDimension(R.dimen.dp_50).toInt()
        params.rightMargin = params.leftMargin

        val compositePageTransformer = CompositePageTransformer()
        compositePageTransformer.addTransformer(ScaleInTransformer())
        compositePageTransformer.addTransformer(MarginPageTransformer(100))
        viewpager.setPageTransformer(compositePageTransformer)

```





## 懒加载

https://mp.weixin.qq.com/s/bBdzJMzRoBDUVeeLFd4z2w

Fragment#setUserVisibleHint(isVisibleToUser: Boolean)方法已被标记为废弃了，并且从注释中可以看到使用 FragmentTransaction#setMaxLifecycle(Fragment, Lifecycle.State)方法来替换setUserVisibleHint方法。

setMaxLifecycle实在Androidx 1.1.0中新增加的一个方法。setMaxLifecycle从名字上来看意思是设置一个最大的生命周期，因为这个方法是在FragmentTransaction中，因此我们可以知道应该是为Fragment来设置一个最大的生命周期

ViewPager2的懒加载通过setMaxLifecycle来实现。

在Lifecycle的State中定义了五种生命周期状态

* DESTROYED
* INITIALIZED
* CREATED
* STARTED
* RESUMED

而在setMaxLifecycle中接收的生命周期状态要求不能低于CREATED，否则会抛出一个IllegalArgumentException的异常



意思就是如果fragment的生命周期 > MaxLifecycle

**案例1**：如果fragment的生命周期处于onResume状态，而设置的setMaxLifecycle是CREATED,那么已经执行了onResume的Fragment，将其最大生命周期设置为CREATED后会执行onPause->onStop->onDestoryView。也就是回退到了onCreate的状态

如果fragment的生命周期 <= MaxLifecycle

那么fragment的生命周期就会停在MaxLifecycle那里



ViewPager2也是同样的原理，只要设置**offscreenPageLimit = 1**

```
abstract class TestLifecycleFragment : Fragment() {
    private var isFirstLoad = true

    override fun onResume() {
        super.onResume()
        if (isFirstLoad) {
            isFirstLoad = false
            loadData()
        }
    }

    abstract fun loadData()
}
```

