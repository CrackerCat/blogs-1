## 概述

2017年的Google I/O大会上，Google推出了一系列譬如 [Lifecycle、ViewModel、LiveData](https://developer.android.com/jetpack/)等一系列 **更适合用于MVVM模式开发** 的架构组件。

本文的主角就是 **[ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel)** ，也许有朋友会提出质疑：

> **[ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel)** 这么简单的东西，从API的使用到源码分析，相关内容都烂大街了，你这篇文章还能翻出什么花来？

我无法反驳，事实上，阅读本文的您可能对MVVM的代码已经 **驾轻就熟**，甚至是经历了完整项目的洗礼，但我依然想做一次大胆地写作尝试—— **即使对于MVVM模式的思想噗之以鼻，或者已经熟练使用MVVM，本文也尽量让您有所收获，至少阅读体验不那么枯燥**。

## ViewModel的前世今生

**ViewModel**，或者说 [MVVM](https://baike.baidu.com/item/MVVM/96310?fr=aladdin) (Model-View-ViewModel)，并非是一个新鲜的词汇，它的定义最早起源于前端，代表着 **数据驱动视图** 的思想。

比如说，我们可以通过一个`String`类型的状态来表示一个`TextView`，同理，我们也可以通过一个`List<T>`类型的状态来维护一个`RecyclerView`的列表——在实际开发中我们通过观察这些数据的状态，来维护UI的自动更新，这就是 **数据驱动视图（观察者模式）**。

每当`String`的数据状态发生变更，View层就能检测并自动执行UI的更新，同理，每当列表的数据源`List<T>`发生变更，`RecyclerView`也会自动刷新列表：

![](https://github.com/qingmei2/qingmei2-blogs-art/blob/master/android/jetpack/viewModel/image1/image.4u7rkd6aqe8.png?raw=true)


对于开发者来讲，在开发过程中可以大幅减少UI层和Model层相互调用的代码，转而将**更多的重心投入到业务代码的编写**。

**ViewModel** 的概念就是这样被提出来的，我对它的形容类似一个 **状态存储器** ， 它存储着UI中各种各样的状态, 以 **登录界面** 为例，我们很容易想到最简单的两种状态 ：

```kotlin
class LoginViewModel {
    val username: String  // 用户名输入框中的内容
    val password: String  // 密码输入框中的内容
}
```

先不纠结于代码的细节，现在我们知道了ViewModel的重心是对 **数据状态**的维护。接下来我们来看看，在17年之前Google还没有推出ViewModel组件之前，Android领域内MVVM **百花齐放的各种形态** 吧。

### 1.群雄割据时代的百花齐放

说到MVVM就不得不提Google在2015年IO大会上提出的`DataBinding`库，它的发布直接促进了MVVM在Android领域的发展，开发者可以直接通过将数据状态通过 **伪Java代码** 的形式绑定在`xml`布局文件中，从而将MVVM模式的开发流程形成一个 **闭环**：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
       <data>
           <variable
               name="user"
               type="User" />
       </data>
      <TextView
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:text="@{ user.name }"
          android:textSize="20sp" />
</layout>
```

通过 **伪Java代码** 将UI的逻辑直接粗暴的添加进`xml`布局文件中达到和`View`的绑定，`DataBinding`这种实现方式引起了 **强烈的争论**。直至如今，依然有很多开发者无法接受`DataBinding`，这是完全可以理解的，因为它确实 **很难定位语法的错误和运行时的崩溃原因**。

MVVM模式并不一定依赖于`DataBinding`，但是除了`DataBinding`，开发者当时并没有足够多的选择——直至目前，仍然有部分的MVVM开发者坚持不使用 `DataBinding`，取而代之使用生态圈极为丰富的`RxJava`（或者其他）代替 `DataBinding`的数据绑定。

如果说当时对于 **数据绑定** 的库至少还有官方的`DataBinding`可供参考，`ViewModel`的规范化则是非常困难——基于`ViewModel`层进行状态的管理这个基本的约束，不同的项目、不同的依赖库加上不同的开发者，最终代码中对于 **状态管理** 的实现方式都有很大的不同。

比如，有的开发者，将 **ViewModel** 层像 **MVP** 一样定义为一个接口：

```kotlin
interface IViewModel

open class BaseViewModel: IViewModel
```

也有开发者(比如这个[repo](https://github.com/hitherejoe/MVVM_Hacker_News/blob/master/app/src/main/java/com/hitherejoe/mvvm_hackernews/viewModel/CommentViewModel.java))直接将ViewModel层继承了可观察的属性（比如`dataBinding`库的`BaseObservable`）,并持有`Context`的引用：

```Java
public class CommentViewModel extends BaseObservable {

    @BindingAdapter("containerMargin")
    public static void setContainerMargin(View view, boolean isTopLevelComment) {
        //...
    }
}
```

**一千个人有一千个哈姆雷特，不同的MVVM也有截然不同的实现方式**，这种百花齐放的代码风格、难以严格统一的 **开发流派** 导致代码质量的参差不齐，代码的可读性更是天差地别。

再加上`DataBinding`本身导致代码阅读性的降低，真可谓南门北派华山论剑，各种思想喷涌而出——从思想的碰撞交流来讲，这并非坏事，但是对于当时想学习MVVM的我来讲，实在是看得眼花缭乱，在学习接触的过程中，我也不可避免的走了许多弯路。

### 2.Google对于ViewModel的规范化尝试

我们都知道Google在去年的 **I/O** 大会非常隆重地推出了一系列的 **架构组件**， [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel)正是其中之一，也是本文的主角。

有趣的是，相比较于惹眼的 `Lifecycle` 和 `LiveData`， `ViewModel` 显得非常低调，它主要提供了这些特性：

* 配置更改期间自动保留其数据 (比如屏幕的横竖旋转)
* `Activity`、`Fragment`等UI组件之间的通信

如果让我直接吹捧`ViewModel`多么多么优秀，我会非常犯难，因为它表面展现的这些功能实在不够惹眼，但是有幸截止目前为止，我花费了一些笔墨阐述了`ViewModel`在这之前的故事——**它们是接下来正文不可缺少的铺垫**。

### 3.ViewModel在这之前的窘境

也许您尚未意识到，在官方的`ViewModel`发布之前，MVVM开发模式中，ViewModel层的一些窘境，但实际上我已经尽力通过叙述的方式将这些问题描述出来：

#### 3.1 更规范化的抽象接口

在官方的`ViewModel`发布之前，`ViewModel`层的基类多种多样，内部的依赖和公共逻辑更是五花八门。新的`ViewModel`组件直接对`ViewModel`层进行了标准化的规范，即使用`ViewModel`(或者其子类`AndroidViewModel`)。

同时，Google官方建议`ViewModel`尽量保证 **纯的业务代码**，不要持有任何View层(`Activity`或者`Fragment`)或`Lifecycle`的引用，这样保证了`ViewModel`内部代码的可测试性，避免因为`Context`等相关的引用导致测试代码的难以编写（比如，MVP中Presenter层代码的测试就需要额外成本，比如依赖注入或者Mock，以保证单元测试的进行）。

#### 3.2 更便于保存数据

由系统响应用户交互或者重建组件，用户无法操控。当组件被销毁并重建后，原来组件相关的数据也会丢失——最简单的例子就是**屏幕的旋转**，如果数据类型比较简单，同时数据量也不大，可以通过`onSaveInstanceState()`存储数据，组件重建之后通过`onCreate()`，从中读取`Bundle`恢复数据。但如果是大量数据，不方便序列化及反序列化，则上述方法将不适用。

`ViewModel`的扩展类则会在这种情况下自动保留其数据，如果`Activity`被重新创建了，它会收到被之前相同`ViewModel`实例。当所属`Activity`终止后，框架调用`ViewModel`的`onCleared()`方法释放对应资源：

![](https://github.com/qingmei2/qingmei2-blogs-art/blob/master/android/jetpack/viewModel/image.i28ly3qq5y8.png?raw=true)

这样看来，`ViewModel`是有一定的 **作用域** 的，它不会在指定的作用域内生成更多的实例，从而节省了更多关于 **状态维护**（数据的存储、序列化和反序列化）的代码。

`ViewModel`在对应的 **作用域** 内保持生命周期内的 **局部单例**，这就引发一个更好用的特性，那就是`Fragment`、`Activity`等UI组件间的通信。

### 3.3 更方便UI组件之间的通信

一个`Activity`中的多个`Fragment`相互通讯是很常见的，如果`ViewModel`的实例化作用域为`Activity`的生命周期，则两个`Fragment`可以持有同一个ViewModel的实例，这也就意味着**数据状态的共享**:

```Java
public class AFragment extends Fragment {
    private CommonViewModel model;
    public void onActivityCreated() {
        model = ViewModelProviders.of(getActivity()).get(CommonViewModel.class);
    }
}

public class BFragment extends Fragment {
    private CommonViewModel model;
    public void onActivityCreated() {
        model = ViewModelProviders.of(getActivity()).get(CommonViewModel.class);
    }
}
```

> 上面两个Fragment `getActivity()`返回的是同一个宿主`Activity`，因此两个`Fragment`之间返回的是同一个`ViewModel`。

我不知道正在阅读本文的您，有没有冒出这样一个想法：

> ViewModel提供的这些特性，为什么感觉互相之间没有联系呢？

这就引发下面这个问题，那就是：

> 这些特性的本质是什么？

### 4. ViewModel：对状态的持有和维护

`ViewModel`层的根本职责，就是负责维护**UI的状态**，追根究底就是维护对应的**数据**——毕竟，无论是MVP还是MVVM，UI的展示就是对数据的渲染。

* 1.定义了`ViewModel`的基类，并建议通过持有`LiveData`维护保存数据的状态；
* 2.`ViewModel`不会随着`Activity`的屏幕旋转而销毁，减少了**维护状态**的代码成本（数据的存储和读取、序列化和反序列化）；
* 3.在对应的作用域内，保正只生产出对应的唯一实例，**多个`Fragment`维护相同的数据状态**，极大减少了UI组件之间的**数据传递**的代码成本。

现在我们对于`ViewModel`的职责和思想都有了一定的了解，按理说接下来我们应该阐述如何使用`ViewModel`了，但我想先等等，因为我觉得相比API的使用，**掌握其本质的思想**会让你在接下来的代码实践中**如鱼得水**。

## 不，不是源码解析...

通过库提供的API接口作为开始，阅读其内部的源码，这是标准掌握代码内部原理的思路，这种方式的时间成本极高，即使有相关源码分析的博客进行引导，文章中大片大片的源码和注释也足以让人望而却步，**于是我理所当然这么想**：

> 先学会怎么用，再抽空系统学习它的原理和思想吧......

发现没有，这和上学时候的学习方式竟然**截然相反**，甚至说**本末倒置**也不奇怪——任何一个物理或者数学公式，在使用它做题之前，对它背后的基础理论都应该是优先去**系统性学习掌握**的（比如，数学公式的学习一般都需要先通过一定方式推导和证明），这样我才能拿着这个知识点对课后的习题**举一反三**。这就好比，如果一个老师直接告诉你一个公式，然后啥都不说让你做题，这个老师一定是不合格的。

我也不是很喜欢大篇幅地复制源码，我准备换个角度，站在Google工程师的角度看看怎么样设计出一个`ViewModel`。

## 站在更高的视角，设计ViewModel

现在我们是Google工程师，让我们再回顾一下`ViewModel`应起到的作用：

* 1.规范化了`ViewModel`的基类；
* 2.`ViewModel`不会随着`Activity`的屏幕旋转而销毁；
* 3.在对应的作用域内，保正只生产出对应的唯一实例，保证UI组件间的通信。

### 1.设计基类

这个简直太简单了：

```Java
public abstract class ViewModel {

    protected void onCleared() {
    }
}
```
我们定义一个抽象的`ViewModel`基类，并定义一个`onCleared()`方法以便于释放对应的资源，接下来，开发者只需要让他的`XXXViewModel`继承这个抽象的`ViewModel`基类即可。

### 2.保证数据不随屏幕旋转而销毁

这是一个很神奇的功能，但它的实现方式却非常简单,我们先了解这样一个知识点:

> `setRetainInstance(boolean)` 是`Fragment`中的一个方法。将这个方法设置为true就可以使当前`Fragment`在`Activity`重建时存活下来

这似乎和我们的功能非常吻合，于是我们不禁这样想，可不可以让`Activity`持有这样一个不可见的`Fragment`(我们干脆叫他`HolderFragment`)，并让这个`HolderFragment`调用`setRetainInstance(boolean)`方法并持有`ViewModel`——这样当`Activity`因为屏幕的旋转销毁并重建时，该`Fragment`存储的`ViewModel`自然不会被随之销毁回收了:

```Java
public class HolderFragment extends Fragment {

     public HolderFragment() { setRetainInstance(true); }

      private ViewModel mViewModel;
      // getter、setter...
}
```

当然，考虑到一个复杂的UI组件可能会持有多个`ViewModel`，我们更应该让这个不可见的`HolderFragment`持有一个`ViewModel`的数组（或者Map）——我们干脆封装一个叫`ViewModelStore`的容器对象，用来承载和代理所有`ViewModel`的管理：

```Java
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
    // put(), get(), clear()....
}

public class HolderFragment extends Fragment {

      public HolderFragment() { setRetainInstance(true); }

      private ViewModelStore mViewModelStore = new ViewModelStore();
}
```

好了，接下来需要做的就是，在实例化`ViewModel`的时候：

1.当前`Activity`如果没有持有`HolderFragment`，就实例化并持有一个`HolderFragment`
2.`Activity`获取到`HolderFragment`，并让`HolderFragment`将`ViewModel`存进`HashMap`中。

这样，具有生命周期的`Activity`在旋转屏幕销毁重建时，因为不可见的`HolderFragment`中的`ViewModelStore`容器持有了`ViewModel`，`ViewModel`和其内部的状态并没有被回收销毁。

这需要一个条件，在实例化`ViewModel`的时候，我们似乎还需要一个`Activity`的引用，这样才能保证 **获取或者实例化内部的`HolderFragment`并将`ViewModel`进行存储**。

于是我们设计了这样一个的API，在`ViewModel`的实例化时，加入所需的`Activity`依赖：

```Java
CommonViewModel viewModel = ViewModelProviders.of(activity).get(CommonViewModel.class)
```

我们注入了`Activity`，因此`HolderFragment`的实例化就交给内部的代码执行：

```Java
HolderFragment holderFragmentFor(FragmentActivity activity) {
     FragmentManager fm = activity.getSupportFragmentManager();
     HolderFragment holder = findHolderFragment(fm);
     if (holder != null) {
          return holder;
      }
      holder = createHolderFragment(fm);
      return holder;
}
```

这之后，因为我们传入了一个`ViewModel`的`Class`对象，我们默认就可以通过反射的方式实例化对应的`ViewModel`，并交给`HolderFragment`中的`ViewModelStore`容器存起来：

```Java
public <T extends ViewModel> T get(Class<T> modelClass) {
      // 通过反射的方式实例化ViewModel，并存储进ViewModelStore
      viewModel = modelClass.getConstructor(Application.class).newInstance(mApplication);
      mViewModelStore.put(key, viewModel);
      return (T) viewModel;
 }
```

### 3.在对应的作用域内，保正只生产出对应的唯一实例

如何保证在不同的Fragment中，通过以下代码生成同一个ViewModel的实例呢？

```Java
public class AFragment extends Fragment {
    private CommonViewModel model;
    public void onActivityCreated() {
        model = ViewModelProviders.of(getActivity()).get(CommonViewModel.class);
    }
}

public class BFragment extends Fragment {
    private CommonViewModel model;
    public void onActivityCreated() {
        model = ViewModelProviders.of(getActivity()).get(CommonViewModel.class);
    }
}
```

其实很简单，只需要在上一步实例化`ViewModel`的`get()`方法中加一个判断就行了：

```Java
public <T extends ViewModel> T get(Class<T> modelClass) {
      // 先从ViewModelStore容器中去找是否存在ViewModel的实例
      ViewModel viewModel = mViewModelStore.get(key);

      // 若ViewModel已经存在，就直接返回
      if (modelClass.isInstance(viewModel)) {
            return (T) viewModel;
      }

      // 若不存在，再通过反射的方式实例化ViewModel，并存储进ViewModelStore
      viewModel = modelClass.getConstructor(Application.class).newInstance(mApplication);
      mViewModelStore.put(key, viewModel);
      return (T) viewModel;
 }
```

现在，我们成功实现了预期的功能——事实上，上文中的代码正是`ViewModel`官方核心部分功能的源码，甚至默认`ViewModel`实例化的API也没有任何改变：

```Java
CommonViewModel viewModel = ViewModelProviders.of(activity).get(CommonViewModel.class);
```

当然，因为篇幅所限，我将源码进行了简单的删减，同时没有讲述构造方法中带参数的`ViewModel`的实例化方式，但对于目前已经掌握了**设计思想**和**原理**的你，学习这些API的使用几乎不费吹灰之力。

## 总结与思考

`ViewModel`是一个设计非常精巧的组件，它功能并不复杂，相反，它简单的难以置信，你甚至只需要了解实例化`ViewModel`的API如何调用就行了。

同时，它的背后掺杂的思想和理念是值得去反复揣度的。比如，如何保证对状态的规范化管理？如何将纯粹的业务代码通过良好的设计下沉到`ViewModel`中？对于非常复杂的界面，如何将各种各样的功能抽象为数据状态进行解耦和复用？随着MVVM开发的深入化，这些问题都会一个个浮出水面，这时候`ViewModel`组件良好的设计和这些不起眼的小特性就随时有可能成为璀璨夺目的闪光点，帮你攻城拔寨。

**--------------------------广告分割线------------------------------**

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
