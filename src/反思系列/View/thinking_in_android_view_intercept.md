# 反思｜Android 事件拦截机制的设计与实现

> **「反思」** 系列是笔者一个新的尝试，其起源与目录请参考 [这里](https://github.com/qingmei2/android-programming-profile/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md) 。
## 概述

完整的掌握 `Android` 事件分发体系并非易事，其整个流程涉及到了 **系统启动流程**（`SystemServer`）、**输入管理**(`InputManager`)、**系统服务和UI的通信**（`ViewRootImpl` + `Window` + `WindowManagerService`）、`View`层级的 **事件分发机制** 等等一系列的环节。

**事件拦截机制** 是基于`View`层级 **事件分发机制** 的一个进阶性的知识点，本文将对其进行更细致化的讲解。

**事件拦截机制** 本身就相对比较独立，因此本文不需要读者有 **事件分发机制** 相关的预备知识，对后者感兴趣的读者可以参考以下资料：

> [反思 | Android 事件分发机制的设计与实现](https://juejin.im/post/5d66565cf265da03e71b0672)

本文整体结构如下图：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/blogs/algorithm/2/%E4%BA%8B%E4%BB%B6%E6%8B%A6%E6%88%AA0.u3ol7mccb5.png)

## 从事件序列说起

### 1、什么是事件序列

想要说清 **事件分发机制** 和 **事件拦截机制**，**事件序列** 是首先要理解的概念。

什么是事件序列？`Google`官方文档中对其描述为 `The duration of the touch`，顾名思义，我们可以将其理解为 **用户一次完整的触摸操作流程**—— 举例来说，用户单击按钮、用户滑动屏幕、用户长按屏幕中某个UI元素等等，都属于该范畴。

### 2、缘由

为什么 **事件序列** 是一个非常重要的概念？

[上一篇文章](https://juejin.im/post/5d66565cf265da03e71b0672) 中，读者已经了解事件分发的本质原理就是递归，对此简单的实现方式是：每接收一个新的事件，都需要进行一次递归才能找到对应消费事件的`View`，并依次向上返回事件分发的结果。

以每个触摸事件作为最基本的单元，都对`View`树进行一次遍历递归？这对性能的影响显而易见，因此这种设计是有改进空间的。

如何针对这个问题进行改进？将 **事件序列** 作为最基本的单元进行处理则更为合适。

首先，设计者根据用户的行为对`MotionEvent`中添加了一个`Action`的属性以描述该事件的行为：

* `ACTION_DOWN`：手指触摸到屏幕的行为
* `ACTION_MOVE`：手指在屏幕上移动的行为
* `ACTION_UP`：手指离开屏幕的行为
* ...其它Action，比如`ACTION_CANCEL`...

我们知道，针对用户的一次触摸操作，必然对应了一个 **事件序列**，从用户手指接触屏幕，到移动手指，再到抬起手指 ——单个事件序列必然包含`ACTION_DOWN`、`ACTION_MOVE` ... `ACTION_MOVE`、`ACTION_UP` 等多个事件，这其中`ACTION_MOVE`的数量不确定，`ACTION_DOWN`和`ACTION_UP`的数量则为1。

熟悉了 **事件序列** 的概念，设计者就可以着手对现有代码进行设计和改进，其思路如下：当接收到一个`ACTION_DOWN`时，意味着一次完整事件序列的开始，通过递归遍历找到真正对事件进行消费的`Child`，并将其进行保存，这之后接收到`ACTION_MOVE`和`ACTION_UP`行为时，则跳过遍历递归的过程，将事件直接分发对应的消费者：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/blogs/algorithm/2/image.ysuz8jmtsn.png)

由此可见，**事件序列** 在 **事件分发** 的知识体系中的确是非常重要的核心概念（甚至没有之一），其最重要的意义是 **足够节省性能**：用户一次正常的触摸行为，其 **事件序列** 包含了若干个触摸事件，这些事件并非每次都通过递归算法去找到事件的消费者，因为这会消耗非常多的内存——**当事件序列越复杂、或者`View`树的层级嵌套越深，这种优势愈发明显。**

那么，源码的设计者是如何保证通过一次递归算法找到`View`树中对应事件消费者的子`View`，其数据结构又是如何的呢？

认真思考，读者不难得出答案：**链表**。

### 为什么是链表

为什么采用链表，有没有更加简单粗暴的实现方案？

当然，**最符合直觉** 的实现方式似乎是：在通过递归完成第一次事件分发之后，将事件的消费者作为成员保存在当前父`View`中：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/blogs/algorithm/2/image.xg89v9uhaf.png)

不可否认，这样的设计完全可以实现我们需要的效果，但读者仔细思考得知，这种设计最大的问题就是破坏了树形结构的 **内部自治性**。

最顶层`View`直接持有最下层某个`View`的引用合理吗？答案是否定的。首先，这导致`View`层级依赖之间的混乱；其次，顶层`View`本身持有了最下层某个`View`的引用，则这之间若干个层级的`View`的`target`属性都毫无意义。

更能将树结构应用淋漓尽致的方式是构建一个链表：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/blogs/algorithm/2/image.91iq8u49bs7.png)

每个`View`节点都持有事件的下一级消费者，当同一事件序列后续的触摸事件抵达时，不再需要进行消耗性能的`DFS`算法，而是直接交给下一级的子`View`，子`View`则直接交给下下一级的子`View`，直到事件到达真正的消费者：

![](https://raw.githubusercontent.com/qingmei2/qingmei2-blogs-art/master/blogs/algorithm/2/image.4fqbdjylilg.png)

和链表的定义类似，设计者设计了`TouchTarget`类，同时为每一个`ViewGroup`都声明这样一个成员，作为链表的一个结点，以描述当前事件序列的传递方向：

```Java
public abstract class ViewGroup extends View {

  // 链表的下一级结点
  private TouchTarget mFirstTouchTarget;

  private static final class TouchTarget {
      // 描述接下来的触摸事件由哪一个子View接收并分发
      public View child;
  }
}
```

那么这个链表是怎么构建的呢？正如上文所说，当接收到一个`ACTION_DOWN`时，意味着一次完整事件序列的开始，通过递归遍历找到真正对事件进行消费的`Child`

读者需认真揣摩 **事件序列** 的相关概念，因为这个知识点贯穿了整个 **事件分发机制** 流程，可以说是非常核心的知识点；同时，掌握它也是下文快速掌握 **事件拦截机制** 的关键。

## 事件拦截机制

大多数`Android`开发者对 **事件拦截机制** 都不会陌生，读者应该都有了解，`ViewGroup`层级额外设计了`onInterceptTouchEvent()`函数并向外暴露给开发者，以达到让`ViewGroup`不再将触摸事件交给`View`处理，而是自身决定是否消费事件，并将结果反馈给上层级的`ViewGroup`。

### 1、缘由

为什么设计出这样一种拦截机制？其实这是有必要的，以常规的`ScrollView`对应的滑动页面为例，当用户抛出了一个列表的滑动操作，这时，对应的触摸事件序列是否还有必要交给`ScrollView`的子`View`进行处理？

答案是否定的，当`ScrollView`接收到滑动操作时，理所当然，本次滑动操作相关事件都不再需要交给子`View`，而是直接交给`ScrollView`去处理滑动操作。

读者同样需要明白，**并非所有事件序列都会被拦截**——当用户点击`ScrollView`中的某个按钮时，设计者又期望这次的点击操对应的系列事件能够被`ScrollView`分发给子`Button`去处理，这样开发者最终能够在按钮本身的`OnClickListener`中观察到这次点击事件，并进行对应的业务操作。

因此，对于不同类型的`ViewGroup`，开发者需要在不同的场景下，做出是否拦截事件的决定，这种 **父控件根据本身职责去拦截指定场景的事件序列** 的行为，我们称之为 **事件拦截机制**。

### 2、拦截函数：onInterceptTouchEvent()

那么开发者如何做，才能保证  **不同场景的事件被合理的向下分发或直接拦截** 呢？设计者据此提供了 `onInterceptTouchEvent()` 拦截函数：

```Java
public abstract class ViewGroup extends View {

  public boolean onInterceptTouchEvent(MotionEvent ev) {
      // ...
      return false;
  }
}
```

其定义是，当触摸事件到来时，事件首先作为参数传入`onInterceptTouchEvent`函数中，开发者自定义`onInterceptTouchEvent`内部逻辑，以决定是否对该事件进行拦截，并将`boolean`类型的结果进行返回。当返回值为`true`时，该事件序列接下来所有的事件都会被当前的`ViewGroup`拦截；通常情况下，`ViewGroup`的该函数默认返回`false`，即不对事件进行拦截。

以上文为例，我们可以对`ScrollView`添加类似如下策略——当用户发起一个 **点击事件** 的操作时，`onInterceptTouchEvent`返回`false`，将事件交给下游的子控件去决定消费与否；而当用户 **滑动屏幕** 时，则将事件序列进行拦截：

```Java
public class ScrollView extends ViewGroup {

  public boolean onInterceptTouchEvent(MotionEvent ev) {
      // 这里模拟一个抽象的函数代替实际的业务逻辑
      // 实际源码中，这里是根据对触摸事件序列的复杂判断，得出操作是否是滑动事件
      if (isUserScrollAction(ev)) {
        return true;
      } else {
        return false;
      }
  }
}
```

**事件序列** 在这个过程中再次起到了 **至关重要** 的作用。针对单独一个触摸事件——例如 `ACTION_DOWN`或`ACTION_MOVE`而言，我们都无法确定这是否是我们希望拦截的操作。而当我们获取到 **事件序列** 中连续若干个事件后，我们则可以根据手势操作的方向和距离（判断是否是滑动）、触摸屏幕的时间（判断是点击事件还是长按事件）对用户的这次行为进行定义，最终决定是否进行拦截。

——这意味着，当`ScrollView`接收到最初的`ACTION_DOWN`事件时，父控件并没有立即对事件进行拦截，而是交给了子`Button`去消费；而当接收了若干个`ACTION_MOVE`事件时，`ScrollView`的`onInterceptTouchEvent()`函数中判断得出 **本次触摸行为方向朝下，是滑动事件**，然后该函数返回`true`，导致本次和接下来的触摸事件都会被拦截。

等等！到了这里，读者似乎推断出了一个怪异的结论： **针对一个完整事件序列的向下分发过程而言，触摸事件的消费者并不一定只有一个角色**——这似乎不太符合直觉。

但事实的确如此。

既然一个完整的 **事件序列** 其事件可能会交给不同的角色，这是否意味着极端情况下，用户的一次 **滑动行为** 不但会触发了父控件本身的 **滑动** 效果，用户也会同时接收到`Button`子控件的 **点击** 效果？

目前为止的设计中确实存在这个缺陷，因此接下来我们需要增加新的逻辑单元去弥补这个问题，`ACTION_CANCEL`闪亮登场。

### 3、ACTION_CANCEL：弥补与终结

终于来到了`ACTION_CANCEL`的舞台，报幕员对这名演员的介绍是两个单词：**弥补** 和 **终结**。

现在我们希望，当`Button`的父控件`ScrollView`对滑动操作进行了拦截时，`Button`的点击事件不再会被响应。

正常的逻辑处理中，`Button`需要在接收到`ACTION_UP`时，判断整个事件序列持续的时间，如果符合一系列单击操作的前置定义（比如`touchable = true`或`clickable = true`等等），就直接交给单击事件的监听器`View.OnClickListener`去处理。

我们可以将`ACTION_UP`视为 **事件序列** 中的 **终止事件**，但很明显，这个逻辑在 **事件拦截机制** 中并不适用，因为当父控件对事件进行了拦截后，接下来整个序列中所有的事件都转交给了父控件，子控件再也接受不到任何事件，包括`ACTION_UP`。

我们总是希望有始有终（比如期待面试结果的及时反馈），**事件分发机制** 中也是一样，当子控件事件被父控件拦截，子控件也需要一个 **终止事件** 的通知以作出对应的行为。

因此，设计者额外提供了`ACTION_CANCEL`事件，以通知当前的`View`作出对 **事件被拦截** 之后的收尾工作，比如取消点击事件或长按事件相关判断逻辑中的计时器（如果有的话，下同），或者对当前控件滑动距离计算的重置等等，避免了「既发生父控件滑动」又「触发子控件点击」的尴尬场景。

现在，当父控件拦截了触摸事件后，子控件立即接收到一个额外的`ACTION_CANCEL`作为弥补，并草草进行了相关的收尾工作，之后的业务逻辑则统统交给了父控件去处理。

**事件拦截机制** 到此似乎告一段落，读者认真思考，这样的逻辑处理目前已经是完美的了么？

## 拦截机制与反拦截机制

> 父控件:「我无效你的效果。」
> 子控件:「我无效你的无效。」

### 1、压迫与抗争

音乐播放器的进度条控件`SeekBar`提出了严重抗议。

当`SeekBar`与`ScrollView`搭配使用时，前者愕然发现，作为子控件，其最引以为豪的技能——滑动调整音频进度的功能完全被废掉了。

这是当然的，`ScrollView`接收到滑动事件时，会很自然的将接下来相关的所有事件都进行拦截，而作为子控件的`SeekBar`连汤都喝不上。

这是不合理的设计，父控件的权利实在太大了，而子控件对此完全束手无策。因此设计者为`ViewGroup`设计了一个另外一个`API`——`requestDisallowInterceptTouchEvent(boolean)`。

该函数的作用是，命令指定的`ViewGroup` 是否 **不再针对事件序列进行拦截** ，而是正常将事件交给子控件去处理是否消费事件。

以`SeekBar`为例，其完全可以这样设计：

```Java
public abstract class AbsSeekBar extends ProgressBar {

   // ...代码大幅简化，具体逻辑请参考源码...
   @Override
   public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
          // 当接收到ACTION_DOWN事件时，命令父控件不能拦截事件序列
          case MotionEvent.ACTION_DOWN:
              mParent.requestDisallowInterceptTouchEvent(true);
              break;
          // ...
        }
   }
}
```

现在，即使`ScrollView`内部持有对 **滑动操作** 相关的拦截机制，但`SeekBar`依然可以通过更高等级的`API`对其进行压制，从而跳过父控件相关的拦截，并自己消费滑动事件——最终用户得到了他希望得到的操作体验（滑动调节播放进度）。

### 2、更深入性思考

上一小节的叙述本身是存在瑕疵的，通常来说，调节进度的`SeekBar`处理的是横向滑动，而`ScrollView`处理的则是竖向滑动，本质上两者逻辑并不冲突。

这样描述，只是为了让读者能够更容易的理解 **反拦截机制** 对应的`requestDisallowInterceptTouchEvent()`函数设计的目的及意义，对此读者不必深究——当然，读者也可自定义实现一个横向滑动的`HorizontalScrollView`，以得到上一小节中滑动冲突的效果，本文不赘述。

另外一点需要思考的是，当子控件调用了父控件的`requestDisallowInterceptTouchEvent(true)`函数无效化了父控件的拦截机制之后，**父控件拦截机制的无效化需要一直存在吗** ？

答案是否定的，正确的方式是应该在某个时间点 **对父控件拦截机制进行重启**——即调用`requestDisallowInterceptTouchEvent(false)`，这样才能保证在触摸到其它子控件时，父控件依然能够对 **事件拦截机制** 进行正常的运转。

那么这个重置的时间点如何把握，在子控件接收到`ACTION_UP`时调用吗？

在子控件 **事件序列的终止事件中重置状态**，这听起来不错，但是需要注意的是，拦截机制被无效化的状态是存在父控件`ViewGroup`中的，因此换个思路，更好的时机会不会其实是隐藏在`ViewGroup`中的呢？

### 3、更好的时机

设计者最终将重置的时机放在了父控件 **事件序列的起始事件**——`ACTION_DOWN`的处理逻辑中。

```Java
public abstract class ViewGroup extends View {

  @Override
  public boolean dispatchTouchEvent(MotionEvent ev) {
      // ...
      if (actionMasked == MotionEvent.ACTION_DOWN) {
         // 1.这个函数内部将事件拦截功能的开关进行了重置
         resetTouchState();
      }
      // ...2.继续处理事件拦截和事件分发
  }
}
```

这确实是重置拦截机制的更好时机，既保证了其它子控件的触摸事件不会被之前的反拦截机制所影响，同时也维护了`ViewGroup`内部本身的自治性。

这也证明了 **事件序列** 中的起始事件 `ACTION_DOWN` 总是可以被父控件接收到并进行拦截处理，因此，开发者绝大多数情况下不能在 `ViewGroup` 的 `onInterceptTouchEvent()` 中，直接对`ACTION_DOWN`事件返回`true`，因为这将会导致父控件拦截了整个 **事件序列** ，子控件连`ACTION_DOWN`都接受不到，反拦截机制彻底失效。

## 总结

**事件拦截机制** 是一个非常重要的基础知识点，而 **事件序列** 又是其中最核心的概念，无论是 **事件分发** 还是 **事件拦截**，搞懂了 **事件序列** 的意义，其它逻辑概念的理解都不再困难。

## 参考 & 额外的话

> **这一篇文章就能让我理解Android事件拦截机制吗？**

当然不能，在撰写本文的过程中，笔者最终删除了若干更细节知识点的讲解，比如:

* 父控件拦截了事件后，其内部的`mFirstTouchTarget`发生了怎样的变化？（事件传递链表的更新操作）
* 事件的拦截机制常常用于解决开发中的哪些问题？(解决滑动冲突)

等等，这些细节同样十分重要，它们是填充 **事件拦截机制** 完整体系的血与肉，建议读者结合本文与下列相关资料，开启一次更细致的探究之旅。

* 1、Android源码
* [2、「Android开发艺术探索」](https://item.jd.com/11760209.html)


## 关于我

Hello，我是 [却把清梅嗅](https://github.com/qingmei2)，女儿奴，源码的眷者，观众途径序列1，杀人游戏信徒，大头菜投机者，端茶递水工程师。欢迎关注我的 [博客](https://juejin.im/user/588555ff1b69e600591e8462/posts) 或者 [GitHub](https://github.com/qingmei2)。

如果您觉得文章对您有价值，欢迎 ❤️，或通过下方打赏功能，督促我写出更好的文章 :)

* [我的学习体系](https://github.com/qingmei2/blogs)
* **[关于知识付费](https://github.com/qingmei2/blogs/blob/master/appreciation.md)**
* **[关于「反思」系列](https://github.com/qingmei2/blogs/blob/master/src/%E5%8F%8D%E6%80%9D%E7%B3%BB%E5%88%97/%E5%8F%8D%E6%80%9D%7C%E7%B3%BB%E5%88%97%E7%9B%AE%E5%BD%95.md)**
