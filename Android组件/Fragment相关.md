## Fragment生命周期

创建过程: onAttach  onCreate onCreateView

销毁过程: onDestroyView onDestroy onDetach



## FragmentTransaction

#### attach

重新关联一个Fragment。

attach无法像add一样单独使用，单独使用会抛异常。方法存在的意义是对detach后的Fragment进行界面恢复。

------

#### detach

分离指定Fragment的UI视图

当Fragment被detach后，Fragment的生命周期执行完onDestroyView就终止了，这意味着Fragment的实例并没有被销毁，只是UI界面被移除了（注意和remove是有区别的）。

当Fragment被detach后，执行attach操作，会让Fragment从onCreateView开始执行，一直执行到onResume。

------

#### add

添加Fragment

add一个fragment，如果加到的是同一个id的话，有点像我们的Activity栈，启动多个Activity时候，Activity一个个叠在上面，fragment也是类似，一个个fragment叠在上面。

------

#### remove

移除Fragment

**remove()比detach()要彻底一些, 如果不加入到回退栈中, remove()的时候, fragment的生命周期会一直走到onDetach()；如果加入了回退栈，则会只执行到onDestoryView(),Fragment对象还是存在的。**

------

#### replace

可以理解为先把相同id下的Fragment移除掉，然后再加入这个当前的fragment

------

#### show/hide

让一个Fragment隐藏/显示

------

#### commit/commitAllowingStateLoss

大部分是因为自己的代码有闪退异常

**java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState**

你离开当前`Activity`等情况下，系统会调用`onSaveInstanceState()`帮你保存当前`Activity`的状态、数据等，直到再回到该`Activity`之前（`onResume()`之前），你执行`Fragment`事务，就会抛出该异常。然后网上有很多教程，叫你提交的时候使用`commitAllowingStateLoss()`方法，虽然说不会抛出错误，但是如果在`Activity`已经保存状态完之后提交了它，到时候`Ativity`意外崩溃，再恢复数据的时候就不会恢复在`Activity`保存状态之后提交的`fragment`的更新，造成状态丢失了

> 额外补充：
>
> 1.commit()方法并不立即执行transaction中包含的动作,而是把它加入到UI线程队列中. 如果想要立即执行,可以在commit之后立即调用FragmentManager的executePendingTransactions()方法.
>
> 2.commit()方法必须在状态存储之前调用,否则会抛出异常,如果觉得状态丢失没关系, 可以调用commitAllowingStateLoss(). 但是除非万不得已, 一般不推荐用这个方法, 会掩盖很多错误.

------

#### addToBackStack

可以理解为`addToBackStack`把我们前面的`FragmentTransaction`事务（比如add,remove,replace等一系列操作）加入到了回退栈（！！！记住不是把fragment加入到了回退栈），而`popBackStack`是操作回退栈里面的事务

1. 加入回退栈：remove掉的fragment执行onDestoryView，并没有执行onDestory,fragment实例对象还是存在，当回退时候，fragment从onCreateView处执行 
2. 未加入回退栈：remove掉的fragment 执行 onDestoryView和onDestory，彻底销毁移除

