# 1.ViewRoot

* 定义：连接器，对应于`ViewRootImpl`类。实际上不属于View的范畴
* 作用：
  1. 连接`WindowManager` 和 `DecorView`
  2. 完成`View`的三大绘制流程： `measure`、`layout`、`draw`
* 具体描述：
  1. 与WindowManagerService交互通讯，调整窗口大小及布局
  2. 向DecorView派发输入事件
  3. 完成三大绘制流程



# 2.DecorView

* 定义：顶层View，即Android视图树的根节点；同事也是FrameLayout的子类，真正意义上的ViewRoot
* 作用：显示 & 加载布局。View层的事件都先经过DecorView，再传递到View
* 特别说明：内含1个竖直方向的LinearLayout，分为2部分：
  1. 上 = 标题栏（titlebar）
  2. 下 = 内容栏（content）

> 在`Activity`中通过 `setContentView()`所设置的布局文件其实是被加到内容栏之中的，成为其唯一子View = id为content的`FrameLayout`中



# 3.Window

* 定义：承载器
* 作用：承载着视图View的显示
* 具体描述：
  1. Window类=抽象类、实现类=PhoneWindow类
  2. PhoneWindow类中有个内部类DecorView=View的根布局
  3. 通过创建DecorView来加载Activity中设置的布局
* 特别注意：
  1. Window类通过WindowManager将DecorView加载其中
  2. 将DecorView交给ViewRoot进行视图绘制和其他交互



# 4.Activity

* 定义：控制器
* 作用：控制生命周期和处理事件
* 具体描述：
  1. 统筹视图的添加和展示
  2. 通过其他回调方法与Window、View交互
* 特别注意：
  1. 不负责视图控制
  2. 真正控制视图的是Window，是真正代表一个窗口
  3. 一个Acitivity包含一个Window



# 5.之间的关系

![relation](img/view_relation.png?raw=true)

