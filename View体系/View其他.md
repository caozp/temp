## TouchSlop

系统所能识别出的被认为是滑动的最小距离。这是个常量，可以通过如下方式获得

> ViewConfiguration vc= ViewConfiguration.get(context);
>
> vc.getScaledTouchSlop();

ViewConfiguration还有其他方法

> //  获得允许执行fling （抛）的最大速度值
> static int getScaledMaximumFlingVelocity();
> 需要配合VelocityTracker速度追踪来使用
>
> static float getScrollFriction()
> 一个代表了摩擦系数的标量。它应用在flings 或 scrolls 状态。mScroll.setFriction
>
> static int getDoubleTapTimeout()
> 返回一个 双击的毫秒超时时间。
> 它在第一次点击的up事件开始记录，在第二次点击的down事件停止。
> 两次点击之间的时间值小于等于getDoubleTapTimeout()，就表示是一个双击操作



## VelocityTracker追踪速度

速度追踪，用于追踪手指在滑动过程中的速度。首先在onTouchEvent中追踪当前事件的速度

```
VelocityTracker mVelocityTracker = VelocityTracker.obtain();
mVelocityTracker.addMovement(event);
```

当我们想获得当前速度时，可以

```
mVelocityTracker.computeCurrentVelocity(1000);
int xVelocity = mVelocityTracker.getXVelocity();
int yVelocity = mVelocityTracker.getYVelocity();
```

需要注意

1. 获取速度前必须计算速度，也就是调用computeCurrentVelocity方法
2. 速度指的是一段时间内手指滑过的像素数。向左为负值。

不需要时，使用clear方法来重置并回收内存

```
mVelocityTracker.clear();
mVelocityTracker.recycle();
```

## GestureDetector手势检测

辅助检测用户的单击，双击，长按，滑动等行为。

创建GestureDetector对象，并实现OnGestureListener接口

```
 GestureDetector mGestureDetector = new GestureDetector(this);
```

接管View的onTouchEvent方法，在onTouchEvent方法中添加实现

```
boolean consume = mGestureDetector.onTouchEvent(ev);
return consume;
```

示例

```
 // 步骤1：创建手势检测器实例 & 传入OnGestureListener接口（需要复写对应方法）
        mGestureDetector = new GestureDetector(this, new GestureDetector.OnGestureListener() {

            // 1. 用户轻触触摸屏
            public boolean onDown(MotionEvent e) {
                Log.i("MyGesture", "onDown");
                return false;
            }

            // 2. 用户轻触触摸屏，尚未松开或拖动
            // 与onDown()的区别：无松开 / 拖动
            // 即：当用户点击的时，onDown（）就会执行，在按下的瞬间没有松开 / 拖动时onShowPress就会执行
            public void onShowPress(MotionEvent e) {
                Log.i("MyGesture", "onShowPress");
            }

            // 3. 用户长按触摸屏
            public void onLongPress(MotionEvent e) {
                Log.i("MyGesture", "onLongPress");
            }

            // 4. 用户轻击屏幕后抬起
            public boolean onSingleTapUp(MotionEvent e) {
                Log.i("MyGesture", "onSingleTapUp");
                return true;
            }

            // 5. 用户按下触摸屏 & 拖动
            public boolean onScroll(MotionEvent e1, MotionEvent e2,
                                    float distanceX, float distanceY) {
                Log.i("MyGesture", "onScroll:");
                return true;
            }

            // 6. 用户按下触摸屏、快速移动后松开
            // 参数：
            // e1：第1个ACTION_DOWN MotionEvent
            // e2：最后一个ACTION_MOVE MotionEvent
            // velocityX：X轴上的移动速度，像素/秒
            // velocityY：Y轴上的移动速度，像素/秒
            public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX,
                                   float velocityY) {
                Log.i("MyGesture", "onFling");
                return true;
            }

        });

        // 步骤2：让TextView检测手势：重写View的onTouch函数，将触屏事件交给GestureDetector处理，从而对用户手势作出响应
        mTextView = (TextView) findViewById(R.id.textView);
        mTextView.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                mGestureDetector.onTouchEvent(event);
                return false;
            }
        });
```



## OverScroller

在Android手机上，当我们滚动屏幕内容到达内容边界时，如果再滚动就会有一个发光效果。而且界面会进行滚动一小段距离之后再回复原位，这些效果是如何实现的呢？我们需要使用`Scroller`和`scrollTo`的升级版`OverScroller`和`overScrollBy`了，还有发光的`EdgeEffect`类。

[Android Scroll详解(二)：OverScroller实战](https://www.jianshu.com/p/293d0c2f56cb)

## ViewDragHelper

主要用于处理ViewGroup中对子View的拖拽处理,封装了对View的触摸位置，触摸速度，移动距离等的检测和Scroller，通过接口回调的方式通知我们。所以我们需要做的只是用接收来的数据指定这些子View是否需要移动，移动多少等。
三个主要状态：

1. STATE_IDLE：空闲
2. STATE_DRAGGING：正在拖拽
3. STATE_SETTLING：拖拽结束，放置View

## 其他

判断能否向上滑动

```
public static boolean defaultCanScrollUp(View view) {
        if (view == null) {
            return false;
        }
        if (android.os.Build.VERSION.SDK_INT < 14) {
            if (view instanceof AbsListView) {
                final AbsListView absListView = (AbsListView) view;
                return absListView.getChildCount() > 0
                        && (absListView.getFirstVisiblePosition() > 0 || absListView.getChildAt(0)
                        .getTop() < absListView.getPaddingTop());
            } else {
                return ViewCompat.canScrollVertically(view, -1) || view.getScrollY() > 0;
            }
        } else {
            return ViewCompat.canScrollVertically(view, -1);
        }
    }
```

