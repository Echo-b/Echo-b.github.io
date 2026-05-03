# 第七章 PolyData详解（PolyData Deep Dive）

## 本章导读

在第三章中，我们初次遇见了`vtkPolyData`——那是在数据模型概览中，它被描述为VTK中"最灵活、最常用的数据集类型之一"。第六章的"3D几何画廊"综合实战中，你看到了`vtkPolyData`在过滤器、Mapper和Actor之间流转的身影。但你或许还没有机会停下来，仔细审视这个数据类型本身：它如何组织数据？如何用代码构造一个带有顶点、线段、三角形、纹理坐标和法向量的完整几何模型？

本章就是为此而来。我们将对`vtkPolyData`进行一次深入、完整、从原理到实践的审视。你将学习：

1. **PolyData的双层结构**——拓扑（Topology，单元连接关系）与几何（Geometry，点坐标）的分离设计，以及这种分离为何是VTK数据模型中最优雅的设计决策之一。
2. **四种基本图元类型**——Verts（点元）、Lines（线元）、Polys（面元）、Strips（三角带），它们在`vtkCellArray`中如何存储，以及如何被组装到同一个`vtkPolyData`中。
3. **`vtkPoints`与`vtkCellArray`的完整API**——从`InsertNextPoint()`到`InsertNextCell()`，从传统的"两步插入法"到现代的`std::initializer_list`语法。
4. **法向量（Normals）与着色**——光滑着色（Smooth Shading）与平面着色（Flat Shading）的本质区别，`vtkPolyDataNormals`过滤器的工作原理，以及`SetFeatureAngle()`如何控制两者之间的过渡。
5. **纹理坐标（Texture Coordinates）**——如何将二维图像映射到三维曲面上，以及纹理坐标与几何坐标之间的关系。
6. **一个完整的综合代码示例**——从零开始构建一个带高程着色和表面法向量的3D地形曲面，整合本章所有知识点，并包含完整可编译的CMakeLists.txt。

本章是第二卷（进阶篇）的起点。如果说第一卷帮助你"会用VTK"，本章将帮助你"理解VTK如何表示几何世界"。读完本章后，你将对手动构建任意`vtkPolyData`充满信心——无论是程序化生成的数学曲面、从外部传感器采集的点云、还是自定义的CAD几何体。

---

## 7.1 PolyData的结构

### 7.1.1 拓扑与几何的分离

`vtkPolyData`的核心设计理念可以浓缩为一句话：**拓扑（Topology）和几何（Geometry）是分开存储的**。

- **几何**指的是点在三维空间中的实际位置——`(x, y, z)`坐标。在`vtkPolyData`中，几何由`vtkPoints`对象管理。
- **拓扑**指的是点之间的连接关系——哪些点组成一条线段，哪些点组成一个三角形面片。在`vtkPolyData`中，拓扑由四个独立的`vtkCellArray`对象管理。

这种分离意味着：一段描述三角形面片的拓扑数据`{0, 1, 2}`是独立于这三个点的实际坐标的。你可以修改点的坐标（例如在动画中移动顶点），而单元的拓扑定义保持不变。反过来，你也可以修改拓扑（例如将一个四边形拆分为两个三角形）而不需要改变任何点的坐标。

这种设计在科学可视化中至关重要：
- **网格变形**：在有限元分析中，结构受力变形时点坐标改变，但单元的连接关系（哪些点组成一个六面体单元）保持不变。
- **网格细化**：你可以重新三角化一个曲面（改变拓扑），但保留原始采样点的位置（保持几何）。
- **内存效率**：多个单元可以共享同一个点，而不需要为每个单元重复存储点的坐标。

### 7.1.2 四类基本图元

`vtkPolyData`支持四种不同维度的图元类型，每一种都由一个独立的`vtkCellArray`管理：

| 图元类型 | 维度 | 存储位置 | 支持的单元格类型 |
|----------|------|----------|-----------------|
| **Verts（点元）** | 0D | `GetVerts()` / `SetVerts()` | `VTK_VERTEX` (1), `VTK_POLY_VERTEX` (2) |
| **Lines（线元）** | 1D | `GetLines()` / `SetLines()` | `VTK_LINE` (3), `VTK_POLY_LINE` (4) |
| **Polys（面元）** | 2D | `GetPolys()` / `SetPolys()` | `VTK_TRIANGLE` (5), `VTK_QUAD` (9), `VTK_POLYGON` (7) |
| **Strips（三角带）** | 2D | `GetStrips()` / `SetStrips()` | `VTK_TRIANGLE_STRIP` (6) |

一个`vtkPolyData`可以同时包含所有这些图元类型。例如，一个科学可视化场景可能包含：
- 散点图（Verts——每个采样点是一个独立的顶点）
- 粒子轨迹线（Lines——每条轨迹是一条折线段）
- 等值面（Polys——由三角形组成的三维曲面）
- 地形条带（Strips——高效存储规则采样地形）

这种混合能力使得`vtkPolyData`异常灵活。但请注意VTK头文件中的警告：如果要在同一个`vtkPolyData`中混合使用多种类型，必须按照**Verts、Lines、Polys、Strips**的顺序插入单元，否则单元ID的一致性会被破坏，单元数据（Cell Data）的渲染也可能出错。

### 7.1.3 vtkCellArray的内部存储结构（现代格式）

理解`vtkCellArray`的内部存储格式对于高效使用`vtkPolyData`至关重要。在VTK 9.x中，`vtkCellArray`使用一种称为"偏移量+连接性"（Offsets + Connectivity）的双数组结构：

```
假设我们有 3 个单元：
  单元 0: Triangle | 点索引: {0, 1, 2}
  单元 1: Quad     | 点索引: {3, 4, 6, 7}
  单元 2: Line     | 点索引: {5, 8}

内部存储：
  Offsets:      {0, 3, 7, 9}   ← NumCells+1 个值，每个值指向Connectivity中的起始位置
  Connectivity: {0, 1, 2, 3, 4, 6, 7, 5, 8}  ← 所有点ID的连续排列
```

解析方式：
- 单元0：`Offsets[0]=0`, `Offsets[1]=3`，读取`Connectivity[0..2]` → `{0, 1, 2}`，长度=3
- 单元1：`Offsets[1]=3`, `Offsets[2]=7`，读取`Connectivity[3..6]` → `{3, 4, 6, 7}`，长度=4
- 单元2：`Offsets[2]=7`, `Offsets[3]=9`，读取`Connectivity[7..8]` → `{5, 8}`，长度=2

这种格式的优点是：定位任意单元时不需要遍历所有前置单元——通过Offsets数组可以直接跳转到Connectivity的对应位置。这支持了`O(1)`时间复杂度的随机访问。

**与旧格式的对比：** 在VTK的旧存储格式（已废弃但仍被一些老旧代码使用）中，单元长度被直接嵌入Connectivity数组：
```
Connectivity（旧格式）: {3, 0, 1, 2, 4, 3, 4, 6, 7, 2, 5, 8}
                        |--Cell 0--||----Cell 1---||--C2-|
```
旧格式中每个单元的第一个数字是点数。这种格式的随机访问需要遍历并累计，效率较低。VTK 9.x中已默认使用现代Offsets+Connectivity格式（由`VTK_CELL_ARRAY_V2`宏标记）。

### 7.1.4 点索引如何映射到坐标——图解

下面用ASCII图展示一个简单PolyData的完整结构：

```
  vtkPoints (几何层)
  ====================
  索引  坐标
  [0]  (0.0, 0.0, 0.0)    ← 原点
  [1]  (1.0, 0.0, 0.0)    ← X轴方向1个单位
  [2]  (0.0, 1.0, 0.0)    ← Y轴方向1个单位
  [3]  (1.0, 1.0, 0.0)    ← XY对角点
  [4]  (0.5, 0.5, 1.0)    ← 中心上方（金字塔尖）


  vtkCellArray: Polys (拓扑层)
  ==============================
  Offsets:      {0, 3, 6, 10}
  Connectivity: {0, 1, 4,  2, 0, 4,  1, 3, 4, 2}
                 |--△t0--|  |--△t1--|  |--□q0--|

  △t0: 三角形(0,1,4) → 顶点1→2→尖
  △t1: 三角形(2,0,4) → 顶点0→1→尖
  □q0: 四边形(1,3,4,2) → 底面四边形


  三维空间中的视觉效果:
                (0.5, 0.5, 1.0)
                     [4]
                     /|\
                    / | \
                   /  |  \
                  / t0|t1 \
                 /    |    \
         [0]   /_____|_____\  [3]
     (0,0,0)  *------+------* (1,1,0)
              |     / \     |
              |    /   \    |
              |   /  q0 \   |
              |  /       \  |
              *------------*
         [1]               [2]
     (1,0,0)             (0,1,0)
```

**关键理解：** 每个单元并不存储坐标，只存储整数索引。渲染时，Mapper通过索引查阅`vtkPoints`获取实际的`(x, y, z)`坐标。当点坐标发生变化时（如`points->SetPoint(4, 0.6, 0.6, 1.2)`），所有引用该点的单元（t0, t1, q0）的形状会自动随之变化——因为索引不变，但坐标变了。

### 7.1.5 PolyData的内部成员概览

从`vtkPolyData`头文件（`Common/DataModel/vtkPolyData.h`）中可以看到，`vtkPolyData`内部持有以下核心成员：

```cpp
// 从vtkPointSet继承：vtkPoints* Points  —— 点坐标
// 从vtkDataSet继承：vtkPointData  —— 点属性数据（标量、矢量、法向等）
//                      vtkCellData   —— 单元属性数据

// vtkPolyData特有的四个CellArray:
vtkSmartPointer<vtkCellArray> Verts;   // 0D 点元
vtkSmartPointer<vtkCellArray> Lines;   // 1D 线元
vtkSmartPointer<vtkCellArray> Polys;   // 2D 面元（三角形、四边形、多边形）
vtkSmartPointer<vtkCellArray> Strips;  // 2D 三角形带

// 辅助结构（按需构建）:
vtkSmartPointer<CellMap> Cells;        // 全局单元索引映射
vtkSmartPointer<vtkAbstractCellLinks> Links;  // 点到单元的向上链接
```

`Cells`（CellMap）和`Links`并非总是存在。它们在使用`BuildCells()`或`BuildLinks()`时才被构建，用于加速单元随机访问和拓扑查询。直接操作Verts/Lines/Polys/Strips CellArray时，这些缓存结构会失效，需要重新构建。

---

## 7.2 点（Points）与顶点（Verts）

### 7.2.1 vtkPoints——几何的容器

`vtkPoints`是VTK中管理三维点坐标的核心类。它与`vtkCellArray`一样独立于`vtkPolyData`存在——你可以在没有PolyData对象的情况下创建和操作Points。

**核心API：**

```cpp
#include "vtkPoints.h"

// 创建Points对象
vtkNew<vtkPoints> points;

// 方法1：InsertNextPoint —— 在末尾追加一个新点，返回其索引
vtkIdType id0 = points->InsertNextPoint(0.0, 0.0, 0.0);   // 返回 0
vtkIdType id1 = points->InsertNextPoint(1.0, 0.0, 0.0);   // 返回 1
vtkIdType id2 = points->InsertNextPoint(0.5, 0.866, 0.0); // 返回 2

// 方法2：SetPoint —— 设置或修改指定索引处的点坐标（索引必须已存在）
points->SetPoint(1, 2.0, 0.0, 0.0);  // 修改 pt[1] 的坐标

// 方法3：GetPoint —— 获取指定索引处的点坐标
double pt[3];
points->GetPoint(2, pt);  // pt现在 = {0.5, 0.866, 0.0}

// 方法4：GetNumberOfPoints —— 获取点的总数
vtkIdType numPts = points->GetNumberOfPoints();
std::cout << "Total points: " << numPts << std::endl;

// 方法5：访问底层数据（高级用法）
// 获取内部数组的只读指针
vtkDataArray* dataArray = points->GetData();
// 或者获取指定类型的指针（注意：实际类型取决于VTK内部选择）
double* rawPtr = static_cast<vtkDoubleArray*>(points->GetData())->GetPointer(0);
// 遍历所有点：第i个点的x坐标 = rawPtr[i*3], y = rawPtr[i*3+1], z = rawPtr[i*3+2]

// 方法6：SetNumberOfPoints —— 预分配空间
points->SetNumberOfPoints(10000);  // 预分配后可以用SetPoint逐个设置
```

**性能提示：** 当你知道将要插入多少个点时，使用`SetNumberOfPoints()`或`Allocate()`预分配空间可以避免动态扩容带来的多次内存重新分配和数据复制。对于10万+点的大规模点云，这一优化可以将插入速度提升数倍。

**特殊技巧——从外部数组传递数据：**

```cpp
// 场景：你已有一个外部生成的点坐标数组
std::vector<double> coords = {0,0,0, 1,0,0, 0.5,0.866,0, ...};

vtkNew<vtkPoints> points;
// 直接写入底层数据数组
vtkNew<vtkDoubleArray> dataArray;
dataArray->SetNumberOfComponents(3);
dataArray->SetArray(coords.data(), coords.size(), 1); // 1 = 保存引用，不复制
points->SetData(dataArray);
```

### 7.2.2 顶点（Verts）——零维图元

Verts（点元）是最简单的图元类型——每个单元由单个点组成。它们用于点云可视化、粒子效果、或散点图。

**创建单个顶点（VTK_VERTEX）：**

```cpp
#include "vtkCellArray.h"
#include "vtkPolyData.h"

vtkNew<vtkPoints> points;
points->InsertNextPoint(0.0, 0.0, 0.0);
points->InsertNextPoint(1.0, 0.0, 0.0);
points->InsertNextPoint(0.5, 1.0, 0.0);

// 创建Verts的CellArray
vtkNew<vtkCellArray> verts;

// 方法A: 两步法 —— 先声明点数，再逐个插入点索引
verts->InsertNextCell(1);     // 下一个单元有1个点
verts->InsertCellPoint(0);    // 单元0: 点索引0

verts->InsertNextCell(1);     // 下一个单元有1个点
verts->InsertCellPoint(1);    // 单元1: 点索引1

verts->InsertNextCell(1);
verts->InsertCellPoint(2);    // 单元2: 点索引2

// 方法B: 一次性传递整组索引（推荐）
vtkNew<vtkCellArray> vertsB;
vtkIdType pts0[] = {0};
vtkIdType pts1[] = {1};
vtkIdType pts2[] = {2};
vertsB->InsertNextCell(1, pts0);
vertsB->InsertNextCell(1, pts1);
vertsB->InsertNextCell(1, pts2);

// 方法C: 使用C++11 initializer_list（最简洁）
vtkNew<vtkCellArray> vertsC;
vertsC->InsertNextCell({0});
vertsC->InsertNextCell({1});
vertsC->InsertNextCell({2});

// 组装PolyData
vtkNew<vtkPolyData> pointCloud;
pointCloud->SetPoints(points);
pointCloud->SetVerts(vertsC);
```

**创建多顶点（VTK_POLY_VERTEX）：**
`VTK_POLY_VERTEX`是一个包含多个点的"集合顶点"——它与多个独立的`VTK_VERTEX`在内存布局和渲染行为上不同。通常`VTK_VERTEX`更常用，但在某些特定场景（如需要将一组点作为一个整体单元处理）中，`VTK_POLY_VERTEX`更合适。

### 7.2.3 示例：点云可视化

下面是一个完整的点云可视化程序。我们生成100个随机分布的点，并为每个点赋予随机的颜色。

```cpp
// point_cloud_example.cxx
// 生成随机点云并可视化——演示 vtkPolyData 的 Verts 用法

#include "vtkActor.h"
#include "vtkCellArray.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"
#include "vtkNew.h"
#include "vtkPointData.h"
#include "vtkPoints.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkUnsignedCharArray.h"

#include <cstdlib>
#include <ctime>

int main()
{
  // 初始化随机种子
  std::srand(static_cast<unsigned>(std::time(nullptr)));

  const vtkIdType numParticles = 100;

  // ============================================================
  // 第一步：创建几何（点坐标）
  // ============================================================
  vtkNew<vtkPoints> points;
  points->SetNumberOfPoints(numParticles);

  for (vtkIdType i = 0; i < numParticles; ++i)
  {
    // 在 [-5, 5]^3 的立方体内随机生成点
    double x = (std::rand() / (double)RAND_MAX) * 10.0 - 5.0;
    double y = (std::rand() / (double)RAND_MAX) * 10.0 - 5.0;
    double z = (std::rand() / (double)RAND_MAX) * 10.0 - 5.0;
    points->SetPoint(i, x, y, z);
  }

  // ============================================================
  // 第二步：创建拓扑（顶点单元）
  // ============================================================
  vtkNew<vtkCellArray> verts;
  for (vtkIdType i = 0; i < numParticles; ++i)
  {
    // 为每个点创建一个独立的 VTK_VERTEX 单元
    verts->InsertNextCell(1);
    verts->InsertCellPoint(i);
  }

  // ============================================================
  // 第三步：组装 vtkPolyData
  // ============================================================
  vtkNew<vtkPolyData> polyData;
  polyData->SetPoints(points);
  polyData->SetVerts(verts);

  // ============================================================
  // 第四步：创建属性数据（粒子颜色——基于位置）
  // ============================================================
  vtkNew<vtkUnsignedCharArray> colors;
  colors->SetName("Colors");
  colors->SetNumberOfComponents(3); // RGB
  colors->SetNumberOfTuples(numParticles);

  for (vtkIdType i = 0; i < numParticles; ++i)
  {
    double pt[3];
    points->GetPoint(i, pt);
    // 将坐标映射到颜色分量（[-5,5] → [0,255]）
    unsigned char r = static_cast<unsigned char>((pt[0] + 5.0) / 10.0 * 255);
    unsigned char g = static_cast<unsigned char>((pt[1] + 5.0) / 10.0 * 255);
    unsigned char b = static_cast<unsigned char>((pt[2] + 5.0) / 10.0 * 255);
    colors->InsertNextTuple3(r, g, b);
  }

  polyData->GetPointData()->SetScalars(colors);

  // ============================================================
  // 第五步：渲染管道
  // ============================================================
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputData(polyData);
  mapper->ScalarVisibilityOn();            // 开启标量颜色映射
  mapper->SetScalarModeToUsePointData();   // 使用点数据中的标量

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetPointSize(5);   // 设置点的像素大小

  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->SetBackground(0.1, 0.1, 0.15);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(800, 600);
  renderWindow->SetWindowName("Point Cloud Example");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  renderWindow->Render();
  std::cout << "Generated " << numParticles
            << " random particles. Close window to exit." << std::endl;
  interactor->Start();

  return 0;
}
```

**代码要点：**

1. **`SetNumberOfPoints()`预分配**：先调用`SetNumberOfPoints(100)`，然后用`SetPoint(i, x, y, z)`逐个设置。这比循环调用`InsertNextPoint()`的效率更高（因为避免了多次内存重新分配）。

2. **两步法创建Verts**：`InsertNextCell(1)`告诉CellArray"接下来这个单元有1个点"，然后`InsertCellPoint(i)`添加该点的索引。这种模式也适用于Lines和Polys。

3. **`vtkUnsignedCharArray`用于颜色**：VTK通常使用0-255的unsigned char存储RGB颜色。`SetNumberOfComponents(3)`表示每个元组有3个分量（R, G, B）。`InsertNextTuple3(r, g, b)`一次性追加一个RGB元组。

4. **`SetPointSize(5)`**：通过Actor的Property设置顶点在屏幕上的像素大小。注意这个大小是屏幕空间的大小，不随相机距离变化（除非使用`vtkPointGaussianMapper`等更高级的映射器）。

---

## 7.3 线（Lines）与折线（Polylines）

### 7.3.1 单线段（VTK_LINE）

一条线段由两个端点定义，是最简单的1D图元。

```cpp
// 创建一个包含两条线段的PolyData
vtkNew<vtkPoints> points;
points->InsertNextPoint(0.0, 0.0, 0.0);   // 索引 0
points->InsertNextPoint(1.0, 1.0, 0.0);   // 索引 1
points->InsertNextPoint(2.0, 0.0, 0.0);   // 索引 2
points->InsertNextPoint(3.0, 1.0, 0.0);   // 索引 3

vtkNew<vtkCellArray> lines;

// 线段0: 从(0,0,0)到(1,1,0)
lines->InsertNextCell({0, 1});

// 线段1: 从(2,0,0)到(3,1,0)
lines->InsertNextCell({2, 3});

vtkNew<vtkPolyData> polyData;
polyData->SetPoints(points);
polyData->SetLines(lines);
```

### 7.3.2 折线（VTK_POLY_LINE）

折线（Polyline）是由三个或更多点定义的连续线段序列。折线中的每对相邻点之间形成一条线段。

```cpp
// 一条折线穿过多个点：{0, 1, 2, 3, 4, 5}
vtkIdType polylinePts[] = {0, 1, 2, 3, 4, 5};
lines->InsertNextCell(6, polylinePts); // 6个点，形成5段连续线段
```

折线的渲染顺序是：p0→p1, p1→p2, p2→p3, p3→p4, p4→p5。

**折线与多条独立线段的区别：**

```
独立线段 (6条):
  线段0: p0-p1
  线段1: p2-p3
  线段2: p4-p5
  ...

一条折线:
  折线0: p0-p1-p2-p3-p4-p5（5段连接线）

关键差异：
- 折线作为一个整体单元存在，在单元数据（Cell Data）中只有一条记录
- 多条独立线段各自是独立的单元，每条线段在Cell Data中有各自的记录
- 在渲染上两者看起来相同（假设折线没有闭合），但数据语义不同
```

### 7.3.3 线段宽度控制

通过`vtkProperty`可以设置线段在屏幕上的像素宽度：

```cpp
actor->GetProperty()->SetLineWidth(3.0);          // 设置为3像素宽
actor->GetProperty()->SetLineStipplePattern(0x00FF); // 设置虚线模式（可选）
actor->GetProperty()->SetLineStippleRepeatFactor(1);
```

注意：`SetLineWidth()`设置的像素宽度是屏幕空间的大小。无论相机拉近还是拉远，线条在屏幕上的像素宽度保持不变。如果需要世界空间中的线条厚度，应使用`vtkTubeFilter`（将线段转换为管道几何体）。

### 7.3.4 示例：绘制3D路径（螺旋线）

以下程序创建一个三维螺旋路径，展示折线的用法以及线宽的设置。

```cpp
// spiral_path_example.cxx
// 生成3D螺旋线，演示折线（Polyline）和线宽控制

#include "vtkActor.h"
#include "vtkCellArray.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkMath.h"
#include "vtkNamedColors.h"
#include "vtkNew.h"
#include "vtkPoints.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"

#include <cmath>

int main()
{
  const double pi = vtkMath::Pi();
  const int numTurns = 5;               // 螺旋圈数
  const int pointsPerTurn = 100;        // 每圈点数
  const int totalPoints = numTurns * pointsPerTurn + 1;
  const double radius = 2.0;            // 螺旋半径
  const double height = 10.0;           // 总高度

  // ============================================================
  // 第一步：创建螺旋线的点
  // ============================================================
  vtkNew<vtkPoints> points;
  points->SetNumberOfPoints(totalPoints);

  for (int i = 0; i < totalPoints; ++i)
  {
    double t = static_cast<double>(i) / pointsPerTurn;  // 角度（弧度）
    double angle = t * 2.0 * pi;
    double x = radius * std::cos(angle);
    double y = radius * std::sin(angle);
    double z = t * height / numTurns;
    points->SetPoint(i, x, y, z);
  }

  // ============================================================
  // 第二步：创建一条折线连接所有点
  // ============================================================
  vtkNew<vtkCellArray> lines;

  // 构建索引数组 {0, 1, 2, ..., totalPoints-1}
  vtkNew<vtkIdList> polylinePts;
  polylinePts->SetNumberOfIds(totalPoints);
  for (int i = 0; i < totalPoints; ++i)
  {
    polylinePts->SetId(i, i);
  }
  lines->InsertNextCell(polylinePts);

  // ============================================================
  // 第三步：组装 PolyData
  // ============================================================
  vtkNew<vtkPolyData> polyData;
  polyData->SetPoints(points);
  polyData->SetLines(lines);

  // ============================================================
  // 第四步：渲染管道
  // ============================================================
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputData(polyData);

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(1.0, 0.6, 0.0);  // 橙色
  actor->GetProperty()->SetLineWidth(3.0);          // 线宽3像素

  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->SetBackground(0.1, 0.1, 0.15);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(800, 600);
  renderWindow->SetWindowName("3D Spiral Path (Polyline)");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  renderWindow->Render();
  std::cout << "Created a 3D spiral with " << totalPoints
            << " points over " << numTurns << " turns."
            << "\nClose window to exit." << std::endl;
  interactor->Start();

  return 0;
}
```

**关键知识点：**

- **`vtkMath::Pi()`**：VTK提供了常用的数学常量，`vtkMath::Pi()`返回双精度的圆周率值。
- **`vtkIdList`**：这是一个可变长度的整数列表，当你的索引数组长度在运行时确定时非常方便。
- **线宽不变特性**：无论你如何缩放/旋转场景，线条在屏幕上始终是3个像素宽。这不是bug，而是OpenGL线宽渲染的标准行为。

---

## 7.4 多边形（Polygons）与三角带（Triangle Strips）

### 7.4.1 三角形（VTK_TRIANGLE）

三角形是3D图形学中最基本的渲染单元——所有图形管线都以三角形作为渲染基础。

```cpp
// 创建两个三角形：{0,1,2} 和 {0,2,3}
vtkNew<vtkCellArray> triangles;

// 方法A: 两步法
triangles->InsertNextCell(3);
triangles->InsertCellPoint(0);
triangles->InsertCellPoint(1);
triangles->InsertCellPoint(2);

// 方法B: 一次性传递索引数组（推荐）
vtkIdType triPts[] = {0, 2, 3};
triangles->InsertNextCell(3, triPts);

// 方法C: initializer_list（最简洁）
triangles->InsertNextCell({0, 1, 2});

// 组装
vtkNew<vtkPolyData> polyData;
polyData->SetPoints(points);
polyData->SetPolys(triangles);
```

### 7.4.2 四边形（VTK_QUAD）

四边形由四个点定义。VTK要求四边形的四个点共面，否则渲染可能出现伪影（Artifact）。

```cpp
// 四边形: {0, 1, 2, 3}
vtkNew<vtkCellArray> quads;
quads->InsertNextCell({0, 1, 2, 3});
polyData->SetPolys(quads);
```

### 7.4.3 通用多边形（VTK_POLYGON）

`VTK_POLYGON`支持任意数量（>=3）的点定义一个凸多边形。多边形中的所有点必须共面。

```cpp
// 五边形: {0, 1, 2, 3, 4}
vtkIdType pentagonPts[] = {0, 1, 2, 3, 4};
polygons->InsertNextCell(5, pentagonPts);
```

### 7.4.4 三角带（VTK_TRIANGLE_STRIP）

三角带（Triangle Strip）是一种高效存储连续三角形网格的方式。在三角带中，每个新增的点与前两个点形成一个三角形。

**三角带的构建规则：**
- 前三个点`{p0, p1, p2}`定义第一个三角形
- 此后每个新点`p_i`与`p_{i-2}`和`p_{i-1}`形成一个新的三角形
- 为保证一致的面朝向，VTK自动交换每对三角形的顶点顺序

```
  Strips: {0, 1, 2, 3, 4, 5}

  产生的三角形:
    三角形0: {0, 1, 2}  ← (0→1→2)
    三角形1: {2, 1, 3}  ← (1→2→3)  【注意顺序交换】
    三角形2: {2, 3, 4}  ← (2→3→4)
    三角形3: {4, 3, 5}  ← (3→4→5)  【注意顺序交换】

  空间布局示意:
    p0 ---- p2 ---- p4
    |  \  /  \  /  |
    |   \/    \/   |
    |   /\    /\   |
    |  /  \  /  \  |
    p1 ---- p3 ---- p5
```

三角带在存储规则网格地形时特别高效：一个`m x n`的网格只需要`m*n`个点和一个包含`2*m*n`个索引的Strip单元（而不是`2*m*n*3`个索引的独立三角形单元）。这可以节省约2/3的索引存储空间。

**创建三角带的代码：**

```cpp
vtkNew<vtkCellArray> strips;

// 方法A: 两步法
strips->InsertNextCell(6);  // 6个点定义4个三角形
strips->InsertCellPoint(0);
strips->InsertCellPoint(1);
strips->InsertCellPoint(2);
strips->InsertCellPoint(3);
strips->InsertCellPoint(4);
strips->InsertCellPoint(5);

// 方法B: initializer_list
strips->InsertNextCell({0, 1, 2, 3, 4, 5});

polyData->SetStrips(strips);
```

### 7.4.5 顶点顺序与面朝向（右手定则）

**这是VTK渲染中的一个关键概念：顶点的排列顺序决定了面的朝向。**

VTK的`vtkPolyDataMapper`默认渲染正面（Front Face），正面由**右手定则**定义：将右手四指沿顶点排列方向弯曲，拇指指向的方向即为面的法向（向外）。

```
  正面（Front Face）—— 逆时针（Counter-Clockwise, CCW）:
       p0
       /\
      /  \
     /    \
  p1 *----* p2
  
  顶点顺序: {p0, p1, p2}
  法向指向: 屏幕外（朝向观察者）


  反面（Back Face）—— 顺时针（Clockwise, CW）:
       p0
       /\
      /  \
     /    \
  p2 *----* p1
  
  顶点顺序: {p0, p2, p1}
  法向指向: 屏幕内（远离观察者）
```

如果你发现渲染出的面是黑色的（或根本看不见），很可能是顶点的排列顺序反了。有两种解决方法：

1. **翻转顶点顺序**：将`{0, 1, 2}`改为`{0, 2, 1}`。
2. **禁用背面剔除**：`actor->GetProperty()->BackfaceCullingOff()`——但这会使背面也渲染，并可能在光照计算中产生不自然的效果，通常不推荐。

### 7.4.6 示例：手动构建立方体

以下程序手动构建一个单位立方体，使用四边形（VTK_QUAD）定义6个面。

```cpp
// manual_cube_example.cxx
// 手动构建一个立方体——演示四边形面元的顶点顺序和右手定则

#include "vtkActor.h"
#include "vtkCellArray.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"
#include "vtkNew.h"
#include "vtkPoints.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"

int main()
{
  // ============================================================
  // 第一步：定义立方体的8个顶点（单位立方体，中心在原点）
  // ============================================================
  //   p7*---------*p6    Y
  //    /|        /|      |
  // p4*---------*p5|      |
  //   | p3*-----|---*p2   +----X
  //   |/        |/       /
  // p0*---------*p1     Z
  //
  // 坐标: p0(-0.5,-0.5,-0.5), p1(0.5,-0.5,-0.5), p2(0.5,-0.5,0.5), p3(-0.5,-0.5,0.5)
  //       p4(-0.5, 0.5,-0.5), p5(0.5, 0.5,-0.5), p6(0.5, 0.5, 0.5), p7(-0.5, 0.5, 0.5)

  vtkNew<vtkPoints> cubePoints;
  cubePoints->InsertNextPoint(-0.5, -0.5, -0.5); // p0
  cubePoints->InsertNextPoint( 0.5, -0.5, -0.5); // p1
  cubePoints->InsertNextPoint( 0.5, -0.5,  0.5); // p2
  cubePoints->InsertNextPoint(-0.5, -0.5,  0.5); // p3
  cubePoints->InsertNextPoint(-0.5,  0.5, -0.5); // p4
  cubePoints->InsertNextPoint( 0.5,  0.5, -0.5); // p5
  cubePoints->InsertNextPoint( 0.5,  0.5,  0.5); // p6
  cubePoints->InsertNextPoint(-0.5,  0.5,  0.5); // p7

  // ============================================================
  // 第二步：定义6个四边形面（注意顶点顺序遵循右手定则）
  // ============================================================
  vtkNew<vtkCellArray> cubeFaces;

  // 底面 (Y = -0.5): 法向指向 -Y
  // 从外部看是逆时针: {0, 1, 2, 3}
  cubeFaces->InsertNextCell({0, 1, 2, 3});

  // 顶面 (Y = 0.5): 法向指向 +Y
  // 从外部看是逆时针: {4, 7, 6, 5}
  cubeFaces->InsertNextCell({4, 7, 6, 5});

  // 前面 (Z = -0.5): 法向指向 -Z
  cubeFaces->InsertNextCell({0, 4, 5, 1});

  // 后面 (Z = 0.5): 法向指向 +Z
  cubeFaces->InsertNextCell({3, 2, 6, 7});

  // 左面 (X = -0.5): 法向指向 -X
  cubeFaces->InsertNextCell({0, 3, 7, 4});

  // 右面 (X = 0.5): 法向指向 +X
  cubeFaces->InsertNextCell({1, 5, 6, 2});

  // ============================================================
  // 第三步：组装 PolyData
  // ============================================================
  vtkNew<vtkPolyData> cube;
  cube->SetPoints(cubePoints);
  cube->SetPolys(cubeFaces);

  std::cout << "Cube: " << cubePoints->GetNumberOfPoints() << " points, "
            << cubeFaces->GetNumberOfCells() << " faces." << std::endl;

  // ============================================================
  // 第四步：渲染管道
  // ============================================================
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputData(cube);

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(0.3, 0.7, 0.9);    // 浅蓝色
  actor->GetProperty()->SetEdgeVisibility(1);          // 显示边线
  actor->GetProperty()->SetEdgeColor(0.0, 0.0, 0.0);  // 黑色边线
  actor->GetProperty()->SetLineWidth(1.5);

  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->SetBackground(0.15, 0.15, 0.18);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(600, 500);
  renderWindow->SetWindowName("Manually Built Cube (Quads)");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  renderWindow->Render();
  std::cout << "Cube rendered. Rotate to see all 6 faces. Close window to exit."
            << std::endl;
  interactor->Start();

  return 0;
}
```

**顶点顺序验证方法：** 当程序运行后，旋转立方体观察每个面。如果某个面呈现为黑色（或看不见），说明该面的顶点顺序是反的（反面朝向观察者）。此时该面的顶点顺序需要反转——将`{a, b, c, d}`改为`{a, d, c, b}`。

---

## 7.5 法向量（Normals）

### 7.5.1 为什么法向量重要

法向量（Normal Vector）是垂直于曲面的单位向量。它在基于光照的渲染中扮演核心角色：

- **漫反射（Diffuse）**：表面亮度 = 光源方向 · 法向（点积）。法向朝向光源时最亮，背离光源时最暗。
- **镜面反射（Specular）**：高光的位置取决于法向和视线方向的半角向量。
- **背面剔除（Backface Culling）**：法向用于判断多边形是"正面"还是"背面"。背面的多边形在默认情况下不渲染。

**没有法向量的表面将被渲染为纯色——光照对它不产生任何效果。** 这就是为什么有些时候你的模型看起来"扁平"的原因。

### 7.5.2 光滑着色 vs 平面着色

法向量有两种关联方式，对应两种不同的着色效果：

| 着色模式 | 法向存储位置 | 视觉效果 | 表示 |
|----------|-------------|---------|------|
| **平面着色（Flat Shading）** | Cell Data（单元数据） | 每个面片有均匀的亮度，面片之间有明显的光泽不连续 | 低多边形风格 |
| **光滑着色（Smooth Shading）** | Point Data（点数据） | 表面亮度平滑过渡，视觉上模拟曲面 | 真实感渲染 |

```
平面着色（Flat Shading）示意图:
  每个三角形有自己唯一的法向，同一三角形内亮度一致
       ↗n0    ↗n1
      / \    / \
     /   \  /   \     面片之间有明显的亮度分界
    /     \/     \
   *------*------*

光滑着色（Smooth Shading）示意图:
  每个顶点有自己的法向（通常是相邻面的法向平均值）
       ↗n0   ↗n1
      / \   / \
     /   \ /   \      三角形内部亮度通过插值平滑过渡
    /     \/     \
   *------*------*
```

### 7.5.3 手动设置法向量

```cpp
#include "vtkDoubleArray.h"
#include "vtkPointData.h"

// 为每个点创建法向量
vtkNew<vtkDoubleArray> normals;
normals->SetName("Normals");
normals->SetNumberOfComponents(3);
normals->SetNumberOfTuples(numPoints);

// 设置每个点的法向（以球体为例——法向从球心指向表面）
for (vtkIdType i = 0; i < numPoints; ++i)
{
  double pt[3];
  points->GetPoint(i, pt);
  double len = std::sqrt(pt[0]*pt[0] + pt[1]*pt[1] + pt[2]*pt[2]);
  // 单位化：对于以原点为中心的球来说，法向就是归一化的坐标
  normals->SetTuple3(i, pt[0]/len, pt[1]/len, pt[2]/len);
}

// 将法向量关联到Point Data
polyData->GetPointData()->SetNormals(normals);
```

### 7.5.4 vtkPolyDataNormals——自动计算法向量

在大多数实际场景中，你不需要手动设置法向量。VTK提供了`vtkPolyDataNormals`过滤器来自动计算法向量。

```cpp
#include "vtkPolyDataNormals.h"

vtkNew<vtkPolyDataNormals> normalsFilter;
normalsFilter->SetInputConnection(someSource->GetOutputPort());
// 或者
normalsFilter->SetInputData(somePolyData);

// 【关键参数】SetFeatureAngle —— 控制平滑/平面着色的分界线
normalsFilter->SetFeatureAngle(30.0);   // 特征角（度），默认30°

normalsFilter->Update();
vtkPolyData* smoothOutput = normalsFilter->GetOutput();
```

**`SetFeatureAngle()`的工作原理：**

- 该角度定义了"尖锐边缘"（Sharp Edge）的判定阈值。
- 对于网格中的每条边，如果共享该边的两个面的法向夹角**大于**`FeatureAngle`，则该边被视为"特征边"（Feature Edge）。
- 在特征边两侧，顶点法向量**不共享**——每个面上的顶点有自己的法向拷贝，实现平面着色效果。
- 在非特征边（夹角小于阈值）上，顶点法向被平滑（平均），实现光滑着色效果。

```
  特征角 = 30° 示例:

        ╲
    面A  ╲ 边
          ╲_______
           \
    面B     \                   
              \
  
  面A法向与面B法向夹角 = 35° > 30°
  → 这条边是特征边
  → 边上顶点的法向量分离（产生棱角效果）

  如果夹角 = 20° < 30°
  → 这条边不是特征边
  → 边上顶点的法向量被平滑平均
```

**常用FeatureAngle参考值：**

| 值 | 效果 | 适用场景 |
|----|------|---------|
| 0° | 完全平面着色（所有边都是特征边） | 展示网格结构、低多边形风格 |
| 30° | 默认值，尖锐棱角可见，平缓曲面平滑 | 一般用途 |
| 60° | 只有较尖锐的棱角保留 | 有机形状、地形 |
| 180° | 完全光滑着色（所有法向都被平均） | 球体、流线型曲面 |

### 7.5.5 示例：法向量对渲染效果的对比

以下程序同时展示有法向量和无法向量的渲染效果，让你直观理解法向量在光照计算中的作用。

```cpp
// normals_comparison.cxx
// 对比有法向量 vs 无法向量的渲染效果

#include "vtkActor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNamedColors.h"
#include "vtkNew.h"
#include "vtkPointData.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkPolyDataNormals.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkSphereSource.h"
#include "vtkTextActor.h"
#include "vtkTextProperty.h"

int main()
{
  // ============================================================
  // 第一步：创建球体数据源
  // ============================================================
  vtkNew<vtkSphereSource> sphere;
  sphere->SetThetaResolution(16);
  sphere->SetPhiResolution(16);
  sphere->Update();

  vtkPolyData* sphereData = sphere->GetOutput();

  // ============================================================
  // 第二步：创建"无法向"版本（移除PointData中的Normals）
  // ============================================================
  vtkNew<vtkPolyData> noNormals;
  noNormals->DeepCopy(sphereData);  // 深拷贝整个数据集
  // 移除法向量数据（VTK的Cone/Sphere源默认会生成Normals）
  noNormals->GetPointData()->RemoveArray("Normals");

  // ============================================================
  // 第三步：创建"光滑着色"版本
  // ============================================================
  vtkNew<vtkPolyDataNormals> smoothNormals;
  smoothNormals->SetInputData(sphereData);
  smoothNormals->SetFeatureAngle(180.0);  // 全部平滑
  smoothNormals->Update();

  // ============================================================
  // 第四步：创建"平面着色"版本
  // ============================================================
  vtkNew<vtkPolyDataNormals> flatNormals;
  flatNormals->SetInputData(sphereData);
  flatNormals->SetFeatureAngle(0.0);     // 全部平面
  flatNormals->Update();

  // ============================================================
  // 第五步：渲染——三个视口的对比布局
  // ============================================================
  // 左视口 (0.0 - 0.33): 无法向量
  vtkNew<vtkPolyDataMapper> mapperNoNL;
  mapperNoNL->SetInputData(noNormals);

  vtkNew<vtkActor> actorNoNL;
  actorNoNL->SetMapper(mapperNoNL);
  actorNoNL->GetProperty()->SetColor(0.2, 0.6, 1.0); // 蓝色

  vtkNew<vtkRenderer> renNoNL;
  renNoNL->SetViewport(0.0, 0.0, 0.33, 1.0);
  renNoNL->AddActor(actorNoNL);
  renNoNL->SetBackground(0.1, 0.1, 0.15);

  // 中视口 (0.33 - 0.66): 平面着色
  vtkNew<vtkPolyDataMapper> mapperFlat;
  mapperFlat->SetInputConnection(flatNormals->GetOutputPort());

  vtkNew<vtkActor> actorFlat;
  actorFlat->SetMapper(mapperFlat);
  actorFlat->GetProperty()->SetColor(0.2, 0.6, 1.0);

  vtkNew<vtkRenderer> renFlat;
  renFlat->SetViewport(0.33, 0.0, 0.66, 1.0);
  renFlat->AddActor(actorFlat);
  renFlat->SetBackground(0.1, 0.1, 0.15);

  // 右视口 (0.66 - 1.0): 光滑着色
  vtkNew<vtkPolyDataMapper> mapperSmooth;
  mapperSmooth->SetInputConnection(smoothNormals->GetOutputPort());

  vtkNew<vtkActor> actorSmooth;
  actorSmooth->SetMapper(mapperSmooth);
  actorSmooth->GetProperty()->SetColor(0.2, 0.6, 1.0);

  vtkNew<vtkRenderer> renSmooth;
  renSmooth->SetViewport(0.66, 0.0, 1.0, 1.0);
  renSmooth->AddActor(actorSmooth);
  renSmooth->SetBackground(0.1, 0.1, 0.15);

  // 共享相机（三个渲染器使用相同视角）
  renSmooth->SetActiveCamera(renNoNL->GetActiveCamera());

  // ============================================================
  // 第六步：添加标签
  // ============================================================
  auto createLabel = [](const char* text, double x, double y) {
    vtkNew<vtkTextActor> label;
    label->SetInput(text);
    label->GetTextProperty()->SetColor(1.0, 1.0, 1.0);
    label->GetTextProperty()->SetFontSize(14);
    label->GetTextProperty()->BoldOn();
    label->SetPosition(x, y);
    return label;
  };

  renNoNL->AddActor2D(createLabel("No Normals (Flat Color)", 10, 10));
  renFlat->AddActor2D(createLabel("Flat Shading (Angle=0)", 10, 10));
  renSmooth->AddActor2D(createLabel("Smooth Shading (Angle=180)", 10, 10));

  // ============================================================
  // 第七步：渲染窗口
  // ============================================================
  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renNoNL);
  renderWindow->AddRenderer(renFlat);
  renderWindow->AddRenderer(renSmooth);
  renderWindow->SetSize(1200, 400);
  renderWindow->SetWindowName("Normals Comparison: None vs Flat vs Smooth");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  renderWindow->Render();
  std::cout << "Observe the three spheres:\n"
            << "  Left:   No normals — uniform flat color, no lighting effect\n"
            << "  Middle: Flat shading — each triangle has uniform brightness\n"
            << "  Right:  Smooth shading — brightness interpolates across surface\n"
            << "Close window to exit." << std::endl;
  interactor->Start();

  return 0;
}
```

**预期效果：**
- **左球（无法向量）**：整个球体呈现均匀的蓝色，没有明暗变化——光照计算被跳过。
- **中球（平面着色）**：可以清晰地看到每个三角形面片，面片之间有棱角分明的亮度变化。由于仅16x16的分辨率，多边形的边缘非常明显。
- **右球（光滑着色）**：球体表面平滑过渡，看起来像一个真正的球。即使在16x16的低分辨率下，光照过渡也很自然。

---

## 7.6 纹理坐标（Texture Coordinates）

### 7.6.1 纹理坐标的基本概念

纹理坐标（Texture Coordinates，简称TCoords，也常被称为UV坐标）定义了2D纹理图像如何映射到3D几何表面上。纹理坐标通常用(u, v)表示，范围通常在[0, 1]之间：

- **(0, 0)** 对应纹理图像的左下角
- **(1, 1)** 对应纹理图像的右上角

```
  纹理图像 (256x256)                 3D几何体
  +-----------------------+      p3(u=0,v=1) *----* p2(u=1,v=1)
  |                       |                  |\   |
  |   [图像内容]           |                  | \  |
  |                       |                  |  \ |
  |                       |                  |   \|
  +-----------------------+      p0(u=0,v=0) *----* p1(u=1,v=0)
  (0,0)             (1,1)
```

纹理坐标存储在`vtkPointData`中，以名为`"TCoords"`的数组存在，每个点关联一个(u, v)坐标。

### 7.6.2 设置纹理坐标

```cpp
#include "vtkDoubleArray.h"
#include "vtkPointData.h"

vtkNew<vtkDoubleArray> tcoords;
tcoords->SetName("TCoords");
tcoords->SetNumberOfComponents(2);   // 2D纹理坐标 = (u, v)
tcoords->SetNumberOfTuples(numPoints);

// 为一个平面设置纹理坐标
tcoords->SetTuple2(0, 0.0, 0.0);  // 左下角
tcoords->SetTuple2(1, 1.0, 0.0);  // 右下角
tcoords->SetTuple2(2, 1.0, 1.0);  // 右上角
tcoords->SetTuple2(3, 0.0, 1.0);  // 左上角

polyData->GetPointData()->SetTCoords(tcoords);
```

注意：纹理坐标的分量数通常是2（2D纹理），但也可以是1（1D纹理）或3（3D体积纹理）。大多数场景使用2D纹理。

### 7.6.3 示例：为平面应用棋盘格纹理

以下程序创建一个平面并应用程序化生成的棋盘格纹理。

```cpp
// checkerboard_texture_example.cxx
// 演示如何将纹理坐标应用于平面，并使用棋盘格纹理

#include "vtkActor.h"
#include "vtkCellArray.h"
#include "vtkImageData.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPointData.h"
#include "vtkPoints.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkTexture.h"
#include "vtkUnsignedCharArray.h"

int main()
{
  // ============================================================
  // 第一步：创建平面几何体
  // ============================================================
  vtkNew<vtkPoints> planePoints;
  planePoints->InsertNextPoint(-3.0, -2.0, 0.0);  // p0: 左下
  planePoints->InsertNextPoint( 3.0, -2.0, 0.0);  // p1: 右下
  planePoints->InsertNextPoint( 3.0,  2.0, 0.0);  // p2: 右上
  planePoints->InsertNextPoint(-3.0,  2.0, 0.0);  // p3: 左上

  vtkNew<vtkCellArray> planeCell;
  planeCell->InsertNextCell({0, 1, 2, 3});  // 一个四边形

  vtkNew<vtkPolyData> plane;
  plane->SetPoints(planePoints);
  plane->SetPolys(planeCell);

  // ============================================================
  // 第二步：设置纹理坐标（将整个纹理贴在平面上）
  // ============================================================
  vtkNew<vtkDoubleArray> tcoords;
  tcoords->SetName("TCoords");
  tcoords->SetNumberOfComponents(2);
  tcoords->SetNumberOfTuples(4);
  tcoords->SetTuple2(0, 0.0, 0.0);  // p0 → 左下角
  tcoords->SetTuple2(1, 1.0, 0.0);  // p1 → 右下角
  tcoords->SetTuple2(2, 1.0, 1.0);  // p2 → 右上角
  tcoords->SetTuple2(3, 0.0, 1.0);  // p3 → 左上角

  plane->GetPointData()->SetTCoords(tcoords);

  // ============================================================
  // 第三步：程序化生成棋盘格纹理图像
  // ============================================================
  const int texSize = 256;
  const int checkerSize = 32;  // 每个格子的像素大小

  vtkNew<vtkImageData> checkerboard;
  checkerboard->SetDimensions(texSize, texSize, 1);
  checkerboard->AllocateScalars(VTK_UNSIGNED_CHAR, 3); // RGB

  unsigned char* pixels =
    static_cast<unsigned char*>(checkerboard->GetScalarPointer());

  for (int y = 0; y < texSize; ++y)
  {
    for (int x = 0; x < texSize; ++x)
    {
      int idx = (y * texSize + x) * 3;
      bool isWhite = ((x / checkerSize) + (y / checkerSize)) % 2 == 0;
      unsigned char value = isWhite ? 255 : 50;
      pixels[idx + 0] = value;   // R
      pixels[idx + 1] = value;   // G
      pixels[idx + 2] = value;   // B
    }
  }

  // ============================================================
  // 第四步：创建纹理对象
  // ============================================================
  vtkNew<vtkTexture> texture;
  texture->SetInputData(checkerboard);
  texture->InterpolateOn();    // 线性插值（让纹理在放大/缩小时平滑）
  texture->RepeatOn();         // 当纹理坐标超出[0,1]范围时重复纹理

  // ============================================================
  // 第五步：渲染管道
  // ============================================================
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputData(plane);

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->SetTexture(texture);                       // 绑定纹理
  actor->GetProperty()->SetColor(1.0, 1.0, 1.0);   // 白色基底（以便纹理颜色正确呈现）

  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->SetBackground(0.15, 0.15, 0.18);
  renderer->GetActiveCamera()->SetPosition(0, -6, 4);
  renderer->GetActiveCamera()->SetFocalPoint(0, 0, 0);
  renderer->GetActiveCamera()->SetViewUp(0, 0, 1);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(800, 600);
  renderWindow->SetWindowName("Checkerboard Texture on a Plane");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  renderWindow->Render();
  std::cout << "Checkerboard texture applied to a plane." << std::endl;
  std::cout << "Rotate to see the textured surface. Close window to exit."
            << std::endl;
  interactor->Start();

  return 0;
}
```

**纹理关键API说明：**

- **`texture->InterpolateOn()`**：当纹理在屏幕上被放大或缩小时，使用线性插值在相邻纹素之间过渡。关闭后使用最近邻采样（会产生明显的像素化效果）。
- **`texture->RepeatOn()`**：当纹理坐标的值超出[0, 1]范围时，纹理图像会重复平铺，而不是被截断。
- **`actor->SetTexture(texture)`**：将纹理绑定到Actor上。注意这不是Mapper的职责——纹理绑定在Actor级别，与Property一起管理外观属性。

---

## 7.7 代码示例：构建复杂PolyData（3D地形曲面）

本节将综合运用本章所学全部知识，从零开始构建一个3D地形曲面。我们将：

1. 创建一个规则网格的顶点（使用`vtkPoints`）
2. 根据数学函数计算每个顶点的高程（Z坐标）
3. 使用三角带（Triangle Strips）高效构建拓扑
4. 为每个顶点设置高程标量值（用于颜色映射）
5. 使用`vtkPolyDataNormals`自动计算法向量
6. 使用颜色映射表（LookupTable）根据高程着色

### 7.7.1 完整源代码

```cpp
// terrain_surface_example.cxx
// 构建一个3D地形曲面，综合演示 PolyData 的完整用法：
// - Points: 网格顶点
// - Strips: 三角形带拓扑
// - Point Data Scalars: 高程值
// - Point Data Normals: 自动计算的表面法向
// - LookupTable: 按高程着色

#include "vtkActor.h"
#include "vtkCamera.h"
#include "vtkCellArray.h"
#include "vtkDoubleArray.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkLookupTable.h"
#include "vtkMath.h"
#include "vtkNew.h"
#include "vtkPointData.h"
#include "vtkPoints.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkPolyDataNormals.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkScalarBarActor.h"
#include "vtkTextProperty.h"

#include <cmath>
#include <iostream>

// 地形高程函数：Gaussian型山丘叠加
// 返回在 (x, y) 处的地表高度
double TerrainHeight(double x, double y)
{
  // 主山峰
  double peak1 = 4.0 * std::exp(-((x - 1.0) * (x - 1.0) + (y - 1.0) * (y - 1.0)) / 4.0);

  // 次山峰
  double peak2 = 2.5 * std::exp(-((x + 2.0) * (x + 2.0) + (y + 1.5) * (y + 1.5)) / 3.0);

  // 第三山峰
  double peak3 = 3.0 * std::exp(-((x + 1.0) * (x + 1.0) + (y - 2.5) * (y - 2.5)) / 3.5);

  // 微小起伏（噪声模拟）
  double ripple = 0.3 * std::sin(x * 3.0) * std::cos(y * 3.0);

  return peak1 + peak2 + peak3 + ripple;
}

int main()
{
  // ==============================================================
  // 第一步：定义网格参数
  // ==============================================================
  const int gridRows = 150;        // 行数（Y方向）
  const int gridCols = 150;        // 列数（X方向）
  const double xMin = -5.0, xMax = 5.0;
  const double yMin = -5.0, yMax = 5.0;
  const double dx = (xMax - xMin) / (gridCols - 1);
  const double dy = (yMax - yMin) / (gridRows - 1);

  const vtkIdType totalPoints = gridRows * gridCols;
  std::cout << "Generating terrain: " << gridCols << " x " << gridRows
            << " = " << totalPoints << " points" << std::endl;

  // ==============================================================
  // 第二步：创建顶点（几何层）
  // ==============================================================
  vtkNew<vtkPoints> points;
  points->SetNumberOfPoints(totalPoints);

  vtkNew<vtkDoubleArray> elevationScalars;
  elevationScalars->SetName("Elevation");
  elevationScalars->SetNumberOfComponents(1);
  elevationScalars->SetNumberOfTuples(totalPoints);

  for (int j = 0; j < gridRows; ++j)
  {
    double y = yMin + j * dy;
    for (int i = 0; i < gridCols; ++i)
    {
      double x = xMin + i * dx;
      double z = TerrainHeight(x, y);

      vtkIdType ptId = j * gridCols + i;
      points->SetPoint(ptId, x, y, z);
      elevationScalars->SetValue(ptId, z);
    }
  }

  std::cout << "  Min elevation: " << elevationScalars->GetRange()[0]
            << ", Max elevation: " << elevationScalars->GetRange()[1] << std::endl;

  // ==============================================================
  // 第三步：创建三角形带（拓扑层）
  // ==============================================================
  // 每一行网格是一条Triangle Strip
  // 每条Strip包含 gridCols*2 个顶点索引
  vtkNew<vtkCellArray> strips;

  for (int j = 0; j < gridRows - 1; ++j)
  {
    strips->InsertNextCell(gridCols * 2);
    for (int i = 0; i < gridCols; ++i)
    {
      vtkIdType lowerPtId = j * gridCols + i;          // 当前行
      vtkIdType upperPtId = (j + 1) * gridCols + i;   // 下一行
      strips->InsertCellPoint(lowerPtId);
      strips->InsertCellPoint(upperPtId);
    }
  }

  // ==============================================================
  // 第四步：组装 vtkPolyData
  // ==============================================================
  vtkNew<vtkPolyData> terrain;
  terrain->SetPoints(points);
  terrain->SetStrips(strips);
  terrain->GetPointData()->SetScalars(elevationScalars);

  std::cout << "  Created " << strips->GetNumberOfCells()
            << " triangle strips (" << (gridRows - 1) * (gridCols - 1) * 2
            << " triangles total)" << std::endl;

  // ==============================================================
  // 第五步：计算法向量（光滑着色）
  // ==============================================================
  vtkNew<vtkPolyDataNormals> normalsFilter;
  normalsFilter->SetInputData(terrain);
  normalsFilter->SetFeatureAngle(40.0);   // 40度以下的曲面平滑着色
  normalsFilter->SplittingOff();          // 不分裂法向量（保持拓扑不变）
  normalsFilter->ConsistencyOn();         // 确保法向方向一致（全部朝外）
  normalsFilter->Update();

  vtkPolyData* terrainWithNormals = normalsFilter->GetOutput();
  std::cout << "  Normals computed (FeatureAngle="
            << normalsFilter->GetFeatureAngle() << " degrees)" << std::endl;

  // ==============================================================
  // 第六步：创建颜色映射表
  // ==============================================================
  double elevRange[2];
  elevationScalars->GetRange(elevRange);

  vtkNew<vtkLookupTable> lut;
  lut->SetNumberOfTableValues(256);
  lut->SetRange(elevRange[0], elevRange[1]);
  // 使用蓝-绿-黄-红配色方案（模拟地形图）
  // Hue: 0.667(蓝) -> 0.333(绿) -> 0.167(黄) -> 0.0(红)
  lut->SetHueRange(0.667, 0.0);
  lut->SetSaturationRange(0.8, 0.8);
  lut->SetValueRange(0.6, 1.0);
  lut->Build();

  // ==============================================================
  // 第七步：渲染管道
  // ==============================================================
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputConnection(normalsFilter->GetOutputPort());
  mapper->SetLookupTable(lut);
  mapper->SetScalarRange(elevRange[0], elevRange[1]);
  mapper->ScalarVisibilityOn();

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetSpecular(0.3);       // 高光反射系数
  actor->GetProperty()->SetSpecularPower(20);   // 高光锐度（Phong指数）
  actor->GetProperty()->SetAmbient(0.2);        // 环境光
  actor->GetProperty()->SetDiffuse(0.8);        // 漫反射

  // 标量条（Scalar Bar）
  vtkNew<vtkScalarBarActor> scalarBar;
  scalarBar->SetLookupTable(lut);
  scalarBar->SetTitle("Elevation (m)");
  scalarBar->SetNumberOfLabels(8);
  scalarBar->GetTitleTextProperty()->SetColor(0.0, 0.0, 0.0);
  scalarBar->GetTitleTextProperty()->SetFontSize(12);
  scalarBar->GetLabelTextProperty()->SetColor(0.0, 0.0, 0.0);
  scalarBar->GetLabelTextProperty()->SetFontSize(10);

  // ==============================================================
  // 第八步：渲染器与相机设置
  // ==============================================================
  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->AddActor2D(scalarBar);
  renderer->SetBackground(0.2, 0.25, 0.35);     // 天空蓝背景
  renderer->SetBackground2(0.8, 0.85, 0.9);     // 渐变（接近水平线的浅色）
  renderer->GradientBackgroundOn();              // 启用背景渐变

  // 设置相机位置以获得俯瞰视角
  vtkCamera* camera = renderer->GetActiveCamera();
  camera->SetPosition(0.0, -8.0, 5.0);   // 相机位置
  camera->SetFocalPoint(0.0, 0.0, 2.0);  // 注视中心
  camera->SetViewUp(0.0, 0.0, 1.0);      // 上方为Z轴
  camera->Elevation(45);                  // 抬高视角（俯视角度）
  camera->Azimuth(30);                    // 水平旋转

  // ==============================================================
  // 第九步：渲染窗口与交互
  // ==============================================================
  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(1000, 700);
  renderWindow->SetMultiSamples(8);              // 8x MSAA 抗锯齿
  renderWindow->SetWindowName("3D Terrain Surface");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  // ==============================================================
  // 第十步：序列信息输出并启动
  // ==============================================================
  std::cout << "\n========== Terrain Statistics ==========" << std::endl;
  std::cout << "Total Points:       " << terrain->GetNumberOfPoints() << std::endl;
  std::cout << "Total Cells:        " << terrain->GetNumberOfCells() << std::endl;
  std::cout << "Total Strips:       " << terrain->GetNumberOfStrips() << std::endl;

  double bounds[6];
  terrainWithNormals->GetBounds(bounds);
  std::cout << "Bounds:\n"
            << "  X: [" << bounds[0] << ", " << bounds[1] << "]\n"
            << "  Y: [" << bounds[2] << ", " << bounds[3] << "]\n"
            << "  Z: [" << bounds[4] << ", " << bounds[5] << "]" << std::endl;

  // 检查法向量是否已计算
  vtkDataArray* computedNormals =
    terrainWithNormals->GetPointData()->GetNormals();
  if (computedNormals)
  {
    std::cout << "Normals:           Computed ("
              << computedNormals->GetNumberOfTuples() << " tuples)" << std::endl;
  }
  std::cout << "========================================" << std::endl;
  std::cout << "\nClose window to exit." << std::endl;

  renderWindow->Render();
  interactor->Start();

  return 0;
}
```

### 7.7.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8 FATAL_ERROR)
project(TerrainSurface VERSION 1.0.0 LANGUAGES CXX)

# 设置C++标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 查找VTK 9.5.2
find_package(VTK 9.5.2
  COMPONENTS
    CommonCore
    CommonColor
    CommonDataModel
    CommonExecutionModel
    CommonMath
    FiltersCore
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
  REQUIRED
)

# 打印VTK信息（帮助调试）
message(STATUS "VTK_VERSION: ${VTK_VERSION}")
message(STATUS "VTK_RENDERING_BACKEND: OpenGL2")

# 创建可执行文件
add_executable(TerrainSurface terrain_surface_example.cxx)

# 链接VTK库
target_link_libraries(TerrainSurface
  PRIVATE
    ${VTK_LIBRARIES}
)

# VTK模块工厂自动初始化（必须！）
vtk_module_autoinit(
  TARGETS TerrainSurface
  MODULES ${VTK_LIBRARIES}
)
```

### 7.7.3 代码详解

本节逐部分解释地形曲面示例中的关键技术点。

#### 第一步：网格参数定义

```cpp
const int gridRows = 150;
const int gridCols = 150;
```

选择150x150的网格意味着22,500个顶点和44,102个三角形（每个矩形区域由2个三角形组成）。这个分辨率在大多数现代硬件上可以流畅渲染。如果你的机器性能较弱，可以将网格降低到100x100或50x50。

`dx`和`dy`是网格间距，通过`(最大值 - 最小值) / (点数 - 1)`计算，确保边界点恰好落在指定范围上。

#### 第二步：创建顶点与高程标量

使用了一个关键的索引公式：`ptId = j * gridCols + i`。这定义了点在规则网格中的"行优先"（Row-Major）索引方式：

```
  网格 (gridCols=3, gridRows=2):
  j=0: pt[0]  pt[1]  pt[2]   ← 第0行
  j=1: pt[3]  pt[4]  pt[5]   ← 第1行

  其中 pt[j * gridCols + i] = pt[j * 3 + i]
```

`TerrainHeight(x, y)`函数叠加了三个高斯型山峰和一个小幅度的正弦噪声，创建了自然地形的外观。

`elevationScalars`数组在每个点上存储一个标量值（Z坐标即高程）。这个数组既是属性数据（用于颜色映射），也是数据的有机组成部分。

#### 第三步：使用三角带构建拓扑

这是本节最重要的技术决策。对于一个规则网格，我们使用**三角形带（Triangle Strips）**而不是独立三角形，以实现最高的存储效率。

```
  两条相邻行形成一条Strip:

  第j行:  p[j*C+0]  p[j*C+1]  p[j*C+2]  p[j*C+3]  ...
  第j+1行: p[(j+1)*C+0] p[(j+1)*C+1] p[(j+1)*C+2] p[(j+1)*C+3] ...

  一条Strip的索引序列:
    {p0, p0', p1, p1', p2, p2', p3, p3', ...}

  产生的三角形:
    三角1: {p0,  p0', p1}
    三角2: {p1,  p0', p1'}
    三角3: {p1,  p1', p2}
    三角4: {p2,  p1', p2'}
    ...
```

每条Strip对应网格中相邻的两行。对于gridRows=150的网格，有gridRows-1=149条Strip，每条Strip包含`gridCols * 2 = 300`个索引。

**存储效率对比：**
- **独立三角形**：`(gridRows-1) * (gridCols-1) * 2 * 3 = 132,006` 个索引存储
- **三角带**：`(gridRows-1) * gridCols * 2 + (gridRows-1) = 89,400` 个索引存储（加上每个Strip的单元计数）
- 节省约 **32%** 的索引存储空间

#### 第五步：法向量计算

```cpp
vtkNew<vtkPolyDataNormals> normalsFilter;
normalsFilter->SetInputData(terrain);
normalsFilter->SetFeatureAngle(40.0);
normalsFilter->SplittingOff();
normalsFilter->ConsistencyOn();
```

- **`SetFeatureAngle(40.0)`**：特征角设为40度。这意味着相邻面法向夹角小于40度的区域会获得平滑着色（如平缓的山坡），而大于40度的区域会保留尖锐的棱角效果（如山峰）。
- **`SplittingOff()`**：关闭法向分裂。开启时（默认），Filter会在特征边处创建重复的顶点（每个面片有自己的顶点拷贝），这会改变顶点数量。关闭时，它在不改变拓扑的情况下尽可能平滑着色——保持顶点索引和数量不变。
- **`ConsistencyOn()`**：确保所有法向的一致性（统一朝外）。这很重要——如果某些三角形的顶点顺序不一致，某些面的法向可能指向内部，导致渲染时出现"黑洞"。

#### 第六步：颜色映射表

LookupTable定义了从标量值到颜色的映射。`SetHueRange(0.667, 0.0)`创建了从蓝色（低海拔）经过绿色和黄色到红色（高海拔）的配色方案，这是地理可视化中常用的高程配色。

#### 第九步：相机设置

```cpp
camera->Elevation(45);   // 垂直方向上抬45度
camera->Azimuth(30);     // 水平方向旋转30度
```

这些调用在初始相机位置的基础上叠加旋转，提供了从东南方向上方向俯瞰地形的视角。`GradientBackgroundOn()`启用了天空渐变背景——顶部为深蓝灰色，底部为浅蓝灰色，模拟真实天空的视觉效果。

### 7.7.4 运行结果

运行程序后，你将看到一个1000x700像素的窗口，其中显示一个彩色的3D地形曲面：
- 低洼处（Z值低）显示为蓝色
- 中等高度显示为绿色和黄色
- 山峰处（Z值高）显示为橙红色
- 表面有平滑的光照效果（得益于法向量计算）
- 右侧显示高程标量条
- 鼠标可旋转、缩放、平移地形

### 7.7.5 扩展思路

这个地形示例是一个很好的起点。你可以基于此进行以下扩展：

1. **添加等高线（Contour Lines）**：使用`vtkBandedPolyDataContourFilter`在地形表面叠加等高线。
2. **从真实数据加载高程**：读取DEM（Digital Elevation Model）文件，替换`TerrainHeight()`函数。
3. **动态变形**：在计时器回调中随时间修改点坐标，创建动画地形（如波浪传播）。
4. **叠加纹理**：加载卫星影像作为纹理，叠加在地形表面。
5. **添加水体平面**：在某个Z值处创建一个半透明平面，模拟水面效果。

---

## 7.8 本章小结

本章深入剖析了`vtkPolyData`的完整结构和编程接口。让我们回顾核心要点：

### 要点速览

1. **双层分离设计**：`vtkPolyData`将几何（`vtkPoints`——点坐标）与拓扑（`vtkCellArray`——连接关系）分开存储。这种分离使得网格变形、细化、动画等操作变得简单高效，是VTK数据模型中最优雅的设计决策之一。

2. **四类基本图元**：Verts（0D点元）、Lines（1D线元/折线）、Polys（2D三角形/四边形/多边形）、Strips（2D三角形带），分别存储在四个独立的`vtkCellArray`中。一个PolyData可以同时包含所有四种图元。

3. **`vtkCellArray`的现代存储格式**：使用Offsets+Connectivity双数组结构，支持O(1)随机访问。`InsertNextCell()`和`InsertCellPoint()`是构建单元的主要API，C++11的`initializer_list`重载（`InsertNextCell({0, 1, 2})`）提供了最简洁的语法。

4. **顶点顺序决定面朝向**：遵循右手定则——逆时针顶点顺序定义正面（Front Face），法向指向观察者。如果渲染的面"看不见"或呈黑色，首先检查顶点顺序是否正确。

5. **法向量是光照渲染的关键**：没有法向量的表面将渲染为纯色。`vtkPolyDataNormals`过滤器自动计算法向量，`SetFeatureAngle()`控制光滑着色与平面着色之间的分界线。法向量存储在Point Data（光滑着色）或Cell Data（平面着色）中。

6. **纹理坐标映射2D图像到3D表面**：通过`vtkPointData::SetTCoords()`设置纹理坐标，通过`vtkTexture`创建纹理对象，通过`actor->SetTexture()`绑定。`vtkImageData`可用于程序化生成纹理图像。

7. **三角形带是规则网格的高效存储方案**：对于`m x n`网格，使用三角带可以比独立三角形节省约32%的索引存储，同时渲染性能更高。

### 从本章到全书

本章是第二卷（进阶篇）的开门砖。理解了`vtkPolyData`的结构，你将在以下章节中游刃有余：

- **第八章（过滤器的实际应用）**：所有涉及表面网格的过滤器（`vtkClipPolyData`、`vtkSmoothPolyDataFilter`、`vtkDecimatePro`、`vtkContourFilter`）都操作于`vtkPolyData`之上。理解其内部结构将使你能够预测过滤器的行为并调试输出结果。
- **第九章（数据I/O）**：STL、OBJ、PLY、VTK等文件格式本质上都是`vtkPolyData`的不同序列化方式——点坐标+单元连接+属性数据。
- **第十章（颜色映射与标量可视化）**：本章的标量数据（如地形的高程值）在后续章节中被广泛用于驱动颜色映射、变形（Warp）和标签（Glyph）等高级可视化技术。
- **第十二章（高级渲染）**：理解法向量和纹理坐标是掌握PBR（Physically-Based Rendering）材质、法向贴图、环境遮蔽等高级渲染技术的基础。

### 延伸阅读与资源

- vtkPolyData官方文档：https://docs.vtk.org/en/latest/api/CommonDataModel/vtkPolyData.html
- vtkCellArray官方文档：https://docs.vtk.org/en/latest/api/CommonDataModel/vtkCellArray.html
- vtkPolyDataNormals文档：https://docs.vtk.org/en/latest/api/FiltersCore/vtkPolyDataNormals.html
- VTK Examples - Geometric Objects：https://examples.vtk.org/site/Cxx/GeometricObjects/
- VTK Examples - PolyData：https://examples.vtk.org/site/Cxx/PolyData/

### 进入下一章之前

在进入第八章（过滤器的实际应用）之前，建议你：

1. 运行本章的所有代码示例（点云、螺旋线、立方体、法向对比、棋盘格纹理、地形曲面），亲自观察每种图元类型的渲染效果。
2. 修改地形示例中的`TerrainHeight()`函数，创建你自己的地形形状——这是巩固PolyData手动构建技能的最佳练习。
3. 尝试用`VTK_POLYGON`类型创建一个正六边形（六个顶点的多边形），并用`vtkPolyDataNormals`计算其法向。
4. 修改立方体示例中某个面的顶点顺序（故意弄反），观察渲染结果——这能帮助你直观理解右手定则。

> **本章关键概念口诀**："几何存坐标，拓扑存索引；法向定光照，纹理贴图像；顶点逆时针，正面朝向你。"

---

*本章完成。第七章是第二卷的基石——深入理解PolyData的结构和构造方法，将使你在后续章节中面对各种数据操作时从容不迫。*
