# 异步

## 卡的定义-程序未响应

### 单线程

程序没有时间来刷新你的屏幕，正忙于处理别的事件，或者程序是按照顺序执行的，假如程序挂起了，例如sleep等那么程序就停住了

主要因素有两种

- 计算量很大，CPU无法以较快的速度处理数据(现CPU性能强大，平常几乎不会出现这种情况)

- 等待文件读取、服务器响应及用户输入等

### 多线程

在多线程的机制里，每当遇到需要等待的事件，那么就**单独派出去处理**，这样**主线程**就不会因为要**等待事件的发生而被挂起**，那么就不会发生卡顿了



## 事件循环机制(Event Loop)

### Dart的线程

Dart中本身是单线程的模型，但是它提供了Isolate(隔离/孤立)以便实现多线程-它们之间通信只能通过发消息的方式，且它们不能共享变量，好处一个线程的变量是不会被其它线程不小心修改的。



### 事件循环

上述讲解的Dart线程中Dart虽然有多线程，但大多数情况下不需要用到多线程，那如何使用一个线程就可以运行所有代码呢？这时需要用到一个叫做事件循环的机制，事件循环需要做的是一直不断的检查一个叫事件队列(Event queue)的东西，Dart开始执行完所有代码后就会自动退出，一般由main()方法开始运行

<img src="C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230418104111759.png" alt="image-20230418104111759" style="zoom:25%;" />

#### 事件队列(Event Queue)

![image-20230418110925923](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230418110925923.png)

<div align="center">Event Queue</div>

每当有异步操作时，就把相应的操作丢给事件队列，当主线程执行完后就会回来检查事件队列中的事件并执行





#### 微任务队列(Microtask Queue)

![image-20230418111457446](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230418111457446.png)

<div align="center">Microtask Queue</div>

这个微任务队列的级别比事件队列更高，Microtask Queue中的任何任务都会在Event Queue中的下一个任务之前运行，且Microtask Queue内只要还存在任务那么Event Queue就会等待Microtask Queue中的任务完成，通常由系统调用。

~~~dart
void main(){
    scheduleMicrotask(
    	() => print("A");
    )
	print("B");
}
~~~





#### 事件循环图

![image-20230418134147620](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230418134147620.png)



当程序运行时，事件循环会持续运行，直到两个队列中都没有任务可执行时才会停止。但实际上通常事件循环会一直运行，因为会持续的有事件进入队列，例如用户的交互操作、动画更新等。

如果真的没有新的事件进入队列，同时队列中的任务已经执行完毕，那么事件循环会处于空闲状态。在空闲状态下Flutter框架会监视系统事件，例如用户输入等，有新的事件进入队列时，Flutter会再次启动事件循环





![image-20230418134328002](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230418134328002.png)







## Future就像盒子里的巧克力

Future就好像盲盒，在没有打开的时候，你并不知道最终接收到的是什么值，而这个打开的动作就是**then**方法。在then中有 值 的回调

例句

~~~dart
Future<String> getFutureVal(){
    return Future(() => "val");
}
void test(){
   getFutureVa().then((value) => print(value));
}

//输出
//--val--
~~~







#### 小知识点

~~~dart
Future.delayed(Duration(seconds: 2), ()=> "Alice");
~~~

这段代码会被推送到Event Queue 中，是先推送到Event Queue中等待2秒后再执行的。





## 深入详解FutureBuilder组件

示例

~~~dart
FutureBuilder(
	future:Future.delayed(Duration(seconds :2), ()=> "321"),
    builder: (BuildContext context, AsyncSnapshot<dynamic> snapshot) {
    //如果Future任务正在执行，则返回一个旋转等待组件 表示当前任务正在执行
        if(snapshot.connectionState == ConnectionState.waiting){
            return CircularProgressIndicator();
        }
     //如果Future任务完成，则会返回 data 或者 error
        if(snapshot.connectionState == ConnectionState.done){
            return Text("${snapshot.data}");
        }
        
        throw "not have widget";
    }
)
~~~

<div align="center">简化写法</div>

![image-20230419152009133](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230419152009133.png)





Future完成后snapshot中必然有两个值，分别是data和error，这两个不能同时存在，可理解为：当Future完成时data会接收到值，而error必为null，反之亦然，当Future出错时，data为null，error有值。



### future

传入一个Future

### builder

传入一个builder方法，内有两个形参，分别为BuildContext和AsyncSnapshot，当Future发生变化后builder就会自动响应

### initialData

名称即作用，设置一个初始值，当Future并未完成时，为你的snapshot中的data设置一个初始值，但并不是给Future赋值(Future本身还是一个未完成的状态)，initialData的作用就是不一定需要用到 **CircularProgressIndicator** 组件来告知用户程序正在运行，直接用一个默认值来展示

### ConnectionState

- none

  异步任务还未执行，future的参数为null

- waiting

  异步任务正在执行中

- done

  异步任务已完成，future的参数为结果

- active

  在Stream流中，可以判断Stream流是否活跃 







## Stream与StreamBuilder

### 何为Stream

Stream就如它的字面意义，Future过段时间会返回一个值给你，那么Stream则是像小溪的流水一样源源不断的给你返回值





### Stream

默认情况下Stream只能被一个进程监听，如何实现被多个进程监听就需要用到 **broadcast**

~~~dart
StreamController.broadcast()
~~~

但是广播数据流有一个缺陷就是：当没有进程监听该Stream数据流的时候数据是不会被缓存的。

示例

~~~dart
final controller = Controller.broadcast();

当这段Future被开始执行前的5秒内，Stream流是不会被监听到的
Future.delayed(Duration(seconds :5),(){
    controller.stream.listen(){
        //省略以下代码
    }
})
~~~

当这段Future被开始执行前的5秒内，Stream流是不会被监听到的



### StreamController

在StreamController中有 sink 这个值，这个值就如字面意义一样好比水池，可以往里面添加事件，再使用Stream流把事件释放出来。

既然是水池，那必然有开关水池的操作



### 语法糖

可以使用 **async*关键字 **来表示这个方法是一个Stream数据流

再使用 **yield关键字** 来产出Stream数据流

示例

~~~dart
Stream<int> getTime() async* {
    int a= 1;
    while (true) {
      a++;
      await Future.delayed(Duration(seconds: 1));
      yield a;
    }
  }
~~~









### where-过滤

过滤的作用

~~~dart
where((event) => event is int);
~~~

这段代码意思是，当Stream流的数据为类型 int 时，该Stream数据会被过滤掉。



### distinct-去重

每当有Stream数据流返回时，StreamBuilder会被重建，但是遇到重复的数据时仍然重复进行StreamBuilder重建，那么会导致不必要的性能消耗，这时就需要distinct的去重功能，当遇到新的Stream数据流的值与上一个数据流的值重复时，StreamBuilder不会因此而重建，大大降低了性能消耗而不影响用户体验。







## 从用户触发事件流

### 游戏案例

在这里我们制作了一个类似 青蛙过河 的数字游戏

- 界面

  一个1~9按键的键盘和一个从顶部随机出现和随机速度下降然后从上往下的一组算术题，

- 游戏规则

  当屏幕中出现的算术题时，你点击的按键正好对应的算术题的答案时，该算术题就会被重置刷新然后重新出现一道新的算术题。



### 主要设计架构

##### 标题

使用`StreamBuilder`来监听Stream广播的事件，当键盘点击时会发送一条Stream流事件，然后AppBar的子组件Title就可以监听到 发送的Stream事件，实时的显示点击了哪些按键

##### Puzzle-算术题组件

- Puzzle组件接收一个Stream流

- initState方法里初始化数值
  - AnimationController
  - 初始化随机数-使用`Random().nextInt(5) + 1`
  - 初始化随机颜色-`Colors.primaries[Random().nextInt(Colors.primaries.length - 1)]`
  - 初始化出现的位置(即随机PaddingLeft的值)-仍然用到Random
  
  - 嵌套了AnimatedBuilder来监听 `AnimationController` 以便来处理动画事件
  
  - 再嵌套使用`Positioned`里面设置left属性的值，这个传入的值就是 `变量randomPaddingLeft`，这个变量在 `initState` 内就已经赋值了
  
  - 再次设置 `Positioned`里面top属性的值
  
    ```dart
    height * _animationController.value
    ```
  
    `_animationController.value`的值会随着动画而变化，通常值为 0~1 ，再与 `变量height` 相乘，就可以实现 Puzzle 从上而下降落的效果

##### 键盘-KeyPad

- 键盘KeyPad类是一个 `StatefulWidget` 类，它接收一个`StreamController`对象，通过监听该对象的 `Stream流` 实现通信

- 使用 `GridView.count` 生成一个网格列表并限制每行为3个格子，再使用 `List.generate` 生成9个 `MaterialButton` 组件，这样网格列表就会有3行3列，九宫格键盘就设计好了
- 在 `MaterialButton` 中的 `onPressed` 方法中设置了 一个给 `StreamController` 添加一个发送Stream流事件的方法(发送的是键盘按钮的值)



##### 逻辑架构

Puzzle状态管理类中使用初始化状态方法 `initState` 在这里进行逻辑处理

- 首先设置一个重置Puzzle的reset方法且需求为:
  - 刷新Puzzle动画时间即刷新Puzzle下落速度(动画时间越短那么下落速度就越快)
  - 刷新Puzzle组件中 左Padding 的值，即以随机的顶部横轴方位开始运动
  - 刷新Puzzle的颜色
  - 将Puzzle动画重新执行并以 `0.0` 的位置开始动画即从顶部开始运动



- 监听AnimationController

  如果当前Puzzle的动画执行完毕(即当前Puzzle已经下落到屏幕底部的位置那么就算未完成当前算术题)，就把当前Puzzle执行 `reset` 方法

- 未完成 获取屏幕宽高

- 监听接收的 `StreamController`

  当点击键盘按钮时发出的 `Stream流数据`  时，通过逻辑判断 点击的按钮的数字是否等于 算术题的答案 ，如果是则刷新该 Puzzle

















































# 动画

## 如何选择合适的动画控件

![image-20230422161041577](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230422161041577.png)





## 隐式(全自动)动画

### 1.简单的动画















### 2.在不同控件中切换的过渡动画









### 3.更多动画控件及曲线(Curves)

贝塞尔曲线参考网站[Curves class - animation library - Dart API (flutter-io.cn)](https://api.flutter-io.cn/flutter/animation/Curves-class.html)

















## 显式(手动控制)动画

## 其他动画





## 动画的背后机制和原理

### 隐式-全自动动画

隐式动画都继承了一个名为 `ImplicitlyAnimatedWidget` 的类中文名为`隐式动画组件`，而这个`ImplicitlyAnimatedWidget`又继承自`StatefulWidget`，既然是继承自有状态组件，那么就可以去`State`类中查看它做了些什么

<img src="C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230426203331802.png" alt="image-20230426203331802" style="zoom:80%;" />

可以看到它也是用到了`AnimationController` ，这里面很多操作它自动帮我们完成了所以称为全自动动画



### 显示-手动动画

显示动画都继承一个名为 `AnimatedWidget`的类，这个类接收了 类型为`Listenable`的名为`animation`的参数

<img src="C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230426203830631.png" alt="image-20230426203830631" style="zoom: 67%;" />

在深入了解查看它继承的`AnimatedWidget`类也是将顶部传入的 类型为`Listenable`的名为`animation`的参数添加监听事件

<img src="C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230426204329243.png" alt="image-20230426204329243" style="zoom: 67%;" />

![image-20230426204426401](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230426204426401.png)

而这个 `_handleChange`  方法只执行了一个功能

![image-20230426204508040](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230426204508040.png)

那就是 `setState` ，无论是隐式还是显示动画，终归还是靠监听动画控制器，当动画的数值发生变化，那么就直接调用`setState`，对整个控件重新渲染。因为Flutter引擎对渲染控件优化到了极致，即使1秒60次或者120次的`setState`都没有关系













































































## 番外篇

### Hero动画-主动画

Hero动画的主要用途是当页面进行切换或者布局发生变化时，相同的两个组件的变化不会太生硬，会有一个流畅的过渡效果，操作是给目标组件嵌套一个`Hero`的组件并设置标签值`tag`，这两个组件的`tag`一定要相同不然效果不佳，























# Key

## 如果没有Key会发生什么

如果没有`key`那么flutter就分不清相同组件中的具体是哪个组件了，假设有两个`Container`，即使里面的参数不一样，但是没有key，flutter也还是分不清这两个组件





## Widget和Element的对应关系

在Flutter中，`Widget`是UI元素的抽象表示，每个`Widget`描述了UI元素的特定方面，如它的外观、大小、位置、动画和响应事件。而`Element`则是`Widget`的具体实现，用于在渲染树中创建、更新和管理`Widget`的状态。每个`Element`代表了一个具体的UI元素实例，并且具有自己的生命周期和状态。

当一个`Widget`在Flutter的渲染树中被创建时，它会被关联到一个`Element`实例上。这个`Element`实例会保存`Widget`的配置信息以及其它状态信息、例如布局信息、样式等等。当`Widget`的状态发生变化时，`Element`会接受到通知，再根据新的配置项重新构建UI元素，并将新旧元素进行比较，以确定是否需要更新现有的UI元素。



通俗来讲`Widget`就是`Element`的配置项

![image-20230428112811349](C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230428112811349.png)















## 通过Key获取元素的属性



### 通过Key获取元素的宽高或者位移值

- 获取实际渲染的UI元素

  ```dart
  var renderObject = key.currentContext.finRenderObject() as RenderBox;
  ```

- 获取宽高

  ```dart
  renderObject.size.height;
  renderObject.size.width;
  ```

  



# 滚动

## 滚动列表与动态加载

### ListView

`ListView`是一个用于显示垂直或水平的滚动列表。

#### 需要注意的细节

- 当子组件数量有限时，可以放心使用`ListView`
- 当子组件数量较大或者无限大时，建议使用`ListView.builder`来动态构建子组件。

#### 与其它组件不同处

相对于`Column`和`Row`等组件，`ListView`具有以下特点

- 可以自动滚动显示列表内元素 
- 可以显示任意数量的子组件，而不会导致布局溢出的问题
- 可以使用`ScrollController`控制滚动位置



### ListView.builder

`ListView.builder`是一种优化的ListView，它可以动态构建子组件以避免不必要的渲染和内存消耗。它接收一个回调函数`itemBuilder`，该回调函数会在每次需要显示一个子组件时被调用。该回调函数会根据`索引`返回相应的子组件。

#### 与ListView不同处

`ListView.builder`会划分一块区域，这个区域是`屏幕显示的区域`以及`靠近屏幕显示区域一部分区域`，`ListView.builder`会先渲染这个区域内的子组件，大大降低了性能消耗

- 可以动态构建子组件，避免不必要的渲染和内存消耗
- 可以提高列表性能，特别是在大量子组件需要被渲染时





### ListView.separated(带有分隔符的ListView.builder)

`ListView.separated`是一个用于显示垂直列表的组件，它可以在相邻子组件之间添加分隔符。可以用于显示一些具有相同外观和行为的子组件，例如文本行、图像等，这些子组件可以被隔开并嗳垂直方向上滚动





### 控制器

#### ScrollController









### 下拉刷新

#### RefreshIndicator



























## 有趣的列表组件





### ListWheelScrollView

#### 组件属性

- offAxisFraction

  表示车轮水平偏离中心的程度

- useMagnifier

  是否启用放大镜效果

- manification

  放大镜的放大倍率，配合着`useMagnifier`使用

- squeeze-显示比例

  表示车轮上的子控件数量与在同等大小的平面列表上的子控件数量之比，例如，如果高度为100px，[itemExtent]为20px，那么5个项将放在一个等效的平面列表中。当squeeze为1时，RenderListWheelViewport中也会显示5个子控件。当squeeze为2时，RenderListWheelViewport中将显示10个子控件，默认值为1

- perspective-视角

  表示车轮的圆润程度，值为 `0-0.01`之间

- physics-物理属性

  可以设置和ListView一样的物理效果，可以通过设置`FixedExtentScrollPhysics()`来保证每次滚动时一定会停留在某个子组件上，一般用来设计闹钟滚轮

- overAndUnderCenterOpacity-其余组件透明度

  除去中间部分组件外的子组件透明度

- onSelectedItemChanged-回调方法

  监听哪些控件经过中间区域

#### 代码示例

```dart
RotatedBox(
        quarterTurns: 1,
        child: ListWheelScrollView(
          itemExtent: 100,
          physics: FixedExtentScrollPhysics(),
          useMagnifier: true,
          overAndUnderCenterOpacity: .5,
          onSelectedItemChanged: (e){
            print("$e");
          },
          children: List.generate(
            10,
            (index) => Container(
              width: double.infinity,
              alignment: Alignment.center,
              child: RotatedBox(
                  quarterTurns: -1,
                  child: Text("${index + 1}",style: TextStyle(fontSize: 72),)),
            ),
          ),
        ),
      ),
```





### PageView

#### 组件属性

- pageSnapping

  确认是否一定要停留在某个字控件上









































# 布局

## 约束、尺寸、位置 

<img src="C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230510105534898.png" alt="image-20230510105534898" style="zoom:67%;" />







## Flex弹性布局

### 紧约束

有固定尺寸的布局

### 松约束

没有固定尺寸的布局，一般是弹性布局，根据剩余空间自由灵活的控制自己的尺寸

### 布局原理

Flex布局分为两大类，一类是有弹性的一类是没有弹性的，Flex布局的时候会先布局紧约束的组件，由紧约束的组件分配完空间后，松约束(弹性布局组件)的组件再根据剩余空间来分配自己的空间



## Column&Row

Column和Row在向下传递约束的时候，会将自己主轴的约束设置为无限长，所以很容易发生溢出的问题，但是合理的布局就可以避免这个情况，

### 参数

- mainAxisAlignment

  - start

  - end

  - center

  - spaceBetween

    在子组件之间均匀的分布自由空间

    <img src="C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230510161953711.png" alt="image-20230510161953711" style="zoom:33%;" />

  - spaceAround

    将剩余空间均匀地放置在子组件之间，并将该剩余空间的一半放置在第一个和最后一个子组件的前后

    <img src="C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230510162108052.png" alt="image-20230510162108052" style="zoom:33%;" />

  - spaceEvenly

    在子组件之间以及在第一个和最后一个子组件的前后均匀地放置空闲空间

    <img src="C:\Users\27822\AppData\Roaming\Typora\typora-user-images\image-20230510162217673.png" alt="image-20230510162217673" style="zoom:33%;" />











## Stack



### Stack布局原理

- 有位置

- 无位置

  由Position嵌套的组件，Position会接收传递给它的位置信息

Stack布局和Flex布局原理很相似，也分为两大类来决定如何布局，一类是有位置的，一类是没有位置的。

先把没有位置的布局好，之后再把Stack自己的尺寸调整为`没有位置的组件中最大的那一个的尺寸`









## CustomMultiChildLayout-自定义多子组件布局

使用`CustomMultiChildLayout`这个组件可以在没有合适的布局组件可以选择时，我们自己设计一个布局

需要创建一个类继承自`MultiChildLayoutDelegate`



#### 简易的文字下划线效果

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        // This is the theme of your application.
        //
        // Try running your application with "flutter run". You'll see the
        // application has a blue toolbar. Then, without quitting the app, try
        // changing the primarySwatch below to Colors.green and then invoke
        // "hot reload" (press "r" in the console where you ran "flutter run",
        // or simply save your changes to "hot reload" in a Flutter IDE).
        // Notice that the counter didn't reset back to zero; the application
        // is not restarted.
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(),
    );
  }
}

class MyHomePage extends StatefulWidget {
  const MyHomePage({Key? key}) : super(key: key);

  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(),
      body: CustomMultiChildLayout(
        delegate: MyDelegate(),
        children: [

          LayoutId(
            id: "underline",
            child: Container(
              color: Colors.red,
            ),
          ),LayoutId(
            id: "text",
            child: Text("WSBT",style: TextStyle(fontSize: 72),),
          ),
        ],
      ),
    );
  }
}

class MyDelegate extends MultiChildLayoutDelegate {
  static const firstId = "firstId";
  static const secondId = "secondId";

  @override
  void performLayout(Size size) {
    final sizeText = layoutChild(
      "text",
      BoxConstraints.loose(size),
    );
    final textCenterw=(size.width -sizeText.width)/2;
    final textCenterh=(size.height -sizeText.height)/2;
    positionChild("text", Offset(textCenterw, textCenterh));
    final sizeUnderline = layoutChild(
      "underline",
      BoxConstraints.tight(
        Size(sizeText.width, 4),
      ),
    );
    positionChild("underline", Offset(textCenterw, textCenterh+sizeText.height),);
  }

  @override
  bool shouldRelayout(covariant MultiChildLayoutDelegate oldDelegate) {
    return true;
  }
}
```















## 自定义RenderObject

### 步骤

1. 创建一个类并继承自`SingleChildRenderObjectWidget`的widget类，它的`createRenderObject`方法返回一个自定义的RenderObject对象。在这个自定义RenderObject对象中，需要重写`performLayout`和`paint`方法
2. 创建一个继承自`RenderBox`的RenderObject对象，并使用`RenderObjectWithChildMixin`混入。`RenderObjectChildMixin`会帮助我们处理子组件的布局
3. 在`RenderObject`对象中实现`performLayout`方法，进行布局计算。
4. 在`RenderObject`对象中实现`paint`方法，进行渲染绘制。使用`context.paintChild`方法来渲染子组件



### 要点

#### 使用子组件尺寸

如果要使用child的尺寸，需要在`layout`方法中加上`parentUsesSize=true`，这样在布局时，就会考虑子组件的尺寸。

```dart
size=(child as RenderBox).size;
```

因为Flutter中布局是从上向下传递约束，从下往上传递尺寸，如果一个组件已经确定了尺寸，就不需要知道其子组件的尺寸，可以将该组件作为分界点，避免不必要的性能开销。当子组件的尺寸发生变化时，只有该组件及其祖先组件会重新布局。





#### performLayout和paint的关系



<div align="center">它们之间没关系</div>



performLayout是负责计算布局的，而paint是负责把组件渲染在屏幕上面的

即使performLayout把组件的布局计算好了，把组件放在相应的位置上，那如果paint方法不去把组件渲染出来，那么屏幕上也是看不到任何东西的，但是组件仍然占有位置。





### 给自定义RenderObject传递值

需要使用`updateRenderObject`方法

```dart
@override
void updateRenderObject(BuildContext context, convariant RenderObject renderObject) {
    (renderObject as *继承自RenderBox的具体布局渲染组件的类*).被传递的值=传递的值;
}
```









### 例子

```dart
class ShadowBox extends SingleChildRenderObjectWidget {

  double distance;

  ShadowBox({
    required Widget child,
    required double distance,
  }) :distance=distance,super(child: child);

  @override
  RenderObject createRenderObject(BuildContext context) {
    return RenderShadowBox(distance);
  }

  @override
  void updateRenderObject(BuildContext context, covariant RenderObject renderObject) {
    (renderObject as RenderShadowBox).distance=distance;
  }
}

class RenderShadowBox extends RenderBox with RenderObjectWithChildMixin {
  double distance = 8.0;

  RenderShadowBox(this.distance);

  @override
  void performLayout() {
    child!.layout(constraints, parentUsesSize: true);
    size = (child as RenderBox).size;
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    context.paintChild(child!, offset + Offset(20, 20));
    //
    context.pushOpacity(offset + Offset(20, 20), 127, (context, offset) {
      context.paintChild(child!, offset + Offset(distance, distance));
    });
  }
}
```













# Slivers











### SliverPrototypeExtentList

通过`prototypeItem`传入一个组件，然后会自动的获取这个组件的尺寸并应用在每一个列表下的子组件，比设置itemExtent属性要方便很多，因为设置itemExtent属性你并不知道每个组件具体会占用多大的尺寸



### SliverFillViewPort

类似`ListViewPage`





































# Provider



















# 误区

在Java中，异步操作是多线程，但是在dart中，异步操作并不是多线程



