# Flutter UI

## UI 三大元素：Widget、Element、RenderObject

对应于这三个元素有三个owner负责管理它们，分别为WidgetOwner、BuildOwner、PipeLineOwner。

 1. **Widget** 是immutable的，存放渲染内容、视图布局信息，是廉价的纯对象。Widget可以在同一个Widget Tree中复用很多次，因为它只是个description。

 2. **Element** 是mutable的，存放上下文。Element同时持有Widget和RenderObject的实例，是树中特定位置Widget的实例。虽然Widget可以重复使用，但是Flutter Framework仍然会对相同的Widget依次创建全新的Element。一次渲染会生成一个Element Tree并放在内存中，在下次渲染时Flutter Framework 会尝试用新的Widget去更新旧的Element。Element的任务概括了说就是保存了一个树形结构以便更新时做diff、patch和管理组件生命周期，由于widget没有状态，如果想修改widget的某一属性就必须重新创建一遍widget，而StatefuleWidget中包含state，如果重新创建就会丢失，所以需要Element存储这些需要暂存的状态，然后Widget Tree发生变化时，Flutter Framework通过Element的update来根据新widget更新旧Element.我们不需要手动去实现Element。
 3. **RenderObject** 负责视图渲染，实际上所有的布局、绘制和事件响应全部由它负责。RenderObject是由Element子类RenderObjectElement创建出来的，RenderObject会构成一个Render Tree，并且每个RenderObject也都会被保存下来以便在更新时复用  

## **Flutter的刷新流程**

### **一、Element的复用**

虽然createRenderObject方法的实现是在Widget中，但持有RenderObject引用的是Element。

从Element的源码可以看到

```dart
abstract class RenderObjectElement extends Element {
  ...
  @override
  RenderObjectWidget get widget => super.widget;

  @override
  RenderObject get renderObject => _renderObject;
  RenderObject _renderObject;
}
```

Element同时持有两者实例

在StatefulWidget中状态变化想刷新界面时，需要调用setState()方法

```dart
@protected
void setState(VoidCallback fn) {
    ...
    _element.markNeedsBuild();
}
```

第一步：**Element标记自身为dirty=true,并调用owner.scheduleBuildFor(this);通知buildOwner进行处理**

第二步：**buildOwner会将所有dirty的element添加到集合_dirtyElements中，并调用ui.window.scheduleFrame();通知底层渲染引擎安排新的一帧处理**

第三步：**底层引擎最终回调到Dart层，并执行buildOwner的buildScope方法**

```dart
void buildScope(Element context, [VoidCallback callback]) {
         ...
    try {
         ...
        /*
        1.排序---按照Element的深度从小到大，对_dirtyElements进行排序，避免子Widget的重复build。
        */
      _dirtyElements.sort(Element._sort);
        ...
      int dirtyCount = _dirtyElements.length;
      int index = 0;
      while (index < dirtyCount) {
        try {
        /*
        2.遍历执行_dirtyElements当中element的rebuild方法，最终调用performRebuild()
        */
          _dirtyElements[index].rebuild();
        } catch (e, stack) {
        }
        index += 1;
      }
    } finally {
      for (Element element in _dirtyElements) {
        element._inDirtyList = false;
      }
      //3.遍历结束之后，清空dirtyElements集合
      _dirtyElements.clear();
       ...
    }
  }
```

buildScope方法需要传入一个Element的参数，以对这个Element以下(包含)的范围rebuild。

第四步：**执行performRebuild()**

performRebuild()不同的Element有不同的实现，我们暂时只看最常用的两个Element：

>* ComponentElement，是StatefulWidget和StatelessElement的父类
>
>* RenderObjectElement， 是有渲染功能的Element的父类

1.**ComponentElement的performRebuild()**

```dart
void performRebuild() {
    Widget built;
    try {
      built = build();
    }
    ...
    try {
      _child = updateChild(_child, built, slot);
    }
    ...
  }

```

先执行element的build(),如StatefulElement中的build()为Widget build() => state.build(this);就是执行了StatefulWidget中复写的build方法

```dart
Element updateChild(Element child, Widget newWidget, dynamic newSlot) {
  ...
  //1.如果刚build出来的widget等于null，说明这个控件被删除了，child Element可以被删除了。
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }

    if (child != null) {
    //2.如果child的widget和新build出来的一样（Widget复用了），就看下位置一样不，不一样就更新下，一样就直接return了。Element还是旧的Element
      if (child.widget == newWidget) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        return child;
      }
      //3.看下Widget是否可以update，Widget.canUpdate的逻辑是判断key值和运行时类型是否相等。如果满足条件的话，就更新，并返回。
      if (Widget.canUpdate(child.widget, newWidget)) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        child.update(newWidget);
        return child;
      }
      deactivateChild(child);
    }
    //4.如果上述三个条件都没有满足的话，就调用 inflateWidget() 创建新的Element
    return inflateWidget(newWidget, newSlot);
  }
```

再看下inflateWidget

```dart
Element inflateWidget(Widget newWidget, dynamic newSlot) {
    final Key key = newWidget.key;
    if (key is GlobalKey) {
      final Element newChild = _retakeInactiveElement(key, newWidget);
      if (newChild != null) {
        newChild._activateWithParent(this, newSlot);
        final Element updatedChild = updateChild(newChild, newWidget, newSlot);
        return updatedChild;
      }
    }
    final Element newChild = newWidget.createElement();
    newChild.mount(this, newSlot);
    return newChild;
  }
```

从上面源码可以看到，方法内首先会尝试通过GlobalKey去查找可复用的Element，复用失败就调用Widget的方法创建新的Element，然后调用mount方法，将自己挂载到父Element上去，mount之前我们也讲过，会在这个方法里创建新的RenderObject。

2.**RenderObjectElement的performRebuild()**

```dart
@override
  void performRebuild() {
    widget.updateRenderObject(this, renderObject);
    _dirty = false;
  }
```

与ComponentElement的不同之处在于，没有去build，而是调用了updateRenderObject方法更新RenderObject。

靠以上方法便使得Element在中间能应对Widget的多变，保障RenderObject的相对不变

### **二、PipelineOwner对RenderObject的管理**

在底层引擎最终回到Dart层，最终会执行WidgetsBinding 的drawFrame ()

### **三、清理**

drawFrame方法在最后执行buildOwner.finalizeTree();

废弃element之后，调用inflateWidget创建新的element时，还调用了_retakeInactiveElement尝试通过GlobalKey复用element，此时的复用池也是在_inactiveElements当中。