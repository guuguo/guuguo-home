---
title: 【Android进阶】这一次把View绘制流程刻在脑子里！！
---
> 天空看不见云，大火球在上面肆意发光，逼着毛孔慢慢渗出汗水。
> 
> 我离开舒适区，跑出去面试了几次。
> 
> 得到的最多的反馈是不够深入。
> 
> 作为一个五年经验的安卓开发者，欠缺的还有很多。
# 前言
从一个view实例被创建，到展示到屏幕上，都经历了怎么样的一个流程？在安卓开发中，这似乎是一个基本的知识，应该被开发者清楚地认识明白，面试中也作为问题频频出现，然而我还是认识得不深刻。
`Android View`的绘制流程 是View相关的核心知识点。我希望通过这篇文章学习并分享Android View绘制流程的始末。
**并将其刻在脑子里。**
# 目录
本文分为以下流程学习，阅读完本文将会学习到`PhoneWindow`,`WindowManger`,`ViewRootImpl`,`View` 等关键类的联系和作用。对window窗体机制以及绘制流程有所了解。
1. 流程图分析
2. 了解view绘制流程
3. 了解`setContentView`如何附加到内容到页面
# 关键类解释
- `Choreographer`：协调动画、输入和绘图的时间。`Choreographer`从显示子系统接收定时脉冲（例如垂直同步），然后安排工作发生，作为渲染下一个显示帧的一部分。
# 一. 流程图分析
## 1.1 创建Activity到setContentView的窗口附加流程图
下图展示了window的创建到`setContentView`之后的窗体view树变化情况
![activity 设置布局流程](https://upload.cc/i1/2021/07/27/LljAid.png)

## 1.2 view绘制流程图
![绘制流程图](https://upload.cc/i1/2021/07/27/bPq8gy.png)
# 二. view绘制流程
## 2.1 绘制流程分析
在我们调用`requestLayout ` 和 `invalidate`的时候，我们会让view刷新布局和绘制。所以从这两个方法入手，可以完整地走一遍绘制流程。
绘制动画等行为主要通过`Choreographer` 类协调。

1. 调用`requestLayout `和 `invalidate`标记绘制和充布局信息
2. `Choreographer`接受系统垂直同步等脉冲消息，在`scheduleTraversals`方法中回调执行`doTraversal` 开始遍历view树。
3. 触发`ViewRootImpl#performTraversals`完成view树遍历
   1. 如果`layoutRequested` 为true,`measureHierarchy` 中测量 `mView` 及其子`view`
   2. 需要的话，触发`ViewRootImpl#performLayout` 完成布局
   3. 如果`view`没有隐藏且`TreeObserver`中没有拦截绘制，就调用`performDraw`，完成绘制
      1. 计算dirty脏区域
      2. 从mSurface中 获取脏区域的canvas，交给view绘制
## 2.2 `ViewRootImpl` 创建时机
从上面可以看到，所有的绘制和布局都是由`ViewRootImpl#doTraversal`触发，然后对其持有的view树进行遍历绘制。所以一定要了解`ViewRootImpl`和其持有的`DecorView`的创建和关联时机。关键流程如下：
1. `Activity#handleResume` 的时候，调用`WIndowManager#addView`添加`decorView`
2. 调用到`WindowManagerGlobal#addView` 的时候创建`ViewRootImpl`实例。
3. 调用`ViewRootImpl#setView`完成一系列初始化方法
   1. 注册`mDisplayListener `到`DisplayManager`，接收显示更新回调
   2. 调用 `requestLayout `更新一次布局大小和位置信，以确保从系统接收任何其他事件之前进行过一次布局
   3. 通过`WindowSession `调用addToDisplayAsUser，添加window
4. 在接收系统事件的时候，调用scheduleTraversals 绘制view树
WindowMangerGlobal 最终调用的其实都是ViewRootImpl方法。ViewRootImpl在addView关联号DecorView后，还调用了setView方法进行初始化，接收垂直同步脉冲信息，代码如下：
```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView,
        int userId) {
            ...
            mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);
            ...
            // Schedule the first layout -before- adding to the window
            // manager, to make sure we do the relayout before receiving
            // any other events from the system.
            requestLayout();
           ...
           try{
                res = mWindowSession.addToDisplayAsUser(mWindow, mSeq, mWindowAttributes,
                        getHostVisibility(), mDisplay.getDisplayId(), userId, mTmpFrame,
                        mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                        mAttachInfo.mDisplayCutout, inputChannel,
         
            } 
}
```
在初始化的最后，通过`WindowSession` 调用`addToDisplayAsUser`添加了`window`到屏幕显示中。
# 三.  附加contentView到界面 
当我们启动activity，将我们写的xml布局文件显示在屏幕上，其中经历了那些过程呢？我们要在界面上展示内容，有如下几个步骤：
1. 启动activity，在`performLaunchActivity`的时候创建`Activity`并且attach和调用onCreate方法
2. 在attach的时候，创建PhoneWindow实例并持有mWindow引用
3. 调用`setContentView` 以附加内容到windows中
4. 通过确认`decorView `以及 `subDecorView`存在，创建`DecorView`和`subDecorView`
5. 添加`ContentView`到`decorView`树中的 `R.id.content`节点
6. 当`handleResumeActivity`的时候，调用`WindowManager.addView`。关联`View`和`ViewRootImpl`，后续便可以绘制。
## 3.1 创建PhoneWindow
我们先看启动activity的方法，`ActivityThread#performLaunchAcivity`。 从该方法源码中可知，启动activity的方法流程如下：
1. 创建Activity实例 ，在`Instrumentation#newActivity`完成
2. 创建PhoneWindows附加到Activity。在`Activity#attachAcitivity`完成
3. 调用Activity的onCreate生命周期,代码是`Instrumentation#callActivityOnCreate`
4. 在`onCreate`中执行用户自定义的代码，比如`setContentView`。
所以可知，在`activity`准备启动的时候，就已经完成了`PhoneWindows`实例的创建。而接下来就执行到了我们在`Activity#onCreate`中调用`setContentView`方法设置的自定义布局。
## 3.2 setContentView的本质
`activity`在启动之后，我们通常在`onCreate`调用`setContentView`中设置自己的布局文件。我们来具体看看`setContentView`做了什么。
`setContentView`方法本质其实是向`android.R.id.content`添加自己。
我们看`AppCompatDelegateImpl#setContentView`
```java
@Override
public void setContentView(View v, ViewGroup.LayoutParams lp) {
    ///确认好 window decorView 以及 subDecorView
    ensureSubDecor();
    //向 android.R.id.content 添contentView
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    contentParent.addView(v, lp);
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```
这一块代码关键在于向id为`android.R.id.content`的子view中添加`contentView`。
`addView`的过程自然会触发布局的重新渲染。
关键之处还是在于`ensureSubDecor()`方法中对于`decoView `以及`subDecorView`的实例化创建工作。
## 3.3  确认window ，decorView 以及 subDecorView
先看看`AppCompatDelegateImpl#ensureSubDecor()`的主要实现：
```java
private void ensureSubDecor() {
    if (!mSubDecorInstalled) {
        mSubDecor = createSubDecor();
    }
}
private ViewGroup createSubDecor() {
    // Now let's make sure that the Window has installed its decor by retrieving it
    ensureWindow();
    mWindow.getDecorView();

    final LayoutInflater inflater = LayoutInflater.from(mContext);
    ViewGroup subDecor = null;

    //省略其他样式subDecor布局的实例化
    //包含 actionBar floatTitle ActionMode等样式
   subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
  

    //省略状态栏适配代码
    //省略actionBar布局替换代码
    mWindow.setContentView(subDecor);
    return subDecor;
}
```
代码很长，上面是经过省略之后的主要代码。可以看到代码逻辑很清晰：
- 步骤一：确认window并attach(设置背景等操作)
- 步骤二：获取DecorView，因为是第一次调用所以会installDecor(创建DecorView和Window#ContentLyout)
- 步骤三：从xml中实例化出subDecor布局
- 步骤四：设置内容布局: `mWindow.setContentView(subDecor);`
## 3.4 初始化 installDecor 
关键两处代码是`Window#installDecor` 和 `Window#setContentView`。
先看一下`Window#installDecor`的代码：
```java
private void installDecor() {
    mForceDecorInstall = false;
    mDecor = generateDecor(-1);
    if (mContentParent == null) {
        //R.id.content
        mContentParent = generateLayout(mDecor);
        final decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);

        if (decorContentParent != null) {
            //...省略一些decorContentParent的处理
        } else {
            mTitleView = findViewById(R.id.title);
            final View titleContainer = findViewById(R.id.title_container);
            ///省略设置mTitle 设置标题容器显示隐藏
        }

        //设置decor背景
        //省略activity各种动画的实例化
    }
}
```
这一块除了一些标题。动画的初始化之外，最为关键的就是
- 通过`generateDecor()`生成了`DecorView`
- 以及通过`generateLayout()`获取了`ContentLayout`
  - 获取windowStyle的各种属性，并设置Features和WindowManager.LayoutParams.flags等
  - 如果window是顶层容器，获取背景资源等信息
  - 获取各种默认布局实例化( R.layout.screen_simple等)，加到DecorView中。和`AppComptDelegateImpl#createSubDecor`创建的`subDecor`类似。
  - 获取`com.android.internal.R.id.content` 布局，并返回为`ContentLayout`

接下来再看`Window#setContentView`了：
```java
@Override
public void setContentView(View view, ViewGroup.LayoutParams params) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        view.setLayoutParams(params);
        final Scene newScene = new Scene(mContentParent, view);
        transitionTo(newScene);
    } else {
        mContentParent.addView(view, params);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```
关键代码很简单，就是往`mContentParent`中添加view。而从上文可知，mContentParent就是`andorid.R.id.content`的布局。
## 3.5 小结：
分析得知，xml 编写layout布局到展示布局在界面上，经历了这么个流程：
1. #### 启动activity
2. #### 创建PhoneWindow
3. #### 设置布局setContentView
   1. 确认subDecorView的初始化
      1. 初始化生成DecorView
         1. Window中 创建DecorView
         2. Window中 创建样例到代码布局作为DecorView的子布局（比如R.layout.smple）
         3. 返回 `com.android.internal.R.id.content` 作为ContentPrent
         4. Window中 处理`DecorContentParent `布局，或者处理标题等内容
      2. 实例化subDecorView，如R.layout.abc_screen_simple
      3. 设置 `subDecorView`到Window的ContentPrent
   2. 添加实例化的Layout 到android.R.id.*content*
4. #### addView的时候调用 `requestLayout(); invalidate(true);`
   1. `requestLayout`遍历View树到DecorView，调用`ViewRootImpl#requestLayoutDuringLayout`
   2. `invalidate` 判断区域内的view，将需要刷新的view设置为dirty。
5. #### 等待绘制时机（handleResumeActivity之后才会触发绘制），通过`Choreographer` 遍历view树的布局和绘制操作。

据此算是完全搞清楚了`setContentView `的时候经历了什么。也明白了activity如何根据float， title等属性生成不同的布局了。

# 最后
这一篇详细介绍了`view`的绘制系统，同时也是window窗口机制以及 android显示机制的前置知识。view系统是我们ui开发过程中接触最深的android知识。了解绘制原理不止对面试有帮助。对于自己的开发工作也有不小的助力。