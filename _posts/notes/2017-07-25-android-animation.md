---
layout: post
title: "Android 动画笔记"
excerpt: "Android 动画官方文档学习笔记。"
date: 2017-07-25
modified: 2017-07-25
categories: notes
author: zrl
tags:
  - Android
  - Animation
comments: false
share: true
---

## [动画分类](https://developer.android.google.cn/guide/topics/graphics/overview.html)

- 属性动画 Property  Animation

  最为方便强大，推荐使用。

- 视图动画 View Animation

  旧版本的动画方式。

- 绘制动画 Drawable Animation

  即一帧帧绘制画面，万能但仅在必要时使用。

### 属性动画和视图动画的区别

- 视图动画只能作用于 [`View`](https://developer.android.com/reference/android/view/View.html) 对象，属性动画没有这个限制。
- 视图动画只能操纵少数几个属性，例如缩放比例、旋转角度等，许多属性，例如背景颜色，就没法通过视图动画进行操作，属性动画更加通用。
- 视图动画仅仅修改了绘制位置，并没有实际修改属性值，例如用视图动画实现一个按钮移动的效果，按钮可以正确移动，但是用户点按按钮的位置却没有改变。
- 属性动画相对于视图动画而言要复杂一些，对于一些简单情形可以考虑用视图动画解决。

## [属性动画](https://developer.android.google.cn/guide/topics/graphics/prop-animation.html)

属性动画几乎可以实现任何想要的动画效果，非常具有可扩展性并且非常稳健。属性动画可供设定的选项包括了：

- 持续时间（默认 300 ms）

- 时间插值（Time interpolation）

  即指定一个关于时间的函数，使得属性值的计算依赖于这个函数。

- 重复播放、逆向播放

- 动画集合

  可以将一组动画合并成一个集合，然后同时播放或是顺序播放或是延时播放。

- 帧刷新间隔

  默认是 10 ms，可以改成别的值，但最终取决于系统状态。

### 属性动画的工作方式

属性动画通过指定一个对象的属性的改变方式来实现动画，举例来说，如果想要实现一个对象在 x 轴上的横向移动动画，那就让这个对象的 x 轴坐标每隔一个时间间隔变化一点即可。例如下图表示了一个对象在 40 ms 内沿 x 轴移动了 40 px 的动画：

![img](https://developer.android.google.cn/images/animation/animation-linear.png)

这个属性动画让该对象的 x 轴坐标每隔 10 ms 增加 10 px，持续播放了 40 ms，产生了一个匀速移动的动画。当然，属性动画也可以实现非匀速的动画，比如这样：

![img](https://developer.android.google.cn/images/animation/animation-nonlinear.png)

实现属性动画需要了解几个重要的组件，如下图所示：

![img](https://developer.android.google.cn/images/animation/valueanimator.png)

其中 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 对象会记录动画的时间相关信息，例如动画播放了多久以及当前时间点上属性的值。其中封装了一个 [`TimeInterpolator`](https://developer.android.google.cn/reference/android/animation/TimeInterpolator.html)，它定义了动画的插值；还封装了一个 [`TypeEvaluator`](https://developer.android.google.cn/reference/android/animation/TypeEvaluator.html)，它定义了如何去计算属性值。例如在刚刚那个非匀速的移动动画里，就可能使用 [`AccelerateDecelerateInterpolator`](https://developer.android.google.cn/reference/android/view/animation/AccelerateDecelerateInterpolator.html) 来作为 [`TimeInterpolator`](https://developer.android.google.cn/reference/android/animation/TimeInterpolator.html)，使用 [`IntEvaluator`](https://developer.android.google.cn/reference/android/animation/IntEvaluator.html) 来作为 [`TypeEvaluator`](https://developer.android.google.cn/reference/android/animation/TypeEvaluator.html)。

在开始播放动画之前，先给定属性的初始值和结束值，以及动画播放的时长。然后就可以调用 [`start()`](https://developer.android.com/reference/android/animation/ValueAnimator.html#start()) 来开始动画的播放了。在动画播放的过程中，[`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 会基于动画已经播放的时间和动画的总持续时间来计算流逝比例（elapsed fraction）（范围是 0 到 1），它表示了动画完成的比例。例如在前面那个匀速平移动画中， t = 10 ms 的时候流逝比例是 0.25，因为总时长是 40 ms。

当流逝比例被计算出来之后，[`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 会调用 [`TimeInterpolator`](https://developer.android.google.cn/reference/android/animation/TimeInterpolator.html) 来计算一个插值比例（interpolated fraction）。例如在上面非匀速移动的动画里，由于 x 的值一开始在缓慢加速，所以当 t = 10 ms 的时候，插值比例大约为 0.15，比流逝比例 0.25 要小，而在之前的匀速移动的动画里，插值比例则一直等于流逝比例。

当插值比例被计算出来之后，[`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 会调用 [`TypeEvaluator`](https://developer.android.google.cn/reference/android/animation/TypeEvaluator.html) 来基于插值比例、初始值和结束值计算属性的当前值。例如，在上面非匀速的移动动画里，t = 10 ms 时插值比例是 0.15，所以属性值可以被计算为 0.15 * (40 - 0)，也就是 6。

### 属性动画 API 概览

大多属性动画的系统 API 都能在 [`android.animation`](https://developer.android.com/reference/android/animation/package-summary.html) 包下找到。视图动画系统已经在 [`android.view.animation`](https://developer.android.com/reference/android/view/animation/package-summary.html) 包下定义了许多插值器，这些插值器都可以直接被用于属性动画系统。

[`Animator`](https://developer.android.com/reference/android/animation/Animator.html) 类提供了创建动画的基本结构，但这些功能都太过精简和基础，一般来说都不会直接用这个类。下面是它的一些子类：

- [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html)：属性动画的主要时间引擎，它还能计算属性的值。它含有所有动画计算的核心功能，以及每一个动画的时间细节。另外还包含了动画是否重复、接收更新事件的监听者等信息，还能设定自定义的类型计算器。动画属性包含了两方面，一方面是计算动画的值，另一方面是将这个值设定到对象的属性上。[`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 并没有包含后者，所以你需要去监听它计算出来的值，并自己去修改对应的值。
- [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html)：这个是 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 的子类，它允许你去设定目标对象以及动画修改的具体属性值。当计算出新的动画值时，这个类就会去修改对象的属性值。大多情况下，你都会去用这个类，因为它能大幅简化将目标对象动画化的过程。不过它也有一些限制，例如它需要目标对象提供特定的属性访问方法，所以有时候也需要直接去使用 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html)。
- [`AnimatorSet`](https://developer.android.google.cn/reference/android/animation/AnimatorSet.html)：这个类提供了一套机制用于将一组动画合并起来，使得它们能以相互关联的形式播放。

计算器告诉属性动画系统如何计算给定属性的值。它们基于 [`Animator`](https://developer.android.com/reference/android/animation/Animator.html) 类提供的时间数据以及初始值和结束值来计算动画的值。属性动画提供了如下的计算器：

- [`IntEvaluator`](https://developer.android.google.cn/reference/android/animation/IntEvaluator.html)：计算 `int` 属性的默认计算器。
- [`FloatEvaluator`](https://developer.android.google.cn/reference/android/animation/FloatEvaluator.html)：计算 `float` 属性的默认计算器。
- [`ArgbEvaluator`](https://developer.android.google.cn/reference/android/animation/ArgbEvaluator.html)：计算十六进制颜色属性的默认计算器。
- [`TypeEvaluator`](https://developer.android.google.cn/reference/android/animation/TypeEvaluator.html)：用于构建自己的计算器的接口。

时间插值器定义了一个关于时间的函数，用以计算动画值。例如，你可以通过指定一个线性函数来实现匀速移动的动画，你也可以指定一个非线性函数来实现先加速后减速的效果。下面是 [`android.view.animation`](https://developer.android.com/reference/android/view/animation/package-summary.html) 包里包含的插值器。如果这些插值器无法满足你的要求，你可以通过实现 [`TimeInterpolator`](https://developer.android.google.cn/reference/android/animation/TimeInterpolator.html) 接口来创建你自己的版本。

- [`AccelerateDecelerateInterpolator`](https://developer.android.google.cn/reference/android/view/animation/AccelerateDecelerateInterpolator.html)：属性值变化起始慢速中间加速的插值器。
- [`AccelerateInterpolator`](https://developer.android.google.cn/reference/android/view/animation/AccelerateInterpolator.html)：属性值变化开始慢速之后慢慢加速的插值器。
- [`AnticipateInterpolator`](https://developer.android.google.cn/reference/android/view/animation/AnticipateInterpolator.html)：属性值一开始向后变化，然后甩到前面的插值器。
- [`AnticipateOvershootInterpolator`](https://developer.android.google.cn/reference/android/view/animation/AnticipateOvershootInterpolator.html)：属性值一开始向后变化，然后甩到前面超过指定的值，最终回到指定值的插值器。
- [`BounceInterpolator`](https://developer.android.google.cn/reference/android/view/animation/BounceInterpolator.html)：属性值在最终值附近弹跳的插值器。
- [`CycleInterpolator`](https://developer.android.google.cn/reference/android/view/animation/CycleInterpolator.html)：动画重复指定数量次数的插值器。
- [`DecelerateInterpolator`](https://developer.android.google.cn/reference/android/view/animation/DecelerateInterpolator.html)：属性值变化初始速率大，然后减速的插值器。
- [`LinearInterpolator`](https://developer.android.google.cn/reference/android/view/animation/LinearInterpolator.html)：属性值变化速率是常数的插值器。
- [`OvershootInterpolator`](https://developer.android.google.cn/reference/android/view/animation/OvershootInterpolator.html)：先甩到前面，超过最终值后再回来的插值器。
- [`TimeInterpolator`](https://developer.android.google.cn/reference/android/animation/TimeInterpolator.html)：用于实现自定义插值器的接口。

### 使用 `ValueAnimator` 实现动画

[`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 类使你可以将某类型的值动画化，并且可以指定动画时长。通过调用它的工厂方法：[`ofInt()`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html#ofInt(int...))、[`ofFloat()`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html#ofFloat(float...)) 或是 [`ofObject()`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html#ofObject(android.animation.TypeEvaluator, java.lang.Object...)) 来获取一个 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 对象，例如：

```java
ValueAnimator animation = ValueAnimator.ofFloat(0f, 100f);
animation.setDuration(1000);
animation.start();
```

在上面的代码里，当调用 [`start()`](https://developer.android.com/reference/android/animation/ValueAnimator.html#start()) 之后，[`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 就会开始在 100 ms 的持续时间内，从 0 到 100 计算动画的值，下面的代码演示了如何为自定义的类型的值实现动画：

```java
ValueAnimator animation = 
  ValueAnimator.ofObject(new MyTypeEvaluator(), startPropertyValue, endPropertyValue);
animation.setDuration(1000);
animation.start();
```

在上面的代码里，当调用 [`start()`](https://developer.android.com/reference/android/animation/ValueAnimator.html#start()) 之后，[`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 就会开始在 100 ms 的持续时间内，用 `MyTypeEvaluator` 提供的逻辑从 `startPropertyValue` 到 `endPropertyValue` 计算属性值。

[`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 计算的结果可以通过注册 [`AnimatorUpdateListener`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html) 回调的方式来使用：

```java
animation.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
  @Override
  public void onAnimationUpdate(ValueAnimator updatedAnimation) {
    // You can use the animated value in a property that uses the
    // same type as the animation. In this case, you can use the
    // float value in the translationX property.
    float animatedValue = (float)updatedAnimation.getAnimatedValue();
    textView.setTranslationX(animatedValue);
  }
});
```

在 [`onAnimationUpdate()`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html#onAnimationUpdate(android.animation.ValueAnimator)) 方法里你可以获取到更新的动画值，然后将其应用到目标属性里。

### 使用 `ObjectAnimator` 实现动画

[`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html) 是 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 的子类，它整合了 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 的功能与将目标对象的属性动画化的能力。这使得对任意对象的动画化更加容易，你不需要每次都去实现 [`ValueAnimator.AnimatorUpdateListener`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html) 了，因为属性可以自动更新。

初始化一个 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html) 的方式和初始化 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 的方式类似，但你也可以同时指定目标对象和对应属性的字符串：

```java
ObjectAnimator animation = ObjectAnimator.ofFloat(textView, "translationX", 100f);
animation.setDuration(1000);
animation.start();
```

为了使 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html) 正确更新属性，代码必须满足如下要求：

- 动画化的对象属性必须有一个 setter 函数（用驼峰法命名），形式为：`set<PropertyName>()`。由于 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html) 会自动更新属性，所以它必须要有访问这些 setter 方法的权限。例如，如果属性值是 `foo`，那么你需要一个 `setFoo()` 方法。如果这个 setter 方法不存在，你有以下几个解决方案：

  - 如果你有权限就添加一个 setter 方法。
  - 使用一个包装类封装原始对象，并提供一个 setter 方法。
  - 使用 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html)。

- 如果你对 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html) 指定了的工厂方法的 `values...` 参数指定了一个值，那么它会被认为是动画的结束值。所以该目标对象还必须提供一个 getter 方法用以获取属性的初始值。getter 方法必须满足 `get<PropertyName>` 的形式。例如，如果属性名是 `foo`，你需要有一个 `getFoo()` 方法。

- 属性的 getter 方法（如果需要的话）和 setter 方法所操作的值的类型必须要和初始值和最终值的类型相同。例如，如果你用了如下方式构建 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html)：

  ```java
  ObjectAnimator.ofFloat(targetObject, "propName", 1f)
  ```

  那你就必须要提供 `targetObject.setPropName(float)` 和 `targetObject.setPropName(float)` 两个方法。

- 某些属性或者对象可能会要求你去对 `View` 对象调用 [`invalidate()`](https://developer.android.google.cn/reference/android/view/View.html#invalidate()) 方法来强制屏幕去重绘以显示更新后的效果。你需要在 [`onAnimationUpdate()`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html#onAnimationUpdate(android.animation.ValueAnimator)) 方法里完成这件事。例如，对一个 `Drawable` 对象的颜色进行动画化的时候，它的显示效果仅会在它重绘自己的时候产生变化。`View` 中的所有属性 setter 方法，例如 [`setAlpha()`](https://developer.android.google.cn/reference/android/view/View.html#setAlpha(float)) 和 [`setTranslationX()`](https://developer.android.google.cn/reference/android/view/View.html#setTranslationX(float)) 都会导致属性的失效（invalidate），所以你无须调用手动去调用 [`invalidate()`](https://developer.android.google.cn/reference/android/view/View.html#invalidate()) 方法。

### 用 `AnimatorSet` 来编制多个动画

在许多情况下，你会需要根据其他动画的开始或结束来播放一个动画。Android 系统让你能通过 [`AnimatorSet`](https://developer.android.google.cn/reference/android/animation/AnimatorSet.html) 来将多个动画绑定在一起，以便于能让这些动画同时播放或是顺序播放或是在一定的延时之后播放。你还可以将  [`AnimatorSet`](https://developer.android.google.cn/reference/android/animation/AnimatorSet.html) 嵌套进其他的 [`AnimatorSet`](https://developer.android.google.cn/reference/android/animation/AnimatorSet.html) 中。

下面的示例代码实现了：

1. 播放 `bounceAnim`。
2. 同时播放 `squashAnim1`，`squashAnim2`，`stretchAnim1` 和 `stretchAnim2`。
3. 播放 `bounceBackAnim`。
4. 播放 `fadeAnim`。

```java
AnimatorSet bouncer = new AnimatorSet();
bouncer.play(bounceAnim).before(squashAnim1);
bouncer.play(squashAnim1).with(squashAnim2);
bouncer.play(squashAnim1).with(stretchAnim1);
bouncer.play(squashAnim1).with(stretchAnim2);
bouncer.play(bounceBackAnim).after(stretchAnim2);
ValueAnimator fadeAnim = ObjectAnimator.ofFloat(newBall, "alpha", 1f, 0f);
fadeAnim.setDuration(250);
AnimatorSet animatorSet = new AnimatorSet();
animatorSet.play(bouncer).before(fadeAnim);
animatorSet.start();
```

### 动画监听者

你可以在动画播放的过程中监听以下的事件：

- [`Animator.AnimatorListener`](https://developer.android.google.cn/reference/android/animation/Animator.AnimatorListener.html)

  - [`onAnimationStart()`](https://developer.android.google.cn/reference/android/animation/Animator.AnimatorListener.html#onAnimationStart(android.animation.Animator)) - 动画开始时被调用。
  - [`onAnimationEnd()`](https://developer.android.google.cn/reference/android/animation/Animator.AnimatorListener.html#onAnimationEnd(android.animation.Animator)) - 动画结束时被调用。
  - [`onAnimationRepeat()`](https://developer.android.google.cn/reference/android/animation/Animator.AnimatorListener.html#onAnimationRepeat(android.animation.Animator)) - 动画重复时被调用。
  - [`onAnimationCancel()`](https://developer.android.google.cn/reference/android/animation/Animator.AnimatorListener.html#onAnimationCancel(android.animation.Animator)) - 动画被取消时被调用，动画无论因为什么原因结束都会调用 [`onAnimationEnd()`](https://developer.android.google.cn/reference/android/animation/Animator.AnimatorListener.html#onAnimationEnd(android.animation.Animator))，所以动画被取消的时候也会调用该方法。

- [`ValueAnimator.AnimatorUpdateListener`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html)

  - [`onAnimationUpdate()`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html#onAnimationUpdate(android.animation.ValueAnimator)) - 动画的每一帧都会被调用。监听这个事件以使用 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 在动画过程中生成的每一个值。通过 [`getAnimatedValue()`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html#getAnimatedValue()) 方法来获取当前动画的值。如果你使用 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) 的话就必须实现这个监听者了。

    某些属性或者对象可能会要求你去对 `View` 对象调用 [`invalidate()`](https://developer.android.google.cn/reference/android/view/View.html#invalidate()) 方法来强制屏幕去重绘以显示更新后的效果。你需要在 [`onAnimationUpdate()`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html#onAnimationUpdate(android.animation.ValueAnimator)) 方法里完成这件事。例如，对一个 `Drawable` 对象的颜色进行动画化的时候，它的显示效果仅会在它重绘自己的时候产生变化。`View` 中的所有属性 setter 方法，例如 [`setAlpha()`](https://developer.android.google.cn/reference/android/view/View.html#setAlpha(float)) 和 [`setTranslationX()`](https://developer.android.google.cn/reference/android/view/View.html#setTranslationX(float)) 都会导致属性的失效（invalidate），所以你无须调用手动去调用 [`invalidate()`](https://developer.android.google.cn/reference/android/view/View.html#invalidate()) 方法。

如果你不希望实现 [`Animator.AnimatorListener`](https://developer.android.google.cn/reference/android/animation/Animator.AnimatorListener.html) 接口的所有方法，你也可以选择继承 [`AnimatorListenerAdapter`](https://developer.android.google.cn/reference/android/animation/AnimatorListenerAdapter.html) 类而非实现 [`Animator.AnimatorListener`](https://developer.android.google.cn/reference/android/animation/Animator.AnimatorListener.html) 接口。[`AnimatorListenerAdapter`](https://developer.android.google.cn/reference/android/animation/AnimatorListenerAdapter.html) 类提供了对这些方法的空实现，你可以选择你想要的方法进行 override。

例如：

```java
ValueAnimator fadeAnim = ObjectAnimator.ofFloat(newBall, "alpha", 1f, 0f);
fadeAnim.setDuration(250);
fadeAnim.addListener(new AnimatorListenerAdapter() {
public void onAnimationEnd(Animator animation) {
  balls.remove(((ObjectAnimator)animation).getTarget());
}
```

### 使用 `TypeEvaluator`

如果你希望对一个 Android 系统不知道的类型进行动画化，你可以实现 [`TypeEvaluator`](https://developer.android.google.cn/reference/android/animation/TypeEvaluator.html) 接口来创建自己的计算器。Android 系统知道的类型有 `int`、`float` 和颜色，它们分别被 [`IntEvaluator`](https://developer.android.google.cn/reference/android/animation/IntEvaluator.html)、[`FloatEvaluator`](https://developer.android.google.cn/reference/android/animation/FloatEvaluator.html) 和 [`ArgbEvaluator`](https://developer.android.google.cn/reference/android/animation/ArgbEvaluator.html) 支持。

[`TypeEvaluator`](https://developer.android.google.cn/reference/android/animation/TypeEvaluator.html) 接口只需要你实现 [`evaluate()`](https://developer.android.google.cn/reference/android/animation/TypeEvaluator.html#evaluate(float, T, T)) 这一个方法，例如，[`FloatEvaluator`](https://developer.android.google.cn/reference/android/animation/FloatEvaluator.html) 的实现如下所示：

```java
public class FloatEvaluator implements TypeEvaluator {

  public Object evaluate(float fraction, Object startValue, Object endValue) {
    float startFloat = ((Number) startValue).floatValue();
    return startFloat + fraction * (((Number) endValue).floatValue() - startFloat);
  }
}
```

### 使用插值器

一个插值器定义了动画中的特定值如何被用一个关于时间的函数计算出来。例如，你可以指定一个动画在整个动画过程中线性地进行，这意味着动画的移动在整个过程中都是匀速的，或者你也可以指定一个动画去用一个非线性的函数，例如，在动画的开始或结束时使用加速或减速。

动画系统中的插值器从 `Animator` 那里接收到一个用于表示动画中已流逝时间的比例值。插值器根据动画想要提供的效果来修改这个比例值。Android 系统在 [`android.view.animation`](https://developer.android.com/reference/android/view/animation/package-summary.html) 包中提供了一系列常用的插值器，如果这些插值器都不符合你的要求，你可以通过实现 [`TimeInterpolator`](https://developer.android.google.cn/reference/android/animation/TimeInterpolator.html) 接口来创建你自己的插值器。

作为例子，下面比较了 [`AccelerateDecelerateInterpolator`](https://developer.android.google.cn/reference/android/view/animation/AccelerateDecelerateInterpolator.html) 和 [`LinearInterpolator`](https://developer.android.google.cn/reference/android/view/animation/LinearInterpolator.html) 这两个默认插值器的实现。

**`AccelerateDecelerateInterpolator`**

```java
public float getInterpolation(float input) {
  return (float)(Math.cos((input + 1) * Math.PI) / 2.0f) + 0.5f;
}
```

**`LinearInterpolator`**

```java
public float getInterpolation(float input) {
  return input;
}
```

### 指定关键帧

一个 [`Keyframe`](https://developer.android.google.cn/reference/android/animation/Keyframe.html) 对象与一个 time/value pair 相关联，它使得你可以为动画设置特定时间点上的状态。每一个关键帧还可以拥有其自己的插值器用以控制动画在关键帧之前的行为以及在关键帧的行为。

你需要通过工厂方法 [`ofInt()`](https://developer.android.google.cn/reference/android/animation/Keyframe.html#ofInt(float))、[`ofFloat()`](https://developer.android.google.cn/reference/android/animation/Keyframe.html#ofFloat(float)) 或 [`ofObject()`](https://developer.android.google.cn/reference/android/animation/Keyframe.html#ofObject(float)) 来创建一个 [`Keyframe`](https://developer.android.google.cn/reference/android/animation/Keyframe.html) 对象，然后，你需要调用 [`ofKeyframe()`](https://developer.android.google.cn/reference/android/animation/PropertyValuesHolder.html#ofKeyframe(android.util.Property, android.animation.Keyframe...)) 工厂方法来获取一个 [`PropertyValuesHolder`](https://developer.android.google.cn/reference/android/animation/PropertyValuesHolder.html) 对象。一旦你得到了这个对象，你就可以通过这个 [`PropertyValuesHolder`](https://developer.android.google.cn/reference/android/animation/PropertyValuesHolder.html) 对象和目标对象来获得一个动画了。下面的代码展示了具体的做法：

```java
Keyframe kf0 = Keyframe.ofFloat(0f, 0f);
Keyframe kf1 = Keyframe.ofFloat(.5f, 360f);
Keyframe kf2 = Keyframe.ofFloat(1f, 0f);
PropertyValuesHolder pvhRotation = PropertyValuesHolder.ofKeyframe("rotation", kf0, kf1, kf2);
ObjectAnimator rotationAnim = ObjectAnimator.ofPropertyValuesHolder(target, pvhRotation)
rotationAnim.setDuration(5000ms);
```

### 将 `View` 动画化

属性动画系统允许你将 `View` 对象的动画流线型地组织起来，并且相对于视图动画系统而言还有一些好处。视图动画系统通过改变视图对象的绘制方式来实现对它们的转换。这个过程由 `View` 对象的容器来进行处理，因为 `View` 对象自己并没有这些被操作的属性。这种实现的结果是尽管 `View` 对象被动画化了，但它自身并没有发生改变。这将导致一些奇怪的行为，例如一个对象被绘制到了其他地方，但它仍然存在于原地。在 Android 3.0 里添加了这些新的属性以及相应的 getter 和 setter 方法来消除这个缺点。

属性动画系统可以通过改变 `View` 对象里的实际属性来将其动画化。而且，`View` 对象还会在属性改变的时候自动地调用 [`invalidate()`](https://developer.android.google.cn/reference/android/view/View.html#invalidate()) 方法来刷新屏幕。`View` 类型中与属性动画实现相关的新属性有：

- `translationX` 和 `translationY`：这两个属性控制了 `View` 对象从左到右和从上到下所在的相对位置，这里的相对位置是指相对于它的容器给它设定的位置。
- `rotation`，`rotationX` 和 `rotationY`：这三个属性控制了在 2D（`rotation` 属性）和 3D 下相对于中心点的旋转角度。
- `scaleX` 和 `scaleY`：这两个属性控制了 `View` 对象相对于中心点的 2D 缩放比例。
- `x` 和 `y`：这两个属性用于描述 `View` 对象在容器中的最终位置，也就是其从左到右的位置加上 `translationX` 的值以及从上到下的位置加上 `translationY` 的值。
- `alpha`：表示 `View` 对象的 alpha 透明度。它的值默认为 1（不透明），如果设定为 0 则表示完全透明（不可见）。

你只需要创建一个 property animator 并指定你想动画化的 `View` 对象的属性就可以将其动画化了，例如：

```java
ObjectAnimator.ofFloat(myView, "rotation", 0f, 360f);
```

#### 使用 `ViewPropertyAnimator` 进行动画化

[`ViewPropertyAnimator`](https://developer.android.google.cn/reference/android/view/ViewPropertyAnimator.html) 类提供了一个简单的方法来并行地动画化一个 `View` 对象的多个属性，并且在底层只用到了一个 [`Animator`](https://developer.android.com/reference/android/animation/Animator.html) 对象。它的行为很像一个 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html)，因为它修改了 `View` 对象的实际属性，但它在同时动画化多个属性的时候会比 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html) 更加高效。而且，使用 [`ViewPropertyAnimator`](https://developer.android.google.cn/reference/android/view/ViewPropertyAnimator.html) 的代码会更加清晰易读。下面的代码片段展示了使用多个 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html)、使用一个 [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html) 以及使用 [`ViewPropertyAnimator`](https://developer.android.google.cn/reference/android/view/ViewPropertyAnimator.html) 来同时动画化地改变 `View` 对象的 x 和 y 属性的区别。

**多个 `ObjectAnimator` 对象**

```java
ObjectAnimator animX = ObjectAnimator.ofFloat(myView, "x", 50f);
ObjectAnimator animY = ObjectAnimator.ofFloat(myView, "y", 100f);
AnimatorSet animSetXY = new AnimatorSet();
animSetXY.playTogether(animX, animY);
animSetXY.start();
```

**单个 `ObjectAnimator` 对象**

```java
PropertyValuesHolder pvhX = PropertyValuesHolder.ofFloat("x", 50f);
PropertyValuesHolder pvhY = PropertyValuesHolder.ofFloat("y", 100f);
ObjectAnimator.ofPropertyValuesHolder(myView, pvhX, pvyY).start();
```

**使用 `ViewPropertyAnimator`**

```java
myView.animate().x(50f).y(100f);
```

### 在 XML 中声明动画

属性动画系统让你能够不用编程实现属性动画而是在 XML 中声明属性动画。通过在 XML 中定义你自己的动画，你可以很容易地在多个 `Activity` 里复用你的动画，而且还能更容易地修改动画序列。

为了区分使用新属性动画 API 的代码和使用遗留 [`view animation`](https://developer.android.google.cn/guide/topics/graphics/view-animation.html) 框架的代码，从 Android 3.1 开始，你需要将这些 XML 文件放在 `res/animator/` 文件夹下。

下面是几个属性动画类和它们对应的 XML 标签列表：

- [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html) - `<animator>`
- [`ObjectAnimator`](https://developer.android.google.cn/reference/android/animation/ObjectAnimator.html) - `<objectAnimator>`
- [`AnimatorSet`](https://developer.android.google.cn/reference/android/animation/AnimatorSet.html) - `<set>`

你可以在 [Animation Resources](https://developer.android.google.cn/guide/topics/resources/animation-resource.html#Property) 这个链接里找到你能在 XML 声明中使用的属性。下面的例子顺序播放了两组对象动画，前一组动画里同时播放了两个对象动画：

```xml
<set android:ordering="sequentially">
    <set>
        <objectAnimator
            android:propertyName="x"
            android:duration="500"
            android:valueTo="400"
            android:valueType="intType"/>
        <objectAnimator
            android:propertyName="y"
            android:duration="500"
            android:valueTo="300"
            android:valueType="intType"/>
    </set>
    <objectAnimator
        android:propertyName="alpha"
        android:duration="500"
        android:valueTo="1f"/>
</set>
```

为了运行这个动画，你需要在运行这个动画集合前在代码中将这个 XML 资源填充到 [`AnimatorSet`](https://developer.android.google.cn/reference/android/animation/AnimatorSet.html) 对象里，然后再设定这些动画的目标对象。你可以调用 [`setTarget()`](https://developer.android.google.cn/reference/android/animation/AnimatorSet.html#setTarget(java.lang.Object)) 方法来方便地为 [`AnimatorSet`](https://developer.android.google.cn/reference/android/animation/AnimatorSet.html) 的子类对象设置一个目标对象：

```java
AnimatorSet set = (AnimatorSet) AnimatorInflater.loadAnimator
  (myContext, R.anim.property_animator);
set.setTarget(myObject);
set.start();
```

你也可以在 XML 中声明一个 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html)：

```xml
<animator xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000"
    android:valueType="floatType"
    android:valueFrom="0f"
    android:valueTo="-100f" />
```

为了在你的代码中使用这个 [`ValueAnimator`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.html)，你需要填充一个对象，然后添加一个 [`AnimatorUpdateListener`](https://developer.android.google.cn/reference/android/animation/ValueAnimator.AnimatorUpdateListener.html) 监听器，在回调中获取更新的动画值，然后将其用在你的 `View` 对象上，例如：

```java
ValueAnimator xmlAnimator = (ValueAnimator) AnimatorInflater.loadAnimator
  (this, R.animator.animator);
xmlAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
  @Override
  public void onAnimationUpdate(ValueAnimator updatedAnimation) {
    float animatedValue = (float)updatedAnimation.getAnimatedValue();
    textView.setTranslationX(animatedValue);
  }
});

xmlAnimator.start();
```

## 绘图动画

绘图动画让你能够一个接一个地读取一系列可绘制的资源来构建一个动画。这是一个传统的动画实现方式，这种动画是通过像电影一样按序播放一个包含不同的图片序列来实现的。绘图动画的基础类是 [`AnimationDrawable`](https://developer.android.google.cn/reference/android/graphics/drawable/AnimationDrawable.html) 类。

尽管你可以在你的代码中使用 [`AnimationDrawable`](https://developer.android.google.cn/reference/android/graphics/drawable/AnimationDrawable.html) 类的 API，但更容易完成工作的选择是使用一个 XML 文件来列出组成动画的帧。这类动画的 XML 文件应该被放在你的 Android 工程的 `res/drawable/` 目录下。你需要在这个文件中说明帧的顺序和持续时间。

绘图动画的 XML 文件包含了一个 `<animation-list>` 元素作为根节点，其中浩瀚了一系列的 `<item>` 元素作为其子节点，每一个 `<item>` 元素都定义了动画的一帧：即一个该帧的可绘制资源和持续时间。例如：

```xml
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="true">
    <item android:drawable="@drawable/rocket_thrust1" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust2" android:duration="200" />
    <item android:drawable="@drawable/rocket_thrust3" android:duration="200" />
</animation-list>
```

这个动画仅仅播放了三帧。通过将 `android:oneshot` 属性设置为 `true` 来使得动画仅播放一次并停在最后一帧。如果设定为 `false`，那么这个动画就会循环播放。假设这段 XML 被保存在 `res/drawable/` 目录下的 `rocket_thrust.xml` 文件里，那么它就可以像下面的例子一样添加进 `View` 对象的背景并在被触控时开始播放：

```java
AnimationDrawable rocketAnimation;

public void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  setContentView(R.layout.main);

  ImageView rocketImage = (ImageView) findViewById(R.id.rocket_image);
  rocketImage.setBackgroundResource(R.drawable.rocket_thrust);
  rocketAnimation = (AnimationDrawable) rocketImage.getBackground();
}

public boolean onTouchEvent(MotionEvent event) {
  if (event.getAction() == MotionEvent.ACTION_DOWN) {
    rocketAnimation.start();
    return true;
  }
  return super.onTouchEvent(event);
}
```

需要注意的是，`start()` 方法不可以在 `Activity` 的 `onCreate()` 方法执行期间被调用，因为 [`AnimationDrawable`](https://developer.android.google.cn/reference/android/graphics/drawable/AnimationDrawable.html) 还没有被完全附加到窗口上。如果你希望立即播放动画，你可以选择在 `Activity` 的 [`onWindowFocusChanged()`](https://developer.android.google.cn/reference/android/app/Activity.html#onWindowFocusChanged(boolean)) 方法里调用这个它，这个方法会在 Android 将窗口设为焦点时被回调。