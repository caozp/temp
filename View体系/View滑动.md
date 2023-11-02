实现View滑动有很多种方法。分别是layout，offsetLeftAndRight，LayoutParams，动画，scrollTo

## layout方法

在每次移动时，调用layout方法对屏幕重新布局，从而达到移动的效果。如下自定义View

```
public boolean onTouchEvent(MotionEvent e){
     int x = e.getX();
     int y = e.getY();
     
     switch(e.getAction()){
         case ACTION_DOWN:
             lastX = x;
             lastY = y;
             break;
         case ACTION_MOVE:
             int offsetX = X-lastX;
             int offsetY = y-lastY;
             layout(getLeft()+offsetX,getTop()+offsetY,
                 getRight()+offsetX,getBottom()+offsetY);
             break;
     }
}
```



## offsetLeftAndRight

与layout方法差不多，将ACTION_MOVE替换称如下代码

```
  case ACTION_MOVE:
      int offsetX = X-lastX;
      int offsetY = y-lastY;
      offsetLeftAndRight(offsetX);
      offsetTopAndBottom(offsetY);
      break;
```

## LayoutParams

LayoutParams保存了View的布局参数。通过改变参数来实现View位置的改变。

```
  case ACTION_MOVE:
      int offsetX = X-lastX;
      int offsetY = y-lastY;
      lp.leftMargin = getLeft()+offsetX;
      lp.topMargin = getTop+offsetY;
      setLayoutParams(lp);
      break;
```

## 动画

View动画并不能改变View的位置参数，但是属性动画可以。

```
ObjectAnimator.ofFloat(mView,"translationX",0,300)
              .setDuration(3000)
              .start()
```



## Scroller

Scroller 弹性滑动对象，用来实现View的弹性滑动。首先看`scrollTo`和`scrollBy`

```
    public void scrollTo(int x, int y) {                      //绝对滑动
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            invalidateParentCaches();
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                postInvalidateOnAnimation();
            }
        }
    }

    /**
     * Move the scrolled position of your view. This will cause a call to
     * {@link #onScrollChanged(int, int, int, int)} and the view will be
     * invalidated.
     * @param x the amount of pixels to scroll by horizontally
     * @param y the amount of pixels to scroll by vertically
     */
    public void scrollBy(int x, int y) {              //当前位置相对滑动
        scrollTo(mScrollX + x, mScrollY + y);
    }
```

手指向左，也就是由右向左滑 显示右边的内容，mScrollX等于view左边缘与内容左边缘的距离,mScrollX>0;
手指向上，也就是有下往上滑 显示下边的内容，mScrollY等于view上边缘与内容上边缘的举例,mScrollY>0;

原理:

> Scroller需要和View的computeScroll()方法一起使用才能实现弹性滑动。scroller.startScroll(int startX,int startY,int dx,int dy,int duration);中的参数是滑动的起点坐标，dx,dy 是水平和竖直方向上的滑动距离，duration是完成滑动需要的时间。调用startScroll()滑动还没有开始。startScroll()之后的invalidate()将使View重绘，在View重绘的时候会调用computeScroll()方法，在此方法中调用scroller.computeScrollOffset()判断如果滑动未结束使用scrollTo(),来滑动View，scrollTo()之后再调用invalidate()重新绘制View,在View重绘的时候会又调用computeScroll()方法，如此循环直至滑动结束。多次的小幅度滑动组成了弹性滑动

```
private void smoothScrollto(...){
    mScroller.startScroll(...);  //#1 mScroller.startScroll(...)开始滑动
    invalidate();   //#2 调用invalidate() 请求重绘 
}

//#3 invalidate会导致View的重绘，在View的draw方法中又会调用computeScroll。computeScroll是一个空实现，需要重写此方法

@Override  
public void computeScroll() {  
   if(scroller.computeScrollOffset()){     //#4 判断是否滑动结束,未结束继续 
       scrollTo(scroller.getCurrX(),scroller.getCurrY());  
       //#5 获取scroller当前scrollX和scrollY 然后滑动
       postInvalidate();     //#6 再次请求重绘 又去跳转到#3
   }  
}  
```

## Fling

用户手指快速划过屏幕，然后快速立刻屏幕时，系统会判定用户执行了一个Fling手势。视图会快速滚动，并且在手指立刻屏幕之后也会滚动一段时间。Drag表示手指滑动多少距离，界面跟着显示多少距离，而fling是根据你的滑动方向与轻重，还会自动滑动一段距离。Filing手势在android交互设计中应用非常广泛：电子书的滑动翻页、ListView滑动删除item、滑动解锁等。所以如何检测用户的fling手势是非常重要的。在检测Fling时，你需要检测手指在屏幕上滑动的速度，这是你就需要`VelocityTracker`和`Scroller`这两个类啦。

当滑动速度大于某个值，调用scroller的fling方法。

```
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        .....
        if (mVelocityTracker == null) {
            //检查速度测量器，如果为null，获得一个
            mVelocityTracker = VelocityTracker.obtain();
        }
        int action = MotionEventCompat.getActionMasked(event);
        int index = -1;
        switch (action) {
            case MotionEvent.ACTION_DOWN:
                ......
                if (!mScroller.isFinished()) { //停止屏幕滚动
                    mScroller.abortAnimation();
                }
                .....
                break;
            case MotionEvent.ACTION_MOVE:
                ......
                break;
            case MotionEvent.ACTION_CANCEL:
                endDrag();
                break;
            case MotionEvent.ACTION_UP:
                if (mIsBeingDragged) {
                //当手指立刻屏幕时，获得速度，作为fling的初始速度     mVelocityTracker.computeCurrentVelocity(1000,mMaxFlingSpeed);
                    int initialVelocity = (int)mVelocityTracker.getYVelocity(mActivePointerId);
                    if (Math.abs(initialVelocity) > mMinFlingSpeed) {
                        // 由于坐标轴正方向问题，要加负号。
                        doFling(-initialVelocity);
                    }
                    endDrag();
                }
                break;
            default:
        }
        //每次onTouchEvent处理Event时，都将event交给时间
        //测量器
        if (mVelocityTracker != null) {
            mVelocityTracker.addMovement(event);
        }
        return true;
    }
    private void doFling(int speed) {
        if (mScroller == null) {
            return;
        }
        mScroller.fling(0,getScrollY(),0,speed,0,0,-500,10000);
        invalidate();
    }
    @Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
            postInvalidate();
        }
    }
```

