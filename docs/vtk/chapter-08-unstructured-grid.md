# 第八章 非结构化网格（Unstructured Grid）

## 本章导读

在第七章中，你深入学习了`vtkPolyData`--VTK中最灵活的多边形数据表示，它擅长描述曲面、线段和点集。在第九章中，你将遇见结构化数据家族--`vtkImageData`、`vtkRectilinearGrid`和`vtkStructuredGrid`，它们依靠隐式的IJK索引拓扑来实现极高的内存效率。

本章聚焦于VTK数据模型中**最通用、表达能力最强**的数据集类型：**`vtkUnstructuredGrid`（非结构化网格）**。它的核心特征是：**一个网格中可以同时包含任意类型的单元（Cell）**--四面体、六面体、楔形体、金字塔单元，甚至混合在一起。这是结构化数据和PolyData都无法做到的。

本章将带你完成以下内容：

1. **理解非结构化网格的本质**--为什么需要显式存储拓扑，以及它与vtkPolyData和结构化数据的根本差异。
2. **掌握常用单元类型**--从线性四面体到二次六面体的点计数、拓扑结构和典型应用场景，配有完整的快速参考表。
3. **手动构建非结构化网格**--逐步掌握`vtkPoints` -> `Allocate()` -> `InsertNextCell()` -> `SetPoints()`的构造流程。
4. **常用查询与操作**--获取指定单元、按类型筛选、提取表面、查询单元质量。
5. **一个完整的FEM可视化示例**--手动构建悬臂梁网格，模拟位移场，使用`vtkWarpVector`变形网格，并用查找表着色，包含完整的CMakeLists.txt。

本章是连接"数据表示"与"科学计算可视化"的桥梁。读完本章后，你将对VTK的数据模型形成完整的认知，并具备将外部仿真数据（如FEM结果、CFD网格）导入VTK进行可视化的能力。

---

## 8.1 什么是非结构化网格

`vtkUnstructuredGrid`是VTK中最通用的数据集类型。与其他数据集不同，它不要求网格具有规则的拓扑结构--每个单元可以属于不同的类型，且单元之间的连接关系（Connectivity）是**显式存储**的。

理解这一点的最佳方式是与你已经学过的其他数据集进行对比：

| 特性 | vtkPolyData | vtkStructuredGrid | vtkUnstructuredGrid |
|------|------------|-------------------|---------------------|
| **单元维度** | 0D、1D、2D（点、线、面） | 2D四边形或3D六面体 | 任意维度（含3D体单元） |
| **单元类型** | 固定四类（Verts/Lines/Polys/Strips） | 隐式定义，通常为六面体 | 任意类型，可混合 |
| **拓扑存储** | 显式（vtkCellArray） | 隐式（IJK索引推导） | 显式（每个单元存储点索引） |
| **体单元支持** | 不支持 | 支持（仅六面体） | 支持（四面体、六面体、楔形、金字塔等） |
| **典型应用** | 曲面、轮廓、粒子轨迹 | 贴体网格、变形几何 | FEM结果、CFD网格、工程仿真 |

`vtkUnstructuredGrid`的"非结构化"一词并不意味着网格是混乱的--它指的是网格的拓扑结构不遵循任何预设的规则模式。每一个单元都需要显式地记录"它由哪几个点组成"，而不是像结构化网格那样可以通过简单的数学公式从索引推导出邻居关系。

这种显式拓扑存储带来了灵活性，也带来了内存开销。对于一个有N个点的六面体结构化网格，你可以不存储任何拓扑信息就确定所有单元的连接关系。而对于具有相同几何形状的非结构化网格，你必须为每个六面体存储其8个顶点索引。

**非结构化网格的典型应用场景包括：**

- **有限元法（FEM）结果可视化**：结构分析、热传导、电磁场仿真通常使用四面体或六面体网格，计算结果（位移、应力、温度）定义在网格节点或单元上。
- **计算流体力学（CFD）网格**：边界层附近使用楔形体（棱柱）单元，核心区域使用四面体，壁面附近使用六面体。这些不同类型的单元需要共存于同一个网格中。
- **工程仿真数据交换**：许多仿真软件（ANSYS、Abaqus、OpenFOAM、COMSOL）输出的网格本质上都是非结构化的。将这些数据导入VTK进行可视化，最自然的方式就是使用`vtkUnstructuredGrid`。
- **地质建模与油藏模拟**：地下岩层的几何形状极其不规则，需要使用非结构化的四面体或六面体网格来离散化。

总结来说：当你需要表达任意拓扑的三维体网格，尤其是当单个网格中需要混合多种单元类型时，`vtkUnstructuredGrid`是唯一的选择。

---

## 8.2 常用单元类型详解

VTK定义了超过70种单元类型（在`vtkCellType.h`中枚举），但实际工程中最常用的只有十余种。本节将逐一介绍每种核心单元类型的点计数、拓扑描述和典型应用场景。

### 8.2.1 线性单元（Linear Cells）

线性单元是指单元边上的几何映射是线性的--即单元内的任意点坐标由顶点坐标的线性插值得到。这些单元在工程仿真中最常见。

#### VTK_TETRA（4点）-- 线性四面体

四面体是最简单、最灵活的三维体单元。它由四个三角形面围成，是FEM仿真的"主力单元"--几乎所有的自动网格生成器（如TetGen、NETGEN）都能生成四面体网格。

```
      点3 (p3)
       /|\
      / | \
     /  |  \
    /   |   \
   /    |    \
点0 ----|---- 点2 (p2)
  \     |     /
   \    |    /
    \   |   /
     \  |  /
      \ | /
       \|/
      点1 (p1)
```

- **点数**：4
- **面数**：4（每个面是一个三角形）
- **边数**：6
- **点排列**：点0、1、2构成底面三角形（逆时针，从下方看），点3是顶点
- **典型应用**：FEM结构分析、热传导仿真、电磁场仿真、自动网格生成
- **优点**：可适应任意复杂几何形状，自动网格生成成熟
- **缺点**：同等精度下需要比六面体更多的单元数量

#### VTK_HEXAHEDRON（8点）-- 线性六面体

六面体（也称砖块单元、Brick Element）是结构化网格区域的自然选择。在应力分析和CFD中，六面体单元通常比四面体提供更高的精度。

```
        点7__________点6
        /|          /|
       / |         / |
    点4 --------- 点5 |
      |  |        |  |
      |  点3______|__点2
      | /         | /
      |/          |/
    点0 --------- 点1
```

VTK中六面体顶点的标准排列顺序为：底面逆时针（0-1-2-3），然后顶面逆时针（4-5-6-7），其中点0在点4的下方。

- **点数**：8
- **面数**：6（每个面是一个四边形）
- **边数**：12
- **点排列**：底面`{0,1,2,3}`逆时针，顶面`{4,5,6,7}`逆时针，4在0上方，5在1上方，依此类推
- **典型应用**：结构化网格区域的FEM分析、CFD核心区域、接触力学
- **优点**：计算精度高，单元数量相对较少
- **缺点**：难以自动生成复杂几何的纯六面体网格

#### VTK_WEDGE（6点）-- 楔形体（三棱柱）

楔形体单元在CFD中非常重要--它常用于边界层网格，其中从物面向外延伸的棱柱层自然形成楔形体单元。

```
      点5
      /|\
     / | \
    /  |  \
   /   |   \
  /    |    \
点3 ----|---- 点4
 |      |      |
 |    点2      |
 |    / \      |
 |   /   \     |
 |  /     \    |
 | /       \   |
 |/         \  |
点0 --------- 点1
```

- **点数**：6
- **面数**：5（2个三角形面 + 3个四边形面）
- **边数**：9
- **点排列**：底面三角形`{0,1,2}`逆时针，顶面三角形`{3,4,5}`逆时针，3在0上方，4在1上方，5在2上方
- **典型应用**：CFD边界层网格（从物面三角形挤出）、过渡区域
- **优点**：自然表示挤出几何
- **缺点**：使用场景较为特定

#### VTK_PYRAMID（5点）-- 金字塔单元

金字塔单元的主要作用是**过渡**--在六面体网格区域和四面体网格区域之间建立连接。一个四边形面（底面）可以连接到四面体网格，而四个三角形面可以连接到六面体网格。

```
        点4 (顶点)
        /|\
       / | \
      /  |  \
     /   |   \
    /    |    \
   /     |     \
  /      |      \
 /       |       \
点3 ------------- 点2
 |\              /|
 | \            / |
 |  \          /  |
 |   \        /   |
 |    \      /    |
 |     \    /     |
 |      \  /      |
 |       \/       |
点0 ------------- 点1
```

- **点数**：5
- **面数**：5（1个四边形底面 + 4个三角形侧面）
- **边数**：8
- **点排列**：底面四边形`{0,1,2,3}`逆时针，点4为顶点
- **典型应用**：六面体到四面体的网格过渡区域
- **优点**：唯一的四边形+三角形混合单元，过渡作用不可替代
- **缺点**：单独使用场景较少

#### VTK_VOXEL（8点）-- 体素

`VTK_VOXEL`与`VTK_HEXAHEDRON`看似相同（都是8点六面体），但**点排列顺序不同**。Voxel是专门为`vtkImageData`设计的单元类型。

```
        点6__________点7
        /|          /|
       / |         / |
    点4 --------- 点5 |
      |  |        |  |
      |  点2______|__点3
      | /         | /
      |/          |/
    点0 --------- 点1
```

Voxel的点排列按**IJK顺序**：对于索引空间中的`(i,j,k)`，8个顶点按`(0,0,0), (1,0,0), (0,1,0), (1,1,0), (0,0,1), (1,0,1), (0,1,1), (1,1,1)`的顺序排列。

- **点数**：8
- **与HEXAHEDRON的区别**：仅在于点排列顺序。Voxel的边平行于坐标轴，点的顺序是二进制的（按I、J、K的二进制增长）
- **典型应用**：医学影像（CT/MRI体数据）、规则网格仿真中隐式定义的体素

### 8.2.2 二次单元（Quadratic Cells）

二次单元在每条边的中点增加了一个额外的节点，使得单元内的插值函数从线性提升为二次多项式。这允许用更少的单元捕获更复杂的场分布。

#### VTK_QUADRATIC_TETRA（10点）-- 二次四面体

在标准四面体的4个顶点基础上，每条边的中点增加一个节点（共6个），总共10个点。

```
      点3 (顶点)
       /|\
      / | \
    p7  |  p9    ← p7, p9 是边中点
    /   |   \
   /    p8    \   ← p8 是边中点
  /     |      \
点0 -- p4 ----- 点2
  \     |      /
   \    p6    /    ← p6 是边中点
    \   |   /
    p5  |  p10   ← p5, p10 是边中点
      \ | /
       \|/
      点1
```

点排列顺序（按VTK约定）：
- 0-3：4个顶点（与线性四面体相同顺序）
- 4-9：6个边中点，依次位于边(0,1)、(1,2)、(0,2)、(0,3)、(1,3)、(2,3)上

- **点数**：10
- **典型应用**：高精度FEM分析、需要精确应力/应变梯度的区域
- **优点**：用少量单元即可表示弯曲边界和复杂场分布
- **缺点**：计算成本显著增加

#### VTK_QUADRATIC_HEXAHEDRON（20点）-- 二次六面体（Serendipity单元）

在标准六面体的8个顶点基础上，每条边的中点增加一个节点（12个），总共20个点。注意这不是27点拉格朗日六面体（`VTK_TRIQUADRATIC_HEXAHEDRON`）--后者在面的中心还有节点。

```
        点7__________e18_________点6
        /|                      /|
       / |                     / |
    e15 |                   e13 |
     /  |                   /   |
    /   |                  /    |
  点4 __________e19_________点5   |
   |    |                 |     |
   |    |                 |     |
   |   点3_______e17______|____点2
   e14 /                 e16  /
   |  /                  |   /
   | /                   |  /
   |/                    | /
  点0__________e10_________点1
```

- **点数**：20（8个顶点 + 12个边中点）
- **典型应用**：高精度结构分析、接触力学、需要精确挠度计算的梁/板分析
- **注意**：在VTK的单元分类中，该类型编号为`VTK_QUADRATIC_HEXAHEDRON = 25`

### 8.2.3 单元类型快速参考表

| 单元类型 | VTK枚举值 | 点数 | 面类型 | 典型应用 | 维度 |
|----------|-----------|------|--------|---------|------|
| **VTK_TETRA** | 10 | 4 | 4个三角形 | FEM主力单元 | 3D |
| **VTK_HEXAHEDRON** | 12 | 8 | 6个四边形 | 结构化FEM区域 | 3D |
| **VTK_WEDGE** | 13 | 6 | 2三角+3四边 | CFD边界层 | 3D |
| **VTK_PYRAMID** | 14 | 5 | 1四边+4三角 | 六面体-四面体过渡 | 3D |
| **VTK_VOXEL** | 11 | 8 | 6个四边形 | 医学影像、ImageData | 3D |
| **VTK_QUADRATIC_TETRA** | 24 | 10 | 4个二次三角面 | 高精度FEM | 3D |
| **VTK_QUADRATIC_HEXAHEDRON** | 25 | 20 | 6个二次四边形面 | 高精度结构分析 | 3D |
| **VTK_QUADRATIC_WEDGE** | 26 | 15 | 2个二次三角+3个二次四边 | 高精度CFD边界层 | 3D |
| **VTK_QUADRATIC_PYRAMID** | 27 | 13 | 1个二次四边+4个二次三角 | 高精度过渡区 | 3D |
| **VTK_TRIANGLE** | 5 | 3 | -- | 2D FEM | 2D |
| **VTK_QUAD** | 9 | 4 | -- | 2D FEM | 2D |
| **VTK_LINE** | 3 | 2 | -- | 1D杆单元 | 1D |

> **提示**：VTK单元格类型的完整列表和枚举值可通过`vtkCellType.h`头文件查阅。本节涵盖的是实际工程中最常见的类型。

---

## 8.3 构建非结构化网格

### 8.3.1 构造流程

构建一个`vtkUnstructuredGrid`遵循以下标准步骤：

```
vtkPoints (创建点)
    ↓
vtkUnstructuredGrid::Allocate() (预分配单元内存)
    ↓
InsertNextCell() × N (逐个插入单元)
    ↓
SetPoints() (将点关联到网格)
    ↓
（可选）关联点数据/单元数据（标量、向量等）
```

### 8.3.2 关键API

**`Allocate(vtkIdType numCells, int extSize=1000)`**

预分配单元存储空间。`numCells`指定初始容量，`extSize`指定当容量不足时每次扩展增加的数量。正确预估单元数量可以避免运行时的多次内存重分配。

**`InsertNextCell(int cellType, vtkIdType npts, const vtkIdType* pts)`**

向网格中插入一个单元，这是最核心的API：
- `cellType`：单元类型枚举值（如`VTK_TETRA`、`VTK_HEXAHEDRON`等）
- `npts`：该单元包含的点数（必须与cellType一致）
- `pts`：指向点ID数组的指针，数组长度必须等于npts

返回值是新插入单元的ID（从0开始递增）。

**`SetPoints(vtkPoints* points)`**

将预先构建的`vtkPoints`对象关联到网格。

### 8.3.3 完整示例：构建包含三种混合单元的非结构化网格

以下代码演示了如何手动构建一个同时包含四面体、六面体和楔形体的非结构化网格。

```cpp
// build_mixed_grid.cxx
// 演示手动构建包含多种单元类型的 vtkUnstructuredGrid

#include <vtkNew.h>
#include <vtkPoints.h>
#include <vtkUnstructuredGrid.h>
#include <vtkHexahedron.h>
#include <vtkTetra.h>
#include <vtkWedge.h>
#include <vtkCellTypes.h>
#include <vtkDataSetMapper.h>
#include <vtkActor.h>
#include <vtkRenderer.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkProperty.h>
#include <vtkDataSetSurfaceFilter.h>

int main()
{
  // ============================================================
  // 第一步：创建点
  // ============================================================
  // 我们将构建一个简单的3D场景：
  //   - 一个六面体位于左侧
  //   - 一个四面体通过金字塔过渡到六面体
  //   - 一个楔形体位于右侧
  //
  // 为简化，我们手动定义所有点的坐标

  vtkNew<vtkPoints> points;
  points->InsertNextPoint( 0.0, 0.0, 0.0);  // 点0
  points->InsertNextPoint( 1.0, 0.0, 0.0);  // 点1
  points->InsertNextPoint( 1.0, 1.0, 0.0);  // 点2
  points->InsertNextPoint( 0.0, 1.0, 0.0);  // 点3
  points->InsertNextPoint( 0.0, 0.0, 1.0);  // 点4
  points->InsertNextPoint( 1.0, 0.0, 1.0);  // 点5
  points->InsertNextPoint( 1.0, 1.0, 1.0);  // 点6
  points->InsertNextPoint( 0.0, 1.0, 1.0);  // 点7

  // 四面体的额外顶点（位于六面体右侧上方）
  points->InsertNextPoint( 2.5, 0.5, 2.0);  // 点8

  // 楔形体的顶点（位于最右侧）
  points->InsertNextPoint( 3.0, 0.0, 0.0);  // 点9
  points->InsertNextPoint( 4.0, 0.0, 0.0);  // 点10
  points->InsertNextPoint( 3.5, 0.5, 0.0);  // 点11
  points->InsertNextPoint( 3.0, 0.0, 1.5);  // 点12
  points->InsertNextPoint( 4.0, 0.0, 1.5);  // 点13
  points->InsertNextPoint( 3.5, 0.5, 1.5);  // 点14

  // 金字塔单元的顶点（六面体右侧面 + 四面体左侧的过渡）
  // 使用六面体的右侧面(1,2,6,5)作为底面，点8的投影作为顶点
  points->InsertNextPoint( 2.0, 0.5, 0.5);  // 点15（金字塔顶点）

  // ============================================================
  // 第二步：创建 vtkUnstructuredGrid 并预分配空间
  // ============================================================
  vtkNew<vtkUnstructuredGrid> ugrid;
  ugrid->Allocate(4, 100);  // 预分配4个单元

  // ============================================================
  // 第三步：逐个插入单元
  // ============================================================

  // 单元0：六面体（点0-7）
  //
  //  底面 0-1-2-3（逆时针），顶面 4-5-6-7（逆时针）
  //  4在0上方，5在1上方，6在2上方，7在3上方
  {
    vtkIdType hexPts[8] = {0, 1, 2, 3, 4, 5, 6, 7};
    ugrid->InsertNextCell(VTK_HEXAHEDRON, 8, hexPts);
  }

  // 单元1：四面体（点1, 2, 5, 8）
  //
  //  底面 1-2-5（逆时针，从下方看），顶点 8
  {
    vtkIdType tetPts[4] = {1, 2, 5, 8};
    ugrid->InsertNextCell(VTK_TETRA, 4, tetPts);
  }

  // 单元2：金字塔（点1, 2, 6, 5, 15）
  //
  //  底面 1-2-6-5（逆时针），顶点 15
  {
    vtkIdType pyrPts[5] = {1, 2, 6, 5, 15};
    ugrid->InsertNextCell(VTK_PYRAMID, 5, pyrPts);
  }

  // 单元3：楔形体（点9, 10, 11, 12, 13, 14）
  //
  //  底面三角形 9-10-11（逆时针），顶面三角形 12-13-14（逆时针）
  //  12在9上方，13在10上方，14在11上方
  {
    vtkIdType wedgePts[6] = {9, 10, 11, 12, 13, 14};
    ugrid->InsertNextCell(VTK_WEDGE, 6, wedgePts);
  }

  // ============================================================
  // 第四步：将点关联到网格
  // ============================================================
  ugrid->SetPoints(points);

  // ============================================================
  // 第五步：验证网格信息
  // ============================================================
  std::cout << "=== 非结构化网格信息 ===" << std::endl;
  std::cout << "总点数:     " << ugrid->GetNumberOfPoints() << std::endl;
  std::cout << "总单元数:   " << ugrid->GetNumberOfCells()  << std::endl;

  for (vtkIdType i = 0; i < ugrid->GetNumberOfCells(); ++i)
  {
    int cellType = ugrid->GetCellType(i);
    vtkCell* cell = ugrid->GetCell(i);
    std::cout << "单元[" << i << "] 类型=" << cellType
              << " (" << cell->GetCellTypeAsString(cellType) << ")"
              << " 点数=" << cell->GetNumberOfPoints()
              << std::endl;
  }

  // ============================================================
  // 第六步：可视化
  // ============================================================
  // 使用 vtkDataSetSurfaceFilter 提取表面用于渲染
  vtkNew<vtkDataSetSurfaceFilter> surfaceFilter;
  surfaceFilter->SetInputData(ugrid);
  surfaceFilter->Update();

  vtkNew<vtkDataSetMapper> mapper;
  mapper->SetInputConnection(surfaceFilter->GetOutputPort());

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(0.3, 0.7, 0.9);
  actor->GetProperty()->SetEdgeVisibility(1);
  actor->GetProperty()->SetEdgeColor(0.0, 0.0, 0.0);
  actor->GetProperty()->SetLineWidth(1.0);

  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->SetBackground(0.15, 0.15, 0.18);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(800, 600);
  renderWindow->SetWindowName("Mixed Unstructured Grid (Hex + Tet + Pyramid + Wedge)");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  renderWindow->Render();
  std::cout << "\n渲染完成。旋转视角查看混合网格。关闭窗口退出。" << std::endl;
  interactor->Start();

  return 0;
}
```

### 8.3.4 通过SetCells()批量构建

对于大规模网格（数万至数百万个单元），使用`InsertNextCell()`逐单元插入的效率可能不够高。此时可以使用`SetCells()`方法批量设置：

```cpp
// 批量构建方式（高性能）

vtkNew<vtkCellArray> cellArray;

// 准备 Connectivity 和 Offsets 数组
// (与 vtkCellArray 的现代存储格式一致，参见第七章7.1.3节)
vtkNew<vtkIdTypeArray> connectivity;
vtkNew<vtkIdTypeArray> offsets;
vtkNew<vtkUnsignedCharArray> cellTypes;

// 填充 connectivity、offsets、cellTypes 数组...
// ...

cellArray->SetData(offsets, connectivity);
ugrid->SetCells(cellTypes, cellArray);
ugrid->SetPoints(points);
```

对于超大规模网格，这种方式可以大幅减少函数调用开销，但也更复杂。日常开发中，对少于10万个单元的网格使用`InsertNextCell()`是完全可接受的。

---

## 8.4 常用操作与查询

### 8.4.1 基本查询

```cpp
// 获取单元总数
vtkIdType numCells = ugrid->GetNumberOfCells();

// 获取指定ID的单元类型
int cellType = ugrid->GetCellType(cellId);

// 获取指定ID的单元（返回vtkCell指针，可以进行进一步操作）
vtkCell* cell = ugrid->GetCell(cellId);
int npts = cell->GetNumberOfPoints();
vtkIdList* ptIds = cell->GetPointIds();  // 获取单元的点ID列表

// 遍历单元的所有点
for (vtkIdType j = 0; j < cell->GetNumberOfPoints(); ++j)
{
  vtkIdType ptId = cell->GetPointId(j);
  double pt[3];
  ugrid->GetPoint(ptId, pt);
  // 对点的坐标进行处理...
}
```

### 8.4.2 获取特定类型的所有单元

在实际工作中，经常需要提取网格中某一特定类型的全部单元（例如只查看四面体，或只查看六面体）。VTK没有提供直接的"获取所有某类型单元"的API，但可以通过遍历筛选来实现：

```cpp
#include <vector>

// 获取所有四面体单元的ID
std::vector<vtkIdType> getCellsOfType(vtkUnstructuredGrid* grid, int targetType)
{
  std::vector<vtkIdType> result;
  for (vtkIdType i = 0; i < grid->GetNumberOfCells(); ++i)
  {
    if (grid->GetCellType(i) == targetType)
    {
      result.push_back(i);
    }
  }
  return result;
}

// 使用示例
std::vector<vtkIdType> tetraIds = getCellsOfType(ugrid, VTK_TETRA);
std::cout << "四面体单元数量: " << tetraIds.size() << std::endl;

std::vector<vtkIdType> hexIds = getCellsOfType(ugrid, VTK_HEXAHEDRON);
std::cout << "六面体单元数量: " << hexIds.size() << std::endl;
```

### 8.4.3 提取表面网格

`vtkUnstructuredGrid`不能直接交给Mapper渲染--渲染管线需要的是表面（三角形或多边形）。VTK提供了`vtkDataSetSurfaceFilter`来从任意体网格中提取外表面：

```cpp
#include <vtkDataSetSurfaceFilter.h>

vtkNew<vtkDataSetSurfaceFilter> surfaceFilter;
surfaceFilter->SetInputData(ugrid);
surfaceFilter->Update();

vtkPolyData* surface = surfaceFilter->GetOutput();
// surface 现在是一个 vtkPolyData，包含网格的外表面三角形
// 可以直接用于渲染或进一步处理
```

`vtkDataSetSurfaceFilter`的工作原理是：遍历所有单元的每个面，如果一个面只被一个单元引用（即它是边界上的面），则将其输出为表面三角形。内部单元的面（被两个单元共享）不会被输出。

### 8.4.4 单元质量查询

VTK提供了`vtkMeshQuality`过滤器来评估网格中每个单元的质量。不同的质量度量（Quality Metric）适用于不同类型的单元：

```cpp
#include <vtkMeshQuality.h>

vtkNew<vtkMeshQuality> qualityFilter;
qualityFilter->SetInputData(ugrid);

// 设置质量度量类型
// VTK_QUALITY_ASPECT_RATIO -- 纵横比（值越接近1越好）
// VTK_QUALITY_JACOBIAN      -- 雅可比矩阵条件数
// VTK_QUALITY_SKEW          -- 偏斜度
// VTK_QUALITY_VOLUME        -- 单元体积
qualityFilter->SetTriangleQualityMeasureToAspectRatio();
qualityFilter->SetQuadQualityMeasureToAspectRatio();
qualityFilter->SetTetQualityMeasureToAspectRatio();
qualityFilter->SetHexQualityMeasureToAspectRatio();

qualityFilter->Update();

vtkUnstructuredGrid* qualityResult =
  vtkUnstructuredGrid::SafeDownCast(qualityFilter->GetOutput());

// 质量值存储在单元数据的 "Quality" 数组中
vtkDataArray* qualityArray =
  qualityResult->GetCellData()->GetArray("Quality");

if (qualityArray)
{
  double range[2];
  qualityArray->GetRange(range);
  std::cout << "单元质量范围: [" << range[0] << ", " << range[1] << "]" << std::endl;
}
```

单元质量信息在工程仿真中非常重要--质量差的单元（如过度拉伸或扭曲的四面体）可能导致FEM计算不收敛或精度下降。

---

## 8.5 代码示例：悬臂梁FEM可视化

以下是一个完整的、可编译运行的C++程序。它模拟了悬臂梁（一端固定、另一端加载）的FEM位移场，并使用`vtkWarpVector`将标量位移场转换为几何变形，从而实现FEM后处理中常见的"变形网格+着色"可视化效果。

### 8.5.1 示例说明

本示例包含以下关键步骤：

1. **手动构建悬臂梁网格**：沿X方向生成一系列六面体单元，形成梁的几何形状
2. **模拟位移场**：在梁的自由端施加位移，位移大小沿梁长度线性增加（模拟悬臂梁的弯曲）
3. **使用vtkWarpVector变形网格**：将位移向量作用于网格点，产生视觉上的弯曲变形
4. **通过查找表着色**：将位移幅值映射为颜色，使大位移区域显示为红色，小位移区域显示为蓝色
5. **同时显示原始网格和变形网格**：通过并排或同窗口对比

### 8.5.2 完整C++代码

```cpp
// cantilever_beam_fem.cxx
//
// 悬臂梁FEM可视化示例
//
// 功能：
//   1. 手动构建悬臂梁的非结构化六面体网格
//   2. 模拟FEM位移场（自由端位移最大，固定端为零）
//   3. 使用 vtkWarpVector 变形网格
//   4. 按位移幅值着色
//   5. 完整可编译运行
//
// 编译：
//   cmake -B build && cmake --build build
//   ./build/cantilever_beam_fem

#include <vtkActor.h>
#include <vtkCellData.h>
#include <vtkDoubleArray.h>
#include <vtkDataSetMapper.h>
#include <vtkDataSetSurfaceFilter.h>
#include <vtkInteractorStyleTrackballCamera.h>
#include <vtkLookupTable.h>
#include <vtkNew.h>
#include <vtkPointData.h>
#include <vtkPoints.h>
#include <vtkPolyData.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkScalarBarActor.h>
#include <vtkSmartPointer.h>
#include <vtkTextProperty.h>
#include <vtkUnstructuredGrid.h>
#include <vtkWarpVector.h>

#include <cmath>
#include <iostream>
#include <vector>

int main()
{
  // ============================================================
  // 参数定义
  // ============================================================
  // 梁的几何参数
  const double beamLength = 10.0;    // 梁长度 (X方向)
  const double beamWidth  =  1.0;    // 梁宽度 (Y方向)
  const double beamHeight =  2.0;    // 梁高度 (Z方向)

  // 网格分辨率（沿各方向的单元数）
  const int nx = 20;  // 沿长度方向
  const int ny = 4;   // 沿宽度方向
  const int nz = 6;   // 沿高度方向

  // 最大位移幅值（在自由端）
  const double maxDisplacement = 1.5;

  // ============================================================
  // 第一步：生成网格点
  // ============================================================
  //
  // 点的组织方式：沿X方向逐层排列
  // 对于给定的 (ix, iy, iz)：
  //   x = ix * (beamLength / nx)
  //   y = iy * (beamWidth  / ny)
  //   z = iz * (beamHeight / nz)
  //
  // 点的全局ID = (ix * (ny+1) + iy) * (nz+1) + iz

  const int nxPoints = nx + 1;
  const int nyPoints = ny + 1;
  const int nzPoints = nz + 1;

  vtkNew<vtkPoints> points;

  double dx = beamLength / nx;
  double dy = beamWidth  / ny;
  double dz = beamHeight / nz;

  for (int ix = 0; ix < nxPoints; ++ix)
  {
    for (int iy = 0; iy < nyPoints; ++iy)
    {
      for (int iz = 0; iz < nzPoints; ++iz)
      {
        double x = ix * dx;
        double y = iy * dy;
        double z = iz * dz;
        points->InsertNextPoint(x, y, z);
      }
    }
  }

  // ============================================================
  // 第二步：生成六面体单元
  // ============================================================
  //
  // 每个单元由8个相邻网格点组成
  // 对于单元 (ix, iy, iz)，其8个顶点为：
  //
  //   (ix,   iy,   iz)   -- 点0
  //   (ix+1, iy,   iz)   -- 点1
  //   (ix+1, iy+1, iz)   -- 点2
  //   (ix,   iy+1, iz)   -- 点3
  //   (ix,   iy,   iz+1) -- 点4
  //   (ix+1, iy,   iz+1) -- 点5
  //   (ix+1, iy+1, iz+1) -- 点6
  //   (ix,   iy+1, iz+1) -- 点7
  //
  // 点ID公式：pointId(ix, iy, iz) = (ix * nyPoints + iy) * nzPoints + iz

  auto pointId = [nyPoints, nzPoints](int ix, int iy, int iz) -> vtkIdType
  {
    return static_cast<vtkIdType>((ix * nyPoints + iy)) * nzPoints + iz;
  };

  vtkNew<vtkUnstructuredGrid> beamGrid;
  beamGrid->Allocate(nx * ny * nz, 1000);

  for (int ix = 0; ix < nx; ++ix)
  {
    for (int iy = 0; iy < ny; ++iy)
    {
      for (int iz = 0; iz < nz; ++iz)
      {
        vtkIdType hexPts[8] = {
          pointId(ix,   iy,   iz),     // 底面：0
          pointId(ix+1, iy,   iz),     // 底面：1
          pointId(ix+1, iy+1, iz),     // 底面：2
          pointId(ix,   iy+1, iz),     // 底面：3
          pointId(ix,   iy,   iz+1),   // 顶面：4
          pointId(ix+1, iy,   iz+1),   // 顶面：5
          pointId(ix+1, iy+1, iz+1),   // 顶面：6
          pointId(ix,   iy+1, iz+1)    // 顶面：7
        };
        beamGrid->InsertNextCell(VTK_HEXAHEDRON, 8, hexPts);
      }
    }
  }

  beamGrid->SetPoints(points);

  std::cout << "悬臂梁网格信息:" << std::endl;
  std::cout << "  总点数:   " << beamGrid->GetNumberOfPoints() << std::endl;
  std::cout << "  总单元数: " << beamGrid->GetNumberOfCells()  << std::endl;
  std::cout << "  梁尺寸:   " << beamLength << " x " << beamWidth
            << " x " << beamHeight << std::endl;

  // ============================================================
  // 第三步：模拟FEM位移场
  // ============================================================
  //
  // 模拟悬臂梁在自由端受集中力作用的位移场
  // 简化模型：位移沿X方向线性增加，沿Z方向受抛物线影响
  //
  // 理论背景（欧拉-伯努利梁理论）：
  //   挠度 w(x) = (F * x^2) / (6 * E * I) * (3 * L - x)
  //
  // 为简化可视化，我们使用以下位移公式：
  //   Displacement X: 0 (忽略轴向伸长)
  //   Displacement Y: 0 (忽略横向膨胀)
  //   Displacement Z: Uz_max * (x/L)^2 * (3 - 2 * (x/L))
  //                   其中 Uz_max 是自由端最大挠度
  //
  // 此外再叠加一些Y方向的微小位移来展示泊松效应

  vtkNew<vtkDoubleArray> displacements;
  displacements->SetName("Displacement");
  displacements->SetNumberOfComponents(3);  // 向量（3分量）
  displacements->SetNumberOfTuples(beamGrid->GetNumberOfPoints());

  vtkNew<vtkDoubleArray> displacementMag;
  displacementMag->SetName("DisplacementMagnitude");
  displacementMag->SetNumberOfComponents(1);  // 标量
  displacementMag->SetNumberOfTuples(beamGrid->GetNumberOfPoints());

  for (vtkIdType ptId = 0; ptId < beamGrid->GetNumberOfPoints(); ++ptId)
  {
    double pt[3];
    points->GetPoint(ptId, pt);

    double x = pt[0];
    double y = pt[1];
    double z = pt[2];

    double xNorm = x / beamLength;  // 归一化的X位置 [0, 1]

    // Z方向位移（主弯曲方向）-- 使用三次多项式模拟梁的挠度
    // w(x) = maxDisplacement * (xNorm)^2 * (3 - 2*xNorm)
    double dz_disp = maxDisplacement * xNorm * xNorm * (3.0 - 2.0 * xNorm);

    // Y方向位移（泊松效应引起的横向收缩）
    // 简化为随Z坐标和X位置变化
    double dy_disp = -0.05 * maxDisplacement * xNorm * (y - beamWidth / 2.0);

    // X方向位移（中性轴以上压缩，以下拉伸）
    double dx_disp = -0.02 * maxDisplacement * xNorm * (z - beamHeight / 2.0);

    displacements->SetTuple3(ptId, dx_disp, dy_disp, dz_disp);

    double mag = std::sqrt(dx_disp * dx_disp +
                           dy_disp * dy_disp +
                           dz_disp * dz_disp);
    displacementMag->SetTuple1(ptId, mag);
  }

  // 将位移向量关联到点数据
  beamGrid->GetPointData()->SetVectors(displacements);
  // 将位移幅值关联到点数据（用于着色）
  beamGrid->GetPointData()->SetScalars(displacementMag);

  // ============================================================
  // 第四步：使用 vtkWarpVector 变形网格
  // ============================================================
  //
  // vtkWarpVector 读取Point Data中的向量数组，
  // 并将每个点沿其向量方向移动相应的距离

  vtkNew<vtkWarpVector> warpFilter;
  warpFilter->SetInputData(beamGrid);
  // 默认使用活动向量（active vectors），也可显式指定向量数组名
  warpFilter->SetScaleFactor(1.0);  // 缩放因子：1.0表示使用原始位移量
  warpFilter->Update();

  vtkUnstructuredGrid* warpedGrid = warpFilter->GetUnstructuredGridOutput();

  // ============================================================
  // 第五步：创建查找表
  // ============================================================
  //
  // 使用蓝-绿-红渐变色映射位移幅值
  //   蓝色 = 小位移（固定端）
  //   红色 = 大位移（自由端）

  vtkNew<vtkLookupTable> lut;
  lut->SetNumberOfTableValues(256);
  lut->SetTableRange(0.0, maxDisplacement);  // 映射范围
  lut->SetHueRange(0.667, 0.0);              // 蓝(0.667) -> 红(0.0)
  lut->SetSaturationRange(1.0, 1.0);
  lut->SetValueRange(1.0, 1.0);
  lut->Build();

  // ============================================================
  // 第六步：创建Mapper和Actor
  // ============================================================

  // 6a. 变形网格的表面提取
  vtkNew<vtkDataSetSurfaceFilter> warpedSurface;
  warpedSurface->SetInputConnection(warpFilter->GetOutputPort());
  warpedSurface->Update();

  // 6b. 变形网格的Mapper
  vtkNew<vtkPolyDataMapper> warpedMapper;
  warpedMapper->SetInputConnection(warpedSurface->GetOutputPort());
  warpedMapper->SetLookupTable(lut);
  warpedMapper->SetScalarRange(0.0, maxDisplacement);
  warpedMapper->SetScalarModeToUsePointData();

  // 6c. 变形网格的Actor
  vtkNew<vtkActor> warpedActor;
  warpedActor->SetMapper(warpedMapper);
  warpedActor->GetProperty()->SetEdgeVisibility(0);  // 不显示边线
  warpedActor->GetProperty()->SetSpecular(0.3);
  warpedActor->GetProperty()->SetSpecularPower(30);

  // 6d. 原始（未变形）网格的半透明叠加
  //     用于对比变形前后的差异
  vtkNew<vtkDataSetSurfaceFilter> originalSurface;
  originalSurface->SetInputData(beamGrid);

  vtkNew<vtkPolyDataMapper> originalMapper;
  originalMapper->SetInputConnection(originalSurface->GetOutputPort());
  originalMapper->ScalarVisibilityOff();  // 不参与标量着色

  vtkNew<vtkActor> originalActor;
  originalActor->SetMapper(originalMapper);
  originalActor->GetProperty()->SetColor(0.6, 0.6, 0.6);    // 灰色
  originalActor->GetProperty()->SetOpacity(0.25);             // 半透明
  originalActor->GetProperty()->SetRepresentationToWireframe(); // 线框模式
  // 设置线框的分辨率为1（使用单元的实际边）
  originalActor->GetProperty()->SetLineWidth(0.5);

  // ============================================================
  // 第七步：添加标量条（Scalar Bar）
  // ============================================================
  vtkNew<vtkScalarBarActor> scalarBar;
  scalarBar->SetLookupTable(lut);
  scalarBar->SetTitle("Displacement Magnitude");
  scalarBar->SetNumberOfLabels(6);
  scalarBar->GetTitleTextProperty()->SetColor(1.0, 1.0, 1.0);
  scalarBar->GetTitleTextProperty()->SetFontSize(5);
  scalarBar->GetLabelTextProperty()->SetColor(1.0, 1.0, 1.0);
  scalarBar->GetLabelTextProperty()->SetFontSize(4);

  // ============================================================
  // 第八步：创建渲染场景
  // ============================================================
  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(warpedActor);
  renderer->AddActor(originalActor);
  renderer->AddActor2D(scalarBar);
  renderer->SetBackground(0.12, 0.13, 0.18);

  // 设置相机位置以获得更好的视角
  renderer->GetActiveCamera()->SetPosition(8.0, -6.0, 5.0);
  renderer->GetActiveCamera()->SetFocalPoint(5.0, 0.5, 1.0);
  renderer->GetActiveCamera()->SetViewUp(0.0, 0.0, 1.0);
  renderer->GetActiveCamera()->SetClippingRange(0.1, 100.0);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(1200, 800);
  renderWindow->SetWindowName("Cantilever Beam FEM Visualization (vtkWarpVector)");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  // ============================================================
  // 第九步：打印信息并启动交互
  // ============================================================
  std::cout << "\n悬臂梁FEM可视化" << std::endl;
  std::cout << "  固定端: X = 0 (蓝色区域，位移最小)" << std::endl;
  std::cout << "  自由端: X = " << beamLength
            << " (红色区域，最大位移 = " << maxDisplacement << ")" << std::endl;
  std::cout << "  灰色线框: 原始未变形网格" << std::endl;
  std::cout << "  彩色实体: 变形后网格（按位移幅值着色）" << std::endl;
  std::cout << "\n旋转视角观察梁的弯曲变形。关闭窗口退出。" << std::endl;

  renderWindow->Render();
  interactor->Start();

  return 0;
}
```

### 8.5.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
project(CantileverBeamFEM VERSION 1.0.0 LANGUAGES CXX)

# 设置C++标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 查找VTK 9.5.2及其所需组件
find_package(VTK 9.5.2
  COMPONENTS
    CommonCore
    CommonDataModel
    CommonExecutionModel
    CommonMath
    CommonTransforms
    FiltersCore
    FiltersGeneral
    FiltersGeometry
    FiltersSources
    InteractionStyle
    RenderingAnnotation
    RenderingCore
    RenderingOpenGL2
  REQUIRED
)

# 打印VTK信息
message(STATUS "VTK version: ${VTK_VERSION}")
message(STATUS "VTK rendering backend: OpenGL2")

# 创建可执行文件
add_executable(cantilever_beam_fem cantilever_beam_fem.cxx)

# 链接VTK库
target_link_libraries(cantilever_beam_fem PRIVATE ${VTK_LIBRARIES})

# 自动初始化VTK模块工厂
vtk_module_autoinit(
  TARGETS cantilever_beam_fem
  MODULES ${VTK_LIBRARIES}
)

# 设置可执行文件的输出目录
set_target_properties(cantilever_beam_fem PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
)
```

### 8.5.4 运行说明

**编译：**

```bash
cd cantilever_beam_fem/
cmake -B build
cmake --build build
```

**运行：**

```bash
# Windows
.\build\bin\Release\cantilever_beam_fem.exe

# Linux/macOS
./build/bin/cantilever_beam_fem
```

**预期效果：**

运行程序后，你将看到一个3D渲染窗口，其中包含：

1. **彩色实体梁**：显示变形后的悬臂梁。固定端（X=0）呈蓝色，自由端（X=10）呈红色。颜色渐变反映了位移幅值从固定端到自由端的增大。
2. **灰色半透明线框**：显示原始未变形的梁。这使你能直观地对比变形前后的差异。
3. **标量条**：显示颜色到位移幅值的映射关系。
4. **交互控制**：使用鼠标拖拽旋转、缩放、平移视角。

你可以尝试以下操作来加深理解：
- 修改`maxDisplacement`参数（第66行），观察不同加载强度下的变形效果
- 修改`nx`、`ny`、`nz`参数来改变网格精度
- 修改`SetScaleFactor()`的值为2.0或0.5来放大或缩小位移
- 取消`originalActor`的创建（不显示原始网格），只看变形结果

---

## 8.6 本章小结

本章全面介绍了VTK中最通用的数据类型--`vtkUnstructuredGrid`（非结构化网格），涵盖了从概念理解到实战应用的完整知识链。

### 8.6.1 核心要点回顾

1. **非结构化网格的本质**：`vtkUnstructuredGrid`是VTK中表达能力最强的数据集，支持在单个网格中混合任意类型的单元。与`vtkPolyData`（仅支持0D-2D面单元）和结构化数据（隐式拓扑）不同，非结构化网格的拓扑需要显式存储，这带来了灵活性但增加了内存开销。

2. **单元类型**：VTK定义了70余种单元类型，工程中最常用的包括线性四面体（4点）、六面体（8点）、楔形体（6点）、金字塔（5点）和体素（8点），以及它们对应的二次版本（10点、20点、15点、13点）。选择哪种单元类型取决于仿真方法、几何复杂度和精度需求。

3. **构造流程**：`vtkPoints` -> `Allocate()` -> `InsertNextCell()` -> `SetPoints()` 是手动构建非结构化网格的标准四步法。对于大规模网格，可以使用`SetCells()`批量构建以提高性能。

4. **常用操作**：`GetCell()`和`GetCellType()`用于查询单元信息；`vtkDataSetSurfaceFilter`用于从体网格中提取外表面以进行渲染；`vtkMeshQuality`用于评估单元质量。

5. **FEM可视化模式**：悬臂梁示例展示了科学可视化中最常见的后处理模式--使用`vtkWarpVector`将位移场作用于网格产生变形，使用查找表（Lookup Table）将标量场映射为颜色，叠加原始网格作为参考基准。

### 8.6.2 与前后章节的关系

本章是第二卷（进阶篇）的第二章。第七章讲解了`vtkPolyData`（面数据的表示），第九章将讲解结构化数据（ImageData/RectilinearGrid/StructuredGrid）。至此，你将完整掌握VTK的所有主要数据集类型，为后续的过滤器、体渲染和高级可视化技术打下坚实基础。

### 8.6.3 扩展阅读方向

- `vtkCellType.h`：VTK中所有单元类型的完整枚举定义
- `vtkUnstructuredGridReader` / `vtkUnstructuredGridWriter`：将非结构化网格读写为VTK的XML格式文件（.vtu）
- `vtkExtractUnstructuredGrid`：按区域或单元类型提取子网格
- `vtkAppendFilter`：将多个非结构化网格合并为一个
- `vtkThreshold`：基于标量值提取部分单元（如只显示高应力区域）
- `vtkShrinkFilter`：收缩每个单元以查看内部结构（爆炸视图）
