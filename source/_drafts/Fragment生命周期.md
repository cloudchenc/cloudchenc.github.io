---
title: Fragment生命周期
tags:
categories:
---

onAttach()  

执行该方法时，Fragment与Activity已经完成绑定，该方法有一个Activity类型的参数，代表绑定的Activity，这时候你可以执行诸如mActivity = activity的操作。

onCreate()  

初始化Fragment。可通过参数savedInstanceState获取之前保存的值。

onCreateView()  

初始化Fragment的布局。加载布局和findViewById的操作通常在此函数内完成，但是不建议执行耗时的操作，比如读取数据库数据列表。

onActivityCreated()

执行该方法时，与Fragment绑定的Activity的onCreate方法已经执行完成并返回，在该方法内可以进行与Activity交互的UI操作，所以在该方法之前Activity的onCreate方法并未执行完成，如果提前进行交互操作，会引发空指针异常。

onStart()

执行该方法时，Fragment由不可见变为可见状态。

onResume()

执行该方法时，Fragment处于活动状态，用户可与之交互。

onPause()

执行该方法时，Fragment处于暂停状态，但依然可见，用户不能与之交互。

onStop()

执行该方法时，Fragment完全不可见。

onDestroyView()

销毁与Fragment有关的视图，但未与Activity解除绑定，依然可以通过onCreateView方法重新创建视图。通常在ViewPager+Fragment的方式下会调用此方法。

onDestroy()

销毁Fragment。通常按Back键退出或者Fragment被回收时调用此方法。

onDetach()

解除与Activity的绑定。在onDestroy方法之后调用。


setUserVisibleHint() 与 onHiddenChanged()



