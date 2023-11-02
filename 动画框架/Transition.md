# Transition

在Android 4.4（API级别19）及更高版本中，您可以使用转换框架在当前活动或片段中交换布局时创建动画。 您需要做的就是指定开始和结束布局，以及要使用的动画类型。 然后系统在两个布局之间找出差异并执行动画。 您可以使用它来交换整个UI或移动/替换一些视图。谷歌官网上 给出的一个gif图

![](image/2672902-2c48b83e844df7bc.gif)

## 场景Scene

Transition框架的核心就是根据Scene的不同，帮助开发者自动生成动画。

主要通过

* TransitionManager.go()
* beginDelayedTransition()
* setEnterTransition()/setSharedElementEnterTransition()

起始布局和结束布局均存储在**场景Scene**
中。通常由`getSceneForLayout (ViewGroup sceneRoot,int layoutId,Context context)`获取实例

官方提供了一个[BasicTransition](https://github.com/googlesamples/android-BasicTransition)的demo。从demo中来学习Scene的相关知识。

首先一开始，需要有一个容器，在这个容器内进行场景的切换。如下代码所示，有一个id为scene_root的FrameLayout的容器。



