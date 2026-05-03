# 第3章 数据模型基础 (Data Model Basics)

在第1章和第2章中，我们介绍了VTK的整体架构和可视化管线的工作机制。现在我们将深入VTK的核心——**数据模型**。VTK的强大之处，很大程度上在于其对科学数据的组织方式的精妙设计。理解数据模型是掌握VTK的关键一步。

---

## 3.1 数据模型概览

### 3.1.1 数据对象的根基：vtkDataObject

VTK中所有数据类型的根类是 **`vtkDataObject`**。它是数据在VTK管线中流转的"通用货币"。无论是几何数据、图像数据还是表格数据，它们都继承自 `vtkDataObject`。

`vtkDataObject` 定义了所有VTK数据共有的基本接口和能力：

- **信息对象（vtkInformation）**：存储描述数据的元信息，如数据的空间范围（bounds）、数据是否已被修改（MTime）等。
- **管线信息（PipelineInformation）**：记录数据在管线中的角色，例如片（piece）编号、片总数，用于并行处理。
- **字段数据（FieldData）**：存储在"数据集范围"但不直接关联几何元素的数据，例如时间戳、材料属性等。

一个核心的工程准则贯穿VTK设计：**数据对象只存储数据，不执行算法**。算法是由 `vtkAlgorithm`（第2章中讨论的Source、Filter、Mapper）来执行的。这种"数据—算法"分离的设计让VTK既灵活又健壮。

### 3.1.2 类层次结构

VTK数据模型的继承关系如下：

```
vtkObject
  └── vtkDataObject           ← 所有数据的根基
        ├── vtkDataSet         ← 几何数据的公共基类（本章重点）
        ├── vtkTable           ← 表格数据（类似数据库表）
        ├── vtkGraph           ← 图/网络数据
        ├── vtkTree            ← 树形层次结构数据
        ├── vtkImageData       ← （也直接继承vtkDataSet的一些特性）
        ├── vtkHyperTreeGrid   ← 层次化自适应网格
        └── vtkCompositeDataSet ← 复合数据集
              ├── vtkMultiBlockDataSet
              ├── vtkMultiPieceDataSet
              ├── vtkHierarchicalBoxDataSet
              └── vtkOverlappingAMR
```

大多数科学与工程可视化场景使用的核心类是 **`vtkDataSet`**，它代表**带有几何结构的数据**——即数据同时包含空间位置信息和该位置上的属性值。

### 3.1.3 数据对象 vs 算法对象

回顾第2章的管线概念，必须清晰区分两类VTK对象：

| 类别 | 角色 | 典型类 |
|------|------|--------|
| **数据对象 (Data Object)** | 纯数据容器，存储点和单元信息 | `vtkPolyData`, `vtkImageData`, `vtkUnstructuredGrid` |
| **算法对象 (Algorithm)** | 管线参与者，读取/处理/写入数据 | `vtkAlgorithm` 的子类：`vtkConeSource`, `vtkContourFilter`, `vtkPolyDataMapper` |

数据对象是被动的——它们不知道如何生成自己，只知道如何描述自身。算法对象是主动的——它们消费和产出数据对象。

### 3.1.4 数据集属性（Dataset Attributes）的概念

当我们讨论一个 `vtkDataSet` 时，最重要的概念之一就是**属性数据 (Attribute Data)**。

想象一个三角形网格：网格上有N个顶点和M个三角形单元。除了几何坐标外，每个顶点可能还带有：
- 温度值（标量）
- 速度方向（矢量）
- 法线方向（法向量）

这些附加在几何元素上的信息就是**属性数据**。VTK将属性数据分为两大类：
- **点数据 (Point Data)**：关联到每个点
- **单元数据 (Cell Data)**：关联到每个单元

---

## 3.2 vtkDataSet层级体系

### 3.2.1 vtkDataSet：几何数据的公共基类

`vtkDataSet` 继承自 `vtkDataObject`，是所有**具有显式或隐式几何结构**的数据类型的公共基类。它定义了以下核心概念：

- **Points**：空间中的坐标集合
- **Cells**：由点索引定义的几何体素
- **Point Data**：附加在点上的属性数组
- **Cell Data**：附加在单元上的属性数组

任何 `vtkDataSet` 的子类都可以通过统一的接口访问这些信息。这意味着一个为 `vtkImageData` 编写的过滤器，通常也能处理 `vtkUnstructuredGrid`（只要操作基于 `vtkDataSet` 的通用接口）。

### 3.2.2 五种主要子类

VTK提供了五种主要的 `vtkDataSet` 子类，覆盖了从简单规则网格到复杂非结构化网格的完整谱系。理解每种类型的特点和使用场景至关重要。

#### 3.2.2.1 vtkImageData — 均匀网格

**`vtkImageData`** 表示一个在三个坐标轴方向上均匀分布的规则网格。它是VTK中最简单、最高效的数据集类型。

```
      Y方向 (3个点)
        ↑
        |
        +---+---+
        |   |   |
        +---+---+
        |   |   |
        +---+---+
        +---+---+
        |   |   |
        +---+---+
                → X方向 (4个点)

   Z方向 (2个点，仅一个切片)
```

**关键属性：**

| 属性 | 含义 | 示例 |
|------|------|------|
| `Origin` | 网格起点的3D坐标 | `(0.0, 0.0, 0.0)` |
| `Spacing` | 三个方向上的网格间距 | `(1.0, 0.5, 2.0)` |
| `Extent` | 六个方向上的索引范围 | `(0, 9, 0, 9, 0, 4)` |
| `Dimensions` | 三个方向上的点数 = (nx, ny, nz) | `(10, 10, 5)` |

**计算一个点的坐标：**
```
P[i, j, k] = Origin + (i * Spacing[0], j * Spacing[1], k * Spacing[2])
```

**使用场景：**
- 医学图像（CT、MRI）：规则的像素/体素网格
- 流体模拟：在规则网格上的有限差分计算
- 任何按规则阵列组织的体积数据

由于网格是规则的，`vtkImageData` 不需要显式存储所有点的坐标——只需存储 `Origin`、`Spacing` 和 `Extent`，坐标可以通过上述公式**隐式计算**。这大大节省了内存。

#### 3.2.2.2 vtkRectilinearGrid — 直线网格

**`vtkRectilinearGrid`** 类似 `vtkImageData`，网格线仍沿坐标轴方向，但**每个轴上的间距可以是非均匀的**。

```
      Y方向 (间距不均匀)
        ↑
        |
        +-----+---------+--+
        |     |         |  |
        +-----+---------+--+
        |     |         |  |
        +-----+---------+--+
        |     |         |  |
        +-----+---------+--+
            → X方向 (间距也不均匀)
```

每个坐标轴分别存储一个坐标数组：
```cpp
xCoordinates = {0.0, 1.0, 2.5, 5.0, 10.0}
yCoordinates = {0.0, 0.8, 2.0, 3.0}
zCoordinates = {0.0, 5.0, 12.0}
```

点 `P[i, j, k]` 的坐标为 `(xCoords[i], yCoords[j], zCoords[k])`。

**使用场景：**
- 在特定区域需要更高分辨率的流体模拟
- 边界层网格（靠近壁面间距小，远离壁面间距大）

#### 3.2.2.3 vtkStructuredGrid — 曲线结构网格

**`vtkStructuredGrid`** 保持了规则的拓扑结构（每个点在逻辑上有明确的 `[i, j, k]` 索引），但**几何位置可以是任意的**——网格线不再沿坐标轴，也不一定是直的。

```
  逻辑结构：[0..4] × [0..3]
  几何位置：每个点可以自由放置

    (0,3)  (1,3)  (2,3)  (3,3)  (4,3)
      *------*------*------*------*
      |      |      |      |      |
      *------*------*------*------*   ← 单元边界可以是曲线
      |      |      |      |      |
      *------*------*------*------*
    (0,0)  (1,0)  (2,0)  (3,0)  (4,0)
```

与 `vtkImageData` 和 `vtkRectilinearGrid` 不同，`vtkStructuredGrid` 需要**显式存储所有点的坐标**（通过 `vtkPoints` 对象），但单元之间的连接关系仍然是隐式的——它总是由 `[i, j, k]` 与 `[i+1, j+1, k+1]` 等邻居构成。

**使用场景：**
- 计算流体力学（CFD）中的贴体网格（body-fitted grid）
- 有限元分析中的变形网格

#### 3.2.2.4 vtkPolyData — 多边形数据

**`vtkPolyData`** 是VTK中最灵活、最常用的数据集类型之一。它由以下元素组成：

| 元素 | 说明 |
|------|------|
| **Points (顶点)** | 3D坐标集合 |
| **Verts (点元)** | 独立的点单元（0维） |
| **Lines (线段)** | 由两个点索引定义的线段（1维） |
| **Polys (多边形)** | 由3个或更多点索引定义的多边形（2维） |
| **Strips (三角形带)** | 共享边的三角形序列 |

```
  vtkPolyData 的构成示意：

  Points:  p0 p1 p2 p3 p4 p5 p6
  Verts:   v(p0)
  Lines:   l(p1, p2)
  Polys:   tri(p1, p2, p3), quad(p3, p4, p5, p6)
  Strips:  strip(p0, p1, p2, p3) → 两个三角形
```

`vtkPolyData` 显式存储所有点坐标和单元连接关系。它可以描述任意复杂的曲面和线框结构。

**使用场景：**
- 等值面提取（Marching Cubes 算法输出）
- CAD模型的三角化曲面
- 粒子轨迹线
- 流线可视化

#### 3.2.2.5 vtkUnstructuredGrid — 非结构化网格

**`vtkUnstructuredGrid`** 是最通的数据集类型。它可以包含**任意类型、任意组合的单元**（四面体、六面体、棱柱、金字塔等），适用于有限元分析（FEA）和计算流体力学（CFD）中极其复杂的几何体。

```
  一个混合单元类型的非结构化网格：

       p3         p4
        *---------*
       /|        /|
      / |       / |
  p2 *--+------*p5+----* p7
     |  *------|--*
     | / p0    | / p1
     *---------*/
    p8         p9

  包含：四面体(p0,p1,p2,p3), 六面体(p0,p1,p5,p4,p8,p9,p7,p6), 棱柱, 金字塔...
```

**使用场景：**
- 有限元分析（FEA）
- CFD中的复杂几何体网格
- 地质建模中的不规则地层
- 任何需要灵活单元类型的场景

### 3.2.3 数据集类型选择指南

| 你的数据特征 | 推荐类型 |
|-------------|---------|
| 规则的三维阵列（如CT扫描） | `vtkImageData` |
| 各轴间距不同但沿轴的网格 | `vtkRectilinearGrid` |
| 规则的逻辑结构但几何弯曲 | `vtkStructuredGrid` |
| 三角面片、线段、点云 | `vtkPolyData` |
| 混合单元类型（四面体+六面体+...） | `vtkUnstructuredGrid` |

---

## 3.3 Points与Cells的基本概念

### 3.3.1 Point：3D空间中的坐标点

在VTK中，一个**Point（点）** 就是一个三维空间中的坐标，由 `(x, y, z)` 三个浮点值组成。点是构成所有几何数据集的基石。

```cpp
// 创建一个点
double point[3] = {1.0, 2.5, -3.0};
```

要点：
- 点是纯粹的位置信息，没有"大小"
- 点在数据集内部以数组形式存储（`vtkPoints`），每个点有一个隐式的整数ID（从0开始）
- 点是单元（Cell）的构建块——单元并不包含点坐标，而只包含点ID

### 3.3.2 Cell：由点索引定义的几何体素

一个**Cell（单元）** 是由一个有序的点ID列表定义的几何体素。每个单元有：

- **Cell Type**：一个枚举值，标识单元的种类（三角形、四面体等）
- **Point IDs**：一个有序的整数列表，每个整数是数据集点数组中某个点的索引
- **拓扑 (Topology)**：点之间的连接关系（即"哪些点连在一起"）
- **几何 (Geometry)**：由连接点的实际坐标决定的形状

### 3.3.3 常用单元类型

VTK支持超过20种单元类型。下面列出最常用的九种：

#### 0维单元

```
VTK_VERTEX (1)
    *p0                          单个点。用于点云可视化。
```

#### 1维单元

```
VTK_LINE (3)
    p0 *---------* p1             由两个端点定义的线段。
    p0, p1

VTK_POLY_LINE (4)
    p0 *---* p1---* p2---* p3    由N个点定义的折线段。
    p0, p1, p2, ..., pN-1
```

#### 2维单元

```
VTK_TRIANGLE (5)
        p0
        /\
       /  \
      /    \
  p1 *------* p2               由三个点定义的三角形。
    p0, p1, p2

VTK_QUAD (9)
    p0 *---------* p3
       |         |
       |         |
    p1 *---------* p2           由四个点定义的四边四边形。
    p0, p1, p2, p3

VTK_POLYGON (7)
    p0 *-------* p3
       |       |
       |       |
    p1 *-------* p2            由N个共面点定义的凸多边形。
    p0, p1, p2, ..., pN-1
```

#### 3维单元

```
VTK_TETRA (10)
           p0
           /|\
          / | \
         /  |  \
     p1 *---+---* p2
         \  |  /
          \ | /
           \|/
           p3                 由四个点定义的四面体。
    p0, p1, p2, p3

VTK_HEXAHEDRON (12)
        p4 *---------* p7
          /|        /|
         / |       / |
    p5  *--+------*p6|
        |  *------|--*
        | / p0    | / p3
        |/        |/
    p1  *---------* p2         由八个点定义的六面体（非体素顺序）。
    p0, p1, p2, p3, p4, p5, p6, p7

VTK_VOXEL (11)
        p4 *---------* p5
          /|        /|
         / |       / |
    p6  *--+------*p7|
        |  *------|--*
        | / p0    | / p1
        |/        |/
    p2  *---------* p3         也是六面体，但点顺序不同（像像素/体素排列）。
    p0, p1, p2, p3, p4, p5, p6, p7

VTK_WEDGE (13)
        p0
        /|\
       / | \
      /  |  \
  p1 *---+---* p2
      |  /|\  |
      | / | \ |
      |/  |  \|
  p3  *---+---* p4             由六个点定义的楔形体（三棱柱）。
    p0, p1, p2, p3, p4, p5

VTK_PYRAMID (14)
           p0
          /|\
         / | \
        /  |  \
    p1 *---+---* p2
        \  |  /
         \ | /
          \|/
          p3                   由五个点定义的金字塔形体。
    p0, p1, p2, p3, p4
      (p4在底面中心)
```

### 3.3.4 单元拓扑 vs 单元几何

理解**拓扑 (Topology)** 和**几何 (Geometry)** 的区别对于有效使用VTK至关重要：

| 概念 | 定义 | VTK中的表示 |
|------|------|------------|
| **拓扑** | 点之间的连接关系 | 单元类型 + 点ID列表（整数） |
| **几何** | 点在空间中的实际位置 | 点坐标 `(x, y, z)` |

**为什么这种分离很重要？**

某个四面体单元的拓扑是 `{p0, p1, p2, p3}`，这是一个不变的关系。但如果网格变形了（比如在结构分析中受力弯曲），这四个点的坐标会变化——几何变了，但拓扑不变。

这种分离允许VTK：
- 高效地共享点（多个单元可以引用同一个点）
- 只更新几何信息而不改变拓扑（如动画变形）
- 对同样的拓扑在不同几何上执行操作

### 3.3.5 点索引的工作方式

单元不直接存储坐标，而是存储到点列表的整数索引。

```
  vtkPoints:
    [0]: (0.0, 0.0, 0.0)
    [1]: (1.0, 0.0, 0.0)
    [2]: (0.0, 1.0, 0.0)
    [3]: (1.0, 1.0, 0.0)
    [4]: (0.5, 0.5, 1.0)

  一个四面体单元（VTK_TETRA）:
    类型: 10  (VTK_TETRA)
    点列表: {0, 1, 2, 4}

  解释：
  - 点0是(0,0,0), 点1是(1,0,0), 点2是(0,1,0), 点4是(0.5,0.5,1.0)
  - 这个四面体的四个顶点就是这四个点
  - 如果点4的坐标变为(0.6, 0.6, 1.2)，四面体的形状会跟着变化
    但单元本身的定义（点列表 {0,1,2,4}）不变
```

---

## 3.4 属性数据（Attribute Data）

### 3.4.1 vtkPointData 与 vtkCellData

属性数据是通过 `vtkPointData` 和 `vtkCellData`（两者都是 `vtkDataSetAttributes` 的子类）来附加到数据集上的。它们是数组的容器：

- **`vtkPointData`**：每个数组的元素数量等于数据集的点数。每个数组的元素i对应第i个点。
- **`vtkCellData`**：每个数组的元素数量等于数据集的单元数。每个数组的元素i对应第i个单元。

```cpp
vtkSmartPointer<vtkPolyData> data = ...;  // 某个多边形数据

// 获取点数据
vtkPointData* pointData = data->GetPointData();   // 点数据容器
vtkCellData*  cellData  = data->GetCellData();    // 单元数据容器
```

### 3.4.2 属性类型

VTK定义了五种特殊的属性类型，通过名称识别：

| 属性类型 | 名称约定 | 组件数 | 说明 |
|----------|---------|--------|------|
| **Scalars** | `"Scalars"` | 1 | 单值标量，如温度、压力、密度 |
| **Vectors** | `"Vectors"` | 3 | 三维矢量，如速度、位移、梯度 |
| **Normals** | `"Normals"` | 3 | 曲面法向量，用于光照计算 |
| **Texture Coordinates** | `"TCoords"`| 1-3 | 纹理坐标，用于纹理映射 |
| **Tensors** | `"Tensors"` | 9 (3x3) | 对称张量，如应力、应变张量 |

**命名约定的重要性：**

VTK的许多过滤器、Mapper和渲染器通过名称来查找属性数据。例如：

- `vtkWarpVector` 过滤器默认寻找名为 `"Vectors"` 的数组来变形几何体
- `vtkPolyDataMapper` 默认使用名为 `"Scalars"` 的数组来进行颜色映射
- 光照计算需要名为 `"Normals"` 的数组

你可以设置某个数组为"活跃"属性：

```cpp
// 将名为 "Temperature" 的数组设置为活跃的标量属性
pointData->SetActiveScalars("Temperature");

// 将名为 "Velocity" 的数组设置为活跃的矢量属性
pointData->SetActiveVectors("Velocity");
```

### 3.4.3 属性数据在管线中的流转

属性数据沿管线传播，遵循以下规则：

1. **Source** 生成数据集时，通常会生成初始的属性数据（如 `vtkConeSource` 会生成法线）。
2. **Filter** 处理数据时，会尝试将输入属性传递到输出。如果过滤器生成新的几何体（如 `vtkClipFilter` 剪切单元），它会尽可能在输出上插值或映射原有的属性。
3. **Mapper** 使用活跃的属性来执行渲染（颜色映射、变形、光照）。

```
  Source                 Filter                  Mapper
  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │生成几何体 │ ────→  │处理几何体 │ ────→  │渲染属性   │
  │+ 点/单元属性│       │传播属性   │         │用于着色等  │
  └──────────┘         └──────────┘         └──────────┘
```

### 3.4.4 vtkDataArray 及其子类

`vtkDataArray` 是所有属性数据数组的抽象基类。它的主要子类包括：

```cpp
vtkDataArray
  ├── vtkFloatArray      // float 数组
  ├── vtkDoubleArray     // double 数组
  ├── vtkIntArray        // int 数组
  ├── vtkUnsignedCharArray // unsigned char 数组
  ├── vtkShortArray      // short 数组
  └── vtkBitArray        // 位数组（按位存储Bool值）
```

创建和使用数组：

```cpp
// 创建一个 double 类型的标量数组
vtkSmartPointer<vtkDoubleArray> scalars =
    vtkSmartPointer<vtkDoubleArray>::New();
scalars->SetName("Temperature");
scalars->SetNumberOfComponents(1);  // 1 = 标量
scalars->SetNumberOfTuples(numPoints);

// 填充数据
for (vtkIdType i = 0; i < numPoints; ++i)
{
    scalars->SetValue(i, someFunction(x, y, z));
}

// 附加到点数据
polyData->GetPointData()->SetScalars(scalars);
```

### 3.4.5 访问属性数据的常用方法

```cpp
vtkSmartPointer<vtkDataSet> data = ...;

// ------ Point Data ------

// 获取活跃的标量数组
vtkDataArray* pointScalars = data->GetPointData()->GetScalars();

// 获取活跃的矢量数组
vtkDataArray* pointVectors = data->GetPointData()->GetVectors();

// 获取活跃的法线数组
vtkDataArray* pointNormals = data->GetPointData()->GetNormals();

// 获取活跃的纹理坐标数组
vtkDataArray* pointTCoords = data->GetPointData()->GetTCoords();

// 获取活跃的张量数组
vtkDataArray* pointTensors = data->GetPointData()->GetTensors();

// 按名称获取任意数组
vtkDataArray* customArray =
    data->GetPointData()->GetArray("Displacement");

// 获取点数据中的数组数量
int numArrays = data->GetPointData()->GetNumberOfArrays();

// 遍历所有数组
for (int i = 0; i < numArrays; ++i)
{
    vtkDataArray* arr = data->GetPointData()->GetArray(i);
    std::cout << "Array " << i << ": " << arr->GetName()
              << " (" << arr->GetNumberOfComponents() << " comps)"
              << std::endl;
}

// ------ Cell Data ------

// 同样的模式
vtkDataArray* cellScalars = data->GetCellData()->GetScalars();
vtkDataArray* cellVectors = data->GetCellData()->GetVectors();
vtkDataArray* cellArray   = data->GetCellData()->GetArray("Stress");
```

---

## 3.5 代码示例：探索vtkDataSet

下面是一个完整的C++程序，它创建一个 `vtkConeSource`，提取输出的 `vtkPolyData`，并详细打印数据集的内部结构。

### 3.5.1 源代码

```cpp
// explore_dataset.cxx
// 探索 vtkDataSet 的内部结构
// 使用 vtkConeSource 生成数据，然后遍历点、单元和属性数据

#include <vtkSmartPointer.h>
#include <vtkConeSource.h>
#include <vtkPolyData.h>
#include <vtkPointData.h>
#include <vtkCellData.h>
#include <vtkDataArray.h>
#include <vtkPoints.h>
#include <vtkCellArray.h>
#include <vtkCellTypes.h>
#include <iostream>
#include <iomanip>
#include <string>

// 将单元类型枚举值转换为可读的字符串
std::string CellTypeToString(int cellType)
{
    switch (cellType)
    {
        case VTK_VERTEX:     return "VTK_VERTEX";
        case VTK_POLY_VERTEX:return "VTK_POLY_VERTEX";
        case VTK_LINE:       return "VTK_LINE";
        case VTK_POLY_LINE:  return "VTK_POLY_LINE";
        case VTK_TRIANGLE:   return "VTK_TRIANGLE";
        case VTK_TRIANGLE_STRIP: return "VTK_TRIANGLE_STRIP";
        case VTK_POLYGON:    return "VTK_POLYGON";
        case VTK_PIXEL:      return "VTK_PIXEL";
        case VTK_QUAD:       return "VTK_QUAD";
        case VTK_TETRA:      return "VTK_TETRA";
        case VTK_VOXEL:      return "VTK_VOXEL";
        case VTK_HEXAHEDRON: return "VTK_HEXAHEDRON";
        case VTK_WEDGE:      return "VTK_WEDGE";
        case VTK_PYRAMID:    return "VTK_PYRAMID";
        default:             return "UNKNOWN";
    }
}

// 打印数据集的基本信息
void PrintDataSetInfo(vtkDataSet* data, const std::string& name)
{
    std::cout << "\n========== " << name << " ==========\n";

    vtkIdType numPoints = data->GetNumberOfPoints();
    vtkIdType numCells  = data->GetNumberOfCells();

    std::cout << "Number of Points: " << numPoints << "\n";
    std::cout << "Number of Cells:  " << numCells  << "\n";

    // 打印数据集的范围
    double bounds[6];
    data->GetBounds(bounds);
    std::cout << "Bounds:\n";
    std::cout << "  X: [" << bounds[0] << ", " << bounds[1] << "]\n";
    std::cout << "  Y: [" << bounds[2] << ", " << bounds[3] << "]\n";
    std::cout << "  Z: [" << bounds[4] << ", " << bounds[5] << "]\n";
}

// 打印点数据（属性数组）
void PrintPointData(vtkDataSet* data)
{
    vtkPointData* pd = data->GetPointData();
    if (!pd) return;

    int numArrays = pd->GetNumberOfArrays();
    std::cout << "\n--- Point Data (" << numArrays << " arrays) ---\n";

    for (int i = 0; i < numArrays; ++i)
    {
        vtkDataArray* arr = pd->GetArray(i);
        if (!arr) continue;

        std::cout << "  [" << i << "] Name: \"" << arr->GetName() << "\""
                  << ", Components: " << arr->GetNumberOfComponents()
                  << ", Tuples: " << arr->GetNumberOfTuples()
                  << ", Type: " << arr->GetDataTypeAsString()
                  << "\n";

        // 打印前5个元组（如果存在）
        vtkIdType numToPrint = std::min(arr->GetNumberOfTuples(),
                                        static_cast<vtkIdType>(5));
        if (arr->GetNumberOfComponents() == 1)
        {
            std::cout << "      First " << numToPrint << " values: ";
            for (vtkIdType j = 0; j < numToPrint; ++j)
            {
                double val[1];
                arr->GetTuple(j, val);
                std::cout << val[0] << " ";
            }
            std::cout << "\n";
        }
        else if (arr->GetNumberOfComponents() == 3)
        {
            std::cout << "      First " << numToPrint << " tuples:\n";
            for (vtkIdType j = 0; j < numToPrint; ++j)
            {
                double val[3];
                arr->GetTuple(j, val);
                std::cout << "        (" << val[0] << ", "
                          << val[1] << ", " << val[2] << ")\n";
            }
        }
    }
}

// 打印单元数据（属性数组）
void PrintCellData(vtkDataSet* data)
{
    vtkCellData* cd = data->GetCellData();
    if (!cd) return;

    int numArrays = cd->GetNumberOfArrays();
    std::cout << "\n--- Cell Data (" << numArrays << " arrays) ---\n";

    for (int i = 0; i < numArrays; ++i)
    {
        vtkDataArray* arr = cd->GetArray(i);
        if (!arr) continue;

        std::cout << "  [" << i << "] Name: \"" << arr->GetName() << "\""
                  << ", Components: " << arr->GetNumberOfComponents()
                  << ", Tuples: " << arr->GetNumberOfTuples()
                  << ", Type: " << arr->GetDataTypeAsString()
                  << "\n";
    }
}

// 打印前N个点的坐标
void PrintFirstNPoints(vtkDataSet* data, vtkIdType n)
{
    vtkPoints* points = data->GetPoints();
    if (!points) return;

    vtkIdType numToPrint = std::min(data->GetNumberOfPoints(), n);

    std::cout << "\n--- First " << numToPrint << " Point Coordinates ---\n";
    for (vtkIdType i = 0; i < numToPrint; ++i)
    {
        double pt[3];
        points->GetPoint(i, pt);
        std::cout << "  Point[" << i << "]: ("
                  << std::fixed << std::setprecision(4)
                  << pt[0] << ", " << pt[1] << ", " << pt[2] << ")\n";
    }
}

// 统计并打印单元类型分布
void PrintCellTypeStats(vtkDataSet* data)
{
    vtkIdType numCells = data->GetNumberOfCells();
    std::cout << "\n--- Cell Type Distribution ---\n";

    // 使用 std::map 统计每种单元类型的数量
    std::map<int, vtkIdType> typeCount;

    for (vtkIdType i = 0; i < numCells; ++i)
    {
        int ctype = data->GetCellType(i);
        typeCount[ctype]++;
    }

    for (const auto& entry : typeCount)
    {
        double percentage = 100.0 * entry.second / numCells;
        std::cout << "  " << CellTypeToString(entry.first)
                  << ": " << entry.second
                  << " (" << std::fixed << std::setprecision(1)
                  << percentage << "%)\n";
    }
}

// 打印前N个单元的详细信息
void PrintFirstNCells(vtkDataSet* data, vtkIdType n)
{
    vtkIdType numToPrint = std::min(data->GetNumberOfCells(), n);

    std::cout << "\n--- First " << numToPrint << " Cells ---\n";
    for (vtkIdType i = 0; i < numToPrint; ++i)
    {
        int ctype = data->GetCellType(i);
        vtkIdType npts;
        const vtkIdType* pts;

        // 通过 legacy 接口获取单元的连接关系
        vtkCell* cell = data->GetCell(i);
        npts = cell->GetNumberOfPoints();
        pts  = cell->GetPointIds()->GetPointer(0);

        std::cout << "  Cell[" << i << "]: "
                  << CellTypeToString(ctype)
                  << ", " << npts << " points: {";
        for (vtkIdType j = 0; j < npts; ++j)
        {
            std::cout << pts[j];
            if (j < npts - 1) std::cout << ", ";
        }
        std::cout << "}\n";
    }
}

int main()
{
    // 1. 创建 Cone Source
    vtkSmartPointer<vtkConeSource> coneSource =
        vtkSmartPointer<vtkConeSource>::New();
    coneSource->SetResolution(6);          // 6个侧面分段
    coneSource->SetHeight(3.0);
    coneSource->SetRadius(1.0);
    coneSource->Update();

    // 2. 获取输出数据集
    vtkPolyData* output = coneSource->GetOutput();

    // 3. 打印基本信息
    PrintDataSetInfo(output, "vtkConeSource Output");

    // 4. 打印前若干点的坐标
    PrintFirstNPoints(output, 5);

    // 5. 打印单元类型统计
    PrintCellTypeStats(output);

    // 6. 打印前若干个单元的详情
    PrintFirstNCells(output, 5);

    // 7. 打印点上的属性数据
    PrintPointData(output);

    // 8. 打印单元上的属性数据
    PrintCellData(output);

    std::cout << "\n========== End of Exploration ==========\n";
    return 0;
}
```

### 3.5.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.12)
project(ExploreDataSet)

# 查找 VTK
find_package(VTK REQUIRED COMPONENTS
    CommonCore
    CommonDataModel
    FiltersSources
)

# 包含 VTK 目录
include(${VTK_USE_FILE})

# 创建可执行文件
add_executable(explore_dataset explore_dataset.cxx)

# 链接 VTK 库
target_link_libraries(explore_dataset
    VTK::CommonCore
    VTK::CommonDataModel
    VTK::FiltersSources
)
```

### 3.5.3 预期输出

运行该程序后，你将看到类似以下的输出（具体数值因版本略有差异）：

```
========== vtkConeSource Output ==========
Number of Points: 7
Number of Cells:  13
Bounds:
  X: [-1.0000, 1.0000]
  Y: [-1.0000, 1.0000]
  Z: [0.0000, 3.0000]

--- First 5 Point Coordinates ---
  Point[0]: ( 1.0000,  0.0000, 0.0000)
  Point[1]: ( 0.5000,  0.8660, 0.0000)
  Point[2]: (-0.5000,  0.8660, 0.0000)
  Point[3]: (-1.0000,  0.0000, 0.0000)
  Point[4]: (-0.5000, -0.8660, 0.0000)

--- Cell Type Distribution ---
  VTK_TRIANGLE: 6 (46.2%)
  VTK_LINE: 6 (46.2%)
  VTK_VERTEX: 1 (7.7%)

--- First 5 Cells ---
  Cell[0]: VTK_TRIANGLE, 3 points: {1, 0, 6}
  Cell[1]: VTK_TRIANGLE, 3 points: {6, 0, 5}
  ...

--- Point Data (1 arrays) ---
  [0] Name: "Normals", Components: 3, Tuples: 7, Type: float
      First 5 tuples:
        (0.3446, 0.0000, -0.3446)
        ...

========== End of Exploration ==========
```

这个示例演示了如何以编程方式检查VTK数据集的完整内部结构——点、单元、类型和所有关联的属性数据。

---

## 3.6 代码示例：手动构建简单数据集

前面的示例展示了如何读取和检查已有数据集。本节演示如何从头开始**手动构建**一个 `vtkPolyData`，并将其渲染出来。这是理解VTK数据模型的最直接方式。

### 3.6.1 问题描述

我们将手动构建一个由三个三角形组成的简单"山峰"形状，并赋予每个顶点一个标量值（模拟高程值用于着色），然后渲染它。

```
       p1 (0.0, 1.0, 0.0)
        /\
       /  \
      /    \
     /  t0  \
    /        \
p0 *----------* p2
(0.0,0.0,0.0)  (1.0,0.0,0.0)

       p1
        /\
       /  \
      /    \
     /  t1  \
    /        \
p2 *----------* p3
(1.0,0.0,0.0)  (2.0,0.0,0.0)

       p1
        /\
       /  \
      /    \
     /  t2  \
    /        \
p3 *----------* p4
(2.0,0.0,0.0)  (3.0,0.0,0.0)
```

### 3.6.2 源代码

```cpp
// manual_polydata.cxx
// 手动构建 vtkPolyData：创建点、单元和属性数据，并渲染

#include <vtkSmartPointer.h>
#include <vtkPoints.h>
#include <vtkCellArray.h>
#include <vtkPolyData.h>
#include <vtkDoubleArray.h>
#include <vtkPointData.h>
#include <vtkPolyDataMapper.h>
#include <vtkActor.h>
#include <vtkRenderer.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkNamedColors.h>
#include <vtkProperty.h>
#include <vtkScalarBarActor.h>
#include <vtkLookupTable.h>

int main()
{
    // ============================================================
    // 第1步：创建点坐标
    // ============================================================
    vtkSmartPointer<vtkPoints> points =
        vtkSmartPointer<vtkPoints>::New();

    // 插入5个点。每个点的ID（从0开始）由插入顺序决定。
    points->InsertNextPoint(0.0, 0.0, 0.0);   // Point 0
    points->InsertNextPoint(0.0, 1.0, 0.0);   // Point 1 (山峰顶点)
    points->InsertNextPoint(1.0, 0.0, 0.0);   // Point 2
    points->InsertNextPoint(2.0, 0.0, 0.0);   // Point 3
    points->InsertNextPoint(3.0, 0.0, 0.0);   // Point 4

    std::cout << "Created " << points->GetNumberOfPoints()
              << " points.\n";

    // ============================================================
    // 第2步：创建单元（三角形）
    // ============================================================
    vtkSmartPointer<vtkCellArray> triangles =
        vtkSmartPointer<vtkCellArray>::New();

    // 三角形0：顶点(p0, p1, p2)
    vtkSmartPointer<vtkTriangle> triangle0 =
        vtkSmartPointer<vtkTriangle>::New();
    triangle0->GetPointIds()->SetId(0, 0);  // 第1个顶点用Point 0
    triangle0->GetPointIds()->SetId(1, 1);  // 第2个顶点用Point 1
    triangle0->GetPointIds()->SetId(2, 2);  // 第3个顶点用Point 2
    triangles->InsertNextCell(triangle0);

    // 三角形1：顶点(p1, p2, p3)
    vtkSmartPointer<vtkTriangle> triangle1 =
        vtkSmartPointer<vtkTriangle>::New();
    triangle1->GetPointIds()->SetId(0, 1);  // Point 1
    triangle1->GetPointIds()->SetId(1, 2);  // Point 2
    triangle1->GetPointIds()->SetId(2, 3);  // Point 3
    triangles->InsertNextCell(triangle1);

    // 三角形2：顶点(p1, p3, p4)
    vtkSmartPointer<vtkTriangle> triangle2 =
        vtkSmartPointer<vtkTriangle>::New();
    triangle2->GetPointIds()->SetId(0, 1);  // Point 1
    triangle2->GetPointIds()->SetId(1, 3);  // Point 3
    triangle2->GetPointIds()->SetId(2, 4);  // Point 4
    triangles->InsertNextCell(triangle2);

    std::cout << "Created " << triangles->GetNumberOfCells()
              << " triangles.\n";

    // ============================================================
    // 第3步：组建 vtkPolyData
    // ============================================================
    vtkSmartPointer<vtkPolyData> polyData =
        vtkSmartPointer<vtkPolyData>::New();

    // 设置点
    polyData->SetPoints(points);

    // 设置多边形单元（三角形）
    polyData->SetPolys(triangles);

    // ============================================================
    // 第4步：创建点属性数据（标量值）
    // ============================================================
    vtkSmartPointer<vtkDoubleArray> elevation =
        vtkSmartPointer<vtkDoubleArray>::New();
    elevation->SetName("Elevation");
    elevation->SetNumberOfComponents(1);      // 标量 = 1个分量
    elevation->SetNumberOfTuples(points->GetNumberOfPoints()); // 每个点一个值

    // 给每个点赋值——模拟"高度"
    // 山峰顶点(p1)最高，地面点(p0, p2, p3, p4) = 0
    elevation->SetValue(0, 0.0);  // p0
    elevation->SetValue(1, 1.0);  // p1 (山峰顶点)
    elevation->SetValue(2, 0.0);  // p2
    elevation->SetValue(3, 0.0);  // p3
    elevation->SetValue(4, 0.0);  // p4

    // 将标量数组附加到点数据
    polyData->GetPointData()->SetScalars(elevation);

    // ============================================================
    // 第5步：验证数据并打印信息
    // ============================================================
    std::cout << "\n=== Dataset Summary ===\n";
    std::cout << "Points:  " << polyData->GetNumberOfPoints()  << "\n";
    std::cout << "Cells:   " << polyData->GetNumberOfCells()   << "\n";

    double bounds[6];
    polyData->GetBounds(bounds);
    std::cout << "Bounds:  X[" << bounds[0] << ", " << bounds[1]
              << "] Y[" << bounds[2] << ", " << bounds[3]
              << "] Z[" << bounds[4] << ", " << bounds[5] << "]\n";

    // 打印每个点的坐标和标量值
    std::cout << "\n--- Points and Scalars ---\n";
    for (vtkIdType i = 0; i < polyData->GetNumberOfPoints(); ++i)
    {
        double pt[3];
        polyData->GetPoint(i, pt);
        double val = elevation->GetValue(i);
        std::cout << "  Point[" << i << "]: ("
                  << pt[0] << ", " << pt[1] << ", " << pt[2]
                  << ")  Elevation = " << val << "\n";
    }

    // 打印每个单元
    std::cout << "\n--- Cells ---\n";
    for (vtkIdType i = 0; i < polyData->GetNumberOfCells(); ++i)
    {
        vtkCell* cell = polyData->GetCell(i);
        vtkIdType npts = cell->GetNumberOfPoints();
        std::cout << "  Cell[" << i << "]: "
                  << cell->GetCellType() << " (VTK_TRIANGLE), "
                  << npts << " points: {";
        for (vtkIdType j = 0; j < npts; ++j)
        {
            std::cout << cell->GetPointId(j);
            if (j < npts - 1) std::cout << ", ";
        }
        std::cout << "}\n";
    }

    // ============================================================
    // 第6步：创建渲染管线
    // ============================================================
    // Mapper
    vtkSmartPointer<vtkPolyDataMapper> mapper =
        vtkSmartPointer<vtkPolyDataMapper>::New();
    mapper->SetInputData(polyData);
    mapper->SetScalarRange(0.0, 1.0);  // 标量值的范围

    // Actor
    vtkSmartPointer<vtkActor> actor =
        vtkSmartPointer<vtkActor>::New();
    actor->SetMapper(mapper);
    actor->GetProperty()->SetEdgeVisibility(true);  // 显示单元边线
    actor->GetProperty()->SetEdgeColor(0.0, 0.0, 0.0);
    actor->GetProperty()->SetLineWidth(2.0);

    // 设置颜色映射
    vtkSmartPointer<vtkLookupTable> lut =
        vtkSmartPointer<vtkLookupTable>::New();
    lut->SetNumberOfTableValues(256);
    lut->SetRange(0.0, 1.0);
    lut->SetHueRange(0.667, 0.0);  // 蓝色(低)到红色(高)
    lut->Build();
    mapper->SetLookupTable(lut);

    // 标量条
    vtkSmartPointer<vtkScalarBarActor> scalarBar =
        vtkSmartPointer<vtkScalarBarActor>::New();
    scalarBar->SetLookupTable(lut);
    scalarBar->SetTitle("Elevation");
    scalarBar->SetNumberOfLabels(5);

    // Renderer
    vtkSmartPointer<vtkRenderer> renderer =
        vtkSmartPointer<vtkRenderer>::New();
    renderer->AddActor(actor);
    renderer->AddActor2D(scalarBar);
    renderer->SetBackground(0.95, 0.95, 0.95);

    // RenderWindow
    vtkSmartPointer<vtkRenderWindow> renderWindow =
        vtkSmartPointer<vtkRenderWindow>::New();
    renderWindow->AddRenderer(renderer);
    renderWindow->SetSize(800, 600);
    renderWindow->SetWindowName("Manually Constructed vtkPolyData");

    // Interactor
    vtkSmartPointer<vtkRenderWindowInteractor> interactor =
        vtkSmartPointer<vtkRenderWindowInteractor>::New();
    interactor->SetRenderWindow(renderWindow);

    // 渲染并开始交互
    renderWindow->Render();
    std::cout << "\n=== Rendering Started ===\n";
    std::cout << "Interact with the window. Press 'q' or close the window to exit.\n";
    interactor->Start();

    return 0;
}
```

### 3.6.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.12)
project(ManualPolyData)

# 查找 VTK
find_package(VTK REQUIRED COMPONENTS
    CommonCore
    CommonDataModel
    CommonColor
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
)

# 包含 VTK 目录
include(${VTK_USE_FILE})

# 创建可执行文件
add_executable(manual_polydata manual_polydata.cxx)

# 链接 VTK 库
target_link_libraries(manual_polydata
    VTK::CommonCore
    VTK::CommonDataModel
    VTK::CommonColor
    VTK::InteractionStyle
    VTK::RenderingCore
    VTK::RenderingOpenGL2
)
```

### 3.6.4 代码解析

这个示例演示了手动构造 `vtkPolyData` 的完整过程，分为六个关键步骤：

**步骤1 — 创建点坐标：**
使用 `vtkPoints::InsertNextPoint(x, y, z)` 逐个添加点。每个点的索引由插入顺序自动分配（从0开始）。

**步骤2 — 创建单元：**
每个 `vtkTriangle` 通过 `GetPointIds()->SetId(localIndex, globalPointId)` 来引用点列表中的点。`localIndex` 是三角形内部顶点编号（0, 1, 2），`globalPointId` 是 `vtkPoints` 中的索引。所有三角形插入到一个 `vtkCellArray` 中。

**步骤3 — 组建 vtkPolyData：**
通过 `SetPoints()` 和 `SetPolys()` 将点和单元关联到数据集。

**步骤4 — 创建属性数据：**
创建 `vtkDoubleArray`，为每个点设置标量值，然后通过 `GetPointData()->SetScalars()` 附加到点数据上。VTK Mapper 会自动使用这个活跃标量数组进行颜色映射。

**步骤5 — 验证数据：**
遍历点和单元并打印信息，确保数据按预期构建。这是调试时的好习惯。

**步骤6 — 可视化管线：**
建立标准的 Mapper → Actor → Renderer → RenderWindow → Interactor 管线。设置颜色映射表 (`vtkLookupTable`) 和标量条 (`vtkScalarBarActor`) 来增强可视化效果。

### 3.6.5 运行结果

运行后你将看到一个渲染窗口，显示三个三角形组成的"山峰"形状：

- 山峰顶点（point 1，Y=1.0）显示为红色（标量值=1.0，高程最大）
- 底面四个顶点显示为蓝色（标量值=0.0，地面高程）
- 三角形内部颜色通过线性插值自动过渡
- 所有单元的边线以黑色显示（启用了 EdgeVisibility）
- 左侧显示高程标量条

---

## 3.7 本章小结

本章深入介绍了VTK的数据模型基础，这是后续学习和使用VTK的核心知识。让我们回顾关键要点：

1. **数据模型根基**：`vtkDataObject` 是所有数据类型的基类。数据对象只存储数据，算法对象（`vtkAlgorithm` 的子类）执行处理。理解这种分离是掌握VTK架构的关键。

2. **五种主要数据集类型**：
   - `vtkImageData`：均匀规则网格，最节省内存
   - `vtkRectilinearGrid`：轴对齐但间距可变的网格
   - `vtkStructuredGrid`：拓扑规则但几何任意的曲线网格
   - `vtkPolyData`：最灵活的表面数据类型（点/线/面/三角带）
   - `vtkUnstructuredGrid`：支持任意单元类型的通用网格

3. **Points 和 Cells**：点是3D坐标，单元是由点索引（而非坐标）定义的几何体素。单元类型决定拓扑（连接方式），点坐标决定几何（空间形状）。这种拓扑-几何分离是VTK数据模型的一个核心设计。

4. **属性数据**：通过 `vtkPointData` 和 `vtkCellData` 附加到数据集。五种特殊属性（Scalars、Vectors、Normals、TCoords、Tensors）是VTK过滤器链和渲染系统的"通用语言"。`vtkDataArray` 及其子类提供了类型化的数组存储。

5. **编程接口**：VTK提供了丰富的API来检查和操作数据集。关键方法包括获取点和单元数量、遍历单元类型、访问属性数组等。两个代码示例分别演示了"读取已有数据集"和"从头构建数据集"的实践。

掌握了本章的数据模型知识后，你已经具备了理解VTK中几乎所有数据处理操作的基础。在第4章中，我们将学习如何使用各种过滤器（Filter）来处理和转换这些数据集。

---

*本章完成。建议读者动手运行3.5节和3.6节的代码示例，通过实践加深对数据模型的理解。*
