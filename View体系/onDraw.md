# onDraw

View的绘制过程包括

- 背景background.draw(canvas)
- onDraw
- 绘制child(**dispatchDraw**)
- 装饰(onDrawScrollBars)



View有一个特殊的方法**setWillNotDraw**。

```
    /**
     * If this view doesn't do any drawing on its own, set this flag to
     * allow further optimizations. By default, this flag is not set on
     * View, but could be set on some View subclasses such as ViewGroup.
     *
     * Typically, if you override {@link #onDraw(android.graphics.Canvas)}
     * you should clear this flag.
     *
     * @param willNotDraw whether or not this View draw on its own
     */
    public void setWillNotDraw(boolean willNotDraw) {
        setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
    }
```

如果一个View不需要绘制任何内容，那么设置这个标记位为true以后，系统会进行相应的优化。

## 硬件加速

所谓硬件加速，指的是把某些计算工作交给专门的硬件来做，而不是和普通的计算工作一样交给 CPU 来处理。这样不仅减轻了 CPU 的压力，而且由于有了「专人」的处理，这份计算工作的速度也被加快了。这就是「硬件加速」。

在硬件加速关闭的时候，`Canvas` 绘制的工作方式是：把要绘制的内容写进一个 `Bitmap`，然后在之后的渲染过程中，这个 `Bitmap` 的像素内容被直接用于渲染到屏幕。这种绘制方式的主要计算工作在于把绘制操作转换为像素的过程（例如由一句 `Canvas.drawCircle()` 来获得一个具体的圆的像素信息），这个过程的计算是由 CPU 来完成的。

而在硬件加速开启时，`Canvas` 的工作方式改变了：它只是把绘制的内容转换为 GPU 的操作保存了下来，然后就把它交给 GPU，最终由 GPU 来完成实际的显示工作。



链接：https://juejin.im/post/59bf350cf265da065352cfcf



[GcsSloop的自定义View教程](http://www.gcssloop.com/category/customview)

[HenCoder Android 自定义 View 1-5: 绘制顺序](https://juejin.im/post/599109e45188257c666c60b6)