# 第四章 基本几何体与数据源

## 本章导读

前两章我们学习了VTK的管道架构和数据模型。你知道了Source是管道的起点、Filter是数据的加工车间、Mapper是数据世界与图形世界的翻译官。你也理解了`SetInputConnection()`如何建立算法之间的"活的连接"，以及MTime如何驱动按需执行。

但到目前为止，你只接触过一个Source：`vtkConeSource`。VTK提供了数十种内置的几何体数据源，覆盖了从简单的点、线、面到复杂的箭头、文字等各种基础形状。这些"积木块"是你在构建可视化场景时的基本构件。

本章的核心目标有三个：

1. **掌握VTK内置几何体数据源的完整版图**——了解每种Source的用途、关键参数和适用场景，能够在需要时迅速选择正确的类。
2. **理解程序化数据生成的两种路径**——使用`vtkProgrammableSource`通过回调函数生成数学曲面，以及手动构建`vtkPoints`和`vtkCellArray`来创造任意形状。
3. **通过完整的代码示例巩固管道的实际运用**——在2x2的多视图布局中同时展示四种不同的几何体，综合运用前三章的知识。

本章内容偏重"查阅型"——4.2节对每种Source的介绍可以当作速查手册，在你日后编写VTK程序时回头翻阅。

---

## 4.1 数据源概述

### 4.1.1 Source在管道中的角色

在VTK的管道架构中，**Source（数据源）是管道的入口点——它是数据的生产者，不消费任何输入数据**。这个定义在类继承体系中表现为：Source对象没有输入端口（Input Port），只有输出端口（Output Port）。

```
                        管道
   +--------------------------------------------------+
   |                                                    |
   |  Source --> Filter --> Mapper --> Actor --> ...    |
   |    ^                                               |
   |    |                                               |
   |  0个输入端口   1个或多个输入端口                      |
   |  1个输出端口   1个或多个输出端口                      |
   |                                                    |
   +--------------------------------------------------+
```

从`vtkAlgorithm`的角度来看，Source是一个特殊化的算法：它的`GetNumberOfInputPorts()`返回0，而普通的Filter至少返回1。这个看似简单的差异决定了Source在管道中的独特地位——它是计算链的终点，是`Update()`向上游追溯时最后到达的节点。

回顾第二章的核心概念：当你调用`Update()`时，Executive沿着`SetInputConnection()`的连接链向上游一级一级追溯，直到找到一个没有输入端口的算法——那就是Source。然后，从这个Source开始，数据向下游流动，经过每一级Filter和Mapper的处理，最终到达屏幕。

**Source本身并不特殊——它只是一个没有Input的Filter。** 这一洞察将Source与Filter统一在同一个概念框架下：Filter接收数据、变换后输出；Source不接收数据、生成后输出。在VTK的类继承体系中，许多Source和Filter共享同一个中间基类，它们之间的差异仅仅在于输入端口的有无。

### 4.1.2 内置数据源 vs 手动构建数据

VTK提供两种方式将数据带入管道：

**方式一：使用内置数据源（Built-in Sources）**

这是快速原型开发和教学演示中最常用的方式。VTK在`FiltersSources`模块中内置了数十种几何体数据源，每一种都能通过简洁的参数接口生成对应的几何形状：

```cpp
// 一行代码创建一个球体数据源，参数全部有默认值
vtkNew<vtkSphereSource> sphere;
// sphere此时已经"知道"如何生成一个球体，但数据尚未计算
// 它记住了参数：Radius=0.5, ThetaResolution=8, PhiResolution=8, ...
```

内置数据源的优势在于：
- **简洁**：一行构造，几行参数设置，无需理解底层几何算法。
- **可靠**：经过了VTK社区数十年的测试和维护。
- **参数化**：通过`SetXXX()`方法随时调整形状，MTime自动跟踪变化。
- **管道兼容**：天然支持`SetInputConnection()`，可以直接接入管道。

**方式二：手动构建数据（Manual Data Construction）**

当你需要的形状超出了内置数据源的能力范围——例如自定义的CAD模型轮廓、科学测量得到的离散采样点、或由外部算法生成的特定拓扑结构——你需要手动构建`vtkPoints`和`vtkCellArray`，然后将它们组装成`vtkPolyData`：

```cpp
vtkNew<vtkPoints> points;
points->InsertNextPoint(0.0, 0.0, 0.0);
points->InsertNextPoint(1.0, 0.0, 0.0);
// ... 继续添加点 ...

vtkNew<vtkCellArray> cells;
// ... 手动定义单元拓扑 ...

vtkNew<vtkPolyData> polyData;
polyData->SetPoints(points);
polyData->SetPolys(cells);
```

手动构建数据的能力是VTK灵活性的核心——它意味着你**不局限于任何预定义形状**，可以创建任何你能用点和面描述的三维结构。4.4节将详细展示这一过程。

### 4.1.3 Source::Update() vs 让Render()触发

在本章及后续章节中，你会频繁看到代码中调用`source->Update()`。让我们明确它和`Render()`的区别：

```cpp
vtkNew<vtkConeSource> cone;

// 方式A：手动Update —— 在渲染之前强制执行
cone->Update();
vtkPolyData* data = cone->GetOutput();  // 此时可以安全地访问数据
int npts = data->GetNumberOfPoints();   // 例如：检查生成的点数

// 方式B：让Render()触发 —— 标准管道用法
vtkNew<vtkPolyDataMapper> mapper;
mapper->SetInputConnection(cone->GetOutputPort());
// ... 加到Actor、Renderer中 ...
renderWindow->Render();  // Render()自动向上游触发cone->Update()
```

**什么时候需要手动调用`Update()`？** 答案很简单：当你需要在渲染之前检查或使用数据时。典型场景包括：

1. **调试与验证**：在渲染前确认数据是否正确生成（检查点数、单元数、数据范围等）。
2. **数据导出**：在将数据写入文件（如STL、VTK）之前，需要确保数据已经生成。
3. **脱离渲染的数据处理**：某些应用只使用VTK的数据生成和处理能力，而不进行可视化——此时没有`Render()`来触发`Update()`，你需要手动调用。
4. **教学演示**：在教程代码中显式调用`Update()`来展示管道的执行时机——正如第二章示例中所做的那样。

**原则是：如果你建立了完整的管道并最终调用了`Render()`，那么中间的手动`Update()`就是多余的。** 在实际的应用代码中，手动`Update()`远少于教程代码中的出现频率——教程中使用它来"窥探"管道的中间状态，帮助你理解数据的流动。

### 4.1.4 本章涉及的VTK类速览

下表列出了本章将详细介绍的所有Source类及其所属模块（均位于`FiltersSources`模块组下）：

| 类名 | 生成的几何体 | 输出数据类型 |
|------|-------------|-------------|
| `vtkConeSource` | 圆锥体（可调边数，可变为棱锥） | vtkPolyData |
| `vtkSphereSource` | 球体（经纬线细分） | vtkPolyData |
| `vtkCylinderSource` | 圆柱体（可开关顶/底面） | vtkPolyData |
| `vtkCubeSource` | 长方体 | vtkPolyData |
| `vtkDiskSource` | 圆盘（可设内外半径） | vtkPolyData |
| `vtkArrowSource` | 箭头（可调尖端和杆部比例） | vtkPolyData |
| `vtkRegularPolygonSource` | 正多边形（可设边数） | vtkPolyData |
| `vtkPlaneSource` | 矩形平面（可细分网格） | vtkPolyData |
| `vtkLineSource` | 线段（可细分中间点） | vtkPolyData |
| `vtkTextSource` | 三维文字 | vtkPolyData |
| `vtkProgrammableSource` | 用户自定义（回调函数生成） | vtkPolyData |

这些Source的输出类型均为`vtkPolyData`——这是VTK中最灵活、最常用的数据格式。你可以用`vtkPolyDataMapper`直接映射它们，然后用`vtkActor`在场景中渲染。

---

## 4.2 常用几何体数据源

本节以速查手册的形式介绍十种最常用的内置几何体数据源。对每种Source，我们说明其用途、列出关键参数、并给出一个简短的使用示例。所有示例假定你已经有了第一章中建立的渲染基础设施（Renderer、RenderWindow、RenderWindowInteractor），因此只展示Source的创建和参数设置部分。

### 4.2.1 vtkConeSource —— 圆锥体

**用途：** 生成一个底面为正多边形的锥体（当边数足够大时近似为圆锥）。这是第一章Cone示例中的核心Source。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetHeight(double)` | 1.0 | 锥体高度（Y轴方向） |
| `SetRadius(double)` | 0.5 | 底面半径 |
| `SetResolution(int)` | 6 | 底面多边形的边数（6=六棱锥，32+接近真圆锥） |
| `SetCenter(double, double, double)` | (0, 0, 0) | 锥体几何中心位置 |
| `SetDirection(double, double, double)` | (1, 0, 0) | 锥体轴向（默认X轴方向） |
| `SetCapping(bool)` | true | 是否生成底面（false则只有锥面，无底面） |

**代码示例：**

```cpp
#include "vtkConeSource.h"
#include "vtkNew.h"

// 生成一个高1.5、底面半径0.6、32边的近似圆锥
vtkNew<vtkConeSource> cone;
cone->SetHeight(1.5);
cone->SetRadius(0.6);
cone->SetResolution(32);
cone->SetCenter(0.0, 0.0, 0.0);
cone->SetDirection(0.0, 1.0, 0.0);  // 锥体尖端指向Y轴正方向
// 输出: vtkPolyData (顶点、三角面片、法向量)
```

**实用提示：** 将`SetResolution(4)`与`SetCapping(false)`结合，可以得到一个四棱锥（金字塔形）的侧面。将`SetCapping(false)`与`SetRadius(0)`结合则生成一个没有底面的真圆锥。`SetDirection()`在需要将锥体指向特定方向（如表示向量方向）时非常有用。

---

### 4.2.2 vtkSphereSource —— 球体

**用途：** 生成一个用经纬线（Latitude-Longitude）方式细分的球体表面。这是最常用的几何体之一，常用于表示点数据的位置标记（配合Glyph使用，见后续章节）、碰撞检测的包围球、或组合成分子模型。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetRadius(double)` | 0.5 | 球体半径 |
| `SetThetaResolution(int)` | 8 | 经线方向（绕Y轴）的细分段数 |
| `SetPhiResolution(int)` | 8 | 纬线方向（从北极到南极）的细分段数 |
| `SetStartTheta(double)` | 0.0 | 经度起始角（度） |
| `SetEndTheta(double)` | 360.0 | 经度终止角（度） |
| `SetStartPhi(double)` | 0.0 | 纬度起始角（度，0=北极） |
| `SetEndPhi(double)` | 180.0 | 纬度终止角（度，180=南极） |
| `SetCenter(double, double, double)` | (0, 0, 0) | 球心位置 |
| `SetLatLongTessellation(bool)` | false | 是否使用经纬线细分（true=四边形，false=三角形） |

**代码示例：**

```cpp
#include "vtkSphereSource.h"
#include "vtkNew.h"

// 生成一个半径1.0、经线50段、纬线50段的光滑球体
vtkNew<vtkSphereSource> sphere;
sphere->SetRadius(1.0);
sphere->SetThetaResolution(50);
sphere->SetPhiResolution(50);
sphere->SetCenter(0.0, 0.0, 0.0);
// 输出: vtkPolyData
```

```cpp
// 生成一个半球（通过限制纬度范围）
vtkNew<vtkSphereSource> hemisphere;
hemisphere->SetRadius(1.0);
hemisphere->SetThetaResolution(50);
hemisphere->SetPhiResolution(25);
hemisphere->SetStartPhi(0.0);    // 从北极开始
hemisphere->SetEndPhi(90.0);     // 到赤道结束——生成北半球
```

**实用提示：** `ThetaResolution`和`PhiResolution`是控制球体光滑度的核心参数——值越大球体越光滑，但面片数量以乘积形式增长（面片数 = ThetaResolution x PhiResolution x 2）。对于仅用作位置标记的小球，分辨率设为8-12即可；对于大尺寸的主要展示对象，建议设为32-64。通过调节`StartTheta`/`EndTheta`和`StartPhi`/`EndPhi`可以创建球体的一部分（如半球、四分之一球、球面切片），在需要展示"切开"效果时非常有用。

---

### 4.2.3 vtkCylinderSource —— 圆柱体

**用途：** 生成一个圆柱体，可独立控制是否生成顶面和底面（Capping）。常用于表示柱状结构、管道、或数据可视化中的柱状图元素。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetHeight(double)` | 1.0 | 柱体高度（Y轴方向） |
| `SetRadius(double)` | 0.5 | 柱体半径 |
| `SetResolution(int)` | 6 | 圆周方向的细分段数 |
| `SetCenter(double, double, double)` | (0, 0, 0) | 柱体几何中心 |
| `SetCapping(bool)` | true | 是否生成顶面和底面 |

**代码示例：**

```cpp
#include "vtkCylinderSource.h"
#include "vtkNew.h"

// 生成一个高2.0、半径0.4、64边（光滑曲面）的圆柱体
vtkNew<vtkCylinderSource> cylinder;
cylinder->SetHeight(2.0);
cylinder->SetRadius(0.4);
cylinder->SetResolution(64);
cylinder->SetCenter(0.0, 0.0, 0.0);
cylinder->CappingOn();   // 生成顶面和底面（默认即为开启）
// 输出: vtkPolyData
```

```cpp
// 生成一个没有顶底面的管道（空心圆柱侧面）
vtkNew<vtkCylinderSource> tube;
tube->SetHeight(3.0);
tube->SetRadius(0.5);
tube->SetResolution(32);
tube->CappingOff();  // 只有侧面——一个真正的"管道"
```

**实用提示：** `SetResolution(4)`生成一个四棱柱（截面为正方形），`SetResolution(3)`生成三棱柱。`CappingOff()`在需要展示物体内部或创建开放式管道时非常关键——例如在CAD可视化中展示一个贯穿的孔洞。注意圆柱体的轴向是固定的（沿Y轴），如需其他方向，可以通过`vtkActor::SetOrientation()`在渲染时旋转。

---

### 4.2.4 vtkCubeSource —— 长方体

**用途：** 生成一个轴对齐的长方体（所有面与坐标轴平行）。这是最基础的几何体，广泛用于表示包围盒（Bounding Box）、占位符、或体素化的三维像素。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetXLength(double)` | 1.0 | X方向长度 |
| `SetYLength(double)` | 1.0 | Y方向长度 |
| `SetZLength(double)` | 1.0 | Z方向长度 |
| `SetCenter(double, double, double)` | (0, 0, 0) | 长方体几何中心 |

**代码示例：**

```cpp
#include "vtkCubeSource.h"
#include "vtkNew.h"

// 生成一个2x1x0.5的长方体（类似一块板砖）
vtkNew<vtkCubeSource> cube;
cube->SetXLength(2.0);
cube->SetYLength(1.0);
cube->SetZLength(0.5);
cube->SetCenter(0.0, 0.0, 0.0);
// 输出: vtkPolyData (8个顶点, 12个三角面片)
```

**实用提示：** `vtkCubeSource`生成的始终是轴对齐的长方体。如果你需要一个可以任意旋转的立方体，在渲染时通过`actor->SetOrientation()`来实现，而不是通过Source参数。三个方向的长度独立设置意味着你可以方便地创建各种比例的长方体——这对于数据可视化中的三维柱状图尤其有用。

---

### 4.2.5 vtkDiskSource —— 圆盘

**用途：** 生成一个位于XY平面的圆盘（或圆环）。可以设置内半径和外半径，从而生成从实心圆盘到圆环之间的任意形状。常用于表示二维平面中的圆形区域、垫圈形状、或饼图的扇形片。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetInnerRadius(double)` | 0.0 | 内半径（0=实心圆盘，>0=圆环） |
| `SetOuterRadius(double)` | 0.5 | 外半径 |
| `SetCircumferentialResolution(int)` | 6 | 圆周方向的细分段数 |
| `SetRadialResolution(int)` | 1 | 径向的细分段数 |

**代码示例：**

```cpp
#include "vtkDiskSource.h"
#include "vtkNew.h"

// 生成一个外半径1.0的实心圆盘（内半径为0）
vtkNew<vtkDiskSource> disk;
disk->SetOuterRadius(1.0);
disk->SetInnerRadius(0.0);
disk->SetCircumferentialResolution(64);  // 64边近似圆形
// 输出: vtkPolyData

// 生成一个圆环（垫圈形状）
vtkNew<vtkDiskSource> ring;
ring->SetOuterRadius(1.0);
ring->SetInnerRadius(0.6);     // 内半径 > 0 => 圆环
ring->SetCircumferentialResolution(64);
ring->SetRadialResolution(2);  // 径向2段（内圈和外圈之间有一段过渡）
```

**实用提示：** `vtkDiskSource`生成的圆盘始终位于XY平面（法向为Z轴）。如果需要在其他平面上显示，通过`vtkActor::SetOrientation()`旋转。`CircumferentialResolution`控制圆的精细程度——数值越大越接近真圆。`RadialResolution`在内半径和外半径之间插入更多的顶点环，通常设为1即可，除非你计划对圆盘进行变形处理。

---

### 4.2.6 vtkArrowSource —— 箭头

**用途：** 生成一个三维箭头，由圆柱形的杆部（Shaft）和圆锥形的尖端（Tip）组成。这是矢量场可视化中最常用的符号——通过Glyph机制在数据集的每个点上放置一个箭头来表示该点的矢量方向和大小（详见第八章）。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetTipLength(double)` | 0.35 | 尖端长度（占箭头总长的比例） |
| `SetTipRadius(double)` | 0.1 | 尖端底部半径 |
| `SetShaftRadius(double)` | 0.03 | 杆部半径 |
| `SetTipResolution(int)` | 6 | 尖端的圆周细分（同Cone的Resolution） |
| `SetShaftResolution(int)` | 6 | 杆部的圆周细分（同Cylinder的Resolution） |

**代码示例：**

```cpp
#include "vtkArrowSource.h"
#include "vtkNew.h"

// 生成一个标准箭头
vtkNew<vtkArrowSource> arrow;
arrow->SetTipLength(0.35);     // 尖端占35%的总长
arrow->SetTipRadius(0.1);      // 尖端底部半径
arrow->SetShaftRadius(0.03);   // 杆部半径
arrow->SetTipResolution(32);   // 尖端光滑
arrow->SetShaftResolution(32); // 杆部光滑
// 输出: vtkPolyData (默认方向为X轴正方向)
```

```cpp
// 生成一个粗壮的短箭头（用于突出显示）
vtkNew<vtkArrowSource> boldArrow;
boldArrow->SetTipLength(0.5);    // 尖端占一半
boldArrow->SetTipRadius(0.2);    // 大尖端
boldArrow->SetShaftRadius(0.08); // 粗杆
```

**实用提示：** `vtkArrowSource`生成的箭头默认指向X轴正方向。在矢量场可视化中，`vtkGlyph3D`（详见第八章）会自动根据每个点的矢量数据调整箭头的方向和大小，你不需要手动旋转箭头。直接使用时，通过`actor->SetOrientation()`来改变箭头指向。

---

### 4.2.7 vtkRegularPolygonSource —— 正多边形

**用途：** 生成一个位于XY平面的正多边形（正三角形、正方形、正五边形……）。当边数足够大时趋近于圆。这是最简单的平面几何体Source，常用于生成截面形状、符号标记、或自定义拉伸体的底面。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetNumberOfSides(int)` | 6 | 多边形的边数 |
| `SetRadius(double)` | 0.5 | 外接圆半径 |
| `SetCenter(double, double, double)` | (0, 0, 0) | 多边形中心位置 |
| `SetGeneratePolygon(bool)` | true | 是否生成填充面（false则只生成轮廓线） |
| `SetNormal(double, double, double)` | (0, 0, 1) | 法向量方向（多边形所在平面的朝向） |

**代码示例：**

```cpp
#include "vtkRegularPolygonSource.h"
#include "vtkNew.h"

// 生成一个正五边形
vtkNew<vtkRegularPolygonSource> pentagon;
pentagon->SetNumberOfSides(5);
pentagon->SetRadius(1.0);
pentagon->SetCenter(0.0, 0.0, 0.0);
pentagon->SetGeneratePolygon(true);   // 生成填充面
// 输出: vtkPolyData

// 生成一个仅轮廓的正六边形（用来表示边框）
vtkNew<vtkRegularPolygonSource> outline;
outline->SetNumberOfSides(6);
outline->SetRadius(1.0);
outline->SetGeneratePolygon(false);    // 仅生成轮廓线，不填充
```

**实用提示：** `SetGeneratePolygon(false)`生成的多边形仅包含边界线段，适合用作轮廓标记或线框装饰。将`SetNumberOfSides(4)`与`SetRadius(sqrt(2)/2)`结合可以生成一个正方形——不过对于正方形，`vtkPlaneSource`更为直接。正多边形的法向量可以通过`SetNormal()`调整，这使得你可以在任意平面上生成正多边形。

---

### 4.2.8 vtkPlaneSource —— 矩形平面

**用途：** 生成一个矩形平面（由两个三角形组成），可以进一步细分为四边形网格。这是最灵活的面状几何体Source，广泛用于创建地面平面、截面切片、纹理映射底板、以及作为变形（Warp）操作的输入。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetOrigin(double, double, double)` | (-0.5, -0.5, 0.0) | 平面左下角的原点坐标 |
| `SetPoint1(double, double, double)` | (0.5, -0.5, 0.0) | 定义平面第一条边的终点（从Origin出发） |
| `SetPoint2(double, double, double)` | (-0.5, 0.5, 0.0) | 定义平面第二条边的终点（从Origin出发） |
| `SetXResolution(int)` | 1 | X方向（Point1方向）的细分段数 |
| `SetYResolution(int)` | 1 | Y方向（Point2方向）的细分段数 |
| `SetCenter(double, double, double)` | (0, 0, 0) | （间接）平面的几何中心 |

**代码示例：**

```cpp
#include "vtkPlaneSource.h"
#include "vtkNew.h"

// 生成一个10x10的细分平面（100个四边形），用于变形演示
vtkNew<vtkPlaneSource> plane;
plane->SetOrigin(-5.0, -5.0, 0.0);   // 左下角
plane->SetPoint1(5.0, -5.0, 0.0);    // 右下角（X方向，总长10）
plane->SetPoint2(-5.0, 5.0, 0.0);    // 左上角（Y方向，总长10）
plane->SetXResolution(10);            // X方向10段 => 11x11个顶点
plane->SetYResolution(10);            // Y方向10段 => 共121个顶点, 200个三角形
// 输出: vtkPolyData
```

```cpp
// 生成一个简单的正方形平面（默认参数）
vtkNew<vtkPlaneSource> square;
// 默认: Origin=(-0.5,-0.5,0), Point1=(0.5,-0.5,0), Point2=(-0.5,0.5,0)
// 生成一个1x1的正方形，位于XY平面，中心在原点
// 输出: 4个顶点, 2个三角形
```

**实用提示：** `vtkPlaneSource`的真正威力在于`SetXResolution()`和`SetYResolution()`——它们将平面细分为规则的四边形网格。这个网格随后可以作为`vtkWarpScalar`（根据标量值将顶点沿法向偏移）的输入，将平面变形为三维曲面——这正是标量场可视化的经典技术。平面可以位于空间的任意位置和朝向：通过设置`Origin`、`Point1`和`Point2`，你可以定义任意大小和朝向的矩形平面。

---

### 4.2.9 vtkLineSource —— 线段

**用途：** 生成一条由两个端点定义的线段，可以通过`SetResolution()`在端点之间插入更多的等间距采样点。常用于绘制坐标轴、连接线、或轨迹路径。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetPoint1(double, double, double)` | (-0.5, 0.0, 0.0) | 线段起点 |
| `SetPoint2(double, double, double)` | (0.5, 0.0, 0.0) | 线段终点 |
| `SetResolution(int)` | 1 | 两点之间的细分段数（1=只有起点和终点） |

**代码示例：**

```cpp
#include "vtkLineSource.h"
#include "vtkNew.h"

// 生成一条从(0,0,0)到(1,1,1)的对角线段
vtkNew<vtkLineSource> line;
line->SetPoint1(0.0, 0.0, 0.0);
line->SetPoint2(1.0, 1.0, 1.0);
line->SetResolution(1);  // 只有起点和终点，中间没有采样点
// 输出: vtkPolyData (一条polyline)

// 生成一条有中间采样点的曲线路径（此处为直线，但点被细分了）
vtkNew<vtkLineSource> sampledLine;
sampledLine->SetPoint1(0.0, 0.0, 0.0);
sampledLine->SetPoint2(1.0, 0.0, 0.0);
sampledLine->SetResolution(10);  // 在两点之间插入9个中间点 => 共11个点
// 细分后的线段可用于后续的变形或采样操作
```

**实用提示：** 在VTK中，线段的渲染需要特别设置——默认情况下`vtkPolyDataMapper`会将线段渲染为细线。如果要让线段更加可见，可以通过`actor->GetProperty()->SetLineWidth(3.0)`来设置线宽，以及`actor->GetProperty()->SetColor(r, g, b)`来设置颜色。`SetResolution()`在直线段上插入中间点的目的通常不是为了渲染（因为直线只需要两个端点），而是为了后续处理——例如，将这条线段作为`vtkProbeFilter`的采样路径，在沿线的多个位置采样标量场数据。

---

### 4.2.10 vtkTextSource —— 三维文字

**用途：** 生成指定字符串的三维多边形几何体。文字以折线形式表示，可以在三维场景中作为标签、标题或注释显示。

**关键参数：**

| 方法 | 默认值 | 说明 |
|------|--------|------|
| `SetText(const char*)` | "" | 要生成的文字内容 |
| `SetForegroundColor(double, double, double)` | (1, 1, 1) | 文字颜色（RGB，0-1范围） |
| `SetBackgroundColor(double, double, double)` | (0, 0, 0) | 背景颜色 |
| `SetBackupPropEnabled(bool)` | false | 是否显示背景 |

**代码示例：**

```cpp
#include "vtkTextSource.h"
#include "vtkNew.h"
#include "vtkPolyDataMapper.h"

// 生成三维文字 "VTK 9.5.2"
vtkNew<vtkTextSource> textSource;
textSource->SetText("VTK 9.5.2");
textSource->SetForegroundColor(1.0, 0.8, 0.2); // 金黄色文字
textSource->SetBackgroundColor(0.1, 0.1, 0.3); // 深蓝背景
textSource->BackingOn();  // 显示背景色块

// 注意：vtkTextSource输出的是vtkPolyData，但其中包含的是折线（多段线）
// 因此需要使用合适的mapper
vtkNew<vtkPolyDataMapper> textMapper;
textMapper->SetInputConnection(textSource->GetOutputPort());
```

**实用提示：** `vtkTextSource`生成的是基于矢量字体的三维折线轮廓。文字的质量取决于VTK编译时是否启用了对FreeType等字体渲染库的支持。如果你需要高质量的屏幕文字（始终面向相机），应该优先考虑`vtkTextActor`（用于二维覆盖文字）或`vtkVectorText`（生成三维文字但质量更好）。`vtkTextSource`的优势在于简单——不需要额外的字体文件，直接生成可用的三维文字几何。

---

## 4.3 程序化数据源

### 4.3.1 vtkProgrammableSource 简介

内置数据源覆盖了常见几何形状，但如果你的需求超出了预定义形状的范围——例如生成一个数学曲面、导入自定义算法产生的点云、或从外部传感器读取实时数据——你需要一个能执行自定义代码的数据源。**`vtkProgrammableSource`正是为此而设计的。**

`vtkProgrammableSource`允许你注册一个C++回调函数（或静态方法），该函数将在Source被要求生成数据时被调用。在这个回调函数中，你可以直接操作`vtkPolyData`的输出——创建点、定义单元、附加属性数据——就像你完全控制了一整个Source的内部实现一样。

### 4.3.2 SetExecuteMethod 机制

`vtkProgrammableSource`的核心方法是`SetExecuteMethod()`，它接受两个参数：一个回调函数的函数指针，以及一个可选的`void*`用户数据指针：

```cpp
// 回调函数签名
void MyExecuteFunc(void* arg);
```

这个回调函数在`vtkProgrammableSource`的`RequestData()`阶段被调用——也就是在管道执行到该Source时。在回调函数内部，你需要：

1. 通过`vtkProgrammableSource::GetPolyDataOutput()`获取输出对象的指针。
2. 构建`vtkPoints`并填充顶点坐标。
3. 构建`vtkCellArray`并定义单元拓扑（通常是三角形或多边形）。
4. 将点和单元设置到输出PolyData上。

### 4.3.3 数学曲面示例：z = sin(x) * cos(y)

下面的示例使用`vtkProgrammableSource`生成一个数学曲面——z = sin(x) * cos(y)，这是一个经典的鞍状曲面（波形曲面）。

```cpp
#include "vtkActor.h"
#include "vtkCamera.h"
#include "vtkCellArray.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPoints.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProgrammableSource.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"

#include <cmath>

// ============================================================================
// 回调函数：生成数学曲面 z = sin(x) * cos(y)
// ============================================================================
void GenerateMathSurface(void* arg)
{
  // 获取回调参数——指向vtkProgrammableSource的指针
  vtkProgrammableSource* source = static_cast<vtkProgrammableSource*>(arg);

  // 获取输出PolyData
  vtkPolyData* output = source->GetPolyDataOutput();

  // --- 参数定义 ---
  const int gridSize = 50;            // 网格分辨率（50x50个顶点）
  const double xMin = -3.0, xMax = 3.0;
  const double yMin = -3.0, yMax = 3.0;
  const double dx = (xMax - xMin) / (gridSize - 1);
  const double dy = (yMax - yMin) / (gridSize - 1);

  // --- 第一步：生成顶点 ---
  vtkNew<vtkPoints> points;
  points->SetNumberOfPoints(gridSize * gridSize);

  for (int j = 0; j < gridSize; ++j)
  {
    double y = yMin + j * dy;
    for (int i = 0; i < gridSize; ++i)
    {
      double x = xMin + i * dx;
      double z = std::sin(x) * std::cos(y);  // 数学曲面公式
      points->SetPoint(j * gridSize + i, x, y, z);
    }
  }

  // --- 第二步：生成三角形单元 ---
  vtkNew<vtkCellArray> triangles;

  for (int j = 0; j < gridSize - 1; ++j)
  {
    for (int i = 0; i < gridSize - 1; ++i)
    {
      // 每个四边形分为两个三角形
      // 顶点索引：  (i,j) -- (i+1,j)
      //              |   \     |
      //            (i,j+1) -- (i+1,j+1)
      vtkIdType a = j * gridSize + i;
      vtkIdType b = j * gridSize + (i + 1);
      vtkIdType c = (j + 1) * gridSize + i;
      vtkIdType d = (j + 1) * gridSize + (i + 1);

      // 第一个三角形: a -> b -> c
      vtkNew<vtkTriangle> tri1;
      tri1->GetPointIds()->SetId(0, a);
      tri1->GetPointIds()->SetId(1, b);
      tri1->GetPointIds()->SetId(2, c);
      triangles->InsertNextCell(tri1);

      // 第二个三角形: b -> d -> c
      vtkNew<vtkTriangle> tri2;
      tri2->GetPointIds()->SetId(0, b);
      tri2->GetPointIds()->SetId(1, d);
      tri2->GetPointIds()->SetId(2, c);
      triangles->InsertNextCell(tri2);
    }
  }

  // --- 第三步：组装到输出PolyData ---
  output->SetPoints(points);
  output->SetPolys(triangles);
}

int main(int argc, char* argv[])
{
  // --- 渲染基础设施 ---
  vtkNew<vtkRenderer> renderer;
  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(600, 600);
  renderWindow->SetWindowName("Math Surface: z = sin(x) * cos(y)");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  // --- 创建程序化数据源 ---
  vtkNew<vtkProgrammableSource> progSource;
  progSource->SetExecuteMethod(GenerateMathSurface, progSource);

  // --- 构建管道 ---
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputConnection(progSource->GetOutputPort());

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(0.3, 0.6, 0.9);  // 浅蓝色
  actor->GetProperty()->SetEdgeVisibility(1);       // 显示网格边线
  actor->GetProperty()->SetEdgeColor(0.0, 0.0, 0.2);

  renderer->AddActor(actor);
  renderer->SetBackground(0.15, 0.15, 0.15);       // 深灰背景
  renderer->GetActiveCamera()->SetPosition(8, 6, 10);
  renderer->GetActiveCamera()->SetFocalPoint(0, 0, 0);
  renderer->ResetCamera();

  renderWindow->Render();
  interactor->Start();

  return 0;
}
```

**代码要点解析：**

1. **回调函数的`void* arg`参数**：通过`SetExecuteMethod(func, this)`将`vtkProgrammableSource`自身的指针传递给回调函数，这样回调函数就可以通过`source->GetPolyDataOutput()`获取输出对象。

2. **网格拓扑**：50x50的顶点网格产生了49x49=2401个四边形，每个四边形拆分为2个三角形，共4802个三角形。顶点总数是2500个。这是一个典型的结构化网格生成模式。

3. **使用`vtkNew<vtkTriangle>`辅助构建**：虽然我们可以直接通过`InsertNextCell(3, ids)`等原始接口构建三角形，但使用`vtkTriangle`对象的`GetPointIds()->SetId()`在逻辑上更清晰。两种方式都将在4.4节中详细讲解。

4. **法向量**：本例没有为曲面计算法向量。在实际渲染中，VTK的Mapper可以通过`vtkPolyDataNormals`过滤器自动计算法向量以获得正确的光照效果（在管道中插入一个`vtkPolyDataNormals`过滤器即可）。对于`vtkProgrammableSource`生成的任意曲面，建议在管道中添加法向量计算：

```cpp
vtkNew<vtkPolyDataNormals> normals;
normals->SetInputConnection(progSource->GetOutputPort());
mapper->SetInputConnection(normals->GetOutputPort());
```

**`vtkProgrammableSource`的局限性：** 虽然功能强大，但它有一个显著的缺点——回调函数在每次管道更新时被调用（取决于MTime机制）。如果你的回调函数中包含昂贵的计算，需要注意性能影响。此外，回调函数不能是类的非静态成员方法（除非通过额外的包装），这在与面向对象设计的代码集成时可能不太方便。对于复杂的自定义数据生成，手动构建`vtkPolyData`（见4.4节）通常更为直接和可控。

---

## 4.4 手动构建点与单元

当内置数据源和`vtkProgrammableSource`都不能满足你的需求时——或者当你需要完全掌控数据的每一处细节时——你需要手动构建VTK的数据结构。本节将详细讲解如何从零开始创建`vtkPolyData`，这是VTK中最灵活、最常用的数据格式。

### 4.4.1 构建的基本流程

手动构建`vtkPolyData`的流程遵循以下步骤：

```
1. 创建 vtkPoints 对象
      ↓
2. 添加顶点坐标（InsertNextPoint / SetPoint）
      ↓
3. 创建 vtkCellArray 对象
      ↓
4. 定义单元拓扑（InsertNextCell 配合顶点索引）
      ↓
5. 将 Points 和 CellArray 组装到 vtkPolyData
      ↓
6. （可选）添加属性数据（标量、矢量等）
```

这个流程的核心思想是：**顶点坐标和单元拓扑是分离的**。`vtkPoints`只存储顶点的空间位置（x, y, z），`vtkCellArray`只存储单元如何由顶点组成（顶点索引列表）。这种分离设计使得同一组顶点可以被多个不同的单元集合复用——例如，同一个点集可以同时定义三角形面和线框边。

### 4.4.2 创建 vtkPoints

`vtkPoints`是VTK中存储顶点坐标的标准容器。它内部使用`vtkDataArray`（通常是`vtkFloatArray`）来高效存储坐标数据。

```cpp
#include "vtkPoints.h"
#include "vtkNew.h"

vtkNew<vtkPoints> points;

// 方式一：顺序插入（自动分配索引 0, 1, 2, ...）
points->InsertNextPoint(0.0, 0.0, 0.0);  // 索引 0
points->InsertNextPoint(1.0, 0.0, 0.0);  // 索引 1
points->InsertNextPoint(0.0, 1.0, 0.0);  // 索引 2
points->InsertNextPoint(0.0, 0.0, 1.0);  // 索引 3

// 方式二：指定索引设置（适合提前知道顶点数量的场景）
points->SetNumberOfPoints(4);
points->SetPoint(0, 0.0, 0.0, 0.0);
points->SetPoint(1, 1.0, 0.0, 0.0);
points->SetPoint(2, 0.0, 1.0, 0.0);
points->SetPoint(3, 0.0, 0.0, 1.0);

// 查询
vtkIdType numPoints = points->GetNumberOfPoints();
double* coords = points->GetPoint(0);       // 返回指向内部数组的指针
double x = coords[0], y = coords[1], z = coords[2];
```

`InsertNextPoint()`适用于无法提前预知顶点数量的场景；`SetPoint()`适用于已知顶点数量的场景，且性能更好（避免了动态数组扩容的开销）。

### 4.4.3 创建 vtkCellArray

`vtkCellArray`存储单元的拓扑信息——每个单元是哪些顶点的组合。VTK支持多种单元类型，以下是最常用的几种：

| 单元类型常量 | 含义 | 最少顶点数 | 说明 |
|-------------|------|----------|------|
| `VTK_VERTEX` | 点 | 1 | 孤立的点精灵 |
| `VTK_LINE` | 线段 | 2 | 两个顶点之间的线段 |
| `VTK_POLY_LINE` | 折线 | 2+ | 多个连续的线段 |
| `VTK_TRIANGLE` | 三角形 | 3 | 三个顶点的三角面 |
| `VTK_QUAD` | 四边形 | 4 | 四个顶点的四边形面 |
| `VTK_POLYGON` | 多边形 | 3+ | 任意数量的顶点构成的多边形 |
| `VTK_TETRA` | 四面体 | 4 | 四个顶点的体单元 |
| `VTK_TRIANGLE_STRIP` | 三角形带 | 3+ | 连续三角形共享边 |

构建`vtkCellArray`有以下几种主要方式：

```cpp
#include "vtkCellArray.h"
#include "vtkNew.h"

vtkNew<vtkCellArray> cells;

// ---- 方式一：使用 InsertNextCell 的便捷重载（VTK 9 推荐） ----

// 添加两个三角形构成一个正方形
// 三角形顶点按逆时针排列（从法向量指向的一侧看）
cells->InsertNextCell(3);  // 声明接下来的3个顶点构成一个单元
cells->InsertCellPoint(0); // 三角形的顶点0
cells->InsertCellPoint(1); // 三角形的顶点1
cells->InsertCellPoint(2); // 三角形的顶点2

cells->InsertNextCell(3);
cells->InsertCellPoint(1);
cells->InsertCellPoint(3);
cells->InsertCellPoint(2);

// ---- 方式二：使用 vtkIdList + InsertNextCell ----

vtkNew<vtkIdList> ids;
ids->SetNumberOfIds(3);
ids->SetId(0, 0);
ids->SetId(1, 1);
ids->SetId(2, 2);
cells->InsertNextCell(ids);  // 整批插入

// ---- 方式三：使用原始数组一次性传递大量数据（高性能场景） ----
// 这涉及直接操作vtkCellArray的内部存储，适合数据量极大时的批量构建
```

**关于顶点顺序的重要约定：** 对于面单元（三角形、四边形、多边形），VTK要求顶点按**逆时针顺序**排列（从单元的外部观察）。这个顺序决定了面法向量的方向——法向量根据右手定则由顶点顺序导出。如果顶点顺序混乱，渲染结果中的光照将不正确（面可能看起来"黑"的，因为法向量指向了内部）。

### 4.4.4 组装 vtkPolyData

有了`vtkPoints`和`vtkCellArray`之后，将它们组装成`vtkPolyData`：

```cpp
#include "vtkPolyData.h"

vtkNew<vtkPolyData> polyData;
polyData->SetPoints(points);   // 设置顶点
polyData->SetPolys(cells);     // 设置多边形面单元

// 如果还需要线和点单元：
// polyData->SetLines(lineCells);   // 设置线单元
// polyData->SetVerts(vertCells);    // 设置点单元
```

`vtkPolyData`支持同时包含多种单元类型——例如，同一个PolyData可以既有三角形面（polys）又有边界线（lines）和孤立点（verts）。这些不同的单元类型共享同一套顶点集。

### 4.4.5 完整示例：手动构建一个四面体

下面的示例展示如何从零开始创建一个四面体（由四个三角形面组成的三维体），并在窗口中渲染出来。四面体是最简单的三维体单元（4个顶点、4个三角面），是理解三维几何数据结构的理想教学示例。

```cpp
// ============================================================================
// TetrahedronExample.cxx
// 手动构建一个四面体并渲染
// ============================================================================

#include "vtkActor.h"
#include "vtkCellArray.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPoints.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"

#include <iostream>

int main(int argc, char* argv[])
{
  // ====================================================================
  // 第一步：手动定义四面体的4个顶点
  // ====================================================================
  // 四面体的顶点坐标（一个正四面体，中心在原点附近）
  // 高度约为单位长度
  vtkNew<vtkPoints> points;
  points->InsertNextPoint(0.0,  0.0,  1.0);  // 顶点0：顶部
  points->InsertNextPoint(0.0,  0.943, -0.333); // 顶点1：底部后方
  points->InsertNextPoint(-0.816, -0.471, -0.333); // 顶点2：底部左前方
  points->InsertNextPoint(0.816, -0.471, -0.333);  // 顶点3：底部右前方

  std::cout << "=== 手动构建四面体 ===" << std::endl;
  std::cout << "顶点数量: " << points->GetNumberOfPoints() << std::endl;
  for (vtkIdType i = 0; i < points->GetNumberOfPoints(); ++i)
  {
    double p[3];
    points->GetPoint(i, p);
    std::cout << "  顶点 " << i << ": ("
              << p[0] << ", " << p[1] << ", " << p[2] << ")" << std::endl;
  }

  // ====================================================================
  // 第二步：定义四面体的4个三角面的顶点索引
  // ====================================================================
  // 每个面由3个顶点索引组成，顶点按逆时针排列（从外部看向该面）
  vtkNew<vtkCellArray> triangles;

  // 面0: 底部三角形 (顶点3 -> 顶点2 -> 顶点1) —— 从下方看，逆时针
  vtkNew<vtkTriangle> face0;
  face0->GetPointIds()->SetId(0, 3);
  face0->GetPointIds()->SetId(1, 2);
  face0->GetPointIds()->SetId(2, 1);
  triangles->InsertNextCell(face0);

  // 面1: 侧面 (顶点0, 顶点1, 顶点2)
  vtkNew<vtkTriangle> face1;
  face1->GetPointIds()->SetId(0, 0);
  face1->GetPointIds()->SetId(1, 1);
  face1->GetPointIds()->SetId(2, 2);
  triangles->InsertNextCell(face1);

  // 面2: 侧面 (顶点0, 顶点2, 顶点3)
  vtkNew<vtkTriangle> face2;
  face2->GetPointIds()->SetId(0, 0);
  face2->GetPointIds()->SetId(1, 2);
  face2->GetPointIds()->SetId(2, 3);
  triangles->InsertNextCell(face2);

  // 面3: 侧面 (顶点0, 顶点3, 顶点1)
  vtkNew<vtkTriangle> face3;
  face3->GetPointIds()->SetId(0, 0);
  face3->GetPointIds()->SetId(1, 3);
  face3->GetPointIds()->SetId(2, 1);
  triangles->InsertNextCell(face3);

  std::cout << "三角面数量: " << triangles->GetNumberOfCells() << std::endl;
  std::cout << std::endl;

  // ====================================================================
  // 第三步：组装 vtkPolyData
  // ====================================================================
  vtkNew<vtkPolyData> tetrahedron;
  tetrahedron->SetPoints(points);
  tetrahedron->SetPolys(triangles);

  // ====================================================================
  // 第四步：构建渲染管道
  // ====================================================================
  // 注意：这里使用 SetInputData 而不是 SetInputConnection
  // 因为 tetrahedron 不是一个 vtkAlgorithm —— 它是手动构建的静态数据
  // 这样使用是合理的（见2.2.3节的讨论）
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputData(tetrahedron);

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(0.2, 0.7, 0.3);   // 绿色表面
  actor->GetProperty()->SetEdgeVisibility(1);        // 显示边线
  actor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.1); // 深灰色边线
  actor->GetProperty()->SetLineWidth(2.0);

  // ====================================================================
  // 第五步：设置渲染基础设施并显示
  // ====================================================================
  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->SetBackground(0.15, 0.15, 0.15);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(500, 500);
  renderWindow->SetWindowName("Tetrahedron - Manual Construction");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  renderWindow->Render();
  std::cout << "渲染完成。可旋转场景观察四面体的四个三角面。" << std::endl;
  interactor->Start();

  return 0;
}
```

**代码要点解析：**

1. **SetInputData vs SetInputConnection**：本例使用了`mapper->SetInputData(tetrahedron)`而不是`SetInputConnection()`，因为`tetrahedron`是一个手动创建的`vtkPolyData`对象，不是`vtkAlgorithm`——它没有输出端口，因此不存在`GetOutputPort()`。这种场景正是2.2.3节中讨论的`SetInputData()`合理使用场景之一。

2. **顶点坐标的选取**：正四面体的顶点坐标基于几何计算得出。顶点0位于正上方（z=1），顶点1-3均匀分布在底面的外接圆上，z坐标均为-1/3。这样构成了一个以原点附近为中心的正四面体。

3. **使用`vtkTriangle`辅助对象**：`vtkTriangle`（注意不是`vtkTriangle`——VTK中确实有这个类，全名为`vtkTriangle`）提供了`GetPointIds()`方法，返回一个`vtkIdList`用于设置三个顶点的全局索引。这是一种清晰易读的单元构建方式。

4. **面的可见性取决于顶点顺序**：如果你发现渲染出的四面体某些面是"黑色"的（不响应光照），检查该面对应的三角面顶点是否按逆时针排列（从外部观察时）。

### 4.4.6 更简洁的单元构建方式

对于简单的单元拓扑，你还可以使用更底层的接口，避免创建`vtkTriangle`临时对象：

```cpp
// 方式A：使用 InsertNextCell(npts) + InsertCellPoint
vtkNew<vtkCellArray> cells;
cells->InsertNextCell(3);
cells->InsertCellPoint(0);
cells->InsertCellPoint(1);
cells->InsertCellPoint(2);

// 方式B：使用数组批量插入（最高效）
vtkNew<vtkCellArray> cells;
vtkIdType tri1[3] = {0, 1, 2};
vtkIdType tri2[3] = {1, 3, 2};
cells->InsertNextCell(3, tri1);
cells->InsertNextCell(3, tri2);
```

方式B在需要构建大量单元时性能最优——它避免了为每个单元创建临时`vtkIdList`或`vtkTriangle`对象的开销。

---

## 4.5 代码示例：多数据源对比展示

本节提供一个完整的综合示例，在单个窗口中创建2x2的四视图布局，每个子视图展示一种不同的几何体数据源。这个示例综合运用了前三章的知识（管道连接、Actor配置、Renderer布局），并展示了本章介绍的多种Source的实际效果。

### 4.5.1 完整代码

将以下代码保存为`MultiSourceDemo.cxx`：

```cpp
// ============================================================================
// MultiSourceDemo.cxx
// 第四章综合示例：2x2 四视图展示四种不同几何体数据源
// ============================================================================

#include "vtkActor.h"
#include "vtkCamera.h"
#include "vtkConeSource.h"
#include "vtkCubeSource.h"
#include "vtkCylinderSource.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkSphereSource.h"
#include "vtkTextActor.h"
#include "vtkTextProperty.h"

// ============================================================================
// 辅助函数：创建一个Actor，关联给定的Source
// ============================================================================
vtkSmartPointer<vtkActor> CreateActorForSource(
    vtkAlgorithm* source,
    double r, double g, double b,
    bool showEdges = true)
{
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputConnection(source->GetOutputPort());

  vtkSmartPointer<vtkActor> actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(r, g, b);
  actor->GetProperty()->SetEdgeVisibility(showEdges ? 1 : 0);
  actor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.1);
  actor->GetProperty()->SetLineWidth(1.0);

  return actor;
}

// ============================================================================
// 辅助函数：在指定Renderer中添加文字标签（左上角）
// ============================================================================
void AddLabel(vtkRenderer* renderer, const char* text,
              double r, double g, double b)
{
  vtkNew<vtkTextActor> label;
  label->SetInput(text);
  label->GetTextProperty()->SetFontSize(16);
  label->GetTextProperty()->SetColor(r, g, b);
  label->GetTextProperty()->SetFontFamilyToCourier();
  label->GetTextProperty()->BoldOn();
  label->SetDisplayPosition(10, 10);  // 左下角偏移（像素坐标）
  renderer->AddActor2D(label);
}

int main(int argc, char* argv[])
{
  // ====================================================================
  // 第一步：创建渲染窗口（将容纳4个Renderer）
  // ====================================================================
  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->SetSize(800, 600);
  renderWindow->SetMultiSamples(4);  // 4x MSAA 抗锯齿
  renderWindow->SetWindowName("多数据源对比演示 - 2x2 四视图");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  // ====================================================================
  // 第二步：创建4个Renderer，分别占据窗口的四个象限
  // ====================================================================
  // Viewport坐标: (xmin, ymin, xmax, ymax)，范围 [0,1]
  // 布局：
  //   +-------+-------+
  //   | (0,0) | (1,0) |    其中 (col, row)
  //   | 球体  | 圆柱  |
  //   +-------+-------+
  //   | (0,1) | (1,1) |
  //   | 立方体| 圆锥  |
  //   +-------+-------+

  vtkNew<vtkRenderer> rendererSphere;    // 左上: 球体
  vtkNew<vtkRenderer> rendererCylinder;  // 右上: 圆柱
  vtkNew<vtkRenderer> rendererCube;      // 左下: 立方体
  vtkNew<vtkRenderer> rendererCone;      // 右下: 圆锥

  // 设置各Renderer的viewport
  // viewport = (xmin, ymin, xmax, ymax) —— 归一化坐标
  rendererSphere->SetViewport(0.0, 0.5, 0.5, 1.0);    // 左上
  rendererCylinder->SetViewport(0.5, 0.5, 1.0, 1.0);  // 右上
  rendererCube->SetViewport(0.0, 0.0, 0.5, 0.5);      // 左下
  rendererCone->SetViewport(0.5, 0.0, 1.0, 0.5);      // 右下

  renderWindow->AddRenderer(rendererSphere);
  renderWindow->AddRenderer(rendererCylinder);
  renderWindow->AddRenderer(rendererCube);
  renderWindow->AddRenderer(rendererCone);

  // ====================================================================
  // 第三步：创建4种几何体数据源
  // ====================================================================

  // --- 球体 ---
  vtkNew<vtkSphereSource> sphereSource;
  sphereSource->SetRadius(0.8);
  sphereSource->SetThetaResolution(40);
  sphereSource->SetPhiResolution(40);

  // --- 圆柱体 ---
  vtkNew<vtkCylinderSource> cylinderSource;
  cylinderSource->SetHeight(1.5);
  cylinderSource->SetRadius(0.5);
  cylinderSource->SetResolution(40);

  // --- 立方体 ---
  vtkNew<vtkCubeSource> cubeSource;
  cubeSource->SetXLength(1.2);
  cubeSource->SetYLength(0.8);
  cubeSource->SetZLength(1.0);

  // --- 圆锥体 ---
  vtkNew<vtkConeSource> coneSource;
  coneSource->SetHeight(1.5);
  coneSource->SetRadius(0.6);
  coneSource->SetResolution(40);

  // ====================================================================
  // 第四步：创建Actor（每个Source + Mapper + Actor是一个独立管道分支）
  // ====================================================================
  auto actorSphere = CreateActorForSource(sphereSource, 1.0, 0.3, 0.3);    // 红色
  auto actorCylinder = CreateActorForSource(cylinderSource, 0.3, 0.7, 1.0); // 蓝色
  auto actorCube = CreateActorForSource(cubeSource, 0.3, 1.0, 0.4);         // 绿色
  auto actorCone = CreateActorForSource(coneSource, 1.0, 0.7, 0.1);         // 金色

  // ====================================================================
  // 第五步：将Actor添加到对应的Renderer
  // ====================================================================
  rendererSphere->AddActor(actorSphere);
  rendererCylinder->AddActor(actorCylinder);
  rendererCube->AddActor(actorCube);
  rendererCone->AddActor(actorCone);

  // ====================================================================
  // 第六步：设置每个Renderer的背景色、相机位置和文字标签
  // ====================================================================

  // --- 球体视图（左上） ---
  rendererSphere->SetBackground(0.12, 0.12, 0.18);
  rendererSphere->GetActiveCamera()->SetPosition(0, 0, 2.5);
  rendererSphere->GetActiveCamera()->SetFocalPoint(0, 0, 0);
  AddLabel(rendererSphere, "Sphere", 1.0, 0.8, 0.8);

  // --- 圆柱视图（右上） ---
  rendererCylinder->SetBackground(0.12, 0.12, 0.18);
  rendererCylinder->GetActiveCamera()->SetPosition(0, 0, 2.5);
  rendererCylinder->GetActiveCamera()->SetFocalPoint(0, 0, 0);
  AddLabel(rendererCylinder, "Cylinder", 0.8, 0.9, 1.0);

  // --- 立方体视图（左下） ---
  rendererCube->SetBackground(0.12, 0.12, 0.18);
  rendererCube->GetActiveCamera()->SetPosition(1.5, 1.5, 1.5);
  rendererCube->GetActiveCamera()->SetFocalPoint(0, 0, 0);
  AddLabel(rendererCube, "Cube", 0.8, 1.0, 0.8);

  // --- 圆锥视图（右下） ---
  rendererCone->SetBackground(0.12, 0.12, 0.18);
  rendererCone->GetActiveCamera()->SetPosition(0, 0, 2.5);
  rendererCone->GetActiveCamera()->SetFocalPoint(0, 0, 0);
  AddLabel(rendererCone, "Cone", 1.0, 0.9, 0.7);

  // ====================================================================
  // 第七步：设置共享交互样式
  // ====================================================================
  // 共享一个交互样式使得鼠标操作在四个视图中同时生效
  vtkNew<vtkInteractorStyleTrackballCamera> style;

  // 将每个Renderer设为交互样式的默认Renderer
  // 当鼠标悬停在某个视图上时，交互操作作用于该视图
  interactor->SetInteractorStyle(style);

  // 为每个renderer设置独立的默认样式（确保各自独立响应）
  // 实际上vtkRenderWindowInteractor会自动管理焦点renderer

  // ====================================================================
  // 第八步：首次渲染并启动交互
  // ====================================================================
  renderWindow->Render();

  std::cout << "========================================" << std::endl;
  std::cout << "  多数据源对比演示" << std::endl;
  std::cout << "========================================" << std::endl;
  std::cout << "  左上: vtkSphereSource   - 红色球体" << std::endl;
  std::cout << "  右上: vtkCylinderSource - 蓝色圆柱" << std::endl;
  std::cout << "  左下: vtkCubeSource     - 绿色立方体" << std::endl;
  std::cout << "  右下: vtkConeSource     - 金色圆锥" << std::endl;
  std::cout << std::endl;
  std::cout << "  每个几何体均显示边界线以便观察网格结构。" << std::endl;
  std::cout << "  鼠标操作作用于当前悬停的视图。" << std::endl;
  std::cout << "========================================" << std::endl;

  interactor->Start();

  return 0;
}
```

### 4.5.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(MultiSourceDemo VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(VTK 9.5.2
  COMPONENTS
    CommonCore
    CommonColor
    CommonDataModel
    CommonExecutionModel
    FiltersSources
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
    RenderingFreeType   # vtkTextActor需要
  REQUIRED
)

message(STATUS "VTK version: ${VTK_VERSION}")

add_executable(MultiSourceDemo MultiSourceDemo.cxx)

target_link_libraries(MultiSourceDemo PRIVATE ${VTK_LIBRARIES})

vtk_module_autoinit(
  TARGETS MultiSourceDemo
  MODULES ${VTK_LIBRARIES}
)
```

### 4.5.3 代码详解

本节对MultiSourceDemo示例的关键设计点进行深入分析。

#### Viewport布局

`vtkRenderer::SetViewport(xmin, ymin, xmax, ymax)`是创建多视图布局的核心方法。Viewport的四个参数使用**归一化坐标**（范围0.0到1.0），定义每个Renderer在RenderWindow中所占的矩形区域：

```
(0,1) +-------+-------+ (1,1)
      |       |       |
      | Sphere |Cylin- |
      |        |der    |
0.5 -- +-------+-------+  <-- y=0.5 水平分割线
      |       |       |
      | Cube  | Cone  |
      |       |       |
(0,0) +-------+-------+ (1,0)
      ^       ^       ^
      x=0    x=0.5   x=1.0
      垂直分割线
```

- 左上（Sphere）：   (0.0, 0.5, 0.5, 1.0)
- 右上（Cylinder）： (0.5, 0.5, 1.0, 1.0)
- 左下（Cube）：     (0.0, 0.0, 0.5, 0.5)
- 右下（Cone）：     (0.5, 0.0, 1.0, 0.5)

VTK中viewport的原点在**左下角**（与屏幕坐标Y轴向上的惯例一致）。这与某些二维图形库（Y轴向下）不同，需要注意。

#### CreateActorForSource 辅助函数

```cpp
vtkSmartPointer<vtkActor> CreateActorForSource(
    vtkAlgorithm* source, double r, double g, double b, bool showEdges)
```

这个辅助函数封装了最常见的管道构建模式——Source -> Mapper -> Actor。它接受任何`vtkAlgorithm*`（所以所有Source都可以用），内部通过`SetInputConnection()`建立管道连接，确保了按需执行机制的正常运作。

函数返回类型为`vtkSmartPointer<vtkActor>`——使用引用计数智能指针——确保Actor在函数返回后仍然存活（回顾第一章：`vtkNew<>`只能在栈作用域内使用，需要跨函数传递的所有权使用`vtkSmartPointer<>`）。

#### AddLabel 辅助函数与二维Actor

```cpp
void AddLabel(vtkRenderer* renderer, const char* text, double r, double g, double b)
```

这个函数使用`vtkTextActor`在视图中添加二维文字标签。关键区别：

- `renderer->AddActor(actor)`：添加**三维**Actor（`vtkActor`），受到相机透视和旋转的影响。
- `renderer->AddActor2D(actor2D)`：添加**二维**Actor（如`vtkTextActor`），使用屏幕坐标，不受相机影响——文字始终正对屏幕，不会因为旋转场景而翻转。

`SetDisplayPosition(10, 10)`设置标签在viewport中的像素位置（以该Renderer所占区域的左下角为原点）。对于多视图布局，标签只在对应的Renderer区域内可见。

#### 管道结构

本例的数据流拓扑是一个"一对多"的广播型管道分支：

```
                              +---> (Mapper) ---> ActorSphere  ---> RendererSphere
                              |
                              +---> (Mapper) ---> ActorCylinder -> RendererCylinder
                              |
   (无共享Source —— 4个独立管道)
                              |
                              +---> (Mapper) ---> ActorCube    ---> RendererCube
                              |
                              +---> (Mapper) ---> ActorCone    ---> RendererCone
```

四个管道完全独立——每个都有自己的Source、Mapper和Actor。这不是分支（Forking），而是四个并列的管道，分别馈送到四个Renderer中，最终由同一个RenderWindow呈现。

### 4.5.4 实验拓展建议

运行此示例后，建议尝试以下修改：

1. **替换几何体**：将某个视图中Source替换为`vtkArrowSource`、`vtkDiskSource`等本章介绍的其他Source，观察其外观。
2. **调整viewport**：修改`SetViewport`的参数，尝试3x1的水平排列或1x3的垂直排列。
3. **添加Filter**：在某个管道中插入一个Filter（例如`vtkShrinkPolyData`），观察处理后的效果。
4. **共享数据源**：将四个分支全部连接到同一个Source（如一个`vtkSphereSource`），实现真正的管道分叉——同一个球体在不同视图中被渲染为不同颜色。
5. **关闭边界线**：将`showEdges`参数改为`false`，对比有边线和无边线的视觉效果差异。

---

## 4.6 本章小结

本章系统地介绍了VTK中生成数据的三种方式：内置几何体Source、`vtkProgrammableSource`程序化生成、以及手动构建`vtkPoints`和`vtkCellArray`。让我们回顾核心要点：

### 要点速览

1. **Source是管道的起点——是"只有输出、没有输入"的特殊Filter。** 所有Source继承自`vtkAlgorithm`，其`GetNumberOfInputPorts()`返回0。这个看似简单的定义使得Source成为`Update()`向上游追溯的终点，也是数据向下游流动的起点。

2. **VTK提供了十种常用的内置几何体数据源**，覆盖了球体、圆柱、立方体、圆锥、圆盘、箭头、正多边形、平面、线段和文字。每种Source通过固定的参数接口控制形状，参数修改后MTime自动更新，管道自动响应——这是快速原型开发和教学演示的理想选择。

3. **`vtkProgrammableSource`通过`SetExecuteMethod()`注册回调函数**，允许你用自定义的C++代码生成任意形状的`vtkPolyData`。4.3节的数学曲面示例展示了如何使用这一机制生成z = sin(x) * cos(y)曲面。需要记住的是，对于`vtkProgrammableSource`生成的曲面，通常需要在管道中添加`vtkPolyDataNormals`来计算法向量以获得正确的光照。

4. **手动构建`vtkPolyData`的流程是：创建`vtkPoints`放置顶点坐标 -> 创建`vtkCellArray`定义单元拓扑 -> 组装到`vtkPolyData`对象。** 4.4节展示的四面体示例完整地演示了这一流程。手动构建数据后，使用`SetInputData()`而非`SetInputConnection()`将数据传递给Mapper——这是`SetInputData()`的合理使用场景之一。

5. **多视图布局通过`vtkRenderer::SetViewport()`实现**——每个Renderer占据RenderWindow的一个归一化坐标矩形区域。4.5节的综合示例展示了如何在一个窗口中创建2x2的四视图布局，分别展示四种不同颜色和形状的几何体。

### 使用场景速查

| 场景 | 推荐方式 |
|------|---------|
| 快速原型、教学演示、测试数据 | 内置Source（`vtkSphereSource`等） |
| 数学函数可视化曲面 | `vtkProgrammableSource` 或手动构建 |
| 外部算法产生的自定义形状 | 手动构建`vtkPoints` + `vtkCellArray` |
| 文件导入的几何数据 | 使用Reader（详见第七章） |
| 需要完全控制拓扑结构 | 手动构建`vtkPolyData` |
| 组合多个简单形状 | 多个内置Source + `vtkAppendPolyData`合并 |

### 一条核心洞察

> **Source就是Input数量为0的Filter。**

这一洞察将Source与Filter统一在同一个概念框架下。它们共享着相同的管道基础架构（`vtkAlgorithm`、`vtkExecutive`、MTime机制），区别仅仅在于输入端口的数量。当你真正消化了这个观点，VTK的整个类继承体系和管道设计会突然变得清晰而优雅。

### 进入下一章之前

在进入第五章（颜色、材质与光照）之前，建议你：

1. 运行4.5节的MultiSourceDemo示例，观察四种几何体的外观差异，尝试旋转每个视图。
2. 动手实验：在MultiSourceDemo中替换其中一种Source为你感兴趣的其他类型（如`vtkArrowSource`、`vtkDiskSource`）。
3. 尝试手动构建一个不同于四面体的简单形状（如八面体或金字塔），熟悉`vtkPoints`和`vtkCellArray`的API。
4. 思考：如果你需要在场景中放置1000个不同位置、不同大小的球体，你会怎么做？（提示：第八章的Glyph机制正是为此而设计。）

第五章将带你进入VTK的外观定制世界——你将学会如何控制物体的颜色、透明度、材质属性（环境光、漫反射、镜面反射），以及如何在场景中布置光源来增强可视化效果。这些内容将让你从"会画形状"进阶到"能创造有视觉吸引力的场景"。

---

> **本章关键记忆口诀**："内置Source可速查，程序化生成靠回调，手动构建三点一线（Points + CellArray + PolyData），多视图用Viewport分象限。"
