# 第二十章 自定义交互与Widget (Custom Interaction and Widgets)

## 本章导读

在前面的章节中，你通过`vtkRenderWindowInteractor`和`vtkInteractorStyleTrackballCamera`体验了VTK的默认交互能力——鼠标拖拽旋转、缩放、平移场景。这是科学可视化交互的基础层，但对于实际应用而言，你几乎总是需要更丰富的交互方式：

- 在三维场景中放置一个包围盒来选择感兴趣的区域（Region of Interest）。
- 交互式地测量三维空间中两点之间的距离。
- 用一个可拖拽的平面来实时裁剪数据，观察内部结构。
- 在屏幕上放置滑块来控制参数（如等值面的值、时间步）。
- 自定义鼠标和键盘的行为——例如在点击位置打印世界坐标，或切换"测量模式"与"观察模式"。

VTK通过两套互补的系统来满足这些需求：

1. **Widget系统**：提供了一组丰富的"三维控件"——可以在场景中拖拽、旋转、缩放的交互式实体。Widget遵循"Representation（视觉表现）+ Widget（行为逻辑）"的分离设计。
2. **InteractorStyle系统**：允许你继承并覆写默认的交互样式，从而自定义场景对鼠标和键盘事件的响应方式。

本章将系统性地讲解这两套系统，从Widget框架的原理开始，逐一介绍六个最常用的3D Widget，深入Observer回调模式，最后通过一个完整的"交互式裁剪工具"程序将所有概念整合起来。

**前置知识**：本章需要第5章（渲染与交互机）的知识。如果你对Actor/Renderer/Interactor的关系或事件循环机制还不清楚，建议先回顾第5章。

---

## 20.1 Widget系统概览

### 20.1.1 两代Widget架构

VTK的Widget系统经历了两个主要的设计世代。理解它们的区别有助于你在实际开发中选择合适的工具。

**第一代Widget（Legacy Widget）**：以`vtk3DWidget`为基类。这一代Widget将视觉表现和事件处理逻辑合并在一个类中。代表性类包括：
- `vtkBoxWidget` —— 可操纵的正交六面体
- `vtkLineWidget` —— 交互式线段
- `vtkSphereWidget` —— 球形区域
- `vtkImplicitPlaneWidget` —— 交互式隐式平面
- `vtkDistanceWidget` —— 距离测量

第一代Widget的使用模式较为简单直接：创建Widget实例，调用`SetInteractor()`和`SetDefaultRenderer()`绑定到渲染环境，然后调用`On()`激活。但它们的缺点是视觉表现与行为逻辑耦合在一起，难以自定义外观。

**第二代Widget（New Widgets）**：以`vtkAbstractWidget`为基类。这一代Widget引入了关键的设计分离：

```
vtkAbstractWidget (行为逻辑)
    |
    +-- 持有 vtkWidgetRepresentation (视觉表现)
              |
              +-- 是一个 vtkProp，可添加到 vtkRenderer
```

代表性子类包括：
- `vtkBoxWidget2` —— 第二代包围盒Widget
- `vtkSliderWidget` —— 滑块Widget
- `vtkHandleWidget` —— 通用操纵点Widget

第二代Widget的核心优势在于：你可以独立地替换Representation（视觉表现）而不改变交互逻辑。例如，你可以给`vtkSliderWidget`提供不同的Representation来实现水平滑块、垂直滑块、圆形滑块等不同的外观，而行为代码完全复用。

在本教程中，我们主要使用第一代Widget——因为它们更直接、API更简洁，且对于科学可视化中的绝大多数用例而言完全足够。理解它们的原理后，迁移到第二代Widget将是顺畅的。

### 20.1.2 Widget的组成：Representation + 行为逻辑

无论哪一代Widget，一个Widget在概念上由两部分组成：

```
+-- Widget（整体控件）----------------------------------+
|                                                        |
|  +-- Representation（视觉表现）--+   +-- 行为逻辑 -----+ |
|  |                                |   |                 | |
|  |  - 可拖拽的手柄（球体/箭头）   |   |  - 鼠标按下响应 | |
|  |  - 线框/轮廓/透明面片         |   |  - 鼠标移动响应 | |
|  |  - 文字标注                    |   |  - 鼠标释放响应 | |
|  |  - 颜色和材质属性              |   |  - 键盘事件响应 | |
|  +--------------------------------+   +-----------------+ |
+----------------------------------------------------------+
```

**Representation（视觉表现）**：回答"这个Widget长什么样"的问题。Representation是添加到Renderer中的一组`vtkProp`对象（通常是`vtkActor`）。它包括：
- 手柄（Handles）：小的可拾取几何体（通常是球体），用户通过拖拽它们来操纵Widget。
- 轮廓（Outline）：包围盒或边界线，帮助用户感知Widget在三维空间中的位置和范围。
- 辅助几何体：例如法线箭头、距离标注线、平面填充面片等。

**行为逻辑**：回答"这个Widget如何响应交互"的问题。它负责：
- 拦截并处理来自`vtkRenderWindowInteractor`的鼠标和键盘事件。
- 判断事件发生的位置——是否点击了某个手柄？是否在Widget轮廓内部？
- 根据用户的拖拽动作实时更新Representation的几何位置。
- 发出事件通知（StartInteractionEvent、InteractionEvent、EndInteractionEvent），让外部代码能够在Widget状态改变时获得回调。

### 20.1.3 Widget的放置与激活

所有Widget都继承自`vtkInteractorObserver`（第一代Widget通过`vtk3DWidget`间接继承）。这意味着它们共享一套统一的激活和放置机制。

**核心API模式**：

```cpp
// 1. 创建Widget实例
vtkNew<vtkBoxWidget> boxWidget;

// 2. 绑定到交互器和渲染器
boxWidget->SetInteractor(interactor);
boxWidget->SetDefaultRenderer(renderer);

// 3. 设置初始位置（使用数据集的包围盒）
double bounds[6];
dataset->GetBounds(bounds);
boxWidget->PlaceWidget(bounds);

// 4. 激活Widget
boxWidget->On();

// 5. 当不再需要时，关闭Widget
boxWidget->Off();
```

**关键方法说明**：

| 方法 | 说明 |
|------|------|
| `SetInteractor(interactor)` | 将Widget绑定到一个`vtkRenderWindowInteractor`，使其能接收该交互器的事件 |
| `SetDefaultRenderer(renderer)` | 指定Widget默认添加到哪个Renderer中 |
| `PlaceWidget(bounds)` | 设置Widget的初始位置。`bounds`是一个六元数组`[xmin, xmax, ymin, ymax, zmin, zmax]` |
| `On()` | 激活Widget——使其可见、开始响应事件 |
| `Off()` | 关闭Widget——隐藏并停止响应事件 |
| `SetEnabled(1)` / `SetEnabled(0)` | 与`On()`/`Off()`等效 |
| `GetEnabled()` | 查询Widget当前是否处于激活状态 |

**PlaceWidget()的多种重载**：

```cpp
// 使用六元数组
double bounds[6] = {-1.0, 1.0, -1.0, 1.0, -1.0, 1.0};
widget->PlaceWidget(bounds);

// 使用六个独立的参数
widget->PlaceWidget(-1.0, 1.0, -1.0, 1.0, -1.0, 1.0);

// 不使用参数——Widget将尝试从当前绑定的数据集中获取包围盒
widget->PlaceWidget();
```

### 20.1.4 Widget的事件处理流程

理解Widget如何与Interactor和InteractorStyle协作处理事件，是正确使用Widget的关键。

**事件分发顺序**：

```
用户操作（鼠标移动/按键）
    |
    v
vtkRenderWindowInteractor 捕获原始事件
    |
    v
优先级1: 当前激活的Widget（通过vtk3DWidget/AbstractWidget注册的Observer）
    |-- 如果Widget处理了该事件（例如点击在Widget的手柄上），事件被消费
    |
    v (Widget未处理)
优先级2: 当前激活的InteractorStyle（如TrackballCamera）
    |-- 处理标准的场景导航操作（旋转、缩放、平移）
    |
    v (Style也未处理)
事件被丢弃
```

关键原则：**Widget优先于InteractorStyle**。当你在Widget上操作时（例如拖拽包围盒的一个面），TrackballCamera的旋转操作不会同时触发。只有当你点击Widget之外的区域时，标准的场景导航才生效。

这使得Widget可以无缝地融入现有的交互环境——你不需要在Widget操作和场景导航之间做显式的模式切换（对于简单的Widget使用场景而言）。

**Widget内部的事件处理链**：

在Widget内部，事件经过以下步骤被处理：

1. **事件翻译（Event Translation）**：原始VTK事件（如LeftButtonPressEvent）被翻译为Widget事件（如Select、Move、Scale）。`vtkWidgetEventTranslator`负责这一映射。
2. **回调映射（Callback Mapping）**：根据当前Widget状态和Widget事件类型，`vtkWidgetCallbackMapper`查找并调用对应的处理方法。
3. **状态更新**：处理方法更新Widget的内部状态（位置、朝向、大小）。
4. **Representation更新**：根据新状态更新视觉表现几何体。
5. **事件发出**：Widget在自身（通过`InvokeEvent`）发出`vtkCommand::InteractionEvent`等通知。

---

## 20.2 常用3D Widget

本节逐一介绍六个最常用的VTK 3D Widget。对每个Widget，我们将说明其用途、核心API、并提供一个简短的使用示例。

### 20.2.1 vtkBoxWidget —— 六面体包围盒

**用途**：在三维场景中放置一个可交互的正交六面体包围盒。这是最通用的3D Widget之一，适用于：
- 区域选择（Region of Interest, ROI）：框选数据集的子区域。
- 对象变换：获取包围盒的变换矩阵，用于移动/旋转/缩放其他Actor。
- 裁剪/切割：通过`GetPlanes()`获取六个平面，传递给裁剪过滤器。

**交互方式**：
- 左键拖拽六个面的手柄：移动该面（改变包围盒大小）。
- 左键拖拽中心手柄：平移整个包围盒。
- 左键点击面（非手柄区域）并拖拽：旋转包围盒。
- 右键在窗口中上下移动：缩放包围盒。

**核心API**：

```cpp
#include <vtkBoxWidget.h>

vtkNew<vtkBoxWidget> boxWidget;
boxWidget->SetInteractor(interactor);
boxWidget->SetDefaultRenderer(renderer);
boxWidget->PlaceWidget(bounds);

// 获取隐式平面（用于裁剪/切割）
vtkNew<vtkPlanes> planes;
boxWidget->GetPlanes(planes);

// 获取变换矩阵（用于对象变换）
vtkNew<vtkTransform> transform;
boxWidget->GetTransform(transform);

// 获取几何数据（15个关键点 + 6个四边形面）
vtkNew<vtkPolyData> polyData;
boxWidget->GetPolyData(polyData);

// 控制允许的操作类型
boxWidget->TranslationEnabledOn();
boxWidget->ScalingEnabledOn();
boxWidget->RotationEnabledOn();

// 控制法线方向
boxWidget->SetInsideOut(0);    // 法线指向外部（默认）
boxWidget->SetInsideOut(1);    // 法线指向内部

// 外观属性
boxWidget->GetHandleProperty()->SetColor(1, 0, 0);         // 手柄颜色：红色
boxWidget->GetFaceProperty()->SetOpacity(0.3);              // 面半透明
boxWidget->GetOutlineProperty()->SetColor(1, 1, 1);         // 轮廓颜色：白色

boxWidget->On();
```

### 20.2.2 vtkDistanceWidget —— 三维距离测量

**用途**：在三维空间中测量两点之间的直线距离。适用于：
- 测量模型上的特征尺寸。
- 交互式地检查两个空间位置之间的距离。
- 配合Observer回调，在其他UI中实时显示距离数值。

**交互方式**：
- 第一次左键点击：放置第一个测量点。
- 第二次左键点击：放置第二个测量点（此时Widget进入Manipulate模式）。
- 在Manipulate模式下，拖拽端点手柄来调整测量点的位置。

**核心API**：

```cpp
#include <vtkDistanceWidget.h>
#include <vtkDistanceRepresentation.h>

vtkNew<vtkDistanceWidget> distanceWidget;
distanceWidget->SetInteractor(interactor);
distanceWidget->SetDefaultRenderer(renderer);

// 获取内部Representation以配置外观
vtkDistanceRepresentation* rep =
    distanceWidget->GetDistanceRepresentation();

// 程序化地设置两个端点（世界坐标）
double p1[3] = {0.0, 0.0, 0.0};
double p2[3] = {1.0, 0.0, 0.0};
rep->SetPoint1WorldPosition(p1);
rep->SetPoint2WorldPosition(p2);

// 设置标注格式
rep->SetLabelFormat("%.2f mm");   // 距离标注的printf格式

// 控制标尺颜色
rep->GetLineProperty()->SetColor(1.0, 1.0, 0.0);   // 测量线：黄色

// 切换到已放置模式（跳过交互式放置步骤）
distanceWidget->SetWidgetStateToManipulate();

// 获取当前距离值
double distance = rep->GetDistance();

distanceWidget->On();
```

**关于WidgetState**：`vtkDistanceWidget`有三种状态：
- `Start`（初始）：Widget等待用户通过点击来放置两个点。
- `Define`（定义中）：第一个点已放置，等待第二个点。
- `Manipulate`（操作中）：两个点都已放置，用户可拖拽端点调整位置。

如果你通过程序设置了端点坐标，应调用`SetWidgetStateToManipulate()`跳过交互式放置阶段。

### 20.2.3 vtkImplicitPlaneWidget —— 交互式裁剪平面

**用途**：在三维空间中放置一个可旋转、可平移的无限平面。这是科学可视化中最常用的Widget之一，典型应用包括：
- 实时裁剪（Clipping）：用平面切割数据集，展示内部结构。
- 切割面提取（Cutting）：提取平面与数据集的交线/交面。
- 定义任意方向的截面视角。

**交互方式**：
- 左键拖拽法线箭头（圆锥体）：旋转平面。
- 左键拖拽平面本体：沿法线方向平移平面。
- 中键拖拽平面本体：在平面内任意平移（移动原点）。
- 右键上下移动：缩放外围的包围盒。

**核心API**：

```cpp
#include <vtkImplicitPlaneWidget.h>
#include <vtkPlane.h>

vtkNew<vtkImplicitPlaneWidget> planeWidget;
planeWidget->SetInteractor(interactor);
planeWidget->SetDefaultRenderer(renderer);
planeWidget->PlaceWidget(bounds);

// 设置初始法线方向
planeWidget->SetNormal(1.0, 0.0, 0.0);  // 沿X轴的法线

// 获取隐式平面函数（传递给裁剪/切割过滤器）
vtkNew<vtkPlane> plane;
planeWidget->GetPlane(plane);
// plane->GetNormal() 和 plane->GetOrigin() 可用于过滤器

// 强制法线与坐标轴对齐
planeWidget->SetNormalToXAxis(1);  // 锁定到X轴

// 外观控制
planeWidget->SetDrawPlane(0);      // 隐藏平面半透明面片（避免遮挡数据）
planeWidget->SetTubing(1);         // 启用管道化边缘
planeWidget->GetPlaneProperty()->SetOpacity(0.3);

// 交互能力控制
planeWidget->SetOutlineTranslation(1);  // 允许通过轮廓平移
planeWidget->SetScaleEnabled(1);        // 允许缩放包围盒
planeWidget->SetOriginTranslation(1);   // 允许平移原点

planeWidget->On();
```

**典型配合**：`vtkImplicitPlaneWidget`通常与以下过滤器配合使用：
- `vtkClipPolyData` / `vtkClipDataSet`：使用Widget提供的`vtkPlane`进行裁剪。
- `vtkCutter`：使用Widget提供的`vtkPlane`提取截面。

在20.5节的完整示例中，你将看到这两者的实时配合。

### 20.2.4 vtkSphereWidget —— 球形区域选择

**用途**：在三维空间中放置一个可调整半径的球体。适用于：
- 球形区域选择（选取球内的数据点或单元）。
- 定义球形裁剪区域。
- 创建球形的感兴趣区域。

**交互方式**：
- 左键拖拽球体表面：调整球体半径。
- 左键拖拽球体中心手柄：平移整个球体。

**核心API**：

```cpp
#include <vtkSphereWidget.h>

vtkNew<vtkSphereWidget> sphereWidget;
sphereWidget->SetInteractor(interactor);
sphereWidget->SetDefaultRenderer(renderer);
sphereWidget->PlaceWidget(bounds);

// 获取球体的中心和半径
double center[3];
sphereWidget->GetCenter(center);
double radius = sphereWidget->GetRadius();

// 程序化地设置球体参数
sphereWidget->SetCenter(0.0, 0.0, 0.0);
sphereWidget->SetRadius(2.5);

// 获取球体的隐式函数（用于裁剪/选择）
vtkNew<vtkSphere> sphere;
sphereWidget->GetSphere(sphere);

// 外观控制
sphereWidget->GetSphereProperty()->SetColor(0.2, 0.6, 1.0);
sphereWidget->GetSphereProperty()->SetOpacity(0.3);
sphereWidget->GetHandleProperty()->SetColor(1.0, 0.5, 0.0);

sphereWidget->On();
```

### 20.2.5 vtkSliderWidget —— 二维滑块控件

**用途**：在渲染窗口中放置一个二维滑块，用于控制标量参数。与前面介绍的3D Widget不同，`vtkSliderWidget`是一个2D Widget——它在屏幕空间（而非世界空间）中操作。适用于：
- 控制等值面的值（在最小值到最大值之间滑动）。
- 调整时间步（在时间序列中跳转）。
- 调节任何连续变化的参数（透明度、缩放因子、阈值等）。

**交互方式**：
- 左键拖拽滑块球：连续调整数值。
- 左键点击滑轨的某个位置：滑块跳到该位置。
- 左键点击端点帽：滑块跳到最小或最大值。

**核心API**：

```cpp
#include <vtkSliderWidget.h>
#include <vtkSliderRepresentation2D.h>

vtkNew<vtkSliderRepresentation2D> sliderRep;
sliderRep->SetMinimumValue(0.0);
sliderRep->SetMaximumValue(1.0);
sliderRep->SetValue(0.5);                    // 当前值

// 控制滑块在窗口中的位置（归一化坐标：[0,1]）
sliderRep->GetPoint1Coordinate()
    ->SetCoordinateSystemToNormalizedDisplay();
sliderRep->GetPoint1Coordinate()->SetValue(0.1, 0.1);  // 起点
sliderRep->GetPoint2Coordinate()
    ->SetCoordinateSystemToNormalizedDisplay();
sliderRep->GetPoint2Coordinate()->SetValue(0.5, 0.1);  // 终点

// 控制滑块管道的宽度
sliderRep->SetTubeWidth(0.01);
sliderRep->SetSliderWidth(0.03);

// 标签格式
sliderRep->SetLabelFormat("%.2f");

vtkNew<vtkSliderWidget> sliderWidget;
sliderWidget->SetInteractor(interactor);
sliderWidget->SetRepresentation(sliderRep);
sliderWidget->SetDefaultRenderer(renderer);
sliderWidget->On();
```

**获取滑块值**：通常不在SliderWidget上直接获取值，而是通过回调机制（20.3节）在滑块值改变时自动读取并更新目标对象。

### 20.2.6 vtkLineWidget —— 交互式线段

**用途**：在三维空间中放置一条可交互的线段。适用于：
- 定义探针线（Probe Line）——沿线段采样数据值。
- 定义裁剪轴或对称轴。
- 交互式地指定两个三维位置。

**交互方式**：
- 左键拖拽端点手柄：移动线段的起点或终点。
- 左键拖拽线段中间部分：平移整条线段。

**核心API**：

```cpp
#include <vtkLineWidget.h>

vtkNew<vtkLineWidget> lineWidget;
lineWidget->SetInteractor(interactor);
lineWidget->SetDefaultRenderer(renderer);
lineWidget->PlaceWidget(bounds);

// 设置线段的两个端点
lineWidget->SetPoint1(-1.0, 0.0, 0.0);
lineWidget->SetPoint2(1.0, 0.0, 0.0);

// 获取线段的端点坐标
double p1[3], p2[3];
lineWidget->GetPoint1(p1);
lineWidget->GetPoint2(p2);

// 获取线段数据（vtkPolyData 包含一条线）
vtkNew<vtkPolyData> lineData;
lineWidget->GetPolyData(lineData);

// 控制是否允许端点之外的对齐
lineWidget->SetAlign(1);  // 对齐到最近的点（如果绑定了数据集）

// 外观控制
lineWidget->GetLineProperty()->SetColor(1.0, 1.0, 0.0);
lineWidget->GetLineProperty()->SetLineWidth(3.0);
lineWidget->GetHandleProperty()->SetColor(1.0, 0.0, 0.0);

lineWidget->On();
```

---

## 20.3 Observer回调模式

Widget本身提供交互能力，但真正的"响应"——即当用户拖拽Widget时执行什么操作——是通过**Observer（观察者）回调模式**来实现的。

### 20.3.1 vtkCommand 与 Observer模式

VTK实现了经典的设计模式——**Observer模式**（也称为发布-订阅模式）。在这个模式中：
- **Subject（主题/被观察者）**：是任何继承自`vtkObject`的VTK对象。它维护一个Observer列表，并在特定"事件"发生时通知所有Observer。
- **Observer（观察者/回调）**：是一个回调函数（或函数对象），当Subject发出事件通知时被调用。
- **Event（事件）**：是一个无符号长整数，标识发生了什么。VTK在`vtkCommand.h`中定义了数百个标准事件ID。

在Widget的上下文中：
- **Subject**就是Widget自身（如`vtkBoxWidget`、`vtkDistanceWidget`等）。
- **Events**包括`vtkCommand::InteractionEvent`（Widget被拖拽时）、`vtkCommand::StartInteractionEvent`（开始拖拽）、`vtkCommand::EndInteractionEvent`（结束拖拽）等。
- **Observer**是你编写的回调函数，在Widget状态改变时执行你想要的逻辑。

**AddObserver的基本签名**：

```cpp
// 添加一个Observer，返回一个tag（用于后续移除）
unsigned long tag = object->AddObserver(eventId, callback, clientData);

// 参数说明：
//   eventId   - 事件ID，例如 vtkCommand::InteractionEvent
//   callback  - 回调函数指针（可以是静态函数或成员函数）
//   clientData - 用户自定义数据，将被传递给回调函数（可选）

// 移除一个Observer
object->RemoveObserver(tag);
```

### 20.3.2 静态函数回调

最简单的回调形式是使用**静态函数**（或全局函数）：

```cpp
// 静态回调函数
void MyCallback(vtkObject* caller, unsigned long eventId,
                void* clientData, void* callData)
{
    // caller: 发出事件的对象（即调用 AddObserver 的那个Widget）
    vtkDistanceWidget* widget =
        static_cast<vtkDistanceWidget*>(caller);

    // 获取距离值
    vtkDistanceRepresentation* rep =
        widget->GetDistanceRepresentation();
    double distance = rep->GetDistance();

    std::cout << "Distance: " << distance << std::endl;
}

// 在main()中注册回调
distanceWidget->AddObserver(vtkCommand::InteractionEvent, MyCallback);
```

**静态回调的局限性**：
- 无法直接访问非静态成员（因为没有`this`指针）。
- 通常需要通过`clientData`参数传递上下文对象。

### 20.3.3 使用clientData传递上下文

通过`clientData`可以向回调函数传递任意指针（通常是一个应用上下文结构体），从而让回调函数能够访问更多的应用状态：

```cpp
struct AppContext
{
    vtkSmartPointer<vtkClipPolyData> clipFilter;
    vtkSmartPointer<vtkPlane>        clipPlane;
    vtkSmartPointer<vtkTextActor>    hudText;
};

void ClipPlaneCallback(vtkObject* caller, unsigned long eventId,
                       void* clientData, void* callData)
{
    // 从clientData恢复应用上下文
    AppContext* ctx = static_cast<AppContext*>(clientData);

    // 从Widget获取当前平面参数
    vtkImplicitPlaneWidget* planeWidget =
        static_cast<vtkImplicitPlaneWidget*>(caller);
    planeWidget->GetPlane(ctx->clipPlane);

    // 更新HUD显示
    double* normal = ctx->clipPlane->GetNormal();
    double* origin = ctx->clipPlane->GetOrigin();
    std::ostringstream oss;
    oss << "Normal: (" << normal[0] << ", "
        << normal[1] << ", " << normal[2] << ")\n"
        << "Origin: (" << origin[0] << ", "
        << origin[1] << ", " << origin[2] << ")";
    ctx->hudText->SetInput(oss.str().c_str());

    // 触发渲染更新
    ctx->clipFilter->Modified();
    caller->GetInteractor()->Render();
}

// 注册时传递上下文指针
AppContext ctx;
ctx.clipFilter = clipFilter;
ctx.clipPlane = clipPlane;
ctx.hudText = hudText;

planeWidget->AddObserver(vtkCommand::InteractionEvent,
                         ClipPlaneCallback, &ctx);
```

### 20.3.4 成员函数回调

在实际项目中，你通常希望回调是某个类的成员函数。VTK支持通过`vtkCallbackCommand`或直接使用`AddObserver`的成员函数重载来实现这一点。

**方法一：使用vtkCallbackCommand**

```cpp
class MyObserver
{
public:
    static MyObserver* New() { return new MyObserver; }

    void SetContext(AppContext* ctx) { m_Context = ctx; }

    // 这个静态函数作为桥接
    static void CallbackFunction(vtkObject* caller,
                                 unsigned long eventId,
                                 void* clientData, void* callData)
    {
        MyObserver* self = static_cast<MyObserver*>(clientData);
        self->OnWidgetInteraction(caller, eventId, callData);
    }

    // 真正的处理逻辑
    void OnWidgetInteraction(vtkObject* caller,
                             unsigned long eventId,
                             void* callData)
    {
        // 在这里处理Widget交互
        vtkDistanceWidget* widget =
            static_cast<vtkDistanceWidget*>(caller);
        // ... 更新m_Context中的状态 ...
    }

private:
    AppContext* m_Context = nullptr;
};

// 使用：
vtkNew<MyObserver> observer;
observer->SetContext(&ctx);
distanceWidget->AddObserver(vtkCommand::InteractionEvent,
    MyObserver::CallbackFunction, observer);
```

**方法二：使用AddObserver的Lambda风格**（需要std::function支持）

现代C++中，你可以使用lambda表达式配合`clientData`来实现简洁的回调：

```cpp
AppContext* ctx = new AppContext;  // 注意生命周期管理

distanceWidget->AddObserver(vtkCommand::InteractionEvent,
    [](vtkObject* caller, unsigned long eventId,
       void* clientData, void* callData)
    {
        auto* ctx = static_cast<AppContext*>(clientData);
        auto* widget = static_cast<vtkDistanceWidget*>(caller);
        double d = widget->GetDistanceRepresentation()->GetDistance();
        std::cout << "Distance: " << d << std::endl;
    },
    ctx);
```

**注意**：Lambda只有当没有捕获变量时才能退化为函数指针。如果需要捕获变量，使用`clientData`来传递上下文是最兼容的方式。

### 20.3.5 常用的Widget事件

以下是Widget交互中最常使用的VTK事件ID：

| 事件ID | 含义 | 典型触发时机 |
|--------|------|-------------|
| `vtkCommand::StartInteractionEvent` | 开始交互 | 用户在Widget上按下鼠标按钮 |
| `vtkCommand::InteractionEvent` | 交互进行中 | 用户在Widget上拖拽鼠标（持续触发） |
| `vtkCommand::EndInteractionEvent` | 交互结束 | 用户在Widget上释放鼠标按钮 |
| `vtkCommand::PlacePointEvent` | 点已放置 | 交互式放置Widget的一个端点时 |
| `vtkCommand::ModifiedEvent` | 对象被修改 | Widget的某个参数被程序修改时 |

**回调策略建议**：

- **StartInteractionEvent**：适合做一些准备工作，如保存当前状态、显示辅助信息、暂停某些计算。
- **InteractionEvent**：适合实时更新——例如实时裁剪、实时更新测量数值。这个事件在拖拽时非常高频地触发，因此回调中的逻辑应该尽量轻量。
- **EndInteractionEvent**：适合做收尾工作，如保存最终结果、触发重计算、记录日志。

### 20.3.6 回调示例：实时显示测量距离

下面是一个完整的Observer回调示例，展示如何在拖拽`vtkDistanceWidget`端点时实时更新屏幕上显示的距离值：

```cpp
// 全局或成员变量
vtkSmartPointer<vtkTextActor> g_HudText;

void DistanceInteractionCallback(vtkObject* caller,
                                 unsigned long eventId,
                                 void* clientData, void* callData)
{
    vtkDistanceWidget* widget =
        static_cast<vtkDistanceWidget*>(caller);

    vtkDistanceRepresentation* rep =
        widget->GetDistanceRepresentation();

    double distance = rep->GetDistance();

    // 获取两个端点的世界坐标
    double p1[3], p2[3];
    rep->GetPoint1WorldPosition(p1);
    rep->GetPoint2WorldPosition(p2);

    // 更新HUD文字
    std::ostringstream oss;
    oss << "Distance: " << std::fixed
        << std::setprecision(3) << distance << "\n"
        << "P1: (" << p1[0] << ", " << p1[1] << ", " << p1[2] << ")\n"
        << "P2: (" << p2[0] << ", " << p2[1] << ", " << p2[2] << ")";

    g_HudText->SetInput(oss.str().c_str());

    // 刷新渲染
    widget->GetInteractor()->Render();
}

// 在初始化代码中注册Observer
distanceWidget->AddObserver(vtkCommand::InteractionEvent,
                           DistanceInteractionCallback);
distanceWidget->AddObserver(vtkCommand::EndInteractionEvent,
                           DistanceInteractionCallback);  // 最终值也更新
```

---

## 20.4 自定义交互样式（InteractorStyle）

除了使用Widget之外，VTK的另一个重要的交互定制途径是**自定义InteractorStyle**。通过继承`vtkInteractorStyleTrackballCamera`（或其它内置Style），你可以覆写特定的事件处理方法来响应鼠标和键盘操作。

### 20.4.1 继承自vtkInteractorStyleTrackballCamera

`vtkInteractorStyleTrackballCamera`是VTK最常用的交互样式，提供：
- 左键拖拽：旋转场景（轨迹球旋转）。
- 右键拖拽（或滚轮）：缩放场景。
- 中键拖拽：平移场景。
- 键盘快捷键：`r`重置相机，`w`线框模式，`s`表面模式等。

继承它的基本框架如下：

```cpp
#include <vtkInteractorStyleTrackballCamera.h>
#include <vtkObjectFactory.h>

class MyCustomStyle : public vtkInteractorStyleTrackballCamera
{
public:
    static MyCustomStyle* New();
    vtkTypeMacro(MyCustomStyle, vtkInteractorStyleTrackballCamera);

    // 覆写鼠标事件处理方法
    void OnLeftButtonDown() override;
    void OnRightButtonDown() override;
    void OnMiddleButtonDown() override;
    void OnMouseMove() override;
    void OnKeyPress() override;
    void OnKeyRelease() override;

protected:
    MyCustomStyle() = default;
    ~MyCustomStyle() override = default;

private:
    MyCustomStyle(const MyCustomStyle&) = delete;
    void operator=(const MyCustomStyle&) = delete;
};

// VTK要求使用vtkStandardNewMacro来定义New()实现
vtkStandardNewMacro(MyCustomStyle);
```

### 20.4.2 覆写鼠标事件处理方法

你可以在鼠标事件处理方法中添加自定义逻辑。**关键规则**：如果你处理了事件，不要调用父类的对应方法；如果你不处理（或处理后仍需要默认行为），则调用父类方法。

```cpp
void MyCustomStyle::OnLeftButtonDown()
{
    // 获取当前点击位置的显示坐标（像素坐标）
    int* clickPos = this->GetInteractor()->GetEventPosition();
    std::cout << "Click at display coords: ("
              << clickPos[0] << ", " << clickPos[1] << ")" << std::endl;

    // 获取世界坐标（通过Picker拾取）
    this->GetInteractor()->GetPicker()->Pick(
        clickPos[0], clickPos[1],
        0,  // 总是使用第一个Renderer
        this->GetDefaultRenderer());

    double* worldPos =
        this->GetInteractor()->GetPicker()->GetPickPosition();
    std::cout << "World position: ("
              << worldPos[0] << ", "
              << worldPos[1] << ", "
              << worldPos[2] << ")" << std::endl;

    // 如果你想保留默认的旋转行为，调用父类方法
    // 如果不想（想完全接管左键），则注释掉这一行
    vtkInteractorStyleTrackballCamera::OnLeftButtonDown();
}
```

### 20.4.3 覆写键盘事件处理方法

```cpp
void MyCustomStyle::OnKeyPress()
{
    // 获取按下的键
    std::string key = this->GetInteractor()->GetKeySym();

    if (key == "m")
    {
        std::cout << "Switched to Measure mode." << std::endl;
        // 切换到测量模式的逻辑...
    }
    else if (key == "v")
    {
        std::cout << "Switched to View mode." << std::endl;
        // 切换到观察模式的逻辑...
    }
    else if (key == "c")
    {
        // 打印当前相机参数
        vtkCamera* cam = this->GetDefaultRenderer()->GetActiveCamera();
        double* pos = cam->GetPosition();
        double* fp  = cam->GetFocalPoint();
        double* vu  = cam->GetViewUp();
        std::cout << "Camera -- Position: (" << pos[0] << ", "
                  << pos[1] << ", " << pos[2] << ")\n"
                  << "          FocalPoint: (" << fp[0]  << ", "
                  << fp[1]  << ", " << fp[2]  << ")\n"
                  << "          ViewUp:     (" << vu[0]  << ", "
                  << vu[1]  << ", " << vu[2]  << ")" << std::endl;
    }
    else
    {
        // 对于未处理的键，转发给父类（保留默认快捷键）
        vtkInteractorStyleTrackballCamera::OnKeyPress();
    }
}
```

### 20.4.4 模式切换的实现模式

自定义InteractorStyle的一个常见用途是实现**模式切换**——在不同的交互模式之间切换，每个模式下相同的鼠标操作具有不同的含义。

以下是一种典型的模式切换实现：

```cpp
class ModeSwitchingStyle : public vtkInteractorStyleTrackballCamera
{
public:
    static ModeSwitchingStyle* New();
    vtkTypeMacro(ModeSwitchingStyle, vtkInteractorStyleTrackballCamera);

    enum InteractionMode
    {
        VIEW_MODE = 0,      // 标准视角操控模式
        MEASURE_MODE = 1,   // 测量模式
        PICK_MODE = 2       // 拾取模式
    };

    void SetMode(InteractionMode mode) { m_Mode = mode; }
    InteractionMode GetMode() const { return m_Mode; }

    void OnLeftButtonDown() override
    {
        switch (m_Mode)
        {
        case VIEW_MODE:
            // 标准轨迹球旋转
            vtkInteractorStyleTrackballCamera::OnLeftButtonDown();
            break;

        case MEASURE_MODE:
        {
            // 在测量模式下，左键点击用于放置测量点
            int* clickPos = this->GetInteractor()->GetEventPosition();
            this->GetInteractor()->GetPicker()->Pick(
                clickPos[0], clickPos[1], 0,
                this->GetDefaultRenderer());
            double* worldPos =
                this->GetInteractor()->GetPicker()->GetPickPosition();
            std::cout << "Measure point: ("
                      << worldPos[0] << ", "
                      << worldPos[1] << ", "
                      << worldPos[2] << ")" << std::endl;
            break;
        }

        case PICK_MODE:
        {
            // 在拾取模式下，左键点击用于选择物体
            int* clickPos = this->GetInteractor()->GetEventPosition();
            // ... 拾取逻辑 ...
            std::cout << "Pick mode: object selected." << std::endl;
            break;
        }
        }
    }

    void OnKeyPress() override
    {
        std::string key = this->GetInteractor()->GetKeySym();
        if (key == "v")
        {
            SetMode(VIEW_MODE);
            std::cout << "Mode: VIEW" << std::endl;
        }
        else if (key == "m")
        {
            SetMode(MEASURE_MODE);
            std::cout << "Mode: MEASURE" << std::endl;
        }
        else if (key == "p")
        {
            SetMode(PICK_MODE);
            std::cout << "Mode: PICK" << std::endl;
        }
        else
        {
            vtkInteractorStyleTrackballCamera::OnKeyPress();
        }
    }

protected:
    ModeSwitchingStyle() : m_Mode(VIEW_MODE) {}
    ~ModeSwitchingStyle() override = default;

private:
    InteractionMode m_Mode;
    ModeSwitchingStyle(const ModeSwitchingStyle&) = delete;
    void operator=(const ModeSwitchingStyle&) = delete;
};

vtkStandardNewMacro(ModeSwitchingStyle);
```

### 20.4.5 自定义Style的使用方式

自定义Style创建后，通过`vtkRenderWindowInteractor`的`SetInteractorStyle()`方法与交互器绑定：

```cpp
vtkNew<vtkRenderWindowInteractor> interactor;

// 创建并使用自定义Style
vtkNew<MyCustomStyle> customStyle;
interactor->SetInteractorStyle(customStyle);

// 或者替换默认的TrackballCamera为ModeSwitchingStyle
vtkNew<ModeSwitchingStyle> modeStyle;
interactor->SetInteractorStyle(modeStyle);

// 启动事件循环（照常）
interactor->Initialize();
interactor->Start();
```

---

## 20.5 代码示例：交互式裁剪工具

本节提供一个完整的C++程序`InteractiveClipper.cxx`，将本章的核心概念整合到一个实用的科学可视化工具中。

### 20.5.1 程序概述

**功能描述**：本程序创建一个三维数据集（结构化网格上的球形标量场），提供以下交互式工具：

1. **交互式裁剪平面（vtkImplicitPlaneWidget）**：用户可以拖拽一个半透明的平面来实时裁剪数据，观察内部结构。平面移动时，裁剪结果立即更新。
2. **距离测量工具（vtkDistanceWidget）**：用户可以放置两个测量点来测量裁剪面上的特征尺寸。测量值实时显示在屏幕上。
3. **自定义交互样式（ClipperInteractorStyle）**：支持三种模式——视角操控模式（V）、裁剪模式（C）、测量模式（M）。通过键盘切换。
4. **实时信息显示（HUD）**：在屏幕左上角显示当前模式、裁剪平面参数和测量距离。

**数据流程**：

```
vtkSampleFunction (隐式球体)
    |
    v
vtkImageData (标量场)
    |
    v
vtkClipDataSet (使用Widget提供的vtkPlane进行实时裁剪)
    |
    v
vtkDataSetMapper --> vtkActor (裁剪后的可见部分)
```

### 20.5.2 完整C++代码

```cpp
// ============================================================================
// Chapter 20: Custom Interaction and Widgets
// File: InteractiveClipper.cxx
// Description: Comprehensive demonstration of VTK interactive widgets.
//              Features an interactive clipping plane, distance measurement,
//              mode-switching interactor style, and real-time HUD display.
// VTK Version: 9.5.2
// ============================================================================

#include <vtkActor.h>
#include <vtkCallbackCommand.h>
#include <vtkCamera.h>
#include <vtkClipDataSet.h>
#include <vtkCommand.h>
#include <vtkDataSetMapper.h>
#include <vtkDataSetSurfaceFilter.h>
#include <vtkDistanceRepresentation.h>
#include <vtkDistanceWidget.h>
#include <vtkImplicitPlaneWidget.h>
#include <vtkInteractorStyleTrackballCamera.h>
#include <vtkMinimalStandardRandomSequence.h>
#include <vtkNamedColors.h>
#include <vtkNew.h>
#include <vtkObjectFactory.h>
#include <vtkPlane.h>
#include <vtkPlanes.h>
#include <vtkPointData.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkSampleFunction.h>
#include <vtkSmartPointer.h>
#include <vtkSphere.h>
#include <vtkTextActor.h>
#include <vtkTextProperty.h>
#include <vtkUnstructuredGrid.h>

#include <iomanip>
#include <iostream>
#include <sstream>
#include <string>

// ============================================================================
// Forward declarations
// ============================================================================
struct AppContext;
void ClipPlaneInteractionCallback(vtkObject* caller, unsigned long eventId,
                                  void* clientData, void* callData);
void DistanceInteractionCallback(vtkObject* caller, unsigned long eventId,
                                 void* clientData, void* callData);
void UpdateHUD(AppContext* ctx);

// ============================================================================
// Application context: shared data between main(), callbacks, and style
// ============================================================================
struct AppContext
{
    vtkSmartPointer<vtkClipDataSet>          clipFilter;
    vtkSmartPointer<vtkDataSetSurfaceFilter> surfaceFilter;
    vtkSmartPointer<vtkPlane>                clipPlane;
    vtkSmartPointer<vtkImplicitPlaneWidget>  planeWidget;
    vtkSmartPointer<vtkDistanceWidget>       distanceWidget;
    vtkSmartPointer<vtkRenderer>             renderer;
    vtkSmartPointer<vtkRenderWindow>         renderWindow;
    vtkSmartPointer<vtkRenderWindowInteractor> interactor;
    vtkSmartPointer<vtkTextActor>            hudText;
    vtkSmartPointer<vtkDataSetMapper>        mapper;
    std::string                              currentMode;
    double                                   lastDistance;
};

// ============================================================================
// Custom Interactor Style: supports View / Clip / Measure modes
// ============================================================================
class ClipperInteractorStyle : public vtkInteractorStyleTrackballCamera
{
public:
    static ClipperInteractorStyle* New();
    vtkTypeMacro(ClipperInteractorStyle, vtkInteractorStyleTrackballCamera);

    enum Mode
    {
        VIEW_MODE = 0,
        CLIP_MODE = 1,
        MEASURE_MODE = 2
    };

    void SetContext(AppContext* ctx) { m_Context = ctx; }
    void SetMode(int mode);

    void OnKeyPress() override;

protected:
    ClipperInteractorStyle();
    ~ClipperInteractorStyle() override = default;

private:
    AppContext* m_Context = nullptr;
    int         m_Mode = VIEW_MODE;

    ClipperInteractorStyle(const ClipperInteractorStyle&) = delete;
    void operator=(const ClipperInteractorStyle&) = delete;
};

vtkStandardNewMacro(ClipperInteractorStyle);

ClipperInteractorStyle::ClipperInteractorStyle()
    : m_Mode(VIEW_MODE)
{
}

void ClipperInteractorStyle::SetMode(int mode)
{
    if (m_Context == nullptr) return;

    m_Mode = mode;

    switch (mode)
    {
    case VIEW_MODE:
        m_Context->currentMode = "VIEW";
        m_Context->planeWidget->Off();
        m_Context->distanceWidget->Off();
        std::cout << "[Mode] VIEW -- Standard trackball navigation." << std::endl;
        break;

    case CLIP_MODE:
        m_Context->currentMode = "CLIP";
        m_Context->planeWidget->On();
        m_Context->distanceWidget->Off();
        std::cout << "[Mode] CLIP -- Drag the plane to clip data." << std::endl;
        break;

    case MEASURE_MODE:
        m_Context->currentMode = "MEASURE";
        m_Context->distanceWidget->On();
        m_Context->planeWidget->Off();
        std::cout << "[Mode] MEASURE -- Click two points to measure distance." << std::endl;
        break;
    }

    UpdateHUD(m_Context);
    m_Context->renderWindow->Render();
}

void ClipperInteractorStyle::OnKeyPress()
{
    std::string key = this->GetInteractor()->GetKeySym();

    if (key == "v" || key == "V")
    {
        SetMode(VIEW_MODE);
    }
    else if (key == "c" || key == "C")
    {
        SetMode(CLIP_MODE);
    }
    else if (key == "m" || key == "M")
    {
        SetMode(MEASURE_MODE);
    }
    else if (key == "r" || key == "R")
    {
        // Reset: restore default clipping plane
        if (m_Context && m_Context->planeWidget)
        {
            m_Context->planeWidget->SetNormal(1.0, 0.0, 0.0);
            m_Context->planeWidget->SetOrigin(0.0, 0.0, 0.0);

            double bounds[6];
            m_Context->clipFilter->GetInput()->GetBounds(bounds);
            m_Context->planeWidget->PlaceWidget(bounds);

            m_Context->clipPlane->SetNormal(1.0, 0.0, 0.0);
            m_Context->clipPlane->SetOrigin(0.0, 0.0, 0.0);
            m_Context->clipFilter->Modified();

            std::cout << "[Reset] Clipping plane reset to X-normal at origin." << std::endl;
        }
        m_Context->renderWindow->Render();
    }
    else if (key == "h" || key == "H")
    {
        // Toggle clipping plane visibility (useful when examining clipped result)
        int drawPlane =
            m_Context->planeWidget->GetDrawPlane() ? 0 : 1;
        m_Context->planeWidget->SetDrawPlane(drawPlane);
        std::cout << "[Toggle] Plane visibility: "
                  << (drawPlane ? "ON" : "OFF") << std::endl;
        m_Context->renderWindow->Render();
    }
    else
    {
        // Forward unhandled keys to parent (preserves default shortcuts like 'w', 's')
        vtkInteractorStyleTrackballCamera::OnKeyPress();
    }
}

// ============================================================================
// Callback: Clipping plane interaction -- update clip filter in real-time
// ============================================================================
void ClipPlaneInteractionCallback(vtkObject* caller, unsigned long eventId,
                                  void* clientData, void* callData)
{
    AppContext* ctx = static_cast<AppContext*>(clientData);

    // Extract current plane parameters from the widget
    ctx->planeWidget->GetPlane(ctx->clipPlane);

    // Notify the pipeline that the clip filter's implicit function changed.
    // The clip filter stores a pointer to the plane, so Modified() triggers
    // a re-evaluation on the next Update().
    ctx->clipFilter->Modified();

    // Update the HUD with current plane parameters
    UpdateHUD(ctx);

    // Re-render
    ctx->renderWindow->Render();
}

// ============================================================================
// Callback: Distance widget interaction -- update distance display
// ============================================================================
void DistanceInteractionCallback(vtkObject* caller, unsigned long eventId,
                                 void* clientData, void* callData)
{
    AppContext* ctx = static_cast<AppContext*>(clientData);

    vtkDistanceRepresentation* rep =
        ctx->distanceWidget->GetDistanceRepresentation();

    ctx->lastDistance = rep->GetDistance();

    // Update HUD
    UpdateHUD(ctx);

    // Re-render
    ctx->renderWindow->Render();
}

// ============================================================================
// Helper: Update the HUD text overlay
// ============================================================================
void UpdateHUD(AppContext* ctx)
{
    std::ostringstream oss;
    oss << "=== Interactive Clipper ===\n";
    oss << "Mode: " << ctx->currentMode << "\n";
    oss << "--------------------------------\n";

    // Show current plane parameters
    if (ctx->clipPlane)
    {
        double* normal = ctx->clipPlane->GetNormal();
        double* origin = ctx->clipPlane->GetOrigin();
        oss << "Plane Normal: ("
            << std::fixed << std::setprecision(3)
            << normal[0] << ", " << normal[1] << ", " << normal[2] << ")\n";
        oss << "Plane Origin: ("
            << origin[0] << ", " << origin[1] << ", " << origin[2] << ")\n";
    }

    // Show distance measurement if in measure mode
    if (ctx->currentMode == "MEASURE" && ctx->lastDistance >= 0.0)
    {
        oss << "--------------------------------\n";
        oss << "Distance: "
            << std::fixed << std::setprecision(4)
            << ctx->lastDistance << "\n";
    }

    oss << "--------------------------------\n";
    oss << "Keys: V=View | C=Clip | M=Measure | R=Reset | H=Toggle Plane";

    ctx->hudText->SetInput(oss.str().c_str());
}

// ============================================================================
// Helper: Create a 3D scalar field dataset
// ============================================================================
vtkSmartPointer<vtkImageData> CreateScalarField()
{
    // Create an implicit sphere function
    vtkNew<vtkSphere> sphere;
    sphere->SetCenter(0.0, 0.0, 0.0);
    sphere->SetRadius(3.0);

    // Sample the sphere function onto a 3D grid
    vtkNew<vtkSampleFunction> sample;
    sample->SetImplicitFunction(sphere);
    sample->SetModelBounds(-5.0, 5.0, -5.0, 5.0, -5.0, 5.0);
    sample->SetSampleDimensions(80, 80, 80);
    sample->SetOutputScalarTypeToFloat();
    sample->CappingOff();  // Do not cap the outer values
    sample->Update();

    // Return a copy of the sampled data
    vtkSmartPointer<vtkImageData> data = vtkSmartPointer<vtkImageData>::New();
    data->DeepCopy(sample->GetOutput());

    std::cout << "[Data] Created scalar field: "
              << data->GetDimensions()[0] << " x "
              << data->GetDimensions()[1] << " x "
              << data->GetDimensions()[2]
              << " ("
              << data->GetNumberOfPoints()
              << " points)" << std::endl;

    return data;
}

// ============================================================================
// main()
// ============================================================================
int main(int argc, char* argv[])
{
    // ------------------------------------------------------------------------
    // 1. Create the 3D scalar field dataset
    // ------------------------------------------------------------------------
    auto dataset = CreateScalarField();

    // Get the dataset bounds (used for widget placement)
    double bounds[6];
    dataset->GetBounds(bounds);

    // ------------------------------------------------------------------------
    // 2. Create the clipping plane (vtkPlane) -- the implicit function
    // ------------------------------------------------------------------------
    vtkNew<vtkPlane> clipPlane;
    clipPlane->SetNormal(1.0, 0.0, 0.0);
    clipPlane->SetOrigin(0.0, 0.0, 0.0);

    // ------------------------------------------------------------------------
    // 3. Create the pipeline: Clip -> Surface -> Mapper -> Actor
    // ------------------------------------------------------------------------
    vtkNew<vtkClipDataSet> clipFilter;
    clipFilter->SetInputData(dataset);
    clipFilter->SetClipFunction(clipPlane);
    clipFilter->SetInsideOut(0);  // Keep the side the normal points away from
    clipFilter->GenerateClippedOutputOn(); // Keep both sides available

    // Convert clipped unstructured grid to surface for rendering
    vtkNew<vtkDataSetSurfaceFilter> surfaceFilter;
    surfaceFilter->SetInputConnection(clipFilter->GetOutputPort());

    vtkNew<vtkDataSetMapper> mapper;
    mapper->SetInputConnection(surfaceFilter->GetOutputPort());
    mapper->ScalarVisibilityOn();
    mapper->SetScalarRange(dataset->GetPointData()->GetScalars()->GetRange());

    vtkNew<vtkActor> clippedActor;
    clippedActor->SetMapper(mapper);
    clippedActor->GetProperty()->SetColor(0.4, 0.7, 1.0);
    clippedActor->GetProperty()->SetAmbient(0.3);
    clippedActor->GetProperty()->SetDiffuse(0.7);
    clippedActor->GetProperty()->SetSpecular(0.3);
    clippedActor->GetProperty()->SetSpecularPower(20);

    // ------------------------------------------------------------------------
    // 4. Create the renderer, render window, and interactor
    // ------------------------------------------------------------------------
    vtkNew<vtkRenderer> renderer;
    renderer->SetBackground(0.15, 0.15, 0.2);
    renderer->AddActor(clippedActor);

    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->AddRenderer(renderer);
    renderWindow->SetSize(1200, 800);
    renderWindow->SetWindowName("Chapter 20: Interactive Clipper");

    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    // ------------------------------------------------------------------------
    // 5. Create the implicit plane widget for interactive clipping
    // ------------------------------------------------------------------------
    vtkNew<vtkImplicitPlaneWidget> planeWidget;
    planeWidget->SetInteractor(interactor);
    planeWidget->SetDefaultRenderer(renderer);
    planeWidget->PlaceWidget(bounds);
    planeWidget->SetNormal(1.0, 0.0, 0.0);
    planeWidget->SetOrigin(0.0, 0.0, 0.0);

    // Configure widget appearance
    planeWidget->SetDrawPlane(1);          // Show the semi-transparent plane
    planeWidget->SetTubing(1);             // Tube the intersection edges
    planeWidget->GetPlaneProperty()->SetOpacity(0.2);
    planeWidget->GetPlaneProperty()->SetColor(0.0, 1.0, 1.0);
    planeWidget->GetNormalProperty()->SetColor(1.0, 0.8, 0.0);
    planeWidget->GetSelectedNormalProperty()->SetColor(1.0, 0.2, 0.2);
    planeWidget->GetEdgesProperty()->SetColor(0.0, 1.0, 0.0);
    planeWidget->GetEdgesProperty()->SetLineWidth(2.0);

    // Add observer callback for real-time clipping
    planeWidget->AddObserver(vtkCommand::InteractionEvent,
                             ClipPlaneInteractionCallback);

    // ------------------------------------------------------------------------
    // 6. Create the distance widget for measurement
    // ------------------------------------------------------------------------
    vtkNew<vtkDistanceWidget> distanceWidget;
    distanceWidget->SetInteractor(interactor);
    distanceWidget->SetDefaultRenderer(renderer);

    // Configure the distance representation
    vtkDistanceRepresentation* distRep =
        distanceWidget->GetDistanceRepresentation();
    distRep->SetLabelFormat("Distance: %.3f");
    distRep->GetLineProperty()->SetColor(1.0, 1.0, 0.0);
    distRep->GetLineProperty()->SetLineWidth(2.0);

    // Set initial measurement positions
    double p1[3] = {-2.0, 0.0, 0.0};
    double p2[3] = {2.0, 0.0, 0.0};
    distRep->SetPoint1WorldPosition(p1);
    distRep->SetPoint2WorldPosition(p2);
    distanceWidget->SetWidgetStateToManipulate();

    // Add observer callback for real-time distance display
    distanceWidget->AddObserver(vtkCommand::InteractionEvent,
                                DistanceInteractionCallback);
    distanceWidget->AddObserver(vtkCommand::EndInteractionEvent,
                                DistanceInteractionCallback);

    // ------------------------------------------------------------------------
    // 7. Create the HUD text display
    // ------------------------------------------------------------------------
    vtkNew<vtkTextActor> hudText;
    hudText->GetTextProperty()->SetFontFamilyToCourier();
    hudText->GetTextProperty()->SetFontSize(14);
    hudText->GetTextProperty()->SetColor(0.9, 0.9, 0.9);
    hudText->GetTextProperty()->SetBackgroundColor(0.1, 0.1, 0.15);
    hudText->GetTextProperty()->SetBackgroundOpacity(0.8);
    hudText->SetDisplayPosition(20, 20);
    renderer->AddActor2D(hudText);

    // ------------------------------------------------------------------------
    // 8. Build application context
    // ------------------------------------------------------------------------
    AppContext ctx;
    ctx.clipFilter      = clipFilter;
    ctx.surfaceFilter   = surfaceFilter;
    ctx.clipPlane       = clipPlane;
    ctx.planeWidget     = planeWidget;
    ctx.distanceWidget  = distanceWidget;
    ctx.renderer        = renderer;
    ctx.renderWindow    = renderWindow;
    ctx.interactor      = interactor;
    ctx.hudText         = hudText;
    ctx.mapper          = mapper;
    ctx.currentMode     = "CLIP";
    ctx.lastDistance    = distRep->GetDistance();

    // Update callback client data pointers (override the earlier registrations)
    planeWidget->AddObserver(vtkCommand::InteractionEvent,
                             ClipPlaneInteractionCallback, &ctx);
    planeWidget->AddObserver(vtkCommand::EndInteractionEvent,
                             ClipPlaneInteractionCallback, &ctx);
    distanceWidget->AddObserver(vtkCommand::InteractionEvent,
                                DistanceInteractionCallback, &ctx);
    distanceWidget->AddObserver(vtkCommand::EndInteractionEvent,
                                DistanceInteractionCallback, &ctx);

    // ------------------------------------------------------------------------
    // 9. Set up the custom interactor style
    // ------------------------------------------------------------------------
    vtkNew<ClipperInteractorStyle> customStyle;
    customStyle->SetContext(&ctx);
    interactor->SetInteractorStyle(customStyle);

    // Initialize in CLIP mode (show the clipping plane)
    customStyle->SetMode(ClipperInteractorStyle::CLIP_MODE);

    // Also register the widget callbacks on StartInteractionEvent for
    // proper initial state capture
    planeWidget->AddObserver(vtkCommand::StartInteractionEvent,
                             ClipPlaneInteractionCallback, &ctx);

    // ------------------------------------------------------------------------
    // 10. Set up the camera
    // ------------------------------------------------------------------------
    vtkCamera* camera = renderer->GetActiveCamera();
    camera->SetPosition(8.0, 6.0, 10.0);
    camera->SetFocalPoint(0.0, 0.0, 0.0);
    camera->SetViewUp(0.0, 0.0, 1.0);
    camera->Azimuth(30);
    camera->Elevation(20);

    renderer->ResetCameraClippingRange();

    // Initial HUD update
    UpdateHUD(&ctx);

    // ------------------------------------------------------------------------
    // 11. Print usage instructions and start the event loop
    // ------------------------------------------------------------------------
    std::cout << "\n================================================" << std::endl;
    std::cout << "  Interactive Clipper -- Widgets Demo" << std::endl;
    std::cout << "================================================" << std::endl;
    std::cout << "  V         - View mode (standard navigation)" << std::endl;
    std::cout << "  C         - Clip mode (drag the cyan plane)" << std::endl;
    std::cout << "  M         - Measure mode (click to place points)" << std::endl;
    std::cout << "  R         - Reset clipping plane" << std::endl;
    std::cout << "  H         - Toggle plane visibility" << std::endl;
    std::cout << "  W / S     - Wireframe/Surface display" << std::endl;
    std::cout << "  Left drag - Trackball rotate (View mode)" << std::endl;
    std::cout << "  Right drag - Zoom (View mode)" << std::endl;
    std::cout << "  Middle drag - Pan (View mode)" << std::endl;
    std::cout << "================================================\n" << std::endl;

    renderWindow->Render();
    interactor->Initialize();
    interactor->Start();

    return 0;
}
```

### 20.5.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

# ============================================================================
# Chapter 20: Custom Interaction and Widgets
# File: CMakeLists.txt
# ============================================================================

project(InteractiveClipper)

# --------------------------------------------------------------------------
# Find VTK 9.5.2
# Required components for this chapter:
#   - CommonCore/DataModel/ExecutionModel: VTK foundation
#   - FiltersSources: vtkSampleFunction
#   - FiltersCore/General: vtkClipDataSet, vtkDataSetSurfaceFilter
#   - InteractionWidgets: vtkImplicitPlaneWidget, vtkDistanceWidget
#   - InteractionStyle: vtkInteractorStyleTrackballCamera
#   - RenderingCore/OpenGL2: Rendering backend
#   - RenderingAnnotation: vtkTextActor
#   - RenderingFreeType: Text rendering
# --------------------------------------------------------------------------
find_package(VTK 9.5 REQUIRED COMPONENTS
    CommonCore
    CommonDataModel
    CommonExecutionModel
    CommonMath
    FiltersCore
    FiltersGeneral
    FiltersSources
    InteractionStyle
    InteractionWidgets
    RenderingAnnotation
    RenderingCore
    RenderingFreeType
    RenderingOpenGL2
)

if(NOT VTK_FOUND)
    message(FATAL_ERROR "VTK 9.5 not found. "
        "Set -DVTK_DIR=/path/to/vtk-9.5.2/build")
endif()

message(STATUS "VTK_VERSION: ${VTK_VERSION}")
message(STATUS "VTK_DIR: ${VTK_DIR}")

# --------------------------------------------------------------------------
# Executable
# --------------------------------------------------------------------------
add_executable(InteractiveClipper InteractiveClipper.cxx)

target_link_libraries(InteractiveClipper PRIVATE
    VTK::CommonCore
    VTK::CommonDataModel
    VTK::CommonExecutionModel
    VTK::CommonMath
    VTK::FiltersCore
    VTK::FiltersGeneral
    VTK::FiltersSources
    VTK::InteractionStyle
    VTK::InteractionWidgets
    VTK::RenderingAnnotation
    VTK::RenderingCore
    VTK::RenderingFreeType
    VTK::RenderingOpenGL2
)

# --------------------------------------------------------------------------
# C++ standard
# --------------------------------------------------------------------------
target_compile_features(InteractiveClipper PRIVATE cxx_std_17)

# --------------------------------------------------------------------------
# Platform-specific settings
# --------------------------------------------------------------------------
if(WIN32)
    target_compile_definitions(InteractiveClipper PRIVATE
        _CRT_SECURE_NO_WARNINGS
        NOMINMAX
    )
endif()
```

### 20.5.4 编译与运行

**编译步骤**：

```bash
# 1. 创建构建目录
mkdir build && cd build

# 2. 配置CMake（将路径替换为你的VTK安装路径）
cmake .. -DVTK_DIR=/path/to/vtk-9.5.2/build

# 3. 编译
cmake --build . --config Release

# 4. 运行
./InteractiveClipper        # Linux/macOS
.\Release\InteractiveClipper.exe   # Windows
```

**Windows (Visual Studio)**：

```powershell
mkdir build; cd build
cmake .. -G "Visual Studio 17 2022"
cmake --build . --config Release
.\Release\InteractiveClipper.exe
```

### 20.5.5 预期效果与交互指南

运行程序后，你将看到一个蓝色的三维球形标量场数据，中央有一个青色的裁剪平面：

**视觉呈现**：
- 一个三维体积数据（球体标量场），颜色随标量值变化（蓝色低值到红色高值）。
- 一个半透明的青色平面穿过数据体——这是裁剪平面的视觉表示，带有金色的法线箭头和绿色的边缘线。
- 裁剪平面一侧的数据可见，另一侧被裁掉——展示了数据体的内部结构。
- 屏幕左上角的HUD面板显示当前模式、平面参数和测量距离。

**交互指南**：

| 操作 | 效果 |
|------|------|
| 按 `V` | 切换到**视角模式**。裁剪平面和测量工具都隐藏。鼠标拖拽用于标准轨迹球导航。 |
| 按 `C` | 切换到**裁剪模式**。裁剪平面出现。拖拽法线箭头旋转平面，拖拽平面本体沿法线平移。裁剪结果实时更新。 |
| 按 `M` | 切换到**测量模式**。距离测量Widget出现。点击可重新放置两个测量点，拖拽端点调整位置。距离值实时显示在HUD中。 |
| 按 `R` | 重置裁剪平面到默认位置（X轴法线，原点处）。 |
| 按 `H` | 切换裁剪平面的可见性——在检查裁剪结果时隐藏平面本身。 |
| 左键拖拽（视角模式） | 轨迹球旋转场景。 |
| 右键拖拽（视角模式） | 缩放场景。 |
| 中键拖拽（视角模式） | 平移场景。 |

**关键观察点**：

1. **实时裁剪的响应性**：在裁剪模式下拖拽平面时，裁剪结果应即时更新——这验证了Observer回调在`InteractionEvent`上的实时性。
2. **模式切换的互斥性**：在不同模式之间切换时，相应Widget会被激活或隐藏——这验证了`On()`/`Off()`的正确操作。
3. **Widget与导航的优先级**：在裁剪或测量模式下操作Widget时，场景不会同时旋转——验证了Widget优先于InteractorStyle的事件分发。
4. **HUD的实时更新**：观察左上角的文字在操作Widget时实时更新——验证了回调中对HUD的更新逻辑。

### 20.5.6 扩展思路

这个示例程序可以作为更复杂应用的基础。以下是一些扩展方向：

1. **添加第二个裁剪平面**：使用两个`vtkImplicitPlaneWidget`和`vtkClipClosedSurface`实现双面裁剪，创建"切片"效果。
2. **添加标量栏（Scalar Bar）**：使用`vtkScalarBarActor`在屏幕上显示颜色映射的图例。
3. **保存裁剪结果**：添加一个键盘快捷键，使用`vtkDataSetWriter`将当前裁剪结果保存为`.vtk`文件。
4. **探针线采样**：结合`vtkLineWidget`和`vtkProbeFilter`，沿用户定义的线段采样标量值并绘制X-Y折线图。
5. **多数据集叠加**：同时显示原始数据的线框和裁剪结果，帮助理解空间关系。
6. **使用第二代Widget**：将`vtkImplicitPlaneWidget`替换为`vtkImplicitPlaneWidget2`，体验第二代Widget的Representation分离设计。

---

## 20.6 本章小结

本章系统性地讲解了VTK中的两大交互定制机制——Widget系统和InteractorStyle系统。以下是本章的核心要点回顾：

### 一、Widget系统架构

Widget是VTK中实现"三维控件"的核心机制。你学习了第一代Widget（基于`vtk3DWidget`）和第二代Widget（基于`vtkAbstractWidget`）的区别：

- 第一代Widget将视觉表现和交互逻辑合并在一个类中，API简单直接。所有Widget都通过`SetInteractor()`绑定交互器，通过`PlaceWidget()`设置初始位置，通过`On()`/`Off()`激活/关闭。
- 第二代Widget分离了Representation（视觉表现）和Widget（行为逻辑），提供了更大的定制灵活性。
- Widget在事件分发链中优先于InteractorStyle——当用户在Widget上操作时，场景导航不会同时触发。

### 二、六个常用3D Widget

| Widget | 核心用途 | 关键输出方法 |
|--------|---------|-------------|
| `vtkBoxWidget` | 六面体区域选择、变换 | `GetPlanes()`, `GetTransform()`, `GetPolyData()` |
| `vtkDistanceWidget` | 三维距离测量 | `GetDistanceRepresentation()->GetDistance()` |
| `vtkImplicitPlaneWidget` | 交互式裁剪/切割平面 | `GetPlane()`, `GetPolyData()` |
| `vtkSphereWidget` | 球形区域选择 | `GetSphere()`, `GetCenter()`, `GetRadius()` |
| `vtkSliderWidget` | 二维滑块参数控制 | `GetSliderRepresentation()->GetValue()` |
| `vtkLineWidget` | 交互式线段 | `GetPoint1()`, `GetPoint2()`, `GetPolyData()` |

### 三、Observer回调模式

Observer模式是Widget与应用程序逻辑之间的桥梁：

- 使用`AddObserver(eventId, callback, clientData)`注册回调函数。
- `clientData`是向回调传递应用上下文的关键机制——通常传递一个应用状态结构体的指针。
- 最常用的三个事件是`StartInteractionEvent`（交互开始）、`InteractionEvent`（交互进行中）和`EndInteractionEvent`（交互结束）。
- `InteractionEvent`在拖拽过程中高频触发，回调应保持轻量。繁重的计算工作应放在`EndInteractionEvent`中。

### 四、自定义InteractorStyle

继承`vtkInteractorStyleTrackballCamera`允许你自定义场景对鼠标和键盘的响应：

- 覆写`OnLeftButtonDown()`、`OnRightButtonDown()`、`OnMouseMove()`、`OnKeyPress()`等方法。
- 未处理的事件应转发给父类方法以保留默认行为。
- 模式切换（View/Clip/Measure等）是自定义Style最常见的应用模式——在不同模式下相同的鼠标操作执行不同的功能。

### 五、与实践的联系

你现在掌握的技能可以用于构建以下实际应用：

- **交互式数据探索工具**：结合多个Widget（裁剪平面+距离测量+区域选择），构建类似ParaView的交互式数据探索界面。
- **CAD模型检查工具**：使用距离测量和角度测量Widget在三维模型上进行尺寸标注和公差检查。
- **医学图像可视化**：使用裁剪平面Widget交互式地浏览CT/MRI数据的内部结构，配合滑块调整窗宽窗位。
- **仿真结果后处理**：使用探针线和裁剪平面交互式地检查仿真结果的局部细节。
- **自定义领域专用工具**：基于InteractorStyle实现特定领域的交互逻辑——例如地质建模中的层位拾取、结构分析中的截面定位。

### 关键类与API速查

| 类 | 主要用途 | 关键方法 |
|----|---------|---------|
| `vtk3DWidget` | 第一代Widget基类 | `SetInteractor()`, `On()`, `Off()`, `PlaceWidget()` |
| `vtkAbstractWidget` | 第二代Widget基类 | `SetInteractor()`, `SetEnabled()`, `SetRepresentation()` |
| `vtkBoxWidget` | 六面体区域选择 | `GetPlanes()`, `GetTransform()`, `GetPolyData()` |
| `vtkDistanceWidget` | 距离测量 | `GetDistanceRepresentation()`, `SetWidgetStateToManipulate()` |
| `vtkImplicitPlaneWidget` | 交互式裁剪平面 | `GetPlane()`, `SetNormal()`, `SetOrigin()`, `GetPolyData()` |
| `vtkSphereWidget` | 球形区域选择 | `GetSphere()`, `GetCenter()`, `GetRadius()` |
| `vtkSliderWidget` | 二维滑块 | `SetRepresentation()`, `GetSliderRepresentation()` |
| `vtkLineWidget` | 交互式线段 | `SetPoint1()`, `SetPoint2()`, `GetPolyData()` |
| `vtkCommand` | 事件定义 | `InteractionEvent`, `StartInteractionEvent`, `EndInteractionEvent` |
| `vtkInteractorStyleTrackballCamera` | 标准交互样式 | `OnLeftButtonDown()`, `OnMouseMove()`, `OnKeyPress()` |
| `vtkPlane` | 隐式平面函数 | `SetNormal()`, `SetOrigin()`, `GetNormal()`, `GetOrigin()` |
| `vtkClipDataSet` | 使用隐式函数裁剪 | `SetClipFunction()`, `SetInputData()`, `SetInsideOut()` |
