## 前言

本文是 [Android官方架构组件](https://blog.csdn.net/mq2553299/column/info/24151) 系列的番外篇，因为目前国内关于`DataBinding`双向绑定的博客，讲的实在是五花八门，很多文章看完之后仍然一头雾水，特此专门写一篇文章进行总结。

本文默认读者对`DataBinding`的使用有了初步的了解。

## 什么是双向绑定？

`DataBinding`的本身是对`View`层状态的一种观察者模式的实现，通过让`View`与`ViewModel`层可观察的对象（比如`LiveData`）进行**绑定**，当`ViewModel`层数据发生变化，`View`层也会自动进行UI的更新。

上述我讲的是`DataBinding`最基础的用法，即 **单向绑定** ，其优势在于，将`View`层抽象为一个纯`Java`的可观察者——这意味着`ViewModel`层相关代码是完全可直接用于进行 **单元测试**。

但实际的开发中，**单向绑定**并非是足够的，在一些特定的场景，我们也需要用到 **双向绑定**。

比如说，对于一个`TextView`的内容展示，一般情况下，我们只是用来通过将一个`String`类型的数据对其进行渲染：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/databinding/ex1/image.5r9n5a1goe5.png)

显而易见，**数据的流向是单向的**，换句话说，我们认为`TextView`对`DataSource`只进行了 **读** 操作——如果此时进行了网络请求，我们需要用到`DataSource`某个属性作为参数，我们依然可以毫无顾忌从`DataSource`取值。

但是换一个场景，如果我们把`TextView`换成一个`EditText`，接下来我们需要面对的则截然不同，比如登录界面：
![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/databinding/ex1/image.mpn40f3ori.png)

这似乎没有什么问题，我们依然通过一个`LiveData`对`EditText`进行了单向绑定：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/databinding/ex1/image.ftx2c3o6ade.png)


问题发生了，当我们对 **输入框** 进行编辑，`EditText`的UI发生了变更，但是`LiveData`内的数据却没有更新，当我们想要在`ViewModel`层请求登录的API接口时，我们就必须要去通过`editText.getText()`才能获取用户输入的密码。

于是我们希望，即使是`EditText`的内容发生了变更，但是`LiveData`内的数据也能和`EditText`保持内容的同步——这样我们就不需要让`ViewModel`层持有`View`层的引用，在请求接口时，直接从`LiveData`中取值即可：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/databinding/ex1/image.z5acjk34xt.png)

这就是双向绑定的意义。

## 使用场景是什么

什么适合使用 **双向绑定** 呢，还记得上文中的一句话吗：

> 对于单向绑定来说，**数据的流向是单向的**，换句话说，我们认为`TextView`对`DataSource`只进行了 **读** 操作。

现在我们定义，当 **不确定的操作发生时** ——通常，这种操作代表着用户对UI控件的交互，这时UI的变化需要影响到`ViewModel`层的数据状态（除了 **数据驱动视图** 之外，视图也在驱动数据，以方便作为参数将来进行网络请求等等操作），这时 **双向绑定** 就可以大展身手了。

显然上文中的`EditText`的是 **双向绑定** 经典的使用场景之一，此外，双向绑定的使用场景非常常见，比如`CheckBox`：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/databinding/ex1/image.mzaa1ezojc9.png)

当用户选中了`CheckBox`，我们当然希望`ViewModel`层的`LiveData<Boolean>`状态进行对应的更新，以便将来我们直接从`LiveData`中取值作为参数进行网络请求。

而如果没有双向绑定，用户操作了UI，我们就需要 **手动添加代码保证状态的同步**——比如`checkBox.setOnCheckChangedListener()`，否则，就会在接下来的操作中得到与预期不同的结果。

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/databinding/ex1/image.m3az3aq3zsp.png)


## 听起来好像很麻烦，那么究竟如何使用呢？

幸运的是，Android原生控件中，绝大多数的双向绑定使用场景，`DataBinding`都已经帮我们实现好了：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/android/jetpack/databinding/ex1/image.fys33zgfk07.png)

这意味着我们并不需要去手动实现复杂的双向绑定，以上文的`EditText`为例，我们只需要通过`@={表达式}`进行双向的绑定：

```xml
<EditText
	android:id="@+id/etPassword"
	android:layout_width="match_parent"
	android:layout_height="wrap_content"
	android:text="@={ fragment.viewModel.password }" />
```

相比单向绑定，只需要多一个`=`符号，就能保证`View`层和`ViewModel`层的 **状态同步** 了。

## 难点在哪？

**双向绑定**定义好之后，使用起来很简单，但定义却稍微比单向绑定麻烦一些，即使原生的控件`DataBinding`已经帮助我们实现好了，**对于三方的控件或者自定义控件，还需要我们自己实现**。

本文以`SwipeRefreshLayout`为例，让我们来看看其 **双向绑定** 实现的方式：

```Kotlin
object SwipeRefreshLayoutBinding {

    @JvmStatic
    @BindingAdapter("app:bind_swipeRefreshLayout_refreshing")
    fun setSwipeRefreshLayoutRefreshing(
            swipeRefreshLayout: SwipeRefreshLayout,
            newValue: Boolean
    ) {
        if (swipeRefreshLayout.isRefreshing != newValue)
            swipeRefreshLayout.isRefreshing = newValue
    }

    @JvmStatic
    @InverseBindingAdapter(
            attribute = "app:bind_swipeRefreshLayout_refreshing",
            event = "app:bind_swipeRefreshLayout_refreshingAttrChanged"
    )
    fun isSwipeRefreshLayoutRefreshing(swipeRefreshLayout: SwipeRefreshLayout): Boolean =
            swipeRefreshLayout.isRefreshing

    @JvmStatic
    @BindingAdapter(
            "app:bind_swipeRefreshLayout_refreshingAttrChanged",
            requireAll = false
    )
    fun setOnRefreshListener(
            swipeRefreshLayout: SwipeRefreshLayout,
            bindingListener: InverseBindingListener?
    ) {
        if (bindingListener != null)
            swipeRefreshLayout.setOnRefreshListener {
                bindingListener.onChange()
            }
    }
}
```

有点晦涩，是不是？我们先不要纠结于细节的实现，先来看看代码中是如何使用的吧：

```xml
<androidx.swiperefreshlayout.widget.SwipeRefreshLayout
		android:layout_width="match_parent"
		android:layout_height="match_parent"
		app:bind_swipeRefreshLayout_refreshing="@={ fragment.viewModel.refreshing }">

            <androidx.recyclerview.widget.RecyclerView/>

</androidx.swiperefreshlayout.widget.SwipeRefreshLayout>
```

`refreshing`实际就只是一个`LiveData`：

```Kotlin
val refreshing: MutableLiveData<Boolean> = MutableLiveData()
```

这里的双向绑定，意义在于，当我们为`LiveData`手动设置值时，`SwipeRefreshLayout `的UI也会发生对应的变更；同理，当用户手动下拉执行刷新操作时，`LiveData`的值也会对应的变成为`true`(代表刷新中的状态)。

相比于其它的方式，**双向绑定将`SwipeRefreshLayout`的刷新状态抽象成为了一个`LiveData<Boolean>`** ——我们只需要在xml中定义好，之后就可以在`ViewModel`中围绕这个状态进行代码的编写，不同于`view.setOnRefreshListener()`的方式，这种代码是纯Java的，我们可以针对每一行代码进行纯JVM的单元测试。

> 本小节的所有代码你都可以在 [这里](https://github.com/qingmei2/MVVM-Rhine/blob/master/rhine/src/main/java/com/qingmei2/rhine/binding/support/SwipeRefreshLayoutBinding.kt) 获取。

## 整理思路，按部就班实现双向绑定

说了这么多，但是我们一行代码都还没有实现，不着急，因为编码只是其中的一个步骤，最重要的是 **整理一个流畅的思路**，这样，在接下来的编码阶段，你会如有神助。

### 1.实现单向绑定

我们知道，双向绑定的前提是单向绑定，因此，我们先配置好对应单向绑定的接口：

```kotlin
@JvmStatic
@BindingAdapter("app:bind_swipeRefreshLayout_refreshing")
fun setSwipeRefreshLayoutRefreshing(
        swipeRefreshLayout: SwipeRefreshLayout,
        newValue: Boolean
) {
        swipeRefreshLayout.isRefreshing = newValue
}
```

我们通过将`LiveData`的值和`DataBinding`绑定在一起，每当`LiveData`的状态发生了变更，`SwipeRefreshLayout`的刷新状态也会发生对应的更新。

我们实现了`数据驱动视图`的效果，接下来我们需要思考的是，我们如何才能知道用户会执行下拉操作呢？

### 2.观察View层的状态变更

只有观察到View层的状态变更，我们才能驱动`LiveData`进行对应的更新，其实很简单，通过`swipeRefreshlayout.setOnRefreshListener()`即可：

```Kotlin
@JvmStatic
@BindingAdapter(
        "app:bind_swipeRefreshLayout_refreshingAttrChanged",
        requireAll = false
)
fun setOnRefreshListener(
        swipeRefreshLayout: SwipeRefreshLayout,
        bindingListener: InverseBindingListener?
) {
    if (bindingListener != null)
        swipeRefreshLayout.setOnRefreshListener {
            bindingListener.onChange()   // 1
        }
}
```

注意我注释了 `//1`的地方，每当`swipeRefreshLayout`刷新状态被用户的操作改变，我们都能够在这里监听到，并交给`InverseBindingListener`这个 **信使** 去通知`DataBinding`：

> 嗨！View层的状态发生了变更，你快去通知`LiveData`也进行对应数据的更新呀！

新的问题来了，现在`DataBinding`已经知道需要去通知`LiveData`进行对应数据的更新了，关键是——

### 3. 我要把什么数据交给LiveData?

是的，即使`LiveData`需要进行更新，但是它并不知道要新的状态是什么。

> LiveData: 老哥，你倒是把数据给我啊！

我们急需将`SwipeRefreshLayout`最新状态告诉`LiveData`，因此我们通过`InverseBindingAdapter`注解和 **步骤二** 中去进行对接：

```Kotlin
@JvmStatic
@InverseBindingAdapter(
        attribute = "app:bind_swipeRefreshLayout_refreshing",
        event = "app:bind_swipeRefreshLayout_refreshingAttrChanged"   // 2 【注意！】
)
fun isSwipeRefreshLayoutRefreshing(swipeRefreshLayout: SwipeRefreshLayout): Boolean =
        swipeRefreshLayout.isRefreshing
```

注意到  `//2` 注释的那行代码没有，我们通过相同的`tag`（即`app:bind_swipeRefreshLayout_refreshingAttrChanged`这个字符串，步骤二中我们也声明了相同的字符串），和 **步骤二** 中的代码块形成了绑定对接。

现在，`LiveData`知道如何进行反向的数据更新了：

> 每当用户下拉刷新，`InverseBindingListener`通知`DataBinding`,`LiveData`就会从`swipeRefreshLayout.isRefreshing`得知最新的状态，并进行数据的同步更新。

### 4.不要忘了防止死循环！

细心的你多少已经感觉到了不对劲的地方，现在的双向绑定有一个致命的问题，那就是无限循环会导致的ANR异常。

当`View`层UI状态被改变，`ViewModel`对应发生更新，同时，这个更新又回通知`View`层去刷新UI，这个刷新UI的操作又会通知`ViewModel`去更新.......

因此，为了保证不会无限的死循环导致App的ANR异常的发生，我们需要在最初的代码块中加一个判断，保证，只有View状态发生了变更，才会去更新UI：

```Kotlin
@JvmStatic
@BindingAdapter("app:bind_swipeRefreshLayout_refreshing")
fun setSwipeRefreshLayoutRefreshing(
        swipeRefreshLayout: SwipeRefreshLayout,
        newValue: Boolean
) {
    if (swipeRefreshLayout.isRefreshing != newValue)   // 只有新老状态不同才更新UI
        swipeRefreshLayout.isRefreshing = newValue
}
```

## 小结

本文的初始计划中，还有一个模块是关于 **双向绑定的源码分析**，写到后来又觉得没有必要了，因为即使是 **源码**，也只是将上文中实现的思路啰嗦复述了一遍而已。

双向绑定本身是一个极具争议的功能；事实上，`DataBinding`本身也极具争议——`DataBinding`的好用与否，用或者不用都不重要，重要的是我们需要去正视它展现出来的思想：即如何将一个 **难以测试，状态多变** 的View, 通过代码抽象为 **易于维护和测试** 的纯Java的状态？

`DataBinding`将烦不胜烦的`View`层代码抽象为了易于维护的数据状态，同时极大减少了`View`层向`ViewModel`层抽象的 **胶水代码**，这就是最大的优势。

当然，`DataBinding`并不一定就是正解，事实上，`RxBinding`就是另外一个优秀的解决方案，同样以`SwipeRefreshLayout`为例，我依然可以将其抽象为一个可观察的`Observable<Boolean>`——**前者通过在xml中对数据进行绑定和观察，后者通过`RxJava`对View的状态抽象为一个流，但最终，两者在思想上殊途同归。**

## 系列文章

>  **争取打造 Android Jetpack 讲解的最好的博客系列**：
>* [Android官方架构组件Lifecycle：生命周期组件详解&原理分析](https://juejin.im/post/5c53beaf51882562e27e5ad9)
>* [Android官方架构组件ViewModel:从前世今生到追本溯源](https://juejin.im/post/5c047fd3e51d45666017ff86)
>* [Android官方架构组件LiveData: 观察者模式领域二三事](https://juejin.im/post/5c25753af265da61561f5335)
>* [Android官方架构组件Paging：分页库的设计美学](https://juejin.im/post/5c53ad9e6fb9a049eb3c5cfd)
>* [Android官方架构组件Paging-Ex：为分页列表添加Header和Footer](https://juejin.im/post/5caa0052f265da24ea7d3c2c)
>* [Android官方架构组件Paging-Ex：列表状态的响应式管理](https://juejin.im/post/5ce6ba09e51d4555e372a562)
>* [Android官方架构组件Navigation：大巧不工的Fragment管理框架](https://juejin.im/post/5c53be3951882562d27416c6)  
>* [Android官方架构组件DataBinding-Ex:双向绑定篇](https://juejin.im/post/5c3e04b7f265da611b589574)  

> **Android Jetpack 实战篇**：
>* [开源项目：MVVM+Jetpack实现的Github客户端](https://github.com/qingmei2/MVVM-Rhine)
>* [开源项目：基于MVVM, MVI+Jetpack实现的Github客户端](https://github.com/qingmei2/MVI-Rhine)
>* [总结：使用MVVM尝试开发Github客户端及对编程的一些思考](https://juejin.im/post/5be7bbd9f265da61797458cf)

---

## 关于我

Hello，我是[却把清梅嗅](https://github.com/qingmei2)，如果您觉得文章对您有价值，欢迎 ❤️，也欢迎关注我的[个人博客](https://juejin.im/user/588555ff1b69e600591e8462)或者[Github](https://github.com/qingmei2)。

如果您觉得文章还差了那么点东西，也请通过**关注**督促我写出更好的文章——万一哪天我进步了呢？

* [我的Android学习体系](https://github.com/qingmei2/android-programming-profile)
* [关于文章纠错](https://github.com/qingmei2/Programming-life/blob/master/error_collection.md)
* [关于知识付费](https://github.com/qingmei2/Programming-life/blob/master/appreciation.md)
