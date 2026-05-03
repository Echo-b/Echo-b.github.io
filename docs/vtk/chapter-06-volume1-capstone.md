# 第六章 入门篇综合实战：3D几何画廊

## 本章导读

你在前五章中学到了VTK入门的全部基础知识。第一章让你了解了VTK是什么，搭建了开发环境，跑通了第一个Cone示例。第二章带你深入了管道架构——Source、Filter、Mapper、Actor、Renderer、RenderWindow，以及`SetInputConnection()`和MTime的核心运作机制。第三章展开了VTK的数据模型五层体系（尤其是`vtkPolyData`），以及Points、Cells、属性数据的概念。第四章让你熟悉了十种内置几何数据源，以及手动构建点和单元的方法。第五章讲解了渲染管道的右半侧——Renderer、RenderWindow、Camera、Interactor以及多视口布局。

现在，是时候将这些知识整合成一个完整的应用了。

本章是入门篇（第一卷）的收官之作。我们将构建一个**"3D几何画廊"（3D Shape Gallery）**——一个多视口、多数据源、带实际过滤器处理的交互式可视化程序。它不是演示单一概念的"玩具示例"，而是一个有工程结构的、可用作模板的小型项目。

本章的核心目标有三个：

1. **综合运用第一卷全部知识**——管道架构、数据模型、几何数据源、过滤器、渲染器、相机、多视口布局、交互器，所有这些组件将在一个程序中协同工作。
2. **理解管道分支的实际应用**——一个数据源同时驱动两条渲染分支（原始形状 + 收缩形状），演示VTK管道复用数据的核心优势。
3. **建立编写VTK项目的工程意识**——通过辅助函数封装重复逻辑、清晰的代码组织、视口规划、标签标注等实践，让你的代码从"能跑"升级到"可维护"。

读完本章后，你不仅将看到一个令人满意的综合成果，还将带着扎实的基础进入第二卷（进阶篇）的学习。第二卷将深入PolyData的精细操作、非结构化网格、结构化数据、I/O、高级过滤器、颜色映射、高级渲染以及多数据集场景——但那些都需要建立在第一卷的坚实基础之上。

---

## 6.1 项目概述

### 6.1.1 什么是"3D几何画廊"

"3D几何画廊"是一个桌面应用程序，在一个窗口中同时展示六种几何体，分为两列：

- **左列（3个视口）：原始形状**——以标准表面渲染呈现，让观众看到光滑、完整的几何体外观。
- **右列（3个视口）：收缩版本**——同样的形状经过`vtkShrinkPolyData`过滤器处理后展示，单元边线可见，让观众看到几何体的内部构造（三角形面片的排列方式）。

这种左右对比的布局直观地展示了VTK过滤器的作用：同样的数据源，经过不同的处理，产生截然不同的可视化效果。

### 6.1.2 项目将演示的核心概念

| 概念 | 在项目中的体现 |
|------|--------------|
| **管道架构** | 每条几何管线都是 Source -> Filter -> Mapper -> Actor -> Renderer 的完整实例 |
| **SetInputConnection()** | 所有管道连接均使用管道式连接，保持按需执行 |
| **管道分支** | 同一个Source同时输出到原始Mapper和ShrinkFilter->ShrinkMapper |
| **多数据类型** | 使用vtkSphereSource、vtkCylinderSource、vtkCubeSource三种不同的几何生成器 |
| **过滤器** | vtkShrinkPolyData对每个几何体执行收缩变换，vtkTriangleFilter确保正确的shrink行为 |
| **多视口布局** | 3行x2列共6个视口，使用`SetViewport()`精确分配 |
| **文本标注** | 每个视口上方显示描述性标签 |
| **辅助函数模式** | 用辅助函数封装Actor创建逻辑，消除重复代码 |
| **统一渲染风格** | 一致的背景色、统一的相机初始参数、统一的窗宽线宽 |

### 6.1.3 预期效果描述

程序运行后，你将看到一个约1200x800像素的窗口，背景统一为深蓝灰色。窗口被划分为3行x2列的网格：

```
+----------------------------------+----------------------------------+
|  "原始球体 (Original Sphere)"    |  "收缩球体 (Shrunk Sphere)"      |
|   [光滑球体，浅蓝色]              |   [收缩三角形块，边缘可见]          |
|                                  |                                  |
+----------------------------------+----------------------------------+
|  "原始圆柱 (Original Cylinder)"  |  "收缩圆柱 (Shrunk Cylinder)"    |
|   [光滑圆柱，浅绿色]              |   [收缩三角形块，边缘可见]          |
|                                  |                                  |
+----------------------------------+----------------------------------+
|  "原始立方体 (Original Cube)"    |  "收缩立方体 (Shrunk Cube)"      |
|   [光滑立方体，浅橙色]             |   [收缩三角形块，边缘可见]          |
+----------------------------------+----------------------------------+
```

- **左列**：表面光滑渲染，颜色柔和，展示几何体的完整外观。
- **右列**：三角形面片向中心收缩约15%，边缘线（Edge）可见，展示几何体的网格结构。
- **交互**：每个视口独立响应鼠标操作（旋转、缩放、平移），使用TrackballCamera风格。
- **标签**：每个视口下方有文字标签，指示当前显示的内容。

---

## 6.2 项目架构设计

### 6.2.1 管道全景图

在"3D几何画廊"中，数据流遵循一个可复用的模式。每个几何形状都使用完全相同的管道拓扑，区别仅在于Source的类型和参数：

```
                       可视化管道（每个形状一组）
   +-------------------------------------------------------------------+
   |                                                                   |
   |                        +--- 分支A: 原始渲染 ---+                   |
   |                        |                      |                   |
   |  vtkSphereSource  -----+-> vtkPolyDataMapper -> vtkActor (蓝色)    |
   |  (或其他Source)       |                      |                   |
   |                        |                      |                   |
   |                        +--- 分支B: 收缩渲染 ---+                   |
   |                        |                      |                   |
   |                        +-> vtkTriangleFilter                        |
   |                            |                                      |
   |                            v                                      |
   |                        vtkShrinkPolyData                          |
   |                            |                                      |
   |                            v                                      |
   |                        vtkPolyDataMapper -> vtkActor (收缩,边线)    |
   |                                                                   |
   +-------------------------------------------------------------------+
```

**关键设计决策：管道分支共享同一数据源**

请注意上图中，`vtkSphereSource`（或其他Source）的`GetOutputPort()`同时被传入两个下游：
- 分支A的`vtkPolyDataMapper::SetInputConnection()`
- 分支B的`vtkTriangleFilter::SetInputConnection()`

这就是第二章2.4.1节介绍的管道分支（Forking）。同一份Source数据被两个下游消费者复用：
- 当Source参数被修改时（虽然本项目不做这件事），两个分支都会自动检测MTime变化并更新。
- Source中的数据只需计算一次——无论有多少个下游分支。

**为什么在分支B中使用vtkTriangleFilter？**

`vtkShrinkPolyData`的工作方式是将每个多边形单元的顶点沿单元中心方向收缩。对于像`vtkCubeSource`这样的Source，其输出的四边形（Quad）和三角形（Triangle）混合体在收缩后会产生不均匀的视觉效果。通过在Shrink之前插入`vtkTriangleFilter`，确保所有多边形都被预先三角化，使得收缩效果均匀一致。

### 6.2.2 数据源选择

| 几何体 | Source类 | 关键参数 | 颜色方案 |
|--------|---------|---------|----------|
| 球体 | `vtkSphereSource` | ThetaResolution=32, PhiResolution=32 | 浅蓝 (0.4, 0.6, 0.9) |
| 圆柱 | `vtkCylinderSource` | Resolution=32, Height=2.0 | 浅绿 (0.4, 0.8, 0.5) |
| 立方体 | `vtkCubeSource` | 默认尺寸 | 浅橙 (0.9, 0.7, 0.4) |

三种几何体的选择是有意为之的：
- **球体**代表光滑曲面（全部由三角面片逼近），顶点法向变化连续。
- **圆柱**代表具有直纹面的几何体（侧面可展，顶底为平面），是光滑与平坦的结合。
- **立方体**代表完全的平坦面（六个四边形面），面与面之间存在法向不连续。

这三种形状覆盖了从"全光滑"到"全平坦"的几何频谱，使得收缩效果在三种形状上的视觉对比更加丰富。

### 6.2.3 视口布局规划

```
    0.0              0.5              1.0
  0.0 +----------------+----------------+
      |                |                |
      |   视口 (0,0)   |   视口 (0,1)   |  0.333
      |  原始球体       |  收缩球体       |
      |                |                |
      +----------------+----------------+
      |                |                |
      |   视口 (1,0)   |   视口 (1,1)   |  0.666
      |  原始圆柱       |  收缩圆柱       |
      |                |                |
      +----------------+----------------+
      |                |                |
      |   视口 (2,0)   |   视口 (2,1)   |  1.0
      |  原始立方体     |  收缩立方体     |
      +----------------+----------------+
```

每个视口占据窗口的 `(xmin, ymin, xmax, ymax)` 归一化坐标区域：
- 视口 (row=0, col=0)：原始球体 — `(0.00, 0.666, 0.50, 1.000)`
- 视口 (row=0, col=1)：收缩球体 — `(0.50, 0.666, 1.00, 1.000)`
- 视口 (row=1, col=0)：原始圆柱 — `(0.00, 0.333, 0.50, 0.666)`
- 视口 (row=1, col=1)：收缩圆柱 — `(0.50, 0.333, 1.00, 0.666)`
- 视口 (row=2, col=0)：原始立方体 — `(0.00, 0.000, 0.50, 0.333)`
- 视口 (row=2, col=1)：收缩立方体 — `(0.50, 0.000, 1.00, 0.333)`

### 6.2.4 辅助函数设计

为了避免为每种几何体重复书写15行几乎完全相同的管道连接代码，我们设计两个辅助函数：

**`CreateColoredActor(source, r, g, b)`**
- 接收一个Source对象和RGB颜色值。
- 内部创建Mapper，用`SetInputConnection()`连接。
- 创建Actor，设置Mapper和颜色属性。
- 返回`vtkActor*`供调用者添加到Renderer。

**`CreateShrunkActor(source, r, g, b)`**
- 接收一个Source对象和RGB颜色值。
- 内部创建`vtkTriangleFilter`和`vtkShrinkPolyData`。
- 用管道式连接：Source -> TriangleFilter -> ShrinkFilter -> Mapper。
- 创建Actor，设置Mapper、颜色、**开启边线可见**、设置边线颜色。
- 返回`vtkActor*`。

这两个函数使得主程序的流程清晰如画：

```cpp
// 球体管线
auto sphereSource = vtkSmartPointer<vtkSphereSource>::New();
sphereSource->SetThetaResolution(32);
sphereSource->SetPhiResolution(32);

auto origSphereActor = CreateColoredActor(sphereSource, 0.4, 0.6, 0.9);
auto shrunkSphereActor = CreateShrunkActor(sphereSource, 0.4, 0.6, 0.9);

// 同样的模式用于圆柱和立方体...
```

这样可读性极佳——每个几何体的创建压缩到3-4行，意图一目了然。

---

## 6.3 完整代码

下面是完整的C++源代码。建议你将其保存为`ShapeGallery.cxx`，与项目根目录下的`CMakeLists.txt`配套使用。

```cpp
// ============================================================================
// ShapeGallery.cxx
// VTK 第一卷综合实战：3D几何画廊
// 演示：管道分支、多视口布局、过滤器、文本标注、辅助函数模式
// ============================================================================

#include "vtkActor.h"
#include "vtkCamera.h"
#include "vtkCubeSource.h"
#include "vtkCylinderSource.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"
#include "vtkNew.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkShrinkPolyData.h"
#include "vtkSmartPointer.h"
#include "vtkSphereSource.h"
#include "vtkTextActor.h"
#include "vtkTextProperty.h"
#include "vtkTriangleFilter.h"

#include <array>
#include <string>

// ============================================================================
// 辅助函数1：从数据源创建彩色Actor（原始形状，光滑渲染）
// ============================================================================
// 参数:
//   source - 输入的数据源对象（vtkSphereSource、vtkCylinderSource等）
//   r,g,b  - RGB颜色分量，取值范围 [0.0, 1.0]
// 返回:
//   配置好的vtkActor指针，已设置Mapper和表面颜色
// ============================================================================
vtkSmartPointer<vtkActor> CreateColoredActor(
    vtkAlgorithmOutput* sourceOutput,
    double r, double g, double b)
{
  // --- Mapper：将Source输出的几何数据映射为图形基元 ---
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(sourceOutput);

  // --- Actor：场景中的可视化实体 ---
  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(r, g, b);              // 表面颜色
  actor->GetProperty()->SetDiffuse(1.0);                 // 漫反射系数（最大）
  actor->GetProperty()->SetSpecular(0.3);                // 镜面反射系数
  actor->GetProperty()->SetSpecularPower(10.0);          // 高光锐度

  return actor;
}

// ============================================================================
// 辅助函数2：从数据源创建收缩+边线可见的Actor（展示几何结构）
// ============================================================================
// 管道：Source -> TriangleFilter -> ShrinkPolyData -> Mapper -> Actor
//
// 参数:
//   source - 输入的数据源对象，将被分支复用
//   r,g,b  - RGB颜色分量，取值范围 [0.0, 1.0]
//   shrinkFactor - 收缩因子，0.0=完全收缩到中心，1.0=保持原始尺寸
// 返回:
//   配置好的vtkActor指针，已应用收缩过滤器并开启边线可见
// ============================================================================
vtkSmartPointer<vtkActor> CreateShrunkActor(
    vtkAlgorithmOutput* sourceOutput,
    double r, double g, double b,
    double shrinkFactor = 0.80)
{
  // --- 第一步：三角化 ---
  // 确保所有多边形被转换为三角形，使收缩效果均匀一致
  auto triangleFilter = vtkSmartPointer<vtkTriangleFilter>::New();
  triangleFilter->SetInputConnection(sourceOutput);

  // --- 第二步：收缩过滤器 ---
  // 将每个三角形的顶点沿单元中心方向收缩，露出单元间的缝隙
  auto shrinkFilter = vtkSmartPointer<vtkShrinkPolyData>::New();
  shrinkFilter->SetInputConnection(triangleFilter->GetOutputPort());
  shrinkFilter->SetShrinkFactor(shrinkFactor);

  // --- Mapper ---
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(shrinkFilter->GetOutputPort());

  // --- Actor：展示结构细节的版本 ---
  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(r, g, b);
  actor->GetProperty()->SetDiffuse(0.9);
  actor->GetProperty()->SetSpecular(0.2);

  // ★ 关键：开启边线可见，使逐个三角形单元清晰可见
  actor->GetProperty()->SetEdgeVisibility(1);
  // 边线颜色：深灰色，与表面形成对比但不至于太突兀
  actor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.15);
  actor->GetProperty()->SetLineWidth(1.0);

  return actor;
}

// ============================================================================
// 辅助函数3：创建文本标注Actor
// ============================================================================
// 参数:
//   text     - 要显示的文本内容
//   position - 归一化视口坐标中的位置（与vtkTextActor的Position一致）
//              (x, y) 表示文本左下角在视口中的归一化位置
//   fontSize - 字体大小（像素）
// 返回:
//   配置好的vtkTextActor指针
// ============================================================================
vtkSmartPointer<vtkTextActor> CreateLabel(
    const std::string& text,
    double x, double y,
    int fontSize = 12)
{
  auto textActor = vtkSmartPointer<vtkTextActor>::New();
  textActor->SetInput(text.c_str());
  textActor->SetPosition(x, y);
  textActor->GetTextProperty()->SetFontSize(fontSize);
  textActor->GetTextProperty()->SetColor(1.0, 1.0, 1.0);  // 白色
  textActor->GetTextProperty()->SetFontFamilyToCourier();   // 等宽字体
  textActor->GetTextProperty()->SetJustificationToLeft();
  textActor->GetTextProperty()->SetVerticalJustificationToBottom();

  return textActor;
}

// ============================================================================
// 辅助函数4：为渲染器统一设置初始相机
// ============================================================================
// 对所有的渲染器应用相同的初始相机位置和朝向，以确保视觉一致性
// ============================================================================
void SetupCamera(vtkRenderer* renderer)
{
  auto camera = renderer->GetActiveCamera();
  camera->SetPosition(1.5, 1.5, 3.0);    // 相机位于右上方前方
  camera->SetFocalPoint(0.0, 0.0, 0.0);  // 注视原点
  camera->SetViewUp(0.0, 1.0, 0.0);       // Y轴向上
  camera->SetClippingRange(0.1, 100.0);   // 裁剪平面范围
  // 使用平行投影（正交投影），使所有形状以一致的方式显示
  camera->ParallelProjectionOn();
  camera->SetParallelScale(1.5);
}

// ============================================================================
// 主程序入口
// ============================================================================
int main(int argc, char* argv[])
{
  // ==========================================================================
  // 第一步：创建渲染基础设施
  // ==========================================================================

  // 创建渲染窗口——容纳所有视口的顶层窗口
  auto renderWindow = vtkSmartPointer<vtkRenderWindow>::New();
  renderWindow->SetMultiSamples(4);        // 4x多重采样抗锯齿
  renderWindow->SetSize(1200, 800);        // 窗口客户区大小
  renderWindow->SetWindowName("VTK 3D Shape Gallery -- Volume 1 Capstone");

  // 统一的背景色——深蓝灰色
  const double bgColor[3] = {0.15, 0.20, 0.30};

  // ==========================================================================
  // 第二步：定义视口布局（3行 x 2列）
  // ==========================================================================
  //
  // 每个视口由 (xmin, ymin, xmax, ymax) 四个归一化坐标定义。
  // 6个视口均匀分布在3行x2列的网格中。
  //
  // 布局索引:
  //   左列 (col=0): 原始形状
  //   右列 (col=1): 收缩形状
  //   第0行: 球体
  //   第1行: 圆柱
  //   第2行: 立方体

  const int numRows = 3;
  const int numCols = 2;

  // 创建6个渲染器，一个用于一个视口
  // 使用二维数组存储，方便按行列索引访问
  vtkSmartPointer<vtkRenderer> renderers[numRows][numCols];

  // 初始化每个渲染器并设置其视口区域
  for (int row = 0; row < numRows; ++row)
  {
    for (int col = 0; col < numCols; ++col)
    {
      double ymin = 1.0 - static_cast<double>(row + 1) / numRows;
      double ymax = 1.0 - static_cast<double>(row)     / numRows;
      double xmin =       static_cast<double>(col)     / numCols;
      double xmax =       static_cast<double>(col + 1) / numCols;

      auto renderer = vtkSmartPointer<vtkRenderer>::New();
      renderer->SetViewport(xmin, ymin, xmax, ymax);
      renderer->SetBackground(bgColor);
      SetupCamera(renderer);

      renderers[row][col] = renderer;
      renderWindow->AddRenderer(renderer);
    }
  }

  // ==========================================================================
  // 第三步：定义几何体颜色方案
  // ==========================================================================
  //
  // 为三种几何体分配不同的柔和颜色，使它们在视觉上容易区分

  // 球体 — 浅蓝色（sky blue）
  const double sphereColor[3] = {0.35, 0.58, 0.85};
  // 圆柱 — 浅绿色（mint green）
  const double cylinderColor[3] = {0.35, 0.78, 0.50};
  // 立方体 — 浅橙色（warm peach）
  const double cubeColor[3] = {0.90, 0.65, 0.35};

  // ==========================================================================
  // 第四步：创建球体数据源及其两个渲染分支
  // ==========================================================================

  auto sphereSource = vtkSmartPointer<vtkSphereSource>::New();
  sphereSource->SetThetaResolution(48);   // 经线方向分段数（越高越光滑）
  sphereSource->SetPhiResolution(48);     // 纬线方向分段数
  sphereSource->SetRadius(0.8);           // 球体半径

  // ★ 管道分支：同一个Source的输出端口同时注入原始分支和收缩分支
  //   这就是第二章2.4.1节介绍的"管道分叉"——数据只计算一次，两个分支共享
  auto origSphereActor = CreateColoredActor(
      sphereSource->GetOutputPort(),
      sphereColor[0], sphereColor[1], sphereColor[2]);

  auto shrunkSphereActor = CreateShrunkActor(
      sphereSource->GetOutputPort(),
      sphereColor[0], sphereColor[1], sphereColor[2],
      0.80);  // 收缩因子0.80：每个三角形收缩20%

  // 添加到对应的渲染器
  renderers[0][0]->AddActor(origSphereActor);
  renderers[0][1]->AddActor(shrunkSphereActor);

  // 添加文本标签
  auto labelSphereOrig = CreateLabel("Original Sphere", 0.05, 0.92);
  auto labelSphereShrunk = CreateLabel("Shrunk Sphere", 0.05, 0.92);

  renderers[0][0]->AddActor2D(labelSphereOrig);
  renderers[0][1]->AddActor2D(labelSphereShrunk);

  // ==========================================================================
  // 第五步：创建圆柱数据源及其两个渲染分支
  // ==========================================================================

  auto cylinderSource = vtkSmartPointer<vtkCylinderSource>::New();
  cylinderSource->SetResolution(48);      // 圆周分段数
  cylinderSource->SetHeight(1.8);         // 圆柱高度
  cylinderSource->SetRadius(0.65);        // 圆柱半径
  cylinderSource->CappingOn();            // 生成顶面和底面（默认开启）

  auto origCylinderActor = CreateColoredActor(
      cylinderSource->GetOutputPort(),
      cylinderColor[0], cylinderColor[1], cylinderColor[2]);

  auto shrunkCylinderActor = CreateShrunkActor(
      cylinderSource->GetOutputPort(),
      cylinderColor[0], cylinderColor[1], cylinderColor[2],
      0.80);

  renderers[1][0]->AddActor(origCylinderActor);
  renderers[1][1]->AddActor(shrunkCylinderActor);

  auto labelCylOrig = CreateLabel("Original Cylinder", 0.05, 0.92);
  auto labelCylShrunk = CreateLabel("Shrunk Cylinder", 0.05, 0.92);

  renderers[1][0]->AddActor2D(labelCylOrig);
  renderers[1][1]->AddActor2D(labelCylShrunk);

  // ==========================================================================
  // 第六步：创建立方体数据源及其两个渲染分支
  // ==========================================================================

  auto cubeSource = vtkSmartPointer<vtkCubeSource>::New();
  cubeSource->SetXLength(1.4);            // X方向边长
  cubeSource->SetYLength(1.4);            // Y方向边长
  cubeSource->SetZLength(1.4);            // Z方向边长
  // cubeSource默认中心位于原点 (0, 0, 0)

  auto origCubeActor = CreateColoredActor(
      cubeSource->GetOutputPort(),
      cubeColor[0], cubeColor[1], cubeColor[2]);

  auto shrunkCubeActor = CreateShrunkActor(
      cubeSource->GetOutputPort(),
      cubeColor[0], cubeColor[1], cubeColor[2],
      0.80);

  renderers[2][0]->AddActor(origCubeActor);
  renderers[2][1]->AddActor(shrunkCubeActor);

  auto labelCubeOrig = CreateLabel("Original Cube", 0.05, 0.92);
  auto labelCubeShrunk = CreateLabel("Shrunk Cube", 0.05, 0.92);

  renderers[2][0]->AddActor2D(labelCubeOrig);
  renderers[2][1]->AddActor2D(labelCubeShrunk);

  // ==========================================================================
  // 第七步：创建交互器和交互样式
  // ==========================================================================

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renderWindow);

  // 使用轨迹球相机交互样式
  // 每个渲染器独立响应其所在视口内的鼠标事件
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  // ==========================================================================
  // 第八步：渲染并启动交互循环
  // ==========================================================================

  // 首帧渲染
  renderWindow->Render();

  // 输出程序使用说明到终端
  std::cout << "\n"
            << "========================================\n"
            << "  VTK 3D Shape Gallery\n"
            << "  Volume 1 Capstone Project\n"
            << "========================================\n"
            << "\n"
            << "左列：原始几何体（光滑表面渲染）\n"
            << "右列：收缩几何体（vtkShrinkPolyData，边线可见）\n"
            << "\n"
            << "交互操作：\n"
            << "  - 鼠标左键拖拽：旋转场景\n"
            << "  - 鼠标右键拖拽：缩放（Dolly）\n"
            << "  - 鼠标中键拖拽：平移场景\n"
            << "  - 滚轮：缩放\n"
            << "  - 按 R 键：重置当前视口的相机\n"
            << "\n"
            << "每个视口独立响应交互。\n"
            << "关闭窗口退出程序。\n"
            << "========================================\n"
            << std::endl;

  // 进入事件循环——程序在此阻塞，直到用户关闭窗口
  interactor->Start();

  return 0;
}
```

### 6.3.1 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
project(ShapeGallery VERSION 1.0.0 LANGUAGES CXX)

# 设置C++标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 查找VTK 9.5.2及其所需组件
find_package(VTK 9.5.2
  COMPONENTS
    CommonCore
    CommonColor
    CommonDataModel
    CommonExecutionModel
    FiltersCore
    FiltersSources
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
    RenderingFreeType
  REQUIRED
)

# 打印VTK信息（可选，用于调试确认版本）
message(STATUS "VTK version: ${VTK_VERSION}")
message(STATUS "VTK rendering backend: OpenGL2")

# 创建可执行文件
add_executable(ShapeGallery ShapeGallery.cxx)

# 链接VTK库
target_link_libraries(ShapeGallery
  PRIVATE
    ${VTK_LIBRARIES}
)

# ★ 关键：VTK模块自动初始化
# 这行宏会生成代码在main()之前注册所有需要的对象工厂
# 如果忘记这行，运行时会出现"No override found"错误
vtk_module_autoinit(
  TARGETS ShapeGallery
  MODULES ${VTK_LIBRARIES}
)
```

**新增模块说明：**

| 模块 | 用途 |
|------|------|
| `FiltersCore` | 提供`vtkShrinkPolyData`和`vtkTriangleFilter` |
| `RenderingFreeType` | 提供文本渲染支持（`vtkTextActor`依赖FreeType字体引擎） |

---

## 6.4 代码逐段解析

### 6.4.1 头文件与辅助函数设计

```cpp
#include "vtkSmartPointer.h"
#include "vtkNew.h"
// ... 以及其他头文件
```

程序中同时引入了`vtkSmartPointer`和`vtkNew`。在本章的代码中，为了展示更灵活的所有权管理，源数据对象使用了`vtkSmartPointer`（因为它们需要被传递给辅助函数并可能跨越多个函数作用域），而辅助函数内部的Mapper、Filter等局部对象使用`vtkSmartPointer`以便安全地从函数返回Actor。

回顾第一章1.5.2节的内容：`vtkNew<>`是栈分配的智能指针，适合在单一作用域内使用；`vtkSmartPointer<>`是引用计数智能指针，适合需要在函数之间传递共享所有权。本章使用`vtkSmartPointer`作为统一选择，以简化代码风格。

### 6.4.2 CreateColoredActor——原始形状的辅助函数

```cpp
vtkSmartPointer<vtkActor> CreateColoredActor(
    vtkAlgorithmOutput* sourceOutput,
    double r, double g, double b)
```

这个函数封装了"Source -> Mapper -> Actor"的标准三段式。注意以下设计决策：

**参数使用`vtkAlgorithmOutput*`而非`vtkAlgorithm*`：**
函数接收的是一个输出端口（Output Port），而非算法对象本身。这样做的好处是：
- 调用者可以传入任何算法的输出端口，无论它是Source还是Filter。
- 函数不需要知道上游数据来自什么类型的对象——它只关心"有一个端口可以接"。

这是VTK多态的经典用法：`vtkAlgorithmOutput`是所有算法输出端口的统一类型，使得管道组件可以松散耦合。

**材质属性设置：**
```cpp
actor->GetProperty()->SetDiffuse(1.0);
actor->GetProperty()->SetSpecular(0.3);
actor->GetProperty()->SetSpecularPower(10.0);
```
- `SetDiffuse(1.0)`：满漫反射，使颜色在正面直射时饱和。
- `SetSpecular(0.3)`：适中的镜面高光强度。
- `SetSpecularPower(10.0)`：高光聚拢（值越小高光越弥散，值越大高光越集中尖锐）。10.0产生一个柔和的高光光斑，适合展示曲面。

### 6.4.3 CreateShrunkActor——收缩形状的辅助函数

```cpp
vtkSmartPointer<vtkActor> CreateShrunkActor(
    vtkAlgorithmOutput* sourceOutput,
    double r, double g, double b,
    double shrinkFactor = 0.80)
```

这是本项目中管道最复杂的部分——一条完整的三级管道：

```
Source Output --> vtkTriangleFilter --> vtkShrinkPolyData --> vtkPolyDataMapper --> vtkActor
```

**为什么需要vtkTriangleFilter？**

`vtkCubeSource`生成的几何体包含四边形（VTK_QUAD）单元。如果直接将四边形送入`vtkShrinkPolyData`，收缩后的四边形可能看起来像是变小的四边形而不是一个个独立的三角形碎片。而球体和圆柱的输出中也可能包含三角形带（Triangle Strip），其收缩行为与普通三角形不完全一致。

通过在收缩前插入`vtkTriangleFilter`，确保所有多边形都被转化为标准的独立三角形（VTK_TRIANGLE），这带来两个好处：
1. 收缩效果均匀——每个三角形独立向各自中心收缩。
2. 边线显示准确——所有三角形的三条边都被绘制，不会出现四边形中缺失对角线边线的情况。

**边线可见设置：**
```cpp
actor->GetProperty()->SetEdgeVisibility(1);
actor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.15);
actor->GetProperty()->SetLineWidth(1.0);
```
- `SetEdgeVisibility(1)`是收缩版本的核心视觉差异——它让每个三角形单元的边界以深色线条绘制出来，使得网格结构一目了然。
- 边线颜色选为近乎黑色的深蓝灰色（0.1, 0.1, 0.15），与任何表面颜色都能形成清晰对比而不喧宾夺主。
- 线宽1.0是最细的线条，避免在高分辨率屏幕上显得过于粗糙。

**shrinkFactor参数：**
- 0.0 = 所有三角形完全收缩到它们的几何中心（成为点）。
- 1.0 = 不收缩，保持原始尺寸（但边线仍然可见）。
- 0.80 = 每个三角形向其中心收缩20%，露出单元间的明显缝隙。

0.80是一个经过视觉调试后的值：缝隙足够显眼，能清楚看到网格结构，但形状的整体轮廓仍然保留。

### 6.4.4 CreateLabel——文本标注的辅助函数

```cpp
vtkSmartPointer<vtkTextActor> CreateLabel(
    const std::string& text,
    double x, double y,
    int fontSize = 12)
```

`vtkTextActor`是一个特殊的Actor（继承自`vtkActor2D`，而非`vtkActor`），用于在视口中叠加二维文本。它的关键特点：

- **通过`AddActor2D()`添加**：区别于普通的3D Actor使用`AddActor()`，2D Actor使用`AddActor2D()`添加到Renderer。2D Actor不受3D相机变换影响，始终以屏幕坐标显示。
- **Position是归一化视口坐标**：`SetPosition(0.05, 0.92)` 表示文本左下角位于视口的水平5%、垂直92%位置。这个坐标独立于窗口的像素尺寸——窗口放大缩小时，文本始终保持在视口中的相对位置。
- **字体设置**：`SetFontFamilyToCourier()`选择等宽字体，确保在不同平台上显示一致。`SetJustificationToLeft()`和`SetVerticalJustificationToBottom()`设定文本对齐方式。

### 6.4.5 视口布局——3x2网格的精确构建

```cpp
for (int row = 0; row < numRows; ++row)
{
  for (int col = 0; col < numCols; ++col)
  {
    double ymin = 1.0 - static_cast<double>(row + 1) / numRows;
    double ymax = 1.0 - static_cast<double>(row)     / numRows;
    double xmin =       static_cast<double>(col)     / numCols;
    double xmax =       static_cast<double>(col + 1) / numCols;
    // ...
  }
}
```

这个双重循环计算每个视口的归一化坐标边界。以第0行第0列（球体，原始）为例：
- `xmin = 0/2 = 0.00`, `xmax = 1/2 = 0.50`
- `ymin = 1.0 - 1/3 = 0.666`, `ymax = 1.0 - 0/3 = 1.000`

结果：视口占据窗口的左上1/6区域。这与6.2.3节中规划的手动计算的坐标完全一致。

**为什么y坐标是反向计算的？**

VTK的视口坐标系将原点放在左下角（与OpenGL一致）：(0,0)是左下角，(1,1)是右上角。但我们的布局规划习惯上把第0行视为"最上面一行"。因此：
- 第0行（最上面）的ymax应该接近1.0。
- 第2行（最下面）的ymin应该接近0.0。

`1.0 - row/numRows` 实现了这种反向映射：row=0 时 ymax=1.0，row=2 时 ymin=1.0-3/3=0.0。

### 6.4.6 管道分支——本章最重要的代码段

让我们专注看球体的例子：

```cpp
auto sphereSource = vtkSmartPointer<vtkSphereSource>::New();
sphereSource->SetThetaResolution(48);
// ...

// 分支A：原始渲染
auto origSphereActor = CreateColoredActor(
    sphereSource->GetOutputPort(),
    sphereColor[0], sphereColor[1], sphereColor[2]);

// 分支B：收缩渲染
auto shrunkSphereActor = CreateShrunkActor(
    sphereSource->GetOutputPort(),
    sphereColor[0], sphereColor[1], sphereColor[2],
    0.80);
```

**这里发生的关键事件是：`sphereSource->GetOutputPort()`被调用了两次，每次传入不同的辅助函数。** 这两个函数各自内部调用了`SetInputConnection()`，建立了两个独立的管道分支。

管道连接示意图：

```
                        sphereSource->GetOutputPort()
                                   |
                  +----------------+----------------+
                  |                                 |
                  v                                 v
      CreateColoredActor()              CreateShrunkActor()
                  |                                 |
                  v                                 v
        vtkPolyDataMapper                 vtkTriangleFilter
                  |                                 |
                  v                                 v
           vtkActor (蓝)                     vtkShrinkPolyData
                  |                                 |
                  |                                 v
                  |                        vtkPolyDataMapper
                  |                                 |
                  |                                 v
                  |                          vtkActor (蓝+边线)
                  |                                 |
       +----------+---------------+-----------------+
       |                          |
       v                          v
  renderers[0][0]          renderers[0][1]
  (左上视口)                 (右上视口)
```

**管道分支的关键优势在本项目中体现为：**

1. **数据只生成一次**：球体的几何数据（2304个三角形，对应ThetaResolution=48和PhiResolution=48）由`vtkSphereSource`生成一次。两个分支都使用这份数据，不产生重复计算。
2. **参数传播自动**：如果我们后续调用`sphereSource->SetRadius(1.0)`改变球体大小，两个分支都会在下一次渲染时自动更新——因为MTime机制会在两个分支的Mapper各自向上游追溯时检测到Source的MTime变化。
3. **分支之间独立**：修改原始分支的Actor属性（如颜色）不会影响收缩分支，反之亦然。管道分支共享的是数据（Source的输出），而不是渲染属性。

### 6.4.7 相机配置

```cpp
void SetupCamera(vtkRenderer* renderer)
{
  auto camera = renderer->GetActiveCamera();
  camera->SetPosition(1.5, 1.5, 3.0);
  camera->SetFocalPoint(0.0, 0.0, 0.0);
  camera->SetViewUp(0.0, 1.0, 0.0);
  camera->SetClippingRange(0.1, 100.0);
  camera->ParallelProjectionOn();
  camera->SetParallelScale(1.5);
}
```

所有六个视口使用完全相同的初始相机参数。这确保了视觉上的一致性——当你在不同的视口之间切换观察时，视角不会突然跳变。

**使用平行投影（正交投影）而非透视投影的原因：**

在透视投影中，距离相机越远的物体看起来越小——这会在3x2的布局中产生不一致的视觉效果，因为不同视口中的几何体虽然大小相同，但由于它们在各自视口中与相机的相对距离微妙差异，看起来大小可能不一致。使用平行投影消除了这种透视畸变，使得所有视口中的几何体以统一的尺度呈现。

`SetParallelScale(1.5)`控制了平行投影的"缩放级别"——更小的值产生更大的放大效果。1.5使得半径为0.8的球体和高度为1.8的圆柱都能完整适当地填充视口区域，而不会过大或过小。

### 6.4.8 交互机制

```cpp
auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
interactor->SetInteractorStyle(style);
```

本程序使用TrackballCamera交互样式——与第一章的Cone示例相同。但这里有一个需要澄清的重要细节：

**每个渲染器是否独立响应交互？**

答案是：**是的，但只有一个渲染器在任意时刻响应用户交互。** `vtkInteractorStyleTrackballCamera`在内部使用了一个"当前渲染器"的概念。当用户的鼠标悬停在某个视口上并开始拖拽时，交互器会确定鼠标下的渲染器，并将该渲染器的相机绑定到轨迹球操作。

这意味着：
- 你的鼠标在左上视口（原始球体）时，旋转操作只影响左上视口的相机。
- 你的鼠标在右下视口（收缩立方体）时，旋转操作只影响右下视口的相机。
- 每个视口的相机独立变化，互不影响。

这是一种自然且直观的多视图交互模式——正如你在ParaView或3D Slicer等专业可视化工具中所体验到的。

### 6.4.9 渲染流程

```cpp
renderWindow->Render();
// ... 打印使用说明 ...
interactor->Start();
```

首帧`Render()`确保用户在进入交互循环前就能看到完整的场景。如果没有这一行，窗口在`Start()`之前会保持空白。

在`Render()`执行时，管道的执行顺序是：

1. RenderWindow遍历其注册的6个Renderer。
2. 每个Renderer遍历其Actor列表（原始Actor和文本Actor）。
3. 对于每个可见的3D Actor，Renderer请求其Mapper执行`Update()`。
4. 每个Mapper沿着`SetInputConnection()`连接向上游追溯：
   - 原始Actor的Mapper -> Source（对球体而言是sphereSource）
   - 收缩Actor的Mapper -> ShrinkFilter -> TriangleFilter -> Source
5. 对于共享同一Source的两个分支，Source在第一个分支的追溯中被请求计算，结果被缓存；第二个分支的追溯发现Source的MTime未变化，直接使用缓存结果。
6. 所有数据到达Mapper后，Mapper转换为图形基元。
7. Renderer执行相机投影、光照计算和光栅化。
8. 所有Renderer的输出被合成到RenderWindow的各个视口区域。
9. 最终帧缓冲交换到屏幕。

---

## 6.5 编译与运行

### 6.5.1 目录结构

```
ShapeGallery/
├── CMakeLists.txt       # CMake构建配置文件
└── ShapeGallery.cxx     # 源代码文件（约330行）
```

### 6.5.2 编译步骤

打开终端，进入项目目录并执行以下命令：

```bash
# 1. 进入项目根目录
cd ShapeGallery

# 2. 创建构建目录
mkdir build
cd build

# 3. 运行CMake配置
#    对于通过常规方式安装的VTK（APT/Homebrew/官方安装包）：
cmake ..

#    如果你使用vcpkg：
cmake .. -DCMAKE_TOOLCHAIN_FILE=[path-to-vcpkg]/scripts/buildsystems/vcpkg.cmake

#    如果VTK安装在自定义路径：
cmake .. -DVTK_DIR=[path-to-vtk-install]/lib/cmake/vtk-9.5

# 4. 编译项目
cmake --build . --config Release

#    使用多核加速（Windows/MSVC）：
cmake --build . --config Release --parallel 8

#    在Linux/macOS上：
cmake --build . -- -j 8
```

### 6.5.3 CMake配置成功输出

你应该看到类似以下的关键输出行：

```
-- VTK version: 9.5.2
-- VTK rendering backend: OpenGL2
-- Configuring done
-- Generating done
-- Build files have been written to: .../ShapeGallery/build
```

### 6.5.4 运行程序

**Windows（PowerShell）：**
```powershell
.\Release\ShapeGallery.exe
```

**Linux/macOS：**
```bash
./ShapeGallery
```

### 6.5.5 预期运行效果

程序启动后，你将看到一个1200x800像素的窗口，背景为深蓝灰色。窗口中包含3行x2列共6个视口：

- **左上（0,0）**：浅蓝色光滑球体，标签"Original Sphere"。
- **右上（0,1）**：浅蓝色收缩球体，三角形单元清晰可见，边线为深色，标签"Shrunk Sphere"。
- **左中（1,0）**：浅绿色光滑圆柱（含顶面和底面），标签"Original Cylinder"。
- **右中（1,1）**：浅绿色收缩圆柱，每个三角形单元独立可见，标签"Shrunk Cylinder"。
- **左下（2,0）**：浅橙色光滑立方体，标签"Original Cube"。
- **右下（2,1）**：浅橙色收缩立方体，三角形——而非四边形——单元清晰可见，标签"Shrunk Cube"。

终端将输出程序使用说明。

交互操作（TrackballCamera风格）：
- **鼠标左键拖拽**：旋转当前鼠标所在的视口的场景。
- **鼠标右键上下拖拽**：缩放（Dolly）当前视口。
- **鼠标中键拖拽**：平移当前视口。
- **鼠标滚轮**：缩放当前视口。
- **键盘 R 键**：重置当前活跃视口的相机到初始位置。

### 6.5.6 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|---------|----------|
| 文本标签不显示 | 缺少`RenderingFreeType`组件 | 在CMakeLists.txt的`find_package`中添加`RenderingFreeType` |
| `No override found for vtkPolyDataMapper` | 缺少`vtk_module_autoinit` | 确认CMakeLists.txt中包含`vtk_module_autoinit`宏 |
| 收缩版本无效果（看起来和原始版本一样） | shrinkFactor设置为1.0 | 检查`CreateShrunkActor`的调用参数，确保shrinkFactor < 1.0 |
| 窗口空白（六个视口全白） | Actor未添加到Renderer | 确认每个`renderers[row][col]->AddActor(...)`已正确执行 |
| 链接错误：`undefined reference to vtkShrinkPolyData` | 缺少`FiltersCore`组件 | 在`find_package`中添加`FiltersCore` |
| 收缩版本的立方体看起来不规则 | 缺少`vtkTriangleFilter`预三角化 | 确认`CreateShrunkActor`函数中包含了`vtkTriangleFilter` |
| 编译错误：`vtkTextActor.h`找不到 | 缺少`RenderingFreeType`组件 | 添加`RenderingFreeType`到CMake组件列表 |

---

## 6.6 扩展练习

完成本章项目后，建议你尝试以下扩展练习来加深理解。每个练习都建立在本章代码的基础上，鼓励你亲自动手修改代码并观察效果。

### 6.6.1 练习一：添加第四种几何体

**目标：** 在3x2布局的基础上扩展为包含第四种几何体。

**具体任务：**
1. 修改视口布局从3x2变为4x2（4行x2列），共8个视口。
2. 第四行使用`vtkArrowSource`生成箭头形状（`#include "vtkArrowSource.h"`）。
3. 选择一种新的颜色（如浅紫色 `{0.7, 0.4, 0.85}`）。
4. 调整窗口高度（如从800改为1000）以容纳新的视口行。

**提示：** `vtkArrowSource`默认生成一个从原点指向(1,0,0)的带轴箭头。它没有`SetResolution`参数——箭头的弯曲精度由`SetTipResolution()`和`SetShaftResolution()`控制。

**这个练习将巩固：** 视口布局计算、新数据源的引入、颜色方案设计。

### 6.6.2 练习二：添加键盘回调切换过滤器

**目标：** 通过按键动态切换收缩过滤器的开/关状态，实时观察过滤器的效果。

**具体任务：**
1. 创建一个自定义回调类（继承`vtkCallbackCommand`）或使用`vtkCallbackCommand`。
2. 按键盘`S`键时，在"显示原始Actor"和"显示收缩Actor"之间切换（每个视口）。
3. 移除右列的收缩视口——改为只有3个视口（一行一个），按S键切换显示模式。

**提示：**
```cpp
// 回调函数框架
void KeypressCallbackFunction(
    vtkObject* caller, long unsigned int eventId,
    void* clientData, void* callData)
{
  auto interactor = static_cast<vtkRenderWindowInteractor*>(caller);
  std::string key = interactor->GetKeySym();
  if (key == "s" || key == "S") {
    // 切换每个视口中Actor的可见性
  }
}

// 注册回调
auto callback = vtkSmartPointer<vtkCallbackCommand>::New();
callback->SetCallback(KeypressCallbackFunction);
interactor->AddObserver(vtkCommand::KeyPressEvent, callback);
```

**这个练习将巩固：** 观察者模式、键盘事件处理、Actor可见性控制、管道分支持久化。

### 6.6.3 练习三：使用vtkGlyph3D替代vtkShrinkPolyData

**目标：** 体验不同的过滤器，用球体符号（Glyph）替代收缩效果来展示网格结构。

**具体任务：**
1. 将右列的`CreateShrunkActor`替换为一个新的`CreateGlyphedActor`辅助函数。
2. 在新的辅助函数中，使用`vtkGlyph3D`滤波器：在输入数据的每个点上放置一个小球体，以显示点的分布。
3. 球体符号的半径应小到足以看出网格结构（如0.05），但大到足以可见。

**提示：**
```cpp
vtkSmartPointer<vtkActor> CreateGlyphedActor(
    vtkAlgorithmOutput* sourceOutput,
    double r, double g, double b)
{
  // 创建小球体作为符号
  auto glyphSource = vtkSmartPointer<vtkSphereSource>::New();
  glyphSource->SetRadius(0.05);  // 小半径——仅标记每个点
  glyphSource->SetThetaResolution(8);
  glyphSource->SetPhiResolution(8);

  // Glyph3D在输入数据的每个点上放置符号
  auto glyphFilter = vtkSmartPointer<vtkGlyph3D>::New();
  glyphFilter->SetInputConnection(sourceOutput);
  glyphFilter->SetSourceConnection(glyphSource->GetOutputPort());
  // 注意：Glyph3D需要设置Scaling为off或提供标量数据
  glyphFilter->SetScaleModeToDataScalingOff();

  // ... Mapper, Actor ...
}
```

**这个练习将巩固：** 多输入过滤器（Glyph3D需要两个输入：被采样的数据 + 符号形状）、不同过滤器的视觉差异、管道连接的多端口用法。

### 6.6.4 练习四：添加交互式收缩因子调节

**目标：** 通过鼠标滚轮或滑块实时调节ShrinkFilter的参数，体验MTime驱动的自动更新。

**具体任务：**
1. 创建一个自定义的InteractorStyle子类，重写`OnMouseWheelForward()`和`OnMouseWheelBackward()`方法。
2. 滚轮滚动时，增加或减少所有收缩Actor的ShrinkFactor（范围0.1到1.0）。
3. 在终端打印当前的ShrinkFactor值。
4. 注意：你需要存储对所有`vtkShrinkPolyData`对象的引用（指针），以便在回调中访问它们。

**提示：**
```cpp
class CustomInteractorStyle : public vtkInteractorStyleTrackballCamera
{
public:
  static CustomInteractorStyle* New();
  vtkTypeMacro(CustomInteractorStyle, vtkInteractorStyleTrackballCamera);

  void SetShrinkFilters(
      std::vector<vtkSmartPointer<vtkShrinkPolyData>> filters)
  { shrinkFilters = filters; }

  void OnMouseWheelForward() override
  {
    adjustShrinkFactor(0.05);  // 增加收缩（减小shrinkFactor）
    vtkInteractorStyleTrackballCamera::OnMouseWheelForward();
  }

  void OnMouseWheelBackward() override
  {
    adjustShrinkFactor(-0.05); // 减少收缩（增大shrinkFactor）
    vtkInteractorStyleTrackballCamera::OnMouseWheelBackward();
  }

private:
  void adjustShrinkFactor(double delta)
  {
    for (auto& f : shrinkFilters)
    {
      double current = f->GetShrinkFactor();
      double newVal = current - delta;
      if (newVal < 0.1) newVal = 0.1;
      if (newVal > 1.0) newVal = 1.0;
      f->SetShrinkFactor(newVal);
    }
    // 触发重新渲染
    // 通过Interactor获取RenderWindow并调用Render()
  }

  std::vector<vtkSmartPointer<vtkShrinkPolyData>> shrinkFilters;
};
```

**这个练习将巩固：** 自定义InteractorStyle、MTime驱动的参数更新、Render()的显式触发、实时交互式可视化。

---

## 6.7 第一卷总结与第二卷预告

### 6.7.1 第一卷回顾：你学到了什么

祝贺你完成了入门篇（第一卷）的全部六章内容。让我们回顾你走过的学习路径：

| 章节 | 主题 | 核心收获 |
|------|------|---------|
| **第一章** | VTK概述与环境搭建 | 理解VTK的定位和管道架构概念；完成环境安装与CMake项目配置；运行并理解第一个Cone程序。 |
| **第二章** | 可视化管道 | 掌握管道的六个阶段（Source/Filter/Mapper/Actor/Renderer/RenderWindow）；理解`SetInputConnection()`与`SetInputData()`的根本区别；掌握MTime驱动的按需执行机制；理解管道分支与合并。 |
| **第三章** | 数据模型基础 | 掌握五种主要数据集类型（ImageData/RectilinearGrid/StructuredGrid/PolyData/UnstructuredGrid）；理解Points与Cells的拓扑-几何分离设计；熟悉属性数据体系（Scalars/Vectors/Normals/TCoords/Tensors）；能够编程检查数据集内部结构。 |
| **第四章** | 基本几何体与数据源 | 熟悉十种内置几何数据源及其参数；掌握`vtkProgrammableSource`的程序化数据生成；能够手动构建`vtkPoints`和`vtkCellArray`创建任意形状；完成2x2多视图演示。 |
| **第五章** | 渲染与交互 | 深入理解Renderer/RenderWindow/Interactor的层级关系；掌握多视口布局的`SetViewport()`用法；理解Camera的透视/平行投影及参数控制；熟悉TrackballCamera交互样式。 |
| **第六章** | 入门篇综合实战 | 综合运用全部知识构建完整的"3D几何画廊"应用；实践管道分支、多视口布局、过滤器配置、文本标注、辅助函数封装等工程化技能。 |

通过第一卷的学习，你已经具备了以下能力：

1. **从零搭建VTK开发环境**并编译运行C++可视化程序。
2. **设计并实现可视化管道**——选择正确的Source，插入合适的Filter，连接Mapper和Actor。
3. **理解数据如何流动**——知道何时使用`SetInputConnection()`（默认），何时使用`SetInputData()`（特殊场景）。
4. **创建多视口布局**——在一个窗口中展示多个视图，每个视图拥有独立的Renderer和Camera。
5. **编写结构良好的代码**——使用辅助函数封装重复逻辑，使用智能指针管理VTK对象生命周期。

更重要的是，你现在拥有一个**心智模型（Mental Model）**：当你看到任何VTK代码或文档时，你能够迅速地将它定位在管道的正确位置——"这是一个Source，它产生数据"、"这是一个Filter，它接收数据并输出变换后的数据"、"这是一个Mapper，它将数据翻译为图形"、"这是一个Actor，它代表场景中的实体"。

### 6.7.2 第二卷预告：进阶之路

入门篇的结束不是终点，而是进阶之路的起点。第二卷（进阶篇）将带领你深入VTK更专业、更强大的领域。以下是第二卷各章的预告：

**第七章：PolyData深度操作**
- PolyData的内部数据结构详解（点列表、单元连接性数组）。
- 法向量计算与曲面平滑（`vtkSmoothPolyDataFilter`、`vtkPolyDataNormals`）。
- 网格简化与细分（`vtkDecimatePro`、`vtkLinearSubdivisionFilter`）。
- 拓扑重构（`vtkCleanPolyData`、`vtkStripper`）。

**第八章：非结构化网格（Unstructured Grid）**
- 任意单元类型的处理：四面体、六面体、棱柱、金字塔。
- 有限元分析结果的可视化（应力场、位移场）。
- 网格质量评估与修复。

**第九章：结构化数据与图像数据处理**
- vtkImageData的深度操作。
- 体数据切片与重切片（Reslice）。
- 医学图像数据（DICOM）的加载与显示。
- 三维图像滤波器（高斯平滑、中值滤波、形态学操作）。

**第十章：数据I/O——读写各种文件格式**
- VTK原生格式详解（`.vtk` Legacy、`.vtp`/`.vtu`/`.vts` XML格式）。
- 标准工业格式：STL、OBJ、PLY、VTU。
- 科学计算格式：NetCDF、ExodusII、EnSight。
- 自定义Reader/Writer开发入门。

**第十一章：高级过滤器**
- 等值面提取（Marching Cubes，`vtkContourFilter`）。
- 数据裁剪与提取（`vtkClipDataSet`、`vtkExtractVOI`、`vtkThreshold`）。
- 流线与粒子追踪（`vtkStreamTracer`、`vtkParticleTracer`）。
- 数据重采样与探测（`vtkProbeFilter`、`vtkResampleWithDataSet`）。

**第十二章：标量颜色映射（Scalar Colormap）**
- 查找表（Lookup Table, LUT）的创建与自定义。
- 颜色空间与传递函数（Transfer Function）设计。
- 标量条（Scalar Bar）与图例。
- 对数映射、离散映射、自定义映射策略。

**第十三章：高级渲染技术**
- 体渲染（Volume Rendering）——直接体绘制与最大密度投影（MIP）。
- 透明渲染与深度剥离（Depth Peeling）。
- 环境光遮蔽（SSAO）与阴影映射（Shadow Mapping）。
- 基于物理的渲染（PBR）材质。
- 离屏渲染（Off-screen Rendering）与动画输出。

**第十四章：多数据集与复合可视化**
- `vtkMultiBlockDataSet`——层次化场景管理。
- 复合管道的组织与遍历。
- 多数据集的时间序列动画。
- 场景标注（三维文本、图例、坐标轴）。

**第十五章：性能优化与大规模数据**
- 管道分析与瓶颈诊断。
- LOD（Level of Detail）策略。
- 流式处理（Streaming）大规模数据。
- 多线程处理（SMP）实践。
- VTK与ParaView的协同工作流。

### 6.7.3 进入第二卷前的建议

在进入第二卷之前，建议你完成以下准备工作：

1. **完成6.6节的扩展练习**——至少完成练习一和练习二。亲手修改代码是巩固理解的最有效方式。
2. **回顾你的代码库**——将第一卷中每个章节的示例代码整理到一个项目中，并确保它们都能独立编译运行。这些代码将在第二卷中作为基础不断被引用和扩展。
3. **熟悉VTK文档**——访问 [https://examples.vtk.org/](https://examples.vtk.org/) 浏览官方示例。不需要逐一看完，但浏览目录能让你对"VTK能做什么"有一个更广阔的预期。
4. **选择一个个人项目**——如果你在做科学研究、课程设计、或出于个人兴趣有一个可视化需求，现在就是开始的时候。即使是简单的"加载一个STL文件并旋转观察"也是很好的起点。

### 6.7.4 最后的话

第一卷教给了你VTK的**基本词汇和语法**。你现在能流畅地写出VTK代码——你知道如何创建Source，如何连接Filter，如何设置Mapper和Actor，如何将一切组装到渲染窗口中。

但就像学习一门自然语言一样，流利的表达需要的不仅是词汇和语法，还需要对这门语言的**惯用表达、风格和文化**的熟悉。第二卷将帮助你建立这种更深的熟悉度——你将学习如何处理真实世界的复杂数据，如何优化性能，如何创造出不仅"正确"而且"美观"的可视化效果。

**可视化不仅是一门技术，更是一门艺术。** VTK是你的画笔和调色板——而现在，你已经知道如何握笔了。

---

*第一卷（入门篇）完。感谢你的阅读，第二卷再见。*
