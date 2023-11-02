# View动画

View动画作用对象是View，它支持**4**种动画效果，分别是**平移动画**，**缩放动画**，**旋转动画**和**透明度动画**。四种变换效果分别对应着Animation的四个子类：**TranslateAnimation**，**ScaleAnimation**，**RotateAnimation**，**AlphaAnimation**。四种动画既可以通过`XML`来定义，也可以通过`代码`来动态创建。对于View动画来说，建议采用`XML`来定义

|    名称    |   标签    |        子类        |       效果       |
| :--------: | :-------: | :----------------: | :--------------: |
|  平移动画  | translate | TranslateAnimation |     移动View     |
|  缩放动画  |   scale   |   ScaleAnimation   |  放大或缩小View  |
|  旋转动画  |  rotate   |  RotateAnimation   |     旋转View     |
| 透明度动画 |   alpha   |   AlphaAnimation   | 改变View的透明度 |



## XML定义

首先要创建动画的XML文件，文件的路径是:**res/anim/filename.xml**。

View动画的描述文件有固定的语法的。如下所示

```
<set
    android:duration="10000"                        单位毫秒
    android:fillAfter="true/false"      结束是否停留在当前位置                 
    android:fillBefore="true/false"
    android:interpolator=""                       设定插值器
    android:shareInterpolator="true /false">   是否共用插值器
        
    <alpha
          android:fromAlpha="float"     
          android:toAlpha="float"/>           
    <!--动画开始的透明度（0.0到1.0，0.0是全透明，1.0是不透明） -->
    
    <scale 
          android:fromXScale="float"         水平方向缩放的起始值 比例0.5
          android:toXScale="float"           水平方向缩放的结束值 比例1.2
          android:fromYScale="float"
          android:toYScale="float"
          android:pivotX="float"                    缩放轴点的x坐标
          android:pivotY="float"/>

    <rotate
        android:fromDegrees="float"  
        android:toDegrees="float"          
        android:pivotY="float"              
        android:pivotX="float"/>
    <!--起始旋转角度 0 旋转开始角度，正代表顺时针度数，负代表逆时针度数 -->
        
    <translate
        android:fromXDelta="float"           x的起始值
        android:toXDelta="float"             x的结束值
        android:fromYDelta="float"           y的起始值
        android:toYDelta="float"/>           y的结束值
        
</set>

```

- pivotX，pivotY——pivotX的X轴坐标可以为**数值**、**百分数**、**百分数p**，譬如50表示以当前View左上角坐标加50px为初始点、50%表示以当前View的左上角加上当前View宽高的50%做为初始点、50%p表示在父控件宽高的50%位置做为初始点）
- repeatMode——重复类型有两个值，**reverse**表示倒序回放，**restart**表示从头播放



如何从`XML`中加载动画呢？如下

```
Animation anim= AnimationUtils.loadAnimation(this, R.anim.set);
view.startAnimation(anim)
```

## 代码加载

除了XML加载外，还可以通过代码来实现。

透明度动画，如下所示

```
AlphaAnimation alphaAnimation = new AlphaAnimation(0,1);
alphaAnimation.setDuration(300);
mView.startAnimation(alphaAnimation);
```

平移动画，如下所示

```
 TranslateAnimation translate = new TranslateAnimation(
              Animation.RELATIVE_TO_SELF, 0f, Animation.RELATIVE_TO_SELF, 0f,
              Animation.RELATIVE_TO_SELF, -1f, Animation.RELATIVE_TO_SELF, 0f);
              translate.setDuration(2000);
              mButton.startAnimation(translate);
       
```

`Animation.RELATIVE_TO_SELF`(相对于自身)，`Animation.RELATIVE_TO_PARENT`(相对于父控件(容器)。
另外在上面y是负,在左边x是负，跟手机坐标系一致

## 动画监听

通过Animation的setAnimationListener方法可以为View动画添加过程监听

```
   public interface AnimationListener {
        void onAnimationStart(Animation var1);

        void onAnimationEnd(Animation var1);

        void onAnimationRepeat(Animation var1);
    }
```



## 其他方法

| Animation 类的方法                               | 解释                          |
| :----------------------------------------------- | ----------------------------- |
| reset()                                          | 重置 Animation 的初始化       |
| cancel()                                         | 取消 Animation 动画           |
| start()                                          | 开始 Animation 动画           |
| setAnimationListener(AnimationListener listener) | 给当前 Animation 设置动画监听 |
| hasStarted()                                     | 判断当前 Animation 是否开始   |
| hasEnded()                                       | 判断当前 Animation 是否结束   |



# 补充

- Q:View视图动画为何不能改变View的位置？

  **补间动画执行之后并未改变 View 的真实布局属性值,产生的动画数据实际是应用在canvas**。切记这一点，譬如我们在 Activity 中有一个 Button 在屏幕上方， 我们设置了平移动画移动到屏幕下方然后保持动画最后执行状态呆在屏幕下方，这时如果点击屏幕下方动画执行之后的 Button 是没有任何反应的， 而点击原来屏幕上方没有 Button 的地方却响应的是点击Button的事件

- 在有强依赖onAnimationEnd回调的交互时,如播放完毕才能操作页面时，onAnimationEnd可能因为各种异常没有回调，建议加上超时保护或通过postDelay替代onAnimationEnd。

```
  new Handler().postDelay(new Runnable(){
          public void run(){
                   if(view!=null){
                           v.clearAnimation();  
                  }
          }
  })
```

​       还有动画完成后View无法隐藏，即setVisibity(View.GONE)失效了，这个时候，在View的annimation动画结束时，通过clearAnimation释放资源，清除动画可解决此问题

```
animation.setAnimationListener(new Animation.AnimationListener() {
                    @Override
                    public void onAnimationStart(Animation animation) {

                    }

                    @Override
                    public void onAnimationEnd(Animation animation) {
                        if(mButton!=null){
                            mButton.clearAnimation();  //判断资源是否被释放了
                        }
                    }

                    @Override
                    public void onAnimationRepeat(Animation animation) {

                    }
                });
```

