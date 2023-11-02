# onLayout

Layout时ViewGroup来确定子元素位置的。在ViewGroup的位置被确定之后，它会在onLayout方法中遍历所有子元素，并调用子元素的layout方法，子元素的layout方法又会调用onLayout方法。

先看view的layout方法

```
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);

            ...
        }
        ...
    }
```

首先通过**setFrame确定View的四个顶点位置**，一旦确定，那么View在父容器中的位置也就确定了。接着调用onLayout，这个方法是父容器确定子元素位置的。

总之，当继承自ViewGroup时，重写onLayout方法，通过child.layout来给子元素分配位置。

