# 第十一章 常用过滤器（Common Filters）

## 本章导读

在前十章中，你已经系统学习了VTK的基础架构和数据类型：
- 第一卷（1-6章）建立了管道架构、数据模型、数据源、渲染和交互的坚实基础。
- 第二卷（7-9章）深入了PolyData、UnstructuredGrid和结构化数据的内部结构。
- 第十章带你掌握了I/O操作，实现了从外部世界获取数据的能力。

现在，你将面临一个自然的问题：**拿到了数据之后，如何真正处理它？** 答案就是**过滤器（Filters）**。

在VTK中，过滤器是数据处理的核心引擎。没有过滤器，你只能渲染原始数据源生成的基本几何体——一个巨大的限制。有了过滤器，你可以：

- **提取感兴趣的子集**：从体数据中切出等值面、裁剪掉不需要的区域、按标量范围提取单元。
- **揭示数据的隐藏结构**：生成矢量场的流线、用Glyph表达点数据、提取网格的骨架（中轴线）。
- **改善可视化的质量**：平滑网格、计算法向量、简化复杂模型。
- **执行空间变换**：旋转、平移、缩放、变形——实现更丰富的可视化叙事。

本章将以**实战为主线**，逐一讲解VTK 9.5.2中最常用、最强大的过滤器。每个过滤器都会包含"是什么、怎么用、何时用、代码示例"四个环节。此外，本章还将包含一个**用同一数据源驱动四条过滤器分支**的综合实例——这是对VTK管道分支能力的极致展示。

读完本章后，你将建立起一套心理"过滤器工具箱"——面对一个可视化任务时，你能快速判断需要哪个（或哪几个）过滤器，并知道如何将它们串联成有效的处理流水线。

---

## 11.1 过滤器分类概览

### 11.1.1 过滤器的统一接口

VTK中所有过滤器（Filter）都直接或间接地继承自`vtkAlgorithm`。这意味着你可以用统一的模式来操作任何过滤器：

```cpp
// 统一的连接模式
filter->SetInputConnection(source->GetOutputPort());
// 或
filter->SetInputData(polyData);

// 统一的执行模式
filter->Update();

// 统一的输出获取
vtkPolyData* output = filter->GetOutput();
// 或用于管道继续连接
someMapper->SetInputConnection(filter->GetOutputPort());
```

这种统一性来源于第二章讲述的流水线机制（Pipeline Mechanism）：过滤器不会在你调用`SetInputConnection()`时立即执行，而是等到下游消费者（如Mapper）需要数据时，才沿着管道链按需执行。

### 11.1.2 四大分类

根据功能目标，本章将常用过滤器分为四类：

| 分类 | 功能目标 | 典型过滤器 |
|------|---------|-----------|
| **几何变换** | 改变几何体的空间位置或形状 | `vtkTransformPolyDataFilter`, `vtkWarpScalar`, `vtkShrinkPolyData` |
| **数据提取** | 从数据集中摘取感兴趣的局部 | `vtkClipDataSet`, `vtkContourFilter`, `vtkThreshold`, `vtkCutter`, `vtkExtractEdges` |
| **矢量可视化** | 表达矢量场的方向和强度 | `vtkGlyph3D`, `vtkStreamTracer`, `vtkHedgeHog` |
| **网格处理** | 修改网格的连接关系或几何质量 | `vtkSmoothPolyDataFilter`, `vtkDecimatePro`, `vtkPolyDataNormals`, `vtkTriangleFilter` |

这四类过滤器常常需要组合使用。例如，你可能先用`vtkClipDataSet`裁剪出感兴趣区域，再用`vtkGlyph3D`在该区域内放置箭头来表达矢量场——两条过滤器首尾相连形成处理链。

### 11.1.3 过滤器链的构建模式

VTK中，多个过滤器可以串联成**流水线链**（Pipeline Chain）：

```
Source → Filter1 → Filter2 → ... → FilterN → Mapper → Actor
```

同时，一个Source可以驱动**多个并行的过滤器链**（管道分支）：

```
                         ┌→ FilterA → MapperA → ActorA
Source ──────────────────┤
                         ├→ FilterB → MapperB → ActorB
                         │
                         ├→ FilterC → MapperC → ActorC
                         │
                         └→ FilterD → MapperD → ActorD
```

这正是本章11.6节将要演示的综合实例模式。

---

## 11.2 几何变换过滤器

几何变换过滤器对数据集的几何坐标进行操作——它们改变点在空间中的位置，但不改变拓扑（点之间的连接关系）。

### 11.2.1 vtkShrinkPolyData / vtkShrinkFilter —— 收缩过滤器

**功能描述：** 将数据集中每个单元的顶点沿单元中心方向收缩，使相邻单元之间产生可见的缝隙。这个过滤器不改变数据，而是将数据"拆开"以展示其内部结构。

**核心参数：**
- `SetShrinkFactor(double factor)` —— 收缩因子，取值范围[0.0, 1.0]：
  - 1.0：保持原始尺寸（无收缩）
  - 0.5：每个单元收缩到原始大小的50%
  - 0.0：所有顶点收缩到单元中心（单元消失）

**适用场景：**
- 展示网格的单元构成（在有限元后处理中非常有用）
- 在讲课或演示中显示"这些三角形是如何拼成这个曲面的"
- 在多视口对比中展示模型的内部结构

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkSphereSource.h"
#include "vtkShrinkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"

int main()
{
  // --- 创建球体数据源 ---
  auto sphere = vtkSmartPointer<vtkSphereSource>::New();
  sphere->SetThetaResolution(32);
  sphere->SetPhiResolution(32);

  // --- 收缩过滤器 ---
  auto shrink = vtkSmartPointer<vtkShrinkPolyData>::New();
  shrink->SetInputConnection(sphere->GetOutputPort());
  shrink->SetShrinkFactor(0.7);  // 收缩30%，露出单元间缝隙

  // --- Mapper 和 Actor ---
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(shrink->GetOutputPort());

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(0.3, 0.5, 0.8);
  actor->GetProperty()->SetEdgeVisibility(1);  // 显示单元边线
  actor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.15);
  actor->GetProperty()->SetLineWidth(1.0);

  // --- 渲染设置 ---
  auto colors = vtkSmartPointer<vtkNamedColors>::New();
  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("SteelBlue").GetData());
  renderer->AddActor(actor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

**注意事项：**
- 对于混合多边形类型的网格（如同时包含三角形和四边形），建议先使用`vtkTriangleFilter`将全部单元转化为三角形，以使收缩效果均匀。这就是第六章综合实例中的做法。
- `vtkShrinkPolyData`是专门处理PolyData的版本。对于其他数据集类型，可以使用通用的`vtkShrinkFilter`。

### 11.2.2 vtkTransformPolyDataFilter —— 坐标变换过滤器

**功能描述：** 将`vtkTransform`定义的齐次变换（平移、旋转、缩放）应用到PolyData的每一个点上。与在Actor层面做变换不同，这个过滤器**真的修改了点的坐标**——被变换过的数据可以继续传递给下游过滤器做进一步处理。

**核心参数：**
- `SetTransform(vtkTransform* transform)` —— 设置变换对象

`vtkTransform`本身提供的核心方法：
- `Translate(dx, dy, dz)` —— 平移
- `RotateX(angle)`, `RotateY(angle)`, `RotateZ(angle)` —— 绕坐标轴旋转（角度制）
- `Scale(sx, sy, sz)` —— 各向异性缩放
- `Concatenate(vtkMatrix4x4* mat)` —— 连接任意4x4齐次变换矩阵
- `PostMultiply()` / `PreMultiply()` —— 控制变换的乘法顺序
- `Identity()` —— 重置为单位变换

**适用场景：**
- 需要对同一个几何体在场景中放置多个副本（通过多次应用不同变换）
- 在数据层面进行精确的几何配准（如将多个医学扫描对齐到同一坐标系）
- 变换后的数据需要继续传递给下游过滤器（Actor层面的变换不改变实际坐标）

**Actor变换与Filter变换的区别：**

| 特性 | Actor变换 (vtkActor::SetUserTransform) | Filter变换 (vtkTransformPolyDataFilter) |
|------|---------------------------------------|----------------------------------------|
| 修改对象 | Actor的渲染矩阵（GPU层面） | PolyData的实际点坐标（CPU层面） |
| 下游过滤器能否感知 | 否——下游过滤器看到的是原始坐标 | 是——下游过滤器看到的是变换后的坐标 |
| 性能 | 高（GPU执行） | 较低（每次Update都需变换所有点） |
| 适用 | 静态场景中的物体摆放 | 需要后继数据处理的变换 |

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkConeSource.h"
#include "vtkTransform.h"
#include "vtkTransformPolyDataFilter.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建圆锥数据源 ---
  auto cone = vtkSmartPointer<vtkConeSource>::New();
  cone->SetResolution(20);

  // --- 构建复合变换 ---
  auto transform = vtkSmartPointer<vtkTransform>::New();
  transform->Translate(1.5, 0.0, 0.0);  // 沿X轴平移1.5
  transform->RotateZ(45.0);             // 绕Z轴旋转45度
  transform->Scale(1.0, 1.5, 1.0);      // 在Y方向拉伸1.5倍

  // --- 将变换应用到数据 ---
  auto transformFilter = vtkSmartPointer<vtkTransformPolyDataFilter>::New();
  transformFilter->SetInputConnection(cone->GetOutputPort());
  transformFilter->SetTransform(transform);

  // --- 渲染 ---
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(transformFilter->GetOutputPort());

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(colors->GetColor3d("Coral").GetData());

  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("SlateGray").GetData());
  renderer->AddActor(actor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

### 11.2.3 vtkWarpScalar / vtkWarpVector —— 变形过滤器

**功能描述：** 这两个过滤器根据数据集中点的属性数据来**改变点的几何位置**，是数据可视化中最直观的表达方式之一：

- **vtkWarpScalar**：将每个点沿其**法向量方向**移动一段距离，移动量由该点的标量值决定。典型的应用是"高度场"——标量值越大，点被推得越高。
- **vtkWarpVector**：将每个点沿该点**矢量数据的实际方向**移动一段距离，移动量由矢量的模决定。典型应用是变形分析——显示结构受力后的位移场。

**核心参数：**

vtkWarpScalar：
- `SetScaleFactor(double factor)` —— 缩放因子，将标量值乘以该因子后作为位移量。这是最重要的参数，需要根据数据范围反复调试。
- `SetUseNormal(bool)` —— 是否使用法向量作为位移方向（默认为true）。关闭后可使用自定义方向向量。
- `SetInputArrayToProcess(...)` —— 指定使用点数据中的哪个标量数组。

vtkWarpVector：
- `SetScaleFactor(double factor)` —— 缩放因子，乘以矢量的模得到实际位移量。
- `SetInputArrayToProcess(...)` —— 指定使用点数据中的哪个矢量数组。

**适用场景：**

vtkWarpScalar：
- 将二维标量场可视化三维曲面（如地形高度的三维表达）
- 将压力、温度分布变形为三维"浮雕"图
- 展示模态分析中的振型

vtkWarpVector：
- 固体力学中的位移可视化（放大变形后的结构形态）
- 流体仿真中的流场扰动展示
- 地质形变分析

**代码示例（vtkWarpScalar）：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkPlaneSource.h"
#include "vtkMath.h"
#include "vtkFloatArray.h"
#include "vtkPointData.h"
#include "vtkWarpScalar.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"
#include "vtkSmoothPolyDataFilter.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建二维平面网格 ---
  auto plane = vtkSmartPointer<vtkPlaneSource>::New();
  plane->SetXResolution(30);
  plane->SetYResolution(30);
  plane->SetOrigin(-5.0, -5.0, 0.0);
  plane->SetPoint1(5.0, -5.0, 0.0);
  plane->SetPoint2(-5.0, 5.0, 0.0);

  // --- 在网格点上生成高程标量数据（一个高斯小山） ---
  // 注意：我们需要先Update平面，然后才能修改其点数据
  plane->Update();
  int numPoints = plane->GetOutput()->GetNumberOfPoints();

  auto scalars = vtkSmartPointer<vtkFloatArray>::New();
  scalars->SetName("Elevation");
  scalars->SetNumberOfTuples(numPoints);

  for (int i = 0; i < numPoints; i++)
  {
    double pt[3];
    plane->GetOutput()->GetPoint(i, pt);
    // 生成一个中心在原点的高斯山
    double r2 = pt[0] * pt[0] + pt[1] * pt[1];
    double elevation = 2.0 * exp(-r2 / 2.0);
    scalars->SetTuple1(i, elevation);
  }

  plane->GetOutput()->GetPointData()->SetScalars(scalars);

  // --- 经标量变形（法向偏移） ---
  auto warp = vtkSmartPointer<vtkWarpScalar>::New();
  warp->SetInputData(plane->GetOutput());
  warp->SetScaleFactor(1.0);   // 标量值直接作为位移量

  // --- 平滑处理（使变形后的表面更光滑） ---
  auto smooth = vtkSmartPointer<vtkSmoothPolyDataFilter>::New();
  smooth->SetInputConnection(warp->GetOutputPort());
  smooth->SetNumberOfIterations(10);
  smooth->SetRelaxationFactor(0.1);

  // --- 渲染 ---
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(smooth->GetOutputPort());
  mapper->SetScalarRange(0.0, 2.0);

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(colors->GetColor3d("Gold").GetData());

  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("MidnightBlue").GetData());
  renderer->AddActor(actor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

---

## 11.3 数据提取过滤器

数据提取过滤器能够从完整的数据集中截取你真正感兴趣的部分——裁剪、切割、等值面提取、阈值筛选，这些都是科学可视化中最常用的操作。

### 11.3.1 vtkClipDataSet / vtkClipPolyData —— 裁剪过滤器

**功能描述：** 用一个隐函数（Implicit Function）或裁剪平面将数据集一分为二，保留其中一半（或两半都保留）。隐函数可以是平面、球体、圆柱体、长方体等任意几何形状的函数表达。

**核心参数：**
- `SetClipFunction(vtkImplicitFunction* func)` —— 设置裁剪所用的隐函数。最常用的是`vtkPlane`。
- `SetInsideOut(bool)` —— 控制保留哪一侧：
  - `false`（默认）：保留隐函数值为**正**的一侧
  - `true`：保留隐函数值为**负**的一侧
- `SetValue(double value)` —— 裁剪等值面的值（默认为0.0）
- `SetGenerateClippedOutput(bool)` —— 是否同时输出被裁剪掉的部分（默认false）
- `GetClippedOutputPort()` —— 获取被裁剪部分的输出端口（需先开启`SetGenerateClippedOutput(true)`）

**隐函数类型：**

| 隐函数类 | 描述 | 创建示例 |
|---------|------|---------|
| `vtkPlane` | 无限平面 | `plane->SetOrigin(0,0,0); plane->SetNormal(1,0,0);` |
| `vtkSphere` | 球面 | `sphere->SetCenter(0,0,0); sphere->SetRadius(5.0);` |
| `vtkCylinder` | 无限圆柱 | `cyl->SetCenter(0,0,0); cyl->SetRadius(2.0);` |
| `vtkBox` | 长方体 | `box->SetBounds(xmin,xmax,ymin,ymax,zmin,zmax);` |

**适用场景：**
- 切除模型的一部分以展示内部结构（截面可视化）
- 基于几何范围过滤掉不感兴趣的区域
- 用球体"削"出一个感兴趣的子区域
- 医学可视化中切除遮挡器官查看病灶

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkConeSource.h"
#include "vtkPlane.h"
#include "vtkClipPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkProperty.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建圆锥数据源 ---
  auto cone = vtkSmartPointer<vtkConeSource>::New();
  cone->SetHeight(3.0);
  cone->SetRadius(1.0);
  cone->SetResolution(40);

  // --- 定义裁剪平面：垂直于X轴，穿过原点 ---
  auto clipPlane = vtkSmartPointer<vtkPlane>::New();
  clipPlane->SetOrigin(0.0, 0.0, 0.0);
  clipPlane->SetNormal(1.0, 0.0, 0.0);  // 法向沿+X

  // --- 裁剪过滤器 ---
  auto clipper = vtkSmartPointer<vtkClipPolyData>::New();
  clipper->SetInputConnection(cone->GetOutputPort());
  clipper->SetClipFunction(clipPlane);
  clipper->SetInsideOut(true);   // 保留X<0的一侧（即平面的负侧）
  clipper->SetValue(0.0);
  clipper->SetGenerateClippedOutput(true);  // 同时保留被切掉的部分

  // --- 保留部分的 Mapper 和 Actor ---
  auto keptMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  keptMapper->SetInputConnection(clipper->GetOutputPort());

  auto keptActor = vtkSmartPointer<vtkActor>::New();
  keptActor->SetMapper(keptMapper);
  keptActor->GetProperty()->SetColor(colors->GetColor3d("Tomato").GetData());
  // 让裁剪面可见：将保留部分的裁剪面向-X方向略微平移
  keptActor->GetProperty()->SetEdgeVisibility(1);
  keptActor->GetProperty()->SetEdgeColor(0.3, 0.3, 0.3);

  // --- 被裁剪部分的 Mapper 和 Actor ---
  auto clippedMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  clippedMapper->SetInputConnection(clipper->GetClippedOutputPort());

  auto clippedActor = vtkSmartPointer<vtkActor>::New();
  clippedActor->SetMapper(clippedMapper);
  clippedActor->GetProperty()->SetColor(colors->GetColor3d("Silver").GetData());
  clippedActor->GetProperty()->SetEdgeVisibility(1);
  clippedActor->GetProperty()->SetEdgeColor(0.2, 0.2, 0.2);
  // 将裁剪部分沿X方向平移以清晰展示
  clippedActor->SetPosition(2.5, 0.0, 0.0);

  // --- 渲染 ---
  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("SlateGray").GetData());
  renderer->AddActor(keptActor);
  renderer->AddActor(clippedActor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(1000, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

### 11.3.2 vtkContourFilter —— 等值面/等值线提取

**功能描述：** `vtkContourFilter`是VTK中最标志性的过滤器之一，它从标量场中提取所有标量值等于指定等值面值（Isovalue）的点构成的表面（3D中）或线（2D中）。这个算法的通用名称是**Marching Cubes（移动立方体）**。在医学可视化中，它能从CT数据中提取骨骼表面或皮肤表面；在气象学中，它能提取某个气压值的等压面。

**核心参数：**
- `SetValue(int i, double value)` —— 设置第i个等值面值为value
- `GenerateValues(int numContours, double range[2])` —— 在range范围内均匀生成numContours个等值面值
- `GenerateValues(int numContours, double min, double max)` —— 同上，但用两个标量参数
- `SetNumberOfContours(int n)` —— 设置等值面数量
- `SetComputeNormals(bool)` —— 是否计算法向量（默认true，对光照效果很重要）
- `SetComputeScalars(bool)` —— 是否将原始标量传递给输出（默认true）

**Marching Cubes 算法简述：**

Marching Cubes算法按以下步骤工作：

1. **遍历所有体素**：对于体数据（`vtkImageData`）中的每个由相邻8个点围成的立方体单元（体素），执行以下操作。
2. **判定顶点状态**：对于该体素的8个顶点，如果该顶点的标量值大于等值面值，标记为"内部"（1）；否则标记为"外部"（0）。这样每个体素产生一个8位二进制编码。
3. **查表**：该编码的取值范围是[0, 255]，对应256种可能的拓扑情况。通过查找预定义的表，确定等值面在体素内部产生的三角形顶点位于体素的哪些棱边上。
4. **插值**：对于等值面穿过的每条棱边，用线性插值计算三角形顶点的精确空间位置。
5. **输出**：将所有体素中生成的三角形收集起来，形成最终的等值面PolyData。

**适用场景：**
- 医学影像：从CT/MRI体数据中提取器官轮廓（如骨骼=500HU，皮肤=0HU）
- 气象：提取等压面、等温面
- 计算流体力学：提取涡结构（Q-criterion、lambda2判则的等值面）
- 材料科学：材料界面提取

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkSampleFunction.h"
#include "vtkContourFilter.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkProperty.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建一个隐函数采样生成体数据 ---
  // 使用 vtkSampleFunction 从隐函数生成规则网格上的标量体数据
  // 此处使用一个简单的"距离场"隐函数（球体隐函数）
  auto sphereFunc = vtkSmartPointer<vtkSphere>::New();
  sphereFunc->SetCenter(0.0, 0.0, 0.0);
  sphereFunc->SetRadius(3.0);

  auto sampler = vtkSmartPointer<vtkSampleFunction>::New();
  sampler->SetImplicitFunction(sphereFunc);
  sampler->SetModelBounds(-5.0, 5.0, -5.0, 5.0, -5.0, 5.0);
  sampler->SetSampleDimensions(50, 50, 50);  // 50x50x50的采样网格
  sampler->ComputeNormalsOff();              // 等值面过滤器会自行计算

  // --- 等值面过滤器 ---
  // 标量场 == 0.5 对应球体隐函数中的 r = sqrt(3*3 - 0.5) ≈ 2.92 的球面
  auto contour = vtkSmartPointer<vtkContourFilter>::New();
  contour->SetInputConnection(sampler->GetOutputPort());
  contour->SetValue(0, 0.5);                // 提取标量值=0.5的等值面
  // 也可以生成多个等值面:
  // contour->GenerateValues(5, 0.1, 0.9); // 生成0.1到0.9之间5个均匀分布的等值面

  // --- Mapper 和 Actor ---
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(contour->GetOutputPort());
  mapper->ScalarVisibilityOff();  // 关闭标量颜色映射，使用单色

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(colors->GetColor3d("Peacock").GetData());
  actor->GetProperty()->SetSpecular(0.5);
  actor->GetProperty()->SetSpecularPower(20);

  // --- 渲染 ---
  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("MidnightBlue").GetData());
  renderer->AddActor(actor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

**提示：** `vtkContourFilter`对于`vtkImageData`和`vtkStructuredGrid`使用Marching Cubes算法；对于`vtkPolyData`则使用另一种称为"Edge Tracking"的算法。对于`vtkUnstructuredGrid`，它使用Marching Tetrahedra算法。

### 11.3.3 vtkThreshold —— 阈值过滤器

**功能描述：** 按标量值的范围提取数据集的单元。只有标量值落在指定范围内的单元（Cell）才会被保留在输出中。与`vtkContourFilter`提取的是标量场的"等值面"不同，`vtkThreshold`提取的是"满足条件的所有单元"。

**核心参数：**
- `ThresholdBetween(double lower, double upper)` —— 保留标量值在[lower, upper]之间的所有单元
- `ThresholdByLower(double lower)` —— 保留标量值大于等于lower的所有单元
- `ThresholdByUpper(double upper)` —— 保留标量值小于等于upper的所有单元
- `SetInputArrayToProcess(...)` —— 指定使用哪个标量数组做阈值判断
- `SetAllScalars(bool)` —— 是否要求单元中所有点的标量值都满足条件（默认true；设为false则只要有一个点满足即可）

**适用场景：**
- 提取CFD仿真中高涡量区域的所有单元
- 提取有限元分析中应力超过屈服极限的单元
- 筛选温度场中在某温度范围内的区域
- 按置信度过滤医学图像中的分割结果

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkSphereSource.h"
#include "vtkFloatArray.h"
#include "vtkPointData.h"
#include "vtkThreshold.h"
#include "vtkDataSetMapper.h"
#include "vtkActor.h"
#include "vtkProperty.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建球体数据源并附加标量数据 ---
  auto sphere = vtkSmartPointer<vtkSphereSource>::New();
  sphere->SetThetaResolution(40);
  sphere->SetPhiResolution(40);

  // 为每个点生成一个基于Y坐标的标量值（-1到1对应Y轴高度）
  sphere->Update();
  int numPoints = sphere->GetOutput()->GetNumberOfPoints();

  auto yScalars = vtkSmartPointer<vtkFloatArray>::New();
  yScalars->SetName("YCoordinate");
  yScalars->SetNumberOfTuples(numPoints);

  for (int i = 0; i < numPoints; i++)
  {
    double pt[3];
    sphere->GetOutput()->GetPoint(i, pt);
    yScalars->SetTuple1(i, pt[1]);  // 使用Y坐标作为标量值
  }

  sphere->GetOutput()->GetPointData()->SetScalars(yScalars);

  // --- 阈值过滤器：只保留Y坐标在[0.2, 1.0]之间的单元 ---
  auto threshold = vtkSmartPointer<vtkThreshold>::New();
  threshold->SetInputData(sphere->GetOutput());
  threshold->ThresholdBetween(0.2, 1.0);
  threshold->SetInputArrayToProcess(
      0, 0, 0, vtkDataObject::FIELD_ASSOCIATION_POINTS, "YCoordinate");

  // --- 被阈值过滤后的输出需要使用DataSetMapper（不再是PolyDataMapper） ---
  // 因为阈值过滤器输出的是 vtkUnstructuredGrid
  auto mapper = vtkSmartPointer<vtkDataSetMapper>::New();
  mapper->SetInputConnection(threshold->GetOutputPort());
  mapper->ScalarVisibilityOff();

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(colors->GetColor3d("Coral").GetData());
  actor->GetProperty()->EdgeVisibilityOn();
  actor->GetProperty()->SetEdgeColor(0.2, 0.2, 0.2);

  // --- 渲染 ---
  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("SlateGray").GetData());
  renderer->AddActor(actor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

**重要提示：** `vtkThreshold`的输出类型取决于输入类型：
- 如果输入是`vtkPolyData`，输出是`vtkUnstructuredGrid`
- 如果输入是`vtkImageData`，输出是`vtkUnstructuredGrid`
- 如果输入是`vtkUnstructuredGrid`，输出是`vtkUnstructuredGrid`

输出变成了非结构化网格的原因在于：被阈值"切掉"后的数据拓扑不再是规则的。

### 11.3.4 vtkCutter —— 切片过滤器

**功能描述：** 用一个隐函数平面（通常用`vtkPlane`定义）切割任意数据集，返回该平面与数据集的**交线/交面**生成的PolyData。对于3D体积数据，这相当于获取一个截面；对于3D曲面数据，这相当于获取等切线。

**核心参数：**
- `SetCutFunction(vtkImplicitFunction* func)` —— 设置切割所用的隐函数（通常为`vtkPlane`）
- `SetValue(double value)` —— 隐函数的等值面值（默认为0.0）
- `SetGenerateCutScalars(bool)` —— 是否在输出中包含切割标量值（默认false）
- `SetSortBy(string)` —— 输出线段的排序方式

**与 vtkContourFilter 的区别：**
- `vtkContourFilter`从**标量场内部**提取等值面（标量值来自数据本身的点数据）
- `vtkCutter`使用**外部**的隐函数来切割数据集（与数据本身的标量值无关）

**适用场景：**
- 医学可视化中获取任意角度的CT切片（MPR——多平面重建）
- CFD分析中提取截面上的流场分布
- 地质分析中沿任意平面切开三维地震数据体
- 结构分析中沿指定平面获取剖面

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkCylinderSource.h"
#include "vtkPlane.h"
#include "vtkCutter.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkProperty.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建圆柱体数据源 ---
  auto cylinder = vtkSmartPointer<vtkCylinderSource>::New();
  cylinder->SetHeight(3.0);
  cylinder->SetRadius(1.0);
  cylinder->SetResolution(60);

  // --- 定义切割平面：水平穿过中心（法向沿Z轴） ---
  auto cutPlane = vtkSmartPointer<vtkPlane>::New();
  cutPlane->SetOrigin(0.0, 0.0, 0.5);    // Z=0.5处切割
  cutPlane->SetNormal(0.0, 0.0, 1.0);    // 法向沿+Z

  // --- 切片过滤器 ---
  auto cutter = vtkSmartPointer<vtkCutter>::New();
  cutter->SetInputConnection(cylinder->GetOutputPort());
  cutter->SetCutFunction(cutPlane);
  cutter->SetValue(0.0);

  // --- 原始圆柱的渲染 ---
  auto cylMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  cylMapper->SetInputConnection(cylinder->GetOutputPort());

  auto cylActor = vtkSmartPointer<vtkActor>::New();
  cylActor->SetMapper(cylMapper);
  cylActor->GetProperty()->SetColor(colors->GetColor3d("Silver").GetData());
  cylActor->GetProperty()->SetOpacity(0.3);  // 半透明以显示切口

  // --- 切口曲线的渲染（加粗，颜色鲜明） ---
  auto cutMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  cutMapper->SetInputConnection(cutter->GetOutputPort());

  auto cutActor = vtkSmartPointer<vtkActor>::New();
  cutActor->SetMapper(cutMapper);
  cutActor->GetProperty()->SetColor(colors->GetColor3d("Red").GetData());
  cutActor->GetProperty()->SetLineWidth(3.0);

  // --- 渲染 ---
  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("SlateGray").GetData());
  renderer->AddActor(cylActor);
  renderer->AddActor(cutActor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

### 11.3.5 vtkExtractEdges —— 边缘提取过滤器

**功能描述：** 从任意数据集中提取所有单元的**边线（Wireframe）**，输出为包含线段的PolyData。类似于把实体模型的框架提取出来——你看到的是由线组成的支架结构。

**核心参数：**
- 这个过滤器几乎不需要参数——直接将输入数据集的所有单元边界转换为线段。
- 如果需要保留原始点数据到边上，可以使用`UseAllPointsOn()`。

**适用场景：**
- 快速生成网格的线框渲染
- 在出版物中生成简洁的网格结构示意图
- 将3D单元的边线提取出来做进一步分析

**代码示例：**

```cpp
// 只需要一行代码：
auto edges = vtkSmartPointer<vtkExtractEdges>::New();
edges->SetInputData(someDataSet);  // 适用于任何 vtkDataSet 子类

// 然后用普通的 vtkPolyDataMapper 渲染输出
auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
mapper->SetInputConnection(edges->GetOutputPort());
```

---

## 11.4 矢量可视化过滤器

矢量数据（如速度场、力场、位移场、梯度场）是科学计算中最常见的数据类型之一。但矢量包含方向和大小两个信息维度，如何直观地可视化它们是一个长期的挑战。VTK提供了一套丰富的矢量可视化工具。

### 11.4.1 vtkGlyph3D —— 三维符号图

**功能描述：** `vtkGlyph3D`在每个输入点上放置一个小的几何符号（Glyph），该符号的方向、大小、颜色都可以由该点的属性数据（矢量、标量）控制。这是理解和展示矢量场的最常用方法之一。

**核心参数：**
- `SetSourceConnection(vtkAlgorithmOutput* source)` —— 设置用作Glyph的几何形状。通常用`vtkArrowSource`（箭头）、`vtkConeSource`（圆锥）、`vtkSphereSource`（球体）或`vtkCubeSource`（立方体）。
- `SetInputConnection(vtkAlgorithmOutput* input)` —— 设置包含点的数据集（输入）。
- `SetInputArrayToProcess(int idx, ...)` —— 指定由哪个数组控制Glyph的参数（矢量控制方向、标量控制大小）。
- `SetScaleFactor(double factor)` —— 全局缩放因子。
- `SetVectorModeToUseVector()` —— 由点数据的矢量数组控制Glyph朝向。
- `SetScaleModeToScaleByVector()` —— 由矢量的模控制Glyph大小。
- `SetScaleModeToScalar()` —— 由标量值控制Glyph大小。
- `SetScaleModeToDataScalingOff()` —— 关闭数据驱动缩放，全部用同一个大小。
- `ClampingOn/Off()` —— 是否将缩放因子钳制在指定范围内。
- `SetRange(double min, double max)` —— 缩放的标量/矢量范围。
- `OrientOn()` —— 是否让Glyph方向跟随矢量方向（默认true）。
- `SetColorModeToColorByScalar()` / `SetColorModeToColorByVector()` —— 控制颜色映射的数据来源。

**适用场景：**
- 用箭头展示流场中的速度方向
- 用球体的大小展示标量值的空间分布
- 用彩色锥体展示表面法向量的分布
- 在有限元网格上用小球展示节点处的应力张量

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkPlaneSource.h"
#include "vtkFloatArray.h"
#include "vtkPointData.h"
#include "vtkArrowSource.h"
#include "vtkGlyph3D.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkProperty.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"
#include "vtkMath.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 在XY平面上创建规则点网格 ---
  auto plane = vtkSmartPointer<vtkPlaneSource>::New();
  plane->SetXResolution(12);
  plane->SetYResolution(12);
  plane->SetOrigin(-5.0, -5.0, 0.0);
  plane->SetPoint1(5.0, -5.0, 0.0);
  plane->SetPoint2(-5.0, 5.0, 0.0);
  // 注意：PlaneSource生成的是面片，但Glyph3D会识别其上的每个点
  // 对于点云输入，可以使用 vtkProgrammableSource 或手动构建。

  // --- 为每个点生成矢量数据（旋转场） ---
  plane->Update();
  int numPoints = plane->GetOutput()->GetNumberOfPoints();

  auto vectors = vtkSmartPointer<vtkFloatArray>::New();
  vectors->SetName("Velocity");
  vectors->SetNumberOfComponents(3);
  vectors->SetNumberOfTuples(numPoints);

  for (int i = 0; i < numPoints; i++)
  {
    double pt[3];
    plane->GetOutput()->GetPoint(i, pt);

    // 旋转场：v = (-y, x, 0) —— 绕Z轴的旋转
    double vx = -pt[1];
    double vy = pt[0];
    double vz = 0.0;

    double mag = sqrt(vx * vx + vy * vy);
    // 归一化
    if (mag > 1e-6)
    {
      vx /= mag;
      vy /= mag;
    }

    vectors->SetTuple3(i, vx, vy, vz);
  }

  plane->GetOutput()->GetPointData()->SetVectors(vectors);

  // --- 创建箭头源（用作Glyph形状） ---
  auto arrow = vtkSmartPointer<vtkArrowSource>::New();
  arrow->SetTipLength(0.2);
  arrow->SetTipRadius(0.08);
  arrow->SetShaftRadius(0.03);

  // --- Glyph3D过滤器 ---
  auto glyph = vtkSmartPointer<vtkGlyph3D>::New();
  glyph->SetInputData(plane->GetOutput());
  glyph->SetSourceConnection(arrow->GetOutputPort());
  glyph->SetVectorModeToUseVector();      // 使用点数据的矢量数组
  glyph->SetInputArrayToProcess(
      0, 0, 0, vtkDataObject::FIELD_ASSOCIATION_POINTS, "Velocity");
  glyph->SetScaleModeToScaleByVector();   // 大小由矢量模决定
  glyph->SetScaleFactor(0.6);             // 全局缩放因子
  glyph->OrientOn();                      // 箭头朝向跟随矢量方向
  glyph->ClampingOn();                    // 钳制缩放
  glyph->SetRange(0.2, 0.8);             // 缩放范围

  // --- Mapper 和 Actor ---
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(glyph->GetOutputPort());
  mapper->ScalarVisibilityOff();

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(colors->GetColor3d("DodgerBlue").GetData());

  // --- 渲染 ---
  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("MidnightBlue").GetData());
  renderer->AddActor(actor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

### 11.4.2 vtkStreamTracer —— 流线追踪器

**功能描述：** `vtkStreamTracer`从一组种子点（Seed Points）出发，沿矢量场方向向前（或向后）追踪粒子轨迹，生成代表流线（Streamline）的PolyData。这是CFD（计算流体力学）可视化中最核心的工具之一——它回答了"如果在这里释放一个无质量的粒子，它会飞到哪里？"

**核心参数：**
- `SetInputConnection(vtkAlgorithmOutput* vectorField)` —— 设置包含矢量场的体数据或网格数据。
- `SetSourceConnection(vtkAlgorithmOutput* seedPoints)` —— 设置种子点数据集。种子点可以是一条线（产生流线面）、一个点源（产生单条流线）、或多个离散点。
- `SetIntegrator(vtkInitialValueProblemSolver* integrator)` —— 设置数值积分器：
  - `vtkRungeKutta4` —— 四阶Runge-Kutta法（默认，高精度）
  - `vtkRungeKutta2` —— 二阶RK法（较快）
  - `vtkRungeKutta45` —— 自适应步长的Runge-Kutta-Fehlberg法
- `SetIntegrationDirection(int dir)` —— 积分方向：
  - `vtkStreamTracer::FORWARD`（0）—— 向前（顺流而下）
  - `vtkStreamTracer::BACKWARD`（1）—— 向后（逆流而上）
  - `vtkStreamTracer::BOTH`（2）—— 双向
- `SetMaximumPropagation(double length)` —— 单条流线的最大长度（物理空间距离）。
- `SetMaximumNumberOfSteps(int n)` —— 最大积分步数。
- `SetInitialIntegrationStep(double step)` —— 初始积分步长。
- `SetTerminalSpeed(double speed)` —— 终端速度，低于此速度时停止积分。
- `SetComputeVorticity(bool)` —— 是否计算涡量（默认true）。

**适用场景：**
- CFD后处理：沿速度场追踪流线
- 天气预报图中的风场流线
- 核磁共振扩散张量成像（DTI）中的纤维束追踪
- 电场线/磁力线可视化

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkStructuredGrid.h"
#include "vtkPoints.h"
#include "vtkFloatArray.h"
#include "vtkPointData.h"
#include "vtkPointSource.h"
#include "vtkStreamTracer.h"
#include "vtkRungeKutta4.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkProperty.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"
#include "vtkRibbonFilter.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建一个模拟矢量场（绕Z轴的旋转流 + 径向分量） ---
  // 尺寸：11x11x11的网格，范围[-5,5]^3
  int dims[3] = {11, 11, 11};
  double origin[3] = {-5.0, -5.0, -5.0};
  double spacing[3] = {1.0, 1.0, 1.0};

  auto sgrid = vtkSmartPointer<vtkStructuredGrid>::New();
  sgrid->SetDimensions(dims);

  auto points = vtkSmartPointer<vtkPoints>::New();
  points->SetNumberOfPoints(dims[0] * dims[1] * dims[2]);

  auto vectors = vtkSmartPointer<vtkFloatArray>::New();
  vectors->SetName("Velocity");
  vectors->SetNumberOfComponents(3);
  vectors->SetNumberOfTuples(dims[0] * dims[1] * dims[2]);

  for (int k = 0; k < dims[2]; k++)
  {
    for (int j = 0; j < dims[1]; j++)
    {
      for (int i = 0; i < dims[0]; i++)
      {
        double x = origin[0] + i * spacing[0];
        double y = origin[1] + j * spacing[1];
        double z = origin[2] + k * spacing[2];

        int idx = i + j * dims[0] + k * dims[0] * dims[1];
        points->SetPoint(idx, x, y, z);

        // 绕Z轴的旋转流：(-y, x, 0) 加上一个小Z分量形成螺旋
        double vx = -y;
        double vy = x;
        double vz = 0.5;  // 在Z方向上的恒定速度分量

        vectors->SetTuple3(idx, vx, vy, vz);
      }
    }
  }

  sgrid->SetPoints(points);
  sgrid->GetPointData()->SetVectors(vectors);

  // --- 创建种子点（在XY平面上散布） ---
  auto seedSource = vtkSmartPointer<vtkPointSource>::New();
  seedSource->SetCenter(0.0, 1.0, -4.0);  // 种子点源的中心
  seedSource->SetRadius(2.0);             // 种子散布半径
  seedSource->SetNumberOfPoints(12);      // 12个种子点

  // --- 流线追踪器 ---
  auto integrator = vtkSmartPointer<vtkRungeKutta4>::New();

  auto streamer = vtkSmartPointer<vtkStreamTracer>::New();
  streamer->SetInputData(sgrid);
  streamer->SetSourceConnection(seedSource->GetOutputPort());
  streamer->SetIntegrator(integrator);
  streamer->SetIntegrationDirectionToForward();
  streamer->SetMaximumPropagation(200.0);
  streamer->SetMaximumNumberOfSteps(2000);
  streamer->SetInitialIntegrationStep(0.1);
  streamer->SetTerminalSpeed(1e-12);

  // --- 用RibbonFilter美化流线（把线变成飘带） ---
  auto ribbon = vtkSmartPointer<vtkRibbonFilter>::New();
  ribbon->SetInputConnection(streamer->GetOutputPort());
  ribbon->SetWidth(0.15);
  ribbon->SetWidthFactor(2.0);

  // --- 渲染 ---
  auto streamMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  streamMapper->SetInputConnection(ribbon->GetOutputPort());
  streamMapper->ScalarVisibilityOff();

  auto streamActor = vtkSmartPointer<vtkActor>::New();
  streamActor->SetMapper(streamMapper);
  streamActor->GetProperty()->SetColor(colors->GetColor3d("Orange").GetData());

  // 同时渲染种子点
  auto seedMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  seedMapper->SetInputConnection(seedSource->GetOutputPort());

  auto seedActor = vtkSmartPointer<vtkActor>::New();
  seedActor->SetMapper(seedMapper);
  seedActor->GetProperty()->SetColor(colors->GetColor3d("Lime").GetData());
  seedActor->GetProperty()->SetPointSize(5.0);

  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("MidnightBlue").GetData());
  renderer->AddActor(streamActor);
  renderer->AddActor(seedActor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

### 11.4.3 vtkHedgeHog —— 刺猬图

**功能描述：** `vtkHedgeHog`是最简单的矢量可视化方法——在数据集的每个点上沿该点矢量方向绘制一条有长度的线段，线段的长度正比于矢量的模。因为密集的矢量线看起来像刺猬的刺，所以得名"Hedgehog Plot"（刺猬图）。

**核心参数：**
- `SetScaleFactor(double factor)` —— 全局缩放因子，控制线段长度。
- `SetVectorModeToUseVector()` —— 使用点数据的矢量数组。
- `SetInputArrayToProcess(...)` —— 指定使用哪个矢量数组。

**适用场景：**
- 快速可视化曲面上的法向量分布
- 在稀疏网格上初步探查矢量场结构（密度过高时效果变差）
- 可视化地面风场观测数据（每个观测站一根"风羽"）

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkSphereSource.h"
#include "vtkFloatArray.h"
#include "vtkPointData.h"
#include "vtkHedgeHog.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkProperty.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建低分辨率球体（稀疏点以便刺猬图清晰可见） ---
  auto sphere = vtkSmartPointer<vtkSphereSource>::New();
  sphere->SetThetaResolution(16);
  sphere->SetPhiResolution(16);

  // --- 为每个点附上法向量作为矢量数据 ---
  sphere->Update();
  int numPoints = sphere->GetOutput()->GetNumberOfPoints();

  auto normals = vtkSmartPointer<vtkFloatArray>::New();
  normals->SetName("Normals");
  normals->SetNumberOfComponents(3);
  normals->SetNumberOfTuples(numPoints);

  for (int i = 0; i < numPoints; i++)
  {
    double pt[3];
    sphere->GetOutput()->GetPoint(i, pt);
    // 对于中心在原点的球体，法向量 = 位置向量的归一化
    double r = sqrt(pt[0] * pt[0] + pt[1] * pt[1] + pt[2] * pt[2]);
    if (r > 1e-6)
    {
      normals->SetTuple3(i, pt[0] / r, pt[1] / r, pt[2] / r);
    }
    else
    {
      normals->SetTuple3(i, 0.0, 0.0, 1.0);
    }
  }

  sphere->GetOutput()->GetPointData()->SetVectors(normals);

  // --- 刺猬图过滤器 ---
  auto hedgehog = vtkSmartPointer<vtkHedgeHog>::New();
  hedgehog->SetInputData(sphere->GetOutput());
  hedgehog->SetScaleFactor(0.5);  // 线段的缩放因子
  hedgehog->SetVectorModeToUseVector();

  // --- 同时渲染球体线框作为背景 ---
  auto sphereMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  sphereMapper->SetInputConnection(sphere->GetOutputPort());

  auto sphereActor = vtkSmartPointer<vtkActor>::New();
  sphereActor->SetMapper(sphereMapper);
  sphereActor->GetProperty()->SetColor(colors->GetColor3d("Silver").GetData());
  sphereActor->GetProperty()->SetRepresentationToWireframe();
  sphereActor->GetProperty()->SetOpacity(0.2);

  // --- 刺猬图的渲染 ---
  auto hedgeMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  hedgeMapper->SetInputConnection(hedgehog->GetOutputPort());

  auto hedgeActor = vtkSmartPointer<vtkActor>::New();
  hedgeActor->SetMapper(hedgeMapper);
  hedgeActor->GetProperty()->SetColor(colors->GetColor3d("Red").GetData());
  hedgeActor->GetProperty()->SetLineWidth(1.5);

  // --- 渲染 ---
  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("MidnightBlue").GetData());
  renderer->AddActor(sphereActor);
  renderer->AddActor(hedgeActor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

---

## 11.5 网格处理过滤器

网格处理过滤器用于改善网格的质量、精度或性能。它们修改的是网格的拓扑结构（连接关系）和几何属性（法向量、三角形数量等）。

### 11.5.1 vtkSmoothPolyDataFilter —— 拉普拉斯平滑

**功能描述：** 通过Laplacian平滑算法（将每个顶点向邻居顶点的平均位置移动）来平滑网格曲面。可以有效地去除噪声、消除不规则起伏、使曲面更加光顺。平滑迭代次数越多，曲面越光滑，但也越容易丢失几何细节。

**核心参数：**
- `SetNumberOfIterations(int n)` —— 平滑迭代次数（默认20）。越多越平滑，但也可能过度收缩。
- `SetRelaxationFactor(double factor)` —— 松弛因子（默认0.01，范围[0.0, 2.0]）。值越大，每次迭代顶点移动距离越大。
  - 0.0：顶点不动（无平滑）
  - 0.01-0.1：保守平滑，保留较多细节
  - 0.5-1.0：激进平滑，可能引起体积收缩
- `SetFeatureEdgeSmoothing(bool)` —— 是否平滑特征边（默认false，保留锐边）。
- `SetFeatureAngle(double angle)` —— 特征角定义，超过此角度的边视为特征边（默认45度）。
- `SetBoundarySmoothing(bool)` —— 是否平滑边界（默认true）。
- `SetConvergence(double value)` —— 收敛阈值（默认0.0即不检查收敛）。当顶点移动距离小于此值时提前终止迭代。

**适用场景：**
- 去除三维扫描（激光扫描、结构光扫描）数据中的噪声
- 改善有限元网格的质量
- 在等值面提取后（Marching Cubes生成的网格有时有阶梯状伪影）进行平滑
- 生成美观的可视化曲面

**代码示例：**

```cpp
// 对噪声网格应用平滑
auto smooth = vtkSmartPointer<vtkSmoothPolyDataFilter>::New();
smooth->SetInputConnection(noisySource->GetOutputPort());
smooth->SetNumberOfIterations(50);      // 50次迭代
smooth->SetRelaxationFactor(0.05);      // 保守的松弛因子（每一步只移动5%）
smooth->SetFeatureAngle(60.0);          // 保留超过60度的特征边
smooth->SetBoundarySmoothing(false);    // 不平滑边界
```

**参数调优经验：**
- 对于高噪声数据：减小松弛因子（0.01-0.03），增加迭代次数（100-200）。
- 对于轻微不平的网格：适中的松弛因子（0.1）和较少的迭代次数（10-20）。
- 如果发现模型在平滑后"缩水"，可以尝试交替使用`vtkWindowedSincPolyDataFilter`（一种受控平滑的滤波器）。

### 11.5.2 vtkDecimatePro —— 网格简化

**功能描述：** 通过反复合并边（Edge Collapse）来减少三角形数量，同时尽量保持模型的几何形状。这是生成LOD（Level of Detail，细节层次）的关键技术——你可以在远距离观察时使用简化模型以加快渲染，在近距离观察时使用原始高分辨率模型。

**核心参数：**
- `SetTargetReduction(double reduction)` —— 目标简化率，取值范围[0.0, 1.0]。0.0表示不减少任何三角形，0.9表示减少90%的三角形（只保留10%）。
- `SetPreserveTopology(bool)` —— 是否保持原始拓扑（默认false）。设为true可以防止网格在简化过程中产生孔洞。
- `SetFeatureAngle(double angle)` —— 特征角。超过该角度的边在简化中受到保护（不容易被折叠）。
- `SetMaximumError(double error)` —— 与原始网格之间的最大允许误差。
- `SetSplitting(bool)` —— 是否允许分裂（默认true）。
- `SetBoundaryVertexDeletion(bool)` —— 是否允许删除边界顶点（默认true）。

**适用场景：**
- 大型建筑模型的实时渲染（远处用简化模型）
- 移动设备上的3D可视化（需要降低三角形数量）
- Web端3D模型的带宽优化
- CAD模型中快速预览生成

**代码示例：**

```cpp
#include "vtkSmartPointer.h"
#include "vtkSphereSource.h"
#include "vtkDecimatePro.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkProperty.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"

int main()
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // --- 创建高分辨率球体 ---
  auto sphere = vtkSmartPointer<vtkSphereSource>::New();
  sphere->SetThetaResolution(100);
  sphere->SetPhiResolution(100);

  // 打印原始三角形数量
  sphere->Update();
  std::cout << "原始三角形数: " << sphere->GetOutput()->GetNumberOfPolys()
            << std::endl;

  // --- 网格简化（目标是减少90%的三角形） ---
  auto decimator = vtkSmartPointer<vtkDecimatePro>::New();
  decimator->SetInputConnection(sphere->GetOutputPort());
  decimator->SetTargetReduction(0.9);   // 减面90%
  decimator->SetPreserveTopology(true); // 保持拓扑不出现孔洞
  decimator->SetFeatureAngle(30.0);     // 保护锐边

  decimator->Update();
  std::cout << "简化后三角形数: " << decimator->GetOutput()->GetNumberOfPolys()
            << std::endl;

  // --- Mapper 和 Actor ---
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(decimator->GetOutputPort());

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(colors->GetColor3d("Tomato").GetData());
  actor->GetProperty()->SetEdgeVisibility(1);
  actor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.1);
  actor->GetProperty()->SetRepresentationToWireframe(); // 线框模式以看清简化结果

  // --- 渲染 ---
  auto renderer = vtkSmartPointer<vtkRenderer>::New();
  renderer->SetBackground(colors->GetColor3d("SlateGray").GetData());
  renderer->AddActor(actor);

  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->AddRenderer(renderer);

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  renWin->Render();
  interactor->Start();

  return 0;
}
```

**替代选择：** 如果需要更高的简化质量和更好的边界保持，可以考虑`vtkQuadricDecimation`（基于二次误差度量的简化算法，通常产生更高质量的结果但速度较慢）。

### 11.5.3 vtkPolyDataNormals —— 法向量计算

**功能描述：** 为PolyData计算/重算法向量。在VTK中，法向量对于光照效果至关重要——没有正确的法向量，曲面看起来会"平坦"或"破碎"。`vtkPolyDataNormals`通过分析每个顶点的相邻三角面片来计算平滑的法向量，也可以选择性地将锐边处的顶点拆分为多个副本以产生"硬边"效果。

**核心参数：**
- `SetFeatureAngle(double angle)` —— **这是最重要的参数。** 定义了"特征边"（Feature Edge）的角度阈值（度数制）。两个相邻三角形法向之间的夹角如果超过此角度，它们之间的边将被视为特征边，该边上的顶点会被拆分为两个副本以保留硬边外观。
  - 180度：所有三角形共用法向量（完全平滑）
  - 60-90度：适中的平滑，在锐边处保持清晰
  - 0度：每个三角形独立法向量（完全平面着色，Flat Shading）
- `SplittingOn()/Off()` —— 是否在特征边处拆分顶点（默认true）。
- `ConsistencyOn()/Off()` —— 是否修正法向量朝向的一致性（确保相邻三角形的法向量指向同一侧，默认true）。
- `ComputePointNormalsOn/Off()` —— 是否计算点法向量（默认true）。
- `ComputeCellNormalsOn/Off()` —— 是否计算单元法向量（默认false）。
- `FlipNormalsOn/Off()` —— 是否翻转所有法向量（在处理反转面的模型时有用）。
- `SetNonManifoldTraversal(bool)` —— 是否允许跨非流形边传播法向量（默认true）。

**工作原理简述：**

对于每个顶点，`vtkPolyDataNormals`找到所有包含该顶点的三角形面片。它计算这些三角形面片的**面法向量**（由三角形两条边叉积得到），然后将它们按面积加权平均，得到该顶点的**顶点法向量**。最后，它将平均结果归一化为单位长度。

`SetFeatureAngle`的作用在于：在计算平均时，如果两个相邻三角形的法向夹角超过特征角，则这两个三角形在计算时不会共享顶点法向量——相反，顶点会被"拆分"（Split）为两个副本，每个副本分别从各自的三角形集合接收法向量。这就是"硬边"的物理来源。

**适用场景：**
- 从外部文件导入的模型缺少法向量时，必须运行此过滤器
- 在Marching Cubes之后生成更平滑的等值面
- 需要在特定角度产生硬边效果（如机械零件的棱线）
- 检查模型的朝向一致性

**代码示例：**

```cpp
// 典型用法：加载模型后立即计算法向量
auto normals = vtkSmartPointer<vtkPolyDataNormals>::New();
normals->SetInputConnection(reader->GetOutputPort());
normals->SetFeatureAngle(30.0);   // 30度——产生清晰的机械棱线
normals->SplittingOn();           // 在特征边处拆分顶点
normals->ConsistencyOn();         // 确保法向量朝外

// 然后传递给Mapper
auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
mapper->SetInputConnection(normals->GetOutputPort());
```

### 11.5.4 vtkTriangleFilter —— 三角化过滤器

**功能描述：** 将输入数据集中的所有多边形（三角形、四边形、多边形）转换为**三角形**。这是最简单也最常用的过滤器之一——许多下游算法（如某些Smooth、Decimate、STL导出）要求输入必须全部是三角形。

**核心参数：**
- 这个过滤器几乎不需要配置。直接将输入连接上即可。
- 对于PolyData输入，使用`vtkTriangleFilter`。
- 对于UnstructuredGrid输入，使用`vtkDataSetTriangleFilter`。

**为什么需要三角化？**

- 三角形是唯一保证**共面**的多边形类型（任意三个点始终在同一平面内）。四边形可能不是平面的（四个点不一定共面），这会导致渲染和几何计算中的歧义。
- GPU原生支持三角形渲染——即使你提交了四边形，图形驱动程序也会在后台将其拆分为三角形。
- 许多几何算法（如碰撞检测、光线投射、STL文件格式）只支持三角形。

**代码示例：**

```cpp
// 最简单的一行用法：
auto triangles = vtkSmartPointer<vtkTriangleFilter>::New();
triangles->SetInputConnection(someSource->GetOutputPort());

// 之后的所有管道连接使用 triangles->GetOutputPort()
```

**典型管道组合：**

```cpp
// 从混合多边形的数据源到平滑处理的标准管道
Reader → TriangleFilter → SmoothPolyDataFilter → Normals → Mapper
```

---

## 11.6 代码示例：多过滤器流水线

### 11.6.1 项目概述

本节将构建一个**用同一数据源驱动四条不同过滤器分支**的综合演示程序。这是对VTK管道分支能力的完整展示，也是对本章所学过滤器的集中演练。

程序布局为**2x2视口**，四个视口分别展示：

| 视口位置 | 展示内容 | 过滤器链 |
|---------|---------|---------|
| 左上 (0,0) | 原始球体（带边线） | Source → Mapper（无过滤） |
| 右上 (0,1) | 收缩球体（展示结构） | Source → TriangleFilter → ShrinkFilter → Mapper |
| 左下 (1,0) | 等值面提取（标量场） | Source → ElevationFilter → ContourFilter → Mapper |
| 右下 (1,1) | 向量Glyph（法向量箭头） | Source → MaskPoints → Glyph3D → Mapper |

使用同一个`vtkSphereSource`的`GetOutputPort()`同时连接到四条管道分支，充分展示管道复用的强大能力。

### 11.6.2 管道结构全景图

```
                          vtkSphereSource
                               |
                    GetOutputPort()
                               |
            +------------------+------------------+------------------+
            |                  |                  |                  |
            v                  v                  v                  v
      无过滤器         TriangleFilter      ElevationFilter       MaskPoints
            |                  |                  |                  |
            v                  v                  v                  v
      PolyDataMapper    ShrinkPolyData    ContourFilter        Glyph3D
            |                  |                  |       (ArrowSource→Glyph3D)
            v                  v                  v                  |
       Actor(原始)       PolyDataMapper    PolyDataMapper     PolyDataMapper
            |                  |                  |                  |
            v                  v                  v                  v
       Actor(收缩)       Actor(等值面)      Actor(箭头)
            |                  |                  |                  |
            +------------------+------------------+------------------+
                               |
                       四个Actor分别添加到四个Renderer
                               |
              四个Renderer分别设置不同的视口范围
                               |
                      RenderWindow → Interactor
```

### 11.6.3 完整代码

```cpp
// ============================================================================
// MultiFilterPipeline.cxx
// VTK 第十一章综合示例：多过滤器流水线
//
// 演示核心概念：
//   1. 管道分支：同一数据源驱动4条不同的过滤器链
//   2. 2x2视口布局：使用SetViewport()精确布置
//   3. 多种过滤器组合：Shrink, Contour, Glyph3D
//   4. 文本标注：每个视口上方的说明标签
//   5. 统一交互：四个Renderer共享一个Interactor
// ============================================================================

#include "vtkActor.h"
#include "vtkArrowSource.h"
#include "vtkCamera.h"
#include "vtkContourFilter.h"
#include "vtkElevationFilter.h"
#include "vtkGlyph3D.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkMaskPoints.h"
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
// 辅助函数：创建视口标签
// ============================================================================
// 在指定视口的顶部中央放置一个文本标签
// 参数:
//   renderer - 目标Renderer
//   text     - 标签文本内容
//   yPos     - 文本在视口内的归一化Y位置（0.0=底部, 1.0=顶部）
// ============================================================================
vtkSmartPointer<vtkTextActor> AddViewportLabel(
    vtkRenderer* renderer, const std::string& text, double yPos = 0.92)
{
  auto label = vtkSmartPointer<vtkTextActor>::New();
  label->SetInput(text.c_str());
  label->GetTextProperty()->SetFontSize(14);
  label->GetTextProperty()->SetColor(1.0, 1.0, 1.0);        // 白色文字
  label->GetTextProperty()->BoldOn();
  label->GetTextProperty()->SetJustificationToCentered();
  label->GetTextProperty()->SetVerticalJustificationToTop();

  // 将文本Actor直接添加到Renderer（非视口归一化坐标）
  // 使用vtkTextActor的SetPosition (以像素为单位不直观)
  // 改用SetDisplayPosition——在display坐标中定位
  // 但更简单的方式：使用vtkLegendScaleActor或手动设置
  // 这里我们使用Position在world坐标中的方案：
  // 由于每个视口有不同的相机，这里使用标尺actor定位更方便
  // 实际方案：使用SetPosition在归一化显示坐标中（需手动转换）
  renderer->AddActor2D(label);
  // 在视口坐标系中设置位置（底部居中）
  label->SetPosition(0.5 * (renderer->GetViewport()[0] +
                             renderer->GetViewport()[2]) * 800,
                     0.91 * 600); // 使用固定窗口尺寸

  return label;
}

// ============================================================================
// 辅助函数：创建并配置一个Actor
// ============================================================================
vtkSmartPointer<vtkActor> CreateActor(
    vtkAlgorithmOutput* port,
    double r, double g, double b,
    bool showEdges = false,
    double edgeR = 0.2, double edgeG = 0.2, double edgeB = 0.2,
    double opacity = 1.0)
{
  auto mapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  mapper->SetInputConnection(port);
  mapper->ScalarVisibilityOff();

  auto actor = vtkSmartPointer<vtkActor>::New();
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(r, g, b);
  actor->GetProperty()->SetDiffuse(0.8);
  actor->GetProperty()->SetSpecular(0.3);
  actor->GetProperty()->SetSpecularPower(10.0);
  actor->GetProperty()->SetOpacity(opacity);

  if (showEdges)
  {
    actor->GetProperty()->SetEdgeVisibility(1);
    actor->GetProperty()->SetEdgeColor(edgeR, edgeG, edgeB);
    actor->GetProperty()->SetLineWidth(1.0);
  }

  return actor;
}

// ============================================================================
// main
// ============================================================================
int main(int argc, char* argv[])
{
  auto colors = vtkSmartPointer<vtkNamedColors>::New();

  // =========================================================================
  // Step 1: 创建共享数据源
  // =========================================================================
  // 所有四条管道分支共享同一个球体数据源
  auto sphereSource = vtkSmartPointer<vtkSphereSource>::New();
  sphereSource->SetThetaResolution(48);
  sphereSource->SetPhiResolution(48);
  sphereSource->SetRadius(1.0);

  // 获取输出端口（将被四条分支共享）
  auto sphereOutput = sphereSource->GetOutputPort();

  // =========================================================================
  // Step 2: 构建四条过滤器分支
  // =========================================================================

  // ---- 分支A：原始球体（无过滤器，仅渲染） ----
  auto actorA = CreateActor(sphereOutput, 0.3, 0.6, 0.9, true, 0.1, 0.1, 0.2);

  // ---- 分支B：收缩球体（展示网格结构） ----
  // 管道：Sphere -> TriangleFilter -> ShrinkPolyData -> Mapper
  auto triangleB = vtkSmartPointer<vtkTriangleFilter>::New();
  triangleB->SetInputConnection(sphereOutput);

  auto shrink = vtkSmartPointer<vtkShrinkPolyData>::New();
  shrink->SetInputConnection(triangleB->GetOutputPort());
  shrink->SetShrinkFactor(0.75);

  auto actorB = CreateActor(shrink->GetOutputPort(),
                            0.9, 0.4, 0.3,   // 暖橙色
                            true,             // 显示边线
                            0.15, 0.1, 0.1);  // 深色边线

  // ---- 分支C：等值面提取（先添加标量高程数据） ----
  // 管道：Sphere -> ElevationFilter -> ContourFilter -> Mapper
  auto elevation = vtkSmartPointer<vtkElevationFilter>::New();
  elevation->SetInputConnection(sphereOutput);
  elevation->SetLowPoint(0.0, -1.0, 0.0);
  elevation->SetHighPoint(0.0, 1.0, 0.0);
  // ElevationFilter 自动为每个点生成标量值（基于Y坐标）

  auto contour = vtkSmartPointer<vtkContourFilter>::New();
  contour->SetInputConnection(elevation->GetOutputPort());
  // 生成7个等值面（在Y坐标范围内均匀分布）
  contour->GenerateValues(7, -1.0, 1.0);
  contour->SetComputeNormals(true);

  // 等值面使用标量颜色映射
  auto contourMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  contourMapper->SetInputConnection(contour->GetOutputPort());
  contourMapper->SetScalarRange(-1.0, 1.0);
  // 使用蓝-白-红的颜色查找表
  contourMapper->ScalarVisibilityOn();

  auto actorC = vtkSmartPointer<vtkActor>::New();
  actorC->SetMapper(contourMapper);
  actorC->GetProperty()->SetOpacity(0.85);

  // ---- 分支D：向量Glyph（球体法向量以箭头表示） ----
  // 管道：Sphere -> MaskPoints(降采样) -> Glyph3D(Arrow) -> Mapper
  // MaskPoints作用：减少点数，避免箭头过于密集
  auto maskPoints = vtkSmartPointer<vtkMaskPoints>::New();
  maskPoints->SetInputConnection(sphereOutput);
  maskPoints->SetOnRatio(8);            // 每8个点保留1个
  maskPoints->SetMaximumNumberOfPoints(200);
  maskPoints->RandomModeOff();          // 均匀采样
  maskPoints->GenerateVerticesOn();

  // 创建箭头Glyph源
  auto arrowSource = vtkSmartPointer<vtkArrowSource>::New();
  arrowSource->SetTipLength(0.20);
  arrowSource->SetTipRadius(0.07);
  arrowSource->SetShaftRadius(0.03);

  // Glyph3D: 在降采样后的点上放置箭头
  auto glyph = vtkSmartPointer<vtkGlyph3D>::New();
  glyph->SetInputConnection(maskPoints->GetOutputPort());
  glyph->SetSourceConnection(arrowSource->GetOutputPort());
  glyph->SetVectorModeToUseNormal();    // 使用点法向量作为方向
  glyph->SetScaleModeToDataScalingOff();// 所有箭头统一大小
  glyph->SetScaleFactor(0.25);
  glyph->OrientOn();                    // 箭头朝向跟随法向量
  glyph->SetColorModeToColorByVector(); // 颜色由矢量映射

  auto glyphMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
  glyphMapper->SetInputConnection(glyph->GetOutputPort());
  glyphMapper->ScalarVisibilityOff();

  auto actorD = vtkSmartPointer<vtkActor>::New();
  actorD->SetMapper(glyphMapper);
  actorD->GetProperty()->SetColor(0.2, 0.8, 0.3);  // 绿色箭头
  actorD->GetProperty()->SetSpecular(0.5);
  actorD->GetProperty()->SetSpecularPower(20);

  // =========================================================================
  // Step 3: 创建四个Renderer，配置各自的视口和标签
  // =========================================================================
  double bgColor[3];
  colors->GetColor("MidnightBlue", bgColor);

  // 视口划分：2行x2列
  // 视口坐标格式：(xmin, ymin, xmax, ymax)
  struct ViewportConfig
  {
    double xmin, ymin, xmax, ymax;
    std::string title;
    vtkSmartPointer<vtkActor> actor;
    int labelPosX;  // 标签X位置（像素，基于800x600窗口）
    int labelPosY;  // 标签Y位置（像素）
  };

  // 上半部分: y从0.52到1.0; 下半部分: y从0.0到0.48
  // 左半部分: x从0.0到0.49; 右半部分: x从0.51到1.0
  // 中间留2%的间隙
  std::array<ViewportConfig, 4> viewports = {{
    { 0.00, 0.52, 0.49, 1.00, "A: Original Sphere",        actorA, 195, 580 },
    { 0.51, 0.52, 1.00, 1.00, "B: Shrunk (Show Structure)", actorB, 595, 580 },
    { 0.00, 0.00, 0.49, 0.48, "C: Contour (Isosurfaces)",  actorC, 195, 280 },
    { 0.51, 0.00, 1.00, 0.48, "D: Glyph (Normal Vectors)", actorD, 595, 280 },
  }};

  // 创建四个Renderer
  std::array<vtkSmartPointer<vtkRenderer>, 4> renderers;

  for (int i = 0; i < 4; i++)
  {
    auto& vp = viewports[i];
    auto renderer = vtkSmartPointer<vtkRenderer>::New();
    renderer->SetViewport(vp.xmin, vp.ymin, vp.xmax, vp.ymax);
    renderer->SetBackground(bgColor[0], bgColor[1], bgColor[2]);
    renderer->AddActor(vp.actor);

    // 添加文本标签（使用vtkTextActor —— 在视口归一化坐标中定位）
    auto label = vtkSmartPointer<vtkTextActor>::New();
    label->SetInput(vp.title.c_str());
    label->GetTextProperty()->SetFontSize(13);
    label->GetTextProperty()->SetColor(1.0, 1.0, 1.0);
    label->GetTextProperty()->BoldOn();
    label->GetTextProperty()->SetJustificationToCentered();
    label->GetTextProperty()->SetVerticalJustificationToBottom();
    renderer->AddActor2D(label);

    // vtkTextActor 使用 display 坐标定位（像素）
    // 计算视口中心X坐标：视口左边缘 + 视口宽度的一半
    int centerX = static_cast<int>((vp.xmin + (vp.xmax - vp.xmin) * 0.5) * 800);
    int labelY = static_cast<int>(vp.ymax * 600) - 25;  // 视口顶部下方25像素
    label->SetDisplayPosition(centerX - 100, labelY);

    renderers[i] = renderer;
  }

  // =========================================================================
  // Step 4: 创建RenderWindow，将四个Renderer全部添加
  // =========================================================================
  auto renWin = vtkSmartPointer<vtkRenderWindow>::New();
  renWin->SetSize(800, 600);
  renWin->SetWindowName("Multi-Filter Pipeline Demo (Chapter 11)");

  for (auto& renderer : renderers)
  {
    renWin->AddRenderer(renderer);
  }

  // =========================================================================
  // Step 5: 设置交互器
  // =========================================================================
  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renWin);
  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  // =========================================================================
  // Step 6: 初始化每个视口的相机并渲染
  // =========================================================================
  renWin->Render();

  // 所有视口共享相似的视角
  for (auto& renderer : renderers)
  {
    renderer->GetActiveCamera()->Azimuth(30);
    renderer->GetActiveCamera()->Elevation(20);
    renderer->GetActiveCamera()->Dolly(1.2);
    renderer->ResetCamera();
  }

  renWin->Render();
  interactor->Start();

  return 0;
}
```

### 11.6.4 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(MultiFilterPipeline VERSION 1.0 LANGUAGES CXX)

# --------------------------------------------------------------------------
# VTK依赖配置
# --------------------------------------------------------------------------
find_package(VTK REQUIRED COMPONENTS
  CommonCore
  CommonDataModel
  CommonExecutionModel
  CommonMath
  CommonTransforms
  FiltersCore
  FiltersGeneral
  FiltersSources
  FiltersModeling
  FiltersFlowPaths
  InteractionStyle
  RenderingCore
  RenderingOpenGL2
  RenderingAnnotation
  RenderingFreeType
)
vtk_module_scan(VTK_MODULES modules ${VTK_LIBRARIES})

# --------------------------------------------------------------------------
# 可执行文件
# --------------------------------------------------------------------------
add_executable(MultiFilterPipeline MultiFilterPipeline.cxx)

target_link_libraries(MultiFilterPipeline PRIVATE ${modules})

target_compile_features(MultiFilterPipeline PRIVATE cxx_std_17)

# 从VTK获取包含目录
target_include_directories(MultiFilterPipeline SYSTEM PRIVATE
  ${VTK_INCLUDE_DIRS}
)

# --------------------------------------------------------------------------
# 安装规则
# --------------------------------------------------------------------------
install(TARGETS MultiFilterPipeline
  RUNTIME DESTINATION bin
)
```

### 11.6.5 编译和运行

在项目根目录下执行以下命令：

```bash
# 创建构建目录
mkdir build && cd build

# 配置CMake（需要指定VTK_DIR到你的VTK安装路径）
cmake .. -DVTK_DIR=/path/to/vtk-9.5.2/build

# 编译
cmake --build . --config Release

# 运行
./MultiFilterPipeline      # Linux/macOS
# 或
MultiFilterPipeline.exe    # Windows
```

### 11.6.6 代码要点分析

**1. 管道分支复用数据源：**

```cpp
auto sphereOutput = sphereSource->GetOutputPort();  // 获取一次

// 四条分支共享同一个 sphereOutput
actorA = CreateActor(sphereOutput, ...);           // 分支A直接使用
triangleB->SetInputConnection(sphereOutput);       // 分支B以此为起点
elevation->SetInputConnection(sphereOutput);       // 分支C以此为起点
maskPoints->SetInputConnection(sphereOutput);      // 分支D以此为起点
```

这与第六章的管道分支模式完全一致——但分支数从2条扩展到4条。

**2. MaskPoints过滤器降采样：**

Ball体有48x48=2304个点。如果每个点都放置一个箭头，箭头会非常密集，完全看不出矢量场的结构。`vtkMaskPoints`通过`SetOnRatio(8)`将点数降为原来的1/8，只保留约288个点，箭头变得清晰可辨。

**3. ElevationFilter + ContourFilter组合：**

`vtkElevationFilter`以两个点定义一条参考线，计算每个数据点在该参考线上的投影位置作为标量值。这里使用`SetLowPoint(0,-1,0)`和`SetHighPoint(0,1,0)`，使得Y坐标为每个点的标量值。然后`vtkContourFilter`提取7个等值面，相当于按Y坐标将球体切成7个平行的环。

**4. Actor2D标签定位：**

文本标签使用`AddActor2D()`添加到Renderer，用`SetDisplayPosition()`以像素坐标定位。这样标签位置独立于3D相机变换，始终固定在视口上方。

---

## 11.7 本章小结

### 11.7.1 过滤器选择指南

本章介绍了VTK中最常用的14个过滤器。面对一个可视化任务，如何快速选择合适的过滤器？下表提供指南：

| 你想要做什么 | 推荐过滤器 | 关键参数 |
|-------------|-----------|---------|
| 展示网格单元结构 | `vtkShrinkPolyData` / `vtkShrinkFilter` | `SetShrinkFactor()` |
| 平移/旋转/缩放几何体 | `vtkTransformPolyDataFilter` | `SetTransform(vtkTransform*)` |
| 将标量场映射为曲面高度 | `vtkWarpScalar` | `SetScaleFactor()` |
| 将矢量场映射为几何变形 | `vtkWarpVector` | `SetScaleFactor()` |
| 用平面切掉模型的一部分 | `vtkClipPolyData` / `vtkClipDataSet` | `SetClipFunction(vtkPlane*)`, `SetInsideOut()` |
| 从体数据中提取等值面 | `vtkContourFilter` | `SetValue()` / `GenerateValues()` |
| 按标量范围提取单元 | `vtkThreshold` | `ThresholdBetween()` |
| 获取模型的任意截面 | `vtkCutter` | `SetCutFunction(vtkPlane*)` |
| 提取网格的线框 | `vtkExtractEdges` | （几乎无需参数） |
| 在点上放置箭头/球体/锥体 | `vtkGlyph3D` | `SetSourceConnection()`, `SetScaleMode()`, `OrientOn()` |
| 沿矢量场追踪流线 | `vtkStreamTracer` | `SetSourceConnection(seeds)`, `SetIntegrator()` |
| 快速显示矢量场方向 | `vtkHedgeHog` | `SetScaleFactor()` |
| 平滑有噪声的网格 | `vtkSmoothPolyDataFilter` | `SetNumberOfIterations()`, `SetRelaxationFactor()` |
| 减少三角形数量 | `vtkDecimatePro` | `SetTargetReduction()` |
| 计算/重算法向量 | `vtkPolyDataNormals` | `SetFeatureAngle()`, `SplittingOn()` |
| 将所有多边形转为三角形 | `vtkTriangleFilter` | （几乎无需参数） |

### 11.7.2 核心概念回顾

1. **过滤器的统一范式**：所有过滤器都继承自`vtkAlgorithm`，使用`SetInputConnection()`和`GetOutputPort()`连接，遵循按需执行的流水线机制。

2. **管道分支**：一个数据源可以同时驱动多个下游过滤器链。数据源只需计算一次，多个分支共享结果。这是VTK区别于"立即模式"图形库的核心优势之一。

3. **过滤器链的组合能力**：复杂的可视化效果往往来自于多个过滤器的**串联**：
   - `ElevationFilter → ContourFilter` 产生等值面环
   - `TriangleFilter → ShrinkFilter → Mapper` 产生干净的结构图
   - `MaskPoints → Glyph3D` 产生可控密度的矢量符号图
   - `Reader → TriangleFilter → SmoothFilter → Normals → Mapper` 产生光滑的导入模型

4. **输出类型可能改变**：`vtkThreshold`将`vtkPolyData`输入转换为`vtkUnstructuredGrid`输出。`vtkClipPolyData`的输出仍然是`vtkPolyData`（但底层拓扑已经改变）。理解输出类型对于正确选择后续Mapper至关重要。

5. **法向量是关键**：许多可视化效果（光照、Glyph朝向）依赖正确的法向量。当模型看起来"破碎"或"太暗"时，首先检查`vtkPolyDataNormals`是否正确运行。

### 11.7.3 下一步

本章覆盖了VTK中最常用的过滤器。你现在已经具备了处理绝大多数科学可视化任务的能力。在后续章节中，你将学习：

- **第十二章：颜色映射与标量着色**——如何使用查找表（Lookup Table）和颜色映射将数据值转化为视觉颜色，包括自定义颜色映射、对数映射、离散映射。
- **第十三章：高级渲染技术**——透明度、阴影、体积渲染、多Pass渲染、环境光遮蔽（SSAO）。
- **第十四章：多数据集可视化**——在同一个场景中管理多个不同的数据集，使用`vtkAssembly`构建层级结构。

带着本章建立的过滤器工具箱，你将能够从"拿到数据看个大概"升级到"精准地提取和表达数据中隐藏的信息"——这正是科学可视化的核心价值所在。

---
