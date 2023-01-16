
![www.gaoding.com_design_id=20925982878221356&mode=user.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08c42a36ea6445a0b9da823e1b28d3c9~tplv-k3u1fbpfcp-watermark.image?)

刚刚看完[张风捷特烈](https://juejin.cn/user/149189281194766)的[Flutter 布局探索](https://juejin.cn/book/7075958265250578469/section)小册。感觉受益良多。

看到结局的问题：**如何区分`StatelessWidget` 和 `StatefulWidget` 的使用场景**，不禁开始自问，对于StatefulWidget ，StatelessWidget，以及flutter中Widget的众多子类我真的足够了解吗？

对于自己经常要打交道的东西，如果只是一知半解则不利于进步。

下面就从源码的角度来学习下`flutter`基础的几个`Widget` 都起到了什么作用。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cae8bbe170647b285f46dab12a6b4d1~tplv-k3u1fbpfcp-watermark.image?)

**先给个简单总结：**

- 其中StatelessWidget 和 StatefulWidget 起到了组织组合子组件的作用。
- RenderObjectWidget 起到渲染作用。包含绘制偏移和测量信息。
- ProxyWidget 可以携带信息，以供其他组件使用。

# 一、探索StatelessWidget的组件构建
在使用`StatelessWidget`的时候，通常只需要实现一个build方法。就拿我们常用的`Container`组件举例，他就是`StatelessWidget` 的子类。他的build方法返回的就是各种组件的组合嵌套。
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bce2692604ad4103982db2c8622a5f2b~tplv-k3u1fbpfcp-zoom-1.image)

他的各种成员属性也只是用来配置子组件的组合方式而已。

## 1. StatelessWidget 的build调用时机，以及widget树遍历流程
`Container`组件是`StatelessWidget`的经典子类。

我们通过断点调试看看`Container` 组件`build`方法的调用堆栈

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a71a67e82e1b4851a6353666c696fe8b~tplv-k3u1fbpfcp-zoom-1.image)

在`ComponentElement` 的`performRebuild` 方法调用的时候，触发了`build`方法，从`stateless`中获取了`build`返回的`Widget`，而又在`performRebuild` 调用了`updateChild`方法，对所有的子孙`Element`进行`build`遍历。

> `ComponentElement`是Widget对应元素`StatelessElement`和`StatefulElement`的父类。

我们拉到最初的调用栈。`Element`栈调用的起点在于`attachRootWidget`方法。

还记得我们flutter app开发的起点吗？就是`runApp(App())`方法，开启了整个flutter app。
`attachRootWidget`**方法正是我们在调用`runApp`的时候执行的。**

在其中，执行了`RenderObjectToWidgetAdapter`组件的初始化，将`renderView` 和 `rootWidget`作为入参。并且调用`attachToRenderTree`返回元素树顶点的Element。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52cb09f71ff94256a37e8c63941bb6bd~tplv-k3u1fbpfcp-zoom-1.image)

### 三颗树的顶点
其中`renderView`是`RenderObject`树的顶点，`_renderViewElement`是`Element`树的顶点。匿名的`RenderObjectToWidgetAdapter`则是Widget树的顶点，但是他没有被引用。Widget树的维护依赖于`Element`树，`rootWidget`就是我们的runApp组件节点，被作为参数挂载到`RenderObjectToWidgetAdapter`根组件中，被后续的Element挂载循环使用。

`Element`中也存放了`_parent`变量，所以我们通过`Element`对象可以轻松的追溯到祖先节点。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64408a38fdf9420aa583f7e3942eee8f~tplv-k3u1fbpfcp-zoom-1.image)

我们从上面的分析可以得出`ComponentElement` 的 performRebuild方法是`element.build`传承关键方法  ，mount方法也能由此挂载出所有子树（其他类型的Element实现方案略有不同）

在ComponentElement中。也由`performRebuild`构建出一层层的子孙节点。代码如下，注意红色方框的代码。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d0b5b2be16c4324b82052beab1102d0~tplv-k3u1fbpfcp-zoom-1.image)

第一个红框中是`build()`方法的执行。意味着每次`performRebuild`被调用的时候，子组件都会被`build`出来，由此可知`widget`是唯一的，每次更新都会有新的`Widget`生成。

在`updateChild`的过程中，如果子`element`还未生成，就会调用`widget.createElement()`方法获得`element` 。

我们再看`StatelessWidget` 的源码，实现了`createElement`方法返回了自定义的`StatelessElement`。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec91019559f34dbaac73fdee9d69c09b~tplv-k3u1fbpfcp-zoom-1.image)

生成的子`Element` 都会在`ComponentElement`中被持有，以便后续更新

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c61fd233c12d46729dfc4f84b6695d39~tplv-k3u1fbpfcp-zoom-1.image)

由此可知，`ComponentElement`维系了祖孙关系，其子类Element对应的 StatelessWidget，StatefulWidget，ParentDataWidget 和 InheritedWidget都天然拥有子孙关系能力。

如下所示，`StatefulElement` 是 `ComponentElement` 的子类。
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/592ad9721e6444c4a95360d09c63e816~tplv-k3u1fbpfcp-zoom-1.image)

## 2. `StatelessWidget` 和Element在渲染中的更新
`widget`的创建都是在`element`树遍历的过程中执行的。
`widget`树依赖于`element`树，在`Element`创建的时候widget实例将会被持有。

`StatelessWidget`在布局和渲染流程中依赖`Element`维系，树关系被`Element`挖掘。
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b65e6327f7b34521b0c5fce0f68dbea6~tplv-k3u1fbpfcp-zoom-1.image)

在`Element` `performeRebuild`重新构建的时候，有一个是否更新Element的判定机制，以优化性能。

不管是更新update还是挂载mount，每次子widget都会先`build()`出来。再进行新旧比较。Widget都是一次性的，如果有状态需要保存是由其他方式实现的。
我们再看`updateChild`方法。上面一小节提到在子`element`为空的时候，会在其中`createElement`。而在子`Element`不为空的时候，会根据新旧`Widget` 的不同，进行不同的操作。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05e162af4c7440868862d44e2e8dc400~tplv-k3u1fbpfcp-zoom-1.image)

其中通过新旧`widget`的`equals`判定。决定是否复用之前的`element`。如果复用了`element`，根据`canUpdate`方法的返回值，来执行`child.update`方法。所以我们可以得出这样一个结论。

`widget` 的 `canUpdate` 实现，将很大程度上决定 `Element` 的复用。减少重新绘制，对`State`重新赋值，甚至状态丢失的资源浪费。

## 3. 探索key的作用
`canUpdate`的默认实现中以Widget的类型和key作为关键字进行判断。如果有对key定义，那么Key的一致性就会对widget的更新显得尤为关键。

这也是我们在做性能优化的时候需要注意的。可以利用Key的配置，来控制组件是否需要更新。
```Dart
static bool canUpdate(Widget oldWidget, Widget newWidget) {
  return oldWidget.runtimeType == newWidget.runtimeType
      && oldWidget.key == newWidget.key;
}
```
Key的几种子类基本上都是根据需求，对== 操作符做不同的实现。以更好的自定义 `canUpdate` 的结果。

其中`GlobalKey`比较特殊。作为全局的唯一秘钥。提供了对应 `widget` 的 `BuildContext` 和 `widget` 的访问方式。并针对 `StatefulWidget`。还提供了 `State` 的访问。

以便用户对状态进行全局的更新。比如我们需要在外部使用 `BuildContext` 进行初始化的时候，可以进行这样调用

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be1035a02d2b41aa912a3232336db000~tplv-k3u1fbpfcp-zoom-1.image)

## 4. 小结

通过以上对`StatelessWidget`和`ComponentElement` 的分析，可以得出以下的判断。

`StatelessWidget` 基于 `ComponentElement`。主要功能就是提供了组合各种`widget`的能力，并维持了祖孙的build传承。

当然在探索当中也发现了一些技术债务，由于我们已经知道了statelesswidget的使用场景，对于具体的源码细节先按下不表，在此只记录
- **生命周期_lifecycleState 起到什么作用**
- **_dirty 标记和 markNeedsBuild 的用法和原理是什么**
- **BuildOwner 的作用是什么**


# 二、探索StatefulWidget的动态刷新机制
`StatefulWidget` 和 `StateflessWidget` 有很多共同之处。最主要的原因就是他们创建的元素都是`ComponentElement`的子类，其提供了widget子孙build传承的能力。

可知`StatefulWidget`和`StateflessWidget`一样，也是一个有能力组合各种`widget`的组件。

## 1. State生命周期分析
`StatefulWidget` 定义了`createState`方法。提供了状态刷新能力。
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f6211c186ff4381a45b29c5b521bc88~tplv-k3u1fbpfcp-zoom-1.image)

再次从`StatefullElement`的`build`方法入手。直接调用了`state.build(this)`。代理了`state`的构建行为。

在`performRebuild`方法中也进行了`state.didChangeDependencies`生命周期回调。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/884aa5fcb4874cb39ff3e663612d608c~tplv-k3u1fbpfcp-zoom-1.image)

在State中，除了生命周期方法外， 最重要的就是build方法了。作用和`StatelessWidget`的build方法一致。都是提供了组合widget的能力。
initState则给用户提供了初始化state状态的机会。断点调试看看调用栈如何。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f5c366b2abd46d998a544954d88e5f7~tplv-k3u1fbpfcp-zoom-1.image)

调试中直观看到，在`firstBuld`的时候，`state`的`initState`被调用。并在之后调用了`didChangeDependencies`生命周期方法，和`build`方法。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4066b49073014de29785bcbd8a4390c3~tplv-k3u1fbpfcp-zoom-1.image)

代码中也对方法做了限制，不可以返回Future类型。
所以我们可以在initState中放心做一些初始化工作，没有异步参与，工作将会在build之前完成。

## 2. setState方法刷新页面方式分析
对于setState方法。除开生命周期的判断之外，关键代码只有一句，就是调用了element 的`markNeedsBuild()`
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9bb64d432ee5436cb78555231c806013~tplv-k3u1fbpfcp-zoom-1.image)

该方法将对应的element标记为dirty。并且调用`owner``!.scheduleBuildFor(``this``);`将其加入到 `BuildOwner`的脏列表(**_dirtyElements**)中。
将会在下次帧刷新的时候调用`BuildOwner.owner.buildScope` 重新构建该列表中的元素。

## 3. 小结
StatelessWidget给使用者提供了一个便捷的布局刷新入口，我们可以利用`setState`刷新布局。该方法会将对应`Element`标记为待刷新元素，在下次帧刷新的时候重建布局。状态的改动将会被重建的布局重新获取。

# 三、探索SingleChildRenderObjectWidget
`SingleChildRenderObjectWidget`对应的元素类是`SingleChildRenderObjectElement`。
我们作为开发者，布局过程中`SingleChildRenderObjectWidget` 的子类使用频率非常频繁，布局的约束，偏移和渲染都是由`RenderObjectWidget` 实现的，`SingleChildRenderObjectWidget`继承了`RenderObjectWidget`的渲染能力，并提供了单子传承的能力。布局的过程中该对象的子类不可或缺，flutter框架中也有不少对应的实现类。

`Flutter` 框架中实现的`SingleChildRenderObjectWidget`有以下几种。
1. SizedBox
1. LimitedBox
1. ShaderMask
1. RotatedBox
1. SizedOverflowBox
1. Padding
1. ...

## 1. 探索`SingleChildRenderObjectElement`中对于子widget的挂载和更新

```
SingleChildRenderObjectElement`的`mount` 和 `update`方法都很简单，都是直接调用了`updateChild`方法，传进去的子widget直接是`widget.child
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5e91a3a33994e70a525257d7c796523~tplv-k3u1fbpfcp-zoom-1.image)

这个方法和`ComponentElement`基本上一样，都是利用`canUpdate`的结果进行更新或者是创建子`Element`。

## 1. 以`Padding`为例了解`RenderObjectWidget` 的布局和绘制实现。

#### 名词解释
**RenderObject**：渲染对象，flutter对象布局的约束，绘制，位移全是由该对象实现，RenderObject树的祖孙中传递着约束，以做到布局大小的传承影响。


#### **RenderObject的创建**
`RenderObjectWidget` 会在`mount`挂载的时候，创建`RenderObject`，直接调用`widge.createRenderObject`。我们的约束，绘制，位移全是由`RenderObject`传递和实现的。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b38a9dac62648dcb6daf036341f0365~tplv-k3u1fbpfcp-zoom-1.image)

#### `RenderPadding`的布局实现
以`Padding`为例。`createRenderObject`创建了`RenderPadding`实例，widget的成员原封不动交给了该实例。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f08aee342e0d457dacdf8054bf974c52~tplv-k3u1fbpfcp-zoom-1.image)

约束(`BoxConstraint`)是`Flutter`确定布局大小的方案，各种`RenderObject`对于约束的传递都有自己的实现。

下方是`RenderPadding`的`performLayout`代码。红框标记起来的代码中就展示了`Padding`的约束传承逻辑。

其父布局传给自己约束基础上减去`Padding`再传递给子`RenderObject`。

观察`performLayout`方法可以发现，该方法完成了约束的传递，计算了偏移量Offset，并确定了自己的大小。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66b0fdb1ca304dc3ab33d298a3249dad~tplv-k3u1fbpfcp-zoom-1.image)

确定大小约束之后，就会在paint中绘制自己和子孙。RenderPadding没有自定义绘制，直接使用了父类`RenderShiftedBox`的实现。`RenderShiftedBox` 提供了offset偏移。在绘制子renderObject的时候，为其施加绘制偏移量。有些需要计算子布局偏移的`widget`，如`Padding`，`Align`等，都对`RenderShiftedBox`进行了实现。
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5bca97f72e548d0a775d574ae51d093~tplv-k3u1fbpfcp-zoom-1.image)

可以看到子布局的`offset`存在他的`parentData`中。`PaddingRender`使用的`parentData`是`BoxParentData`，内部提供了offset变量以供父布局使用。

```Dart
/// Parent data used by [RenderBox] and its subclasses.
class BoxParentData extends ParentData {
  /// The offset at which to paint the child in the parent's coordinate system.
  Offset offset = Offset.zero;
  @override
  String toString() => 'offset=$offset';
}
```

所有的`RenderBox`都持有BoxParentData对象，用于存储位移信息，在`setUpPrentData`的时候进行的初始化。红框中的代码展示了这一细节。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55232668f1214422995cf51299f7e4b1~tplv-k3u1fbpfcp-zoom-1.image)

到此，就能了解`RenderObject`是如何被约束`BoxConstraint`，如何被布局`layout`，以及如何被绘制`paint`。

##  1. `RenderObjectElement`的传承方式
`RenderObjectElement` 的父子传承在两个子类中实现，在第1小结中已经提到`SingleChildRenderObjectWidget` 和`ComponentElement`十分类似，只是直接把`widget.child`拿来传承，而不再提供`build`方法以供子组件组合。

`MultiChildRenderObjectElement` 也类似，只不过作为多子组件，三棵树分叉的主要因子，维护的是`children` 列表。
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/815abf1110ca4b94953101433b5705a1~tplv-k3u1fbpfcp-zoom-1.image)

在mount 和 update 的时候，子孙组件会像点了爆竹一样被逐一构建和更新。
## 1. 小结
每个`SingleChildRenderObjectWidget`组件都实现了各自的布局和绘制方案，也各自处理了约束并传递下去。

比如`ColordBox`作为绘制组件，借助了`RenderColord`，绘制了自身颜色，约束则取得是父约束的最小值。`Align`作为定位组件，借助了`RenderPositionedBox`，布局的时候计算了对应的偏移量offset，在绘制子布局的时候使用，约束则在传递的时候转了松约束。

诸如此类，所有组件都利用了对应的`RenderObject`满足了各自布局和渲染的所有需求。我们自己当然也可以自定义对应的`RenderObject`实现自己的布局。

`MultiChildRenderObjectWidget`  和 `SingleChildRenderObjectWidget`类似，只是维护一个子widget变成了多个子widget。

他的`RenderObject`基本上都是`ContainerRenderObjectMixin`和`RenderBox`的子类，内部维护了头尾两个子节点，并利用存储在`parentData`中的双相链表维护所有的子`RenderObject`。

# 四、谈谈ProxyWidget
最后稍微提一下`ProxyWidget`。`ProxyElement`也上`ComponentElement`的子类。和`StatefulWidget` 以及`StatelessWidget`是兄弟关系。也有子孙维系的能力，只不过他的build方法是固定的，返回的就是child。
![UML 图.jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c9c9b5cc1c845dea3b726c93addcf1e~tplv-k3u1fbpfcp-watermark.image?)

## 1. `InheritedWidget`
我们获取 Theme，MediaQuery数据的时候，都是使用了`InheritedWidget`
```Dart
MediaQuery.of(context).size.width;
Theme.of(context).appBarTheme;
```
通过`context` 也就是`Element`实例，获取祖先节点的数据。实现数据共享的效果。

`Element`中维护了祖先的所有`InheritedElement`映射，就可以在需要的时候直接通过子孙`Element`获取。

## 2. ParentDataWidget
`ParentDataWidget`提供了子组件向父组件传递渲染信息的能力。
`Flexible`，`Positioned` 等组件都是ParentDataWidget 的子类。

需要注意的是`：ParentDataWidget`只用于渲染信息的传递

在Element.attachRenderObject的时候会调用updateParentData，然后会辗转调用到对应的ParentDataWidget.applyParentData。可以看出只有子组件是RenderObjectWidget子类的时候才会应用对应的`ParentDataWidget`传递信息。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59a31e74d6234785ab00b07bdb690819~tplv-k3u1fbpfcp-zoom-1.image)

由此可知，只有在子节点渲染的时候，才会应用`RenderObject`的数据传递赋值。
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f831cfdd51e49e29b46a4f48a28723a~tplv-k3u1fbpfcp-zoom-1.image)

子节点的ParentData对象由父布局创建代码如下，创建时机在子节点插入的时候执行。
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86e5e289ccf4407aaa5eb40c2850a392~tplv-k3u1fbpfcp-zoom-1.image)

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d6d0c640acd45bf87bb7a36467f8de2~tplv-k3u1fbpfcp-zoom-1.image)

# 最后
作为开发者，很多时候完成一个任务只会建立在使用的层面。对于为什么这么使用往往不甚了解。

如果我们能更多的学习他的原理。那么如果在开发中碰到问题，我们能够更加得心应手得去解决。

flutter布局渲染的原理以前总是一层雾蒙在我地眼前。但现在，终于有一片薄雾散去，内部轮廓在我面前变得清晰。

坚持学习，见识真实的世界。

## 小试
我们最后尝试一下一个简单地布局，分析其三棵树结构。嵌套结构如下。其中`builder`是`StatelessWidget`。`Column`是 `MultiChildRenderObjectWidget`其他都是`SingleChildRenderObjectWidget`。

```Dart
void main() {
  runApp(Builder(builder: (context) {
    return Column(
      mainAxisSize: MainAxisSize.max,
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        Center(
          child: SizedBox(
            width: 100,
            height: 100,
            child: ColoredBox(color: Colors.blue),
          ),
        ),
        Expanded(
          child: ColoredBox(color: Colors.red),
        ),
      ],
    );
  }));
}
```
展示出来的样式如下。

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2043f25005d8468e99135af98cc9da68~tplv-k3u1fbpfcp-zoom-1.image" alt="img" style="zoom: 67%;" />

分析得出的三棵树如下，源头从`RenderView`而起，然后构建出`RenderObjectToWidgetAdapter`，再构建出`RootRenderObjectElement`。由此从根开始三棵树的循环，直到叶子节点。

`RenderObject`和`Widget`并非一一对应，只有`RenderObjjectWidget`才有，但是`RenderObject`能自动找出自己的组件`RenderObjject` 自动插入到其child中，所以也能自动成树。

![流程图.jpg](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b62bcbedf4945d4ab77358e733a593a~tplv-k3u1fbpfcp-watermark.image?)

至此，我们的`Widget`初步了解完结。

