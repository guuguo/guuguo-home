---
title: 【flutter 起步走】flutter共享数据利器，InheritedWidget原理探秘！
---
![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=N2E1YWM5ZTQ3YTlkNGMwMzhkNjdjMWNhYWViZmMyN2NfZGFrc0dRM2xOZ0JES3ZjaDJlOXJWM2NSbjNpMWhqMGlfVG9rZW46Ym94Y25Tc3VyTmNUQ1VxeERNU29WU0xCOWFjXzE2NDgwMjYwMjk6MTY0ODAyOTYyOV9WNA)

> 知其然，也要知其所以然。最近的搬砖工作中，开发ui页面都是使用flutter，android原生只沦为了后台逻辑处理的后盾。在搬砖过程中，往往只要知道怎么用，便能搭起小房子，而要建的恢弘又大气，还是少不了对于原理的学习。
在接触`flutter`中，`Widget`是我们接触最多的类。我们对于各种界面的搭建用的就是各式各样的`Widget`，有`StatefullWidget`和`StatelessWidget`。印象中`Widget`就是最为ui搭建而存在的，然而在flutter的中，还有一种`widget`不负责ui控制，但却十分重要，那就是`InheritedWidget`。
在我们常用的第三方数据共享库中，都有`InheritedWidget`的影子，比如下面两个。
- flutter_bloc
- Provider
还有开发中用到的很多数据，比如`Theme.of(context)`,`DefaultTextStyle.of(context)`等，都是在父节点配置，子widget全局获取。这都离不开`InheritedWidget`的特性。
既然如此重要，今天就好好学习下`InheritedWidget`的大致原理吧。
# 一，InheritedWidget 介绍
`InheritedWidget`是 Flutter 中非常重要的一个功能型组件，它提供了一种在 widget 树中从上到下共享数据的方式，比如我们在应用的根 widget 中通过`InheritedWidget`共享了一个数据，那么我们便可以在任意子widget 中来获取该共享的数据！这个特性在一些需要在整个 widget 树中共享数据的场景中非常方便！如Flutter SDK中正是通过 InheritedWidget 来共享应用主题（`Theme`）和 Locale (当前语言环境)信息的。
> `InheritedWidget`和 React 中的 context 功能类似，和逐级传递数据相比，它们能实现组件跨级传递数据。`InheritedWidget`的在 widget 树中数据传递方向是从上到下的，这和通知`Notification`的传递方向正好相反。

# 二，组件树中的数据共享用法(利用 ValueNotrifier 和 InhertitedWidget 实现简单的状态管理)
1. 我们首先创建一个MyProvider 继承自InheritedWidget，以存放状态数据，该状态继承自ChangeNotifier，即代码中的`model`变量。
```dart
import 'package:flutter/widgets.dart';

///提供状态存储
class MyProvider<T extends ChangeNotifier> extends InheritedWidget {
  T model;
  MyProvider({Key? key,required Widget child,required this.model}) : super(key: key,child:child);

  @override
  Widget build(BuildContext context) {
    return this.child;
  }

  static MyProvider<T>? of<T extends ChangeNotifier>(BuildContext context){
    return context.dependOnInheritedWidgetOfExactType<MyProvider<T>>();
  }

  @override
  bool updateShouldNotify(covariant MyProvider oldWidget) {
    return false;
  }
}
```
1. 接下来创建一个Customer，用来实现动态刷新，和局部刷新，原理主要是利用changeNotifier监听器更新数据更新，注册对应监听器刷新Customer的子布局。
```Dart
///提供状态消费，在状态ValueNotifier更新的时候，自动刷新Customer组件
class Customer<T extends ChangeNotifier> extends StatefulWidget{
  const Customer({required this.builder,});
  final ProviderBuilder<T> builder;

  @override
  State<Customer<T>> createState() => _CustomerState<T>();
}

class _CustomerState<T extends ChangeNotifier> extends State<Customer<T>> {

  late MyProvider? provider;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    provider=MyProvider.of<T>(context);
    if(provider==null) throw "请在祖先节点提供MyProvider<T extends ChangeNotifier>";
    provider!.model.addListener(() {
      setState(()=>{});
    });
  }
  @override
  Widget build(BuildContext context) {
    return widget.builder(context,MyProvider.of<T>(context)!.model);
  }
}
typedef ProviderBuilder<T extends ChangeNotifier> = Widget Function(BuildContext context,T value);
```
1. 接着创建一个展示用的widget，包含显示数据的子widget 和 点击刷新数据的 按钮，代码如下：
```dart
import 'package:flutter/material.dart';
import 'package:flutter/widgets.dart';
import 'package:free/widget/my_provider.dart';
import 'package:provider/provider.dart';

class TestWidget extends StatefulWidget {
  const TestWidget({Key? key}) : super(key: key);

  @override
  State<TestWidget> createState() => _TestWidgetState();
}
class TestModel extends ChangeNotifier{
  var num=0;

  TestModel() : super();
}
class _TestWidgetState extends State<TestWidget> {
   var model=TestModel();
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: MyProvider<TestModel>(
        model:model,
        child: Builder(
          builder: (context) {
            return Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  MyTextWidget(),
                  MaterialButton(
                    color: Colors.blueAccent,
                      elevation: 10,
                      onPressed: () {
                        model.num++;
                        model.notifyListeners();
                      },
                      child: Text("点击加一")),
                ],
              ),
            );
          }
        ),
      ),
    );
  }
}
class MyTextWidget extends StatelessWidget {
  const MyTextWidget ({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Customer<TestModel>(builder:(c,value) => Text("${value.num}"));
  }
}
```
至此，利用`InheritedWidget`和`ChangeNotifier`实现的简易状态管理库就实现了。该库实现了跨组件共享数据，跨组件更新数据状态，并更新对应的兄弟组件。
这也是一些第三方开源状态库的基本原理。

![img](https://krdl1nfc2s.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWIxZmI5Mzk0ODkxNWI3ZTZjYzY3MDdmYTI5ZjM1ODZfY2FRUk5UbHFranhOUGVDZjhLVkZRMEFuZ3RTeFdnck1fVG9rZW46Ym94Y25IdXJYSm9sOENPakV3OTFQMFdyakFiXzE2NDgwODczMzE6MTY0ODA5MDkzMV9WNA)

# 三，InheritedWidget如何共享数据
先看看我们获取 InheritedWidget实例的**dependOnInheritedWidgetOfExactType**方法，注释翻译如下：
> 获取给定类型T的最近小部件，它必须是具体InheritedWidget子类的类型，并将此构建上下文注册到该小部件，以便当该小部件更改时（或引入该类型的新小部件，或小部件消失离开），这个构建上下文被重建，以便它可以从那个小部件获取新值。
> 这通常从of()静态方法中隐式调用，例如Theme.of 。
> 不应从小部件构造函数或State.initState方法调用此方法，因为如果继承的值发生更改，这些方法将不会再次被调用。为了确保小部件在继承值更改时正确更新自身，只能从构建方法、布局和绘制回调或从State.didChangeDependencies调用（直接或间接）。
> 不应从State.dispose调用此方法，因为此时元素树不再稳定。要从该方法引用祖先，请将对祖先的引用保存在State.didChangeDependencies中。从State.deactivate中使用此方法是安全的，每当从树中删除小部件时都会调用该方法。
> 也可以从交互事件处理程序（例如手势回调）或计时器调用此方法，以获取一次值，如果该值不会被缓存并稍后重用。
> 调用此方法是 O(1)，具有较小的常数因子，但会导致更频繁地重建小部件。
> 一旦小部件通过调用此方法注册对特定类型的依赖关系，它将被重建，并且State.didChangeDependencies将被调用，每当与该小部件相关的更改发生时，直到下一次移动小部件或其祖先之一（例如例如，因为添加或删除了祖先）。
> aspect参数仅在T是支持部分更新的InheritedWidget子类时使用，例如InheritedModel 。它指定此上下文所依赖的继承小部件的“方面。
> **T? dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({ Object? aspect });**

1. 我们找到了`BuildContext`的该方法的具体实现，实际上是在`Element`中，其实现了`BuildContext`。
```Dart
@override
T? dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object? aspect}) {
  assert(_debugCheckStateIsActiveForAncestorLookup());
  final InheritedElement? ancestor = _inheritedWidgets == null ? null : _inheritedWidgets![T];
  if (ancestor != null) {
    return dependOnInheritedElement(ancestor, aspect: aspect) as T;
  }
  _hadUnsatisfiedDependencies = true;
  return null;
}
```
可以看到主要是从 `**_inheritedWidgets**` 中找到对应`Type`的`InheritedElement` 。
1. 我们再去看`InheritedElement`源码，主要关注下`_updateInheritance`方法
```Dart
@override
void _updateInheritance() {
  assert(_lifecycleState == _ElementLifecycle.active);
  final Map<Type, InheritedElement>? incomingWidgets = _parent?._inheritedWidgets;
  if (incomingWidgets != null)
    _inheritedWidgets = HashMap<Type, InheritedElement>.of(incomingWidgets);
  else
    _inheritedWidgets = HashMap<Type, InheritedElement>();
  _inheritedWidgets![widget.runtimeType] = this;
}
```
**小结：**可以看到`_updateInheritance`方法中，其向**_inheritedWidgets**中添加了自己。而**_inheritedWidgets**在**Element**树中会被子Element继承，所以子树就能轻松获取到该`InheritedElement`实例了。
而`_updateInheritance`方法，是在`Element` `mount` 和 `active`方法中被调用的，当`**InheritedElement**`被挂载到element树中或者被激活后执行，就能被子element获取到了。
至此就能大概了解`**InheritedWidget**` 如何共享数据了，主要是利用`InhritedElement`作为中转持有。
# 最后：
最后我们大概了解了`InheritedWidget`的使用和原理。但是要进一步梳理整个流程的时候，就需要梳理Widget，Element和RenderObject三者的关系了。
学无止境，下一把就来了解下flutter的Widget体系。
加油！！