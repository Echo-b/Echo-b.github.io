# 第九章 结构化数据（Structured Data）

## 本章导读

你在第七章学习了`vtkPolyData`--VTK中最灵活、最通用的多边形数据表示。在第八章学习了`vtkUnstructuredGrid`--可以表达任意拓扑的非结构化网格。现在，你将遇到VTK数据模型中的另外三个重要成员：**vtkImageData**（均匀网格）、**vtkRectilinearGrid**（直线网格）和**vtkStructuredGrid**（结构化网格）。

这三者统称为**结构化数据**（Structured Data），它们的共同特征是：**拓扑结构是规则的三维IJK索引格**。也就是说，每个网格点的邻居关系由它在三维索引空间中的位置隐式确定，而不需要显式地存储连接关系（Connectivity）。这一特性使得结构化数据在内存效率和访问速度上远超非结构化数据，同时在许多科学计算领域（医学影像、计算流体力学、气象学等）中占据主导地位。

本章的核心目标有三个：

1. **理解三种结构化数据的本质区别**--它们的间距（Spacing）和几何自由度逐步递增，各自的适用场景也不同。
2. **掌握每种数据类型的创建和使用方法**--从最简单的Uniform Grid到复杂的Curvilinear Grid，逐步深入。
3. **通过一个综合实例对比三种数据类型**--在同一窗口中并排展示三种网格，使你能直观感受它们的几何差异。

读完本章后，你将能够根据手头数据的特性，准确判断应该使用哪种VTK数据结构，并熟练地创建和操作它们。这也将为你后续学习体渲染（Volume Rendering）、等值面提取（Isosurface Extraction）、流线可视化（Streamline Visualization）等技术奠定基础。

---

## 9.1 三种结构化数据概览

### 9.1.1 结构化数据的共同特征

所有结构化数据都共享以下关键属性：

- **拓扑隐式性（Implicit Topology）**：网格点的连接关系不需要显式存储。给定一个网格点索引(i, j, k)，它的6个邻居（如果存在）就是(i-1,j,k)、(i+1,j,k)、(i,j-1,k)、(i,j+1,k)、(i,j,k-1)、(i,j,k+1)。单元也由此自然定义--每8个相邻的网格点围成一个体素（Voxel）或六面体单元。

- **IJK索引空间（IJK Index Space）**：所有点的位置由三个维度的整型索引(i, j, k)来描述。i沿X方向增长、j沿Y方向增长、k沿Z方向增长。

- **逻辑矩形区域**：在一个给定维度上，索引范围是连续的整数区间，不存在间隙或跳跃。例如i从0到nx-1，共nx个采样点。

### 9.1.2 三种类型的关键差异

尽管拓扑相同，三种结构化数据类型在几何表示上有着本质区别：

| 特性 | vtkImageData | vtkRectilinearGrid | vtkStructuredGrid |
|------|-------------|-------------------|------------------|
| **中文名称** | 均匀网格/图像数据 | 直线网格/矩形网格 | 结构化网格/曲线网格 |
| **间距** | 各维度均匀（Uniform） | 各维度内可非均匀，但轴对齐 | 完全任意，无间距概念 |
| **定义方式** | Origin + Spacing + Extent | X/Y/Z三个坐标数组 + Extent | Points数组中所有点的显式坐标 |
| **内存占用** | 最小（不存储坐标） | 中等（只存三个轴坐标） | 最大（存储全部点坐标） |
| **几何自由度** | 最低 | 中等 | 最高 |
| **典型应用** | CT/MRI体数据、位图、规则网格仿真 | CFD拉伸网格、海洋模型变厚度层、对数间距网格 | CFD贴体网格、变形几何、地形跟随网格 |
| **点的物理坐标计算** | origin + index * spacing | (x_coords[i], y_coords[j], z_coords[k]) | points->GetPoint(idx) |
| **单元类型** | 体素(Voxel)或像素(Pixel) | 体素(Voxel) | 六面体(Hexahedron)，可能变形 |

### 9.1.3 如何选择

选择哪种数据结构，取决于你的数据是如何在空间中分布的：

1. **如果各方向的采样间距完全相等（或成固定比例），且网格轴与坐标轴对齐** --> 使用`vtkImageData`。这是内存最高效的选择，因为你几乎不需要存储任何几何信息。

2. **如果每个方向上的间距不一致，但网格线仍然与坐标轴对齐（即点仍位于正交网格的交点上）** --> 使用`vtkRectilinearGrid`。例如，在CFD中边界层附近需要加密网格，而在远场可以稀疏；或者海洋模型中不同深度层的厚度不同。

3. **如果网格点的物理位置是任意指定的，不位于直线正交交点上** --> 使用`vtkStructuredGrid`。例如，沿机翼表面的贴体（body-fitted）网格，或者模拟大变形后的材料网格。

下面我们用一张示意图来理解三者在几何上的递进关系：

```
vtkImageData:               vtkRectilinearGrid:         vtkStructuredGrid:
(均匀间距)                  (非均匀但轴对齐)            (任意点位置)

+---+---+---+---+           +--+----+------+--+         +--+---+---+--+
|   |   |   |   |           |  |    |      |  |         |  |   |    \  \
+---+---+---+---+           +--+----+------+--+         |  |   |     \  \
|   |   |   |   |           |  |    |      |  |         |   \+--+---+--+
+---+---+---+---+           +--+----+------+--+         |   /|   |   |  |
                            |  |    |      |  |         +--+---+---+---+
                            +--+----+------+--+         \   \   \   \
                                                         +--+---+---+
```

---

## 9.2 vtkImageData（均匀网格）

### 9.2.1 数学定义

`vtkImageData`是VTK中最简单、内存效率最高的数据表示。它由三个参数完全定义：

1. **Origin（原点）**：索引(0, 0, 0)处网格点的物理坐标，记为`(ox, oy, oz)`。
2. **Spacing（间距）**：三个维度上相邻网格点之间的物理距离，记为`(dx, dy, dz)`。
3. **Extent（范围）**：六个整数的元组`(imin, imax, jmin, jmax, kmin, kmax)`，定义每个维度上的索引区间。

任意网格点(i, j, k)的物理坐标由以下公式计算：

```
px = ox + i * dx
py = oy + j * dy
pz = oz + k * dz
```

其中`imin <= i <= imax`，`jmin <= j <= jmax`，`kmin <= k <= kmax`。

### 9.2.2 Extent与Dimensions的关系

Extent和Dimensions（尺寸）是两个容易混淆但又密切相关的概念：

- **Extent**：6元组`(imin, imax, jmin, jmax, kmin, kmax)`，定义的是索引的**范围**（范围的两端都包含在内）。索引可以从任意整数开始（包括负数），这在处理分块数据时非常有用。

- **Dimensions**：3元组`(nx, ny, nz)`，定义的是每个维度上的**点数**。它们的关系是：

```
nx = imax - imin + 1
ny = jmax - jmin + 1
nz = kmax - kmin + 1
```

例如，Extent为`(0, 99, 0, 99, 0, 99)`表示每个维度上有100个点，共计`100 * 100 * 100 = 1,000,000`个点。

**点数 vs 单元数：**

| 维度 | 点数公式 | 单元数公式 |
|------|---------|-----------|
| 1D | nx | nx - 1 |
| 2D | nx * ny | (nx-1) * (ny-1) |
| 3D | nx * ny * nz | (nx-1) * (ny-1) * (nz-1) |

### 9.2.3 创建vtkImageData

VTK提供两种等价的创建方式：

**方式一：使用SetExtent() + SetOrigin() + SetSpacing()**

```cpp
auto imageData = vtkSmartPointer<vtkImageData>::New();
imageData->SetExtent(0, 99, 0, 99, 0, 99);  // 100x100x100 网格点
imageData->SetOrigin(0.0, 0.0, 0.0);         // 原点位于世界坐标原点
imageData->SetSpacing(0.1, 0.1, 0.1);        // 每格0.1单位，整体10x10x10
```

**方式二：使用SetDimensions()（会默认Extent从0开始）**

```cpp
auto imageData = vtkSmartPointer<vtkImageData>::New();
imageData->SetDimensions(100, 100, 100);      // 等价于 Extent(0,99, 0,99, 0,99)
imageData->SetOrigin(0.0, 0.0, 0.0);
imageData->SetSpacing(0.1, 0.1, 0.1);
```

### 9.2.4 访问和填充标量数据

要为`vtkImageData`填充标量，你需要先分配标量数组，然后通过索引写入数据。VTK在内存中以**K-最快（K-fastest）**的顺序存储数据，即k索引变化最快，j次之，i最慢。

```cpp
// 为ImageData分配一个双精度标量数组
vtkNew<vtkDoubleArray> scalars;
scalars->SetName("DistanceFromCenter");
scalars->SetNumberOfComponents(1);  // 单分量标量

int* dims = imageData->GetDimensions();
// 总点数 = dims[0] * dims[1] * dims[2] = 1,000,000
vtkIdType numPoints = dims[0] * dims[1] * dims[2];
scalars->SetNumberOfTuples(numPoints);

// 填充数据：计算每个点到中心的距离
double origin[3]; imageData->GetOrigin(origin);
double spacing[3]; imageData->GetSpacing(spacing);

for (int k = 0; k < dims[2]; ++k)
{
  for (int j = 0; j < dims[1]; ++j)
  {
    for (int i = 0; i < dims[0]; ++i)
    {
      double x = origin[0] + i * spacing[0];
      double y = origin[1] + j * spacing[1];
      double z = origin[2] + k * spacing[2];

      // 计算到网格中心的欧几里得距离
      double cx = origin[0] + (dims[0]-1) * spacing[0] / 2.0;
      double cy = origin[1] + (dims[1]-1) * spacing[1] / 2.0;
      double cz = origin[2] + (dims[2]-1) * spacing[2] / 2.0;

      double dist = std::sqrt(
        (x - cx) * (x - cx) +
        (y - cy) * (y - cy) +
        (z - cz) * (z - cz)
      );

      // 计算线性化的点ID
      // K-fastest顺序: id = i*(ny*nz) + j*nz + k
      vtkIdType pointId = (i * dims[1] + j) * dims[2] + k;
      scalars->SetTuple1(pointId, dist);
    }
  }
}

// 将标量数组关联到ImageData的点数据上
imageData->GetPointData()->SetScalars(scalars);
```

`K-fastest`内存布局的理解：

```
对于一个 3x3x3 的volume（extent 0..2,0..2,0..2），内存顺序为：

索引 (i,j,k)：
  (0,0,0), (0,0,1), (0,0,2),   -- i=0, j=0, k从0到2
  (0,1,0), (0,1,1), (0,1,2),   -- i=0, j=1, k从0到2
  (0,2,0), (0,2,1), (0,2,2),   -- i=0, j=2, k从0到2
  (1,0,0), (1,0,1), (1,0,2),   -- i=1, j=0, k从0到2
  ...
```

### 9.2.5 典型应用场景

- **医学影像**：CT扫描产生512x512xN的切片堆叠，每个体素具有均匀的X/Y间距（像素间距），Z间距等于切片厚度。
- **位图图像**：2D图像是ImageData在Z维度extent为(0,0)的特例。
- **规则网格仿真**：使用有限差分法（Finite Difference Method）求解偏微分方程时，计算域通常被离散化为均匀网格。
- **体渲染（Volume Rendering）**：ImageData是体渲染的基础数据结构，标量值被映射为颜色和不透明度。

### 9.2.6 基础代码示例：创建并可视化ImageData

```cpp
// 创建一个50x50x50的ImageData，填充球体距离场
vtkNew<vtkImageData> imageData;
imageData->SetExtent(0, 49, 0, 49, 0, 49);
imageData->SetOrigin(-5.0, -5.0, -5.0);
imageData->SetSpacing(10.0/49, 10.0/49, 10.0/49);

// 填充标量场
int* dims = imageData->GetDimensions();
double origin[3], spacing[3];
imageData->GetOrigin(origin);
imageData->GetSpacing(spacing);

vtkNew<vtkDoubleArray> distScalars;
distScalars->SetName("Distance");
distScalars->SetNumberOfTuples(dims[0] * dims[1] * dims[2]);

for (int i = 0; i < dims[0]; ++i)
{
  for (int j = 0; j < dims[1]; ++j)
  {
    for (int k = 0; k < dims[2]; ++k)
    {
      double x = origin[0] + i * spacing[0];
      double y = origin[1] + j * spacing[1];
      double z = origin[2] + k * spacing[2];
      double dist = std::sqrt(x*x + y*y + z*z);
      vtkIdType id = (i * dims[1] + j) * dims[2] + k;
      distScalars->SetTuple1(id, dist);
    }
  }
}
imageData->GetPointData()->SetScalars(distScalars);

// 提取等值面来可视化（因为ImageData本身无法直接渲染）
vtkNew<vtkContourFilter> contour;
contour->SetInputData(imageData);
contour->SetValue(0, 2.5);  // 提取距离中心2.5单位的等值面
contour->Update();           // 输出变为vtkPolyData

vtkNew<vtkPolyDataMapper> mapper;
mapper->SetInputConnection(contour->GetOutputPort());
mapper->ScalarVisibilityOff();

vtkNew<vtkActor> actor;
actor->SetMapper(mapper);
actor->GetProperty()->SetColor(0.3, 0.6, 1.0);
```

---

## 9.3 vtkRectilinearGrid（直线网格）

### 9.3.1 数学定义

`vtkRectilinearGrid`放松了`vtkImageData`的"均匀间距"约束，但仍保持"轴对齐"（axis-aligned）的特性。它通过三个独立的坐标数组来定义几何：

- **X坐标数组**：`x_coords[i]`，其中`i`的范围为`[imin, imax]`
- **Y坐标数组**：`y_coords[j]`，其中`j`的范围为`[jmin, jmax]`
- **Z坐标数组**：`z_coords[k]`，其中`k`的范围为`[kmin, kmax]`

任意网格点(i, j, k)的物理坐标为：

```
px = x_coords[i]
py = y_coords[j]
pz = z_coords[k]
```

关键在于：这三个数组是**一维的**，且相互独立。这意味着你可以让X方向的间距一端密集、一端稀疏，而Y和Z方向保持均匀--这在工程仿真中非常常见。

### 9.3.2 与vtkImageData的关系

可以这样理解两者之间的关系：

- **vtkImageData**是`vtkRectilinearGrid`的一个**特例**：当三个坐标数组都退化为等间距序列时（即`x_coords[i] = origin_x + i * spacing_x`），两者等价。
- **vtkRectilinearGrid的坐标系仍然是笛卡尔坐标系**：网格线在三个维度上两两正交，只是每条线之间的间距可以不均匀。

实际上，VTK的类继承关系并非如此--`vtkImageData`和`vtkRectilinearGrid`都是`vtkDataSet`的直接子类，它们是并列的。但从概念上讲，ImageData确实是最受约束的（最规整的）情况。

### 9.3.3 创建vtkRectilinearGrid

创建RectilinearGrid需要三个步骤：

1. 设置Extent（定义索引范围）
2. 创建并填充X、Y、Z坐标数组
3. 将坐标数组关联到数据集

```cpp
// 第一步：定义extent
int extent[6] = {0, 19, 0, 14, 0, 9};  // 20x15x10个点

auto grid = vtkSmartPointer<vtkRectilinearGrid>::New();
grid->SetExtent(extent);

// 第二步：创建坐标数组
// X坐标：对数间距——前10个点密集，后10个点稀疏
vtkNew<vtkDoubleArray> xCoords;
xCoords->SetNumberOfTuples(extent[1] - extent[0] + 1);  // 20个点
for (int i = 0; i < 20; ++i)
{
  // 将[0,1]均匀分布的参数t映射为对数空间的坐标
  double t = static_cast<double>(i) / 19.0;  // t范围[0, 1]
  double x = std::pow(10.0, t);              // x范围[1, 10]，对数增长
  xCoords->SetTuple1(i, x);
}

// Y坐标：正弦函数分布——中间密两端疏
vtkNew<vtkDoubleArray> yCoords;
yCoords->SetNumberOfTuples(extent[3] - extent[2] + 1);  // 15个点
for (int j = 0; j < 15; ++j)
{
  double t = static_cast<double>(j) / 14.0;  // t范围[0, 1]
  // 正弦映射：两端密（导数接近0），中间疏
  double angle = -M_PI / 2.0 + t * M_PI;     // 从-pi/2到pi/2
  double y = (std::sin(angle) + 1.0) / 2.0 * 10.0;  // 映射到[0, 10]
  yCoords->SetTuple1(j, y);
}

// Z坐标：均匀间距
vtkNew<vtkDoubleArray> zCoords;
zCoords->SetNumberOfTuples(extent[5] - extent[4] + 1);  // 10个点
for (int k = 0; k < 10; ++k)
{
  zCoords->SetTuple1(k, static_cast<double>(k) * 0.5);  // 间距0.5
}

// 第三步：将坐标数组设置到数据集
grid->SetXCoordinates(xCoords);
grid->SetYCoordinates(yCoords);
grid->SetZCoordinates(zCoords);
```

### 9.3.4 坐标数组的要求

`SetXCoordinates()`、`SetYCoordinates()`和`SetZCoordinates()`接受`vtkDataArray*`（或其子类），但有一些约束：

- **长度必须匹配**：X坐标数组的长度必须等于`imax - imin + 1`，Y/Z坐标数组同理。
- **单调递增**：坐标值可以是非递增的，但不推荐。如果坐标值不是严格递增，某些过滤器可能产生不正确的结果。
- **数据类型灵活**：可以使用`vtkDoubleArray`、`vtkFloatArray`，甚至是整型数组。VTK在内部会处理类型转换。

### 9.3.5 典型应用场景

- **CFD拉伸网格（Stretched Grid）**：在物面边界层附近需要极高分辨率（细网格），而在远场可以使用粗网格。通过在壁面法线方向使用几何级数增长的间距，可以在不增加总网格量的前提下提高关键区域的精度。

- **海洋模型（Ocean Models）**：垂向通常使用变厚度层。表层（混合层）需要高分辨率以捕获复杂的热力学过程，深层可以使用更厚的层以节约计算资源。

- **大气模型**：垂直方向使用气压坐标（pressure coordinates），层间距在对流层底部密集、在平流层稀疏。

- **对数量纲分析**：某些物理量在对数坐标下展示更清晰（如频谱分析），RectilinearGrid允许你在对数空间中均匀采样。

### 9.3.6 代码示例：对数间距RectilinearGrid

```cpp
// 创建一个对数间距的RectilinearGrid并填充标量
int extent[6] = {0, 39, 0, 39, 0, 19};  // 40x40x20

auto rectGrid = vtkSmartPointer<vtkRectilinearGrid>::New();
rectGrid->SetExtent(extent);

// X和Y使用对数间距
vtkNew<vtkDoubleArray> xCoords;
xCoords->SetNumberOfTuples(40);
for (int i = 0; i < 40; ++i)
{
  double t = static_cast<double>(i) / 39.0;
  xCoords->SetTuple1(i, std::pow(10.0, t - 1.0));  // [0.1, 10.0]
}

vtkNew<vtkDoubleArray> yCoords;
yCoords->SetNumberOfTuples(40);
for (int j = 0; j < 40; ++j)
{
  double t = static_cast<double>(j) / 39.0;
  yCoords->SetTuple1(j, std::pow(10.0, t - 1.0));
}

// Z使用均匀间距
vtkNew<vtkDoubleArray> zCoords;
zCoords->SetNumberOfTuples(20);
for (int k = 0; k < 20; ++k)
{
  zCoords->SetTuple1(k, static_cast<double>(k) * 0.5 - 5.0);  // [-5.0, 4.5]
}

rectGrid->SetXCoordinates(xCoords);
rectGrid->SetYCoordinates(yCoords);
rectGrid->SetZCoordinates(zCoords);
```

---

## 9.4 vtkStructuredGrid（结构化网格）

### 9.4.1 数学定义

`vtkStructuredGrid`是三种结构化数据中**最通用**的一种。它保留了IJK拓扑（索引结构），但完全解放了几何位置的约束：每个网格点的(x, y, z)坐标都可以任意指定。

数据结构由两部分组成：

1. **Extent**：6元组`(imin, imax, jmin, jmax, kmin, kmax)`，定义拓扑结构。
2. **vtkPoints**：包含所有网格点物理坐标的显式数组，长度为`nx * ny * nz`。

点的存储顺序同样是**K-fastest**：对于索引(i, j, k)，对应的Points数组下标为：

```
pointId = (i * ny + j) * nz + k
```

### 9.4.2 与RectilinearGrid的递进关系

这一递进关系可以总结为：

```
vtkImageData        : 坐标由 origin + idx*spacing 隐式确定（0个坐标数组）
    ↓ 放松均匀间距约束
vtkRectilinearGrid  : 坐标由三个1D数组确定（3个坐标数组）
    ↓ 放松轴对齐约束
vtkStructuredGrid   : 坐标由全部点的3D数组显式指定（nx*ny*nz个坐标值）
```

### 9.4.3 创建vtkStructuredGrid

```cpp
// 创建一个弯曲的结构化网格（抛物面形状）
int nx = 30, ny = 30, nz = 5;

auto structuredGrid = vtkSmartPointer<vtkStructuredGrid>::New();
structuredGrid->SetExtent(0, nx-1, 0, ny-1, 0, nz-1);

// 创建点数组
vtkNew<vtkPoints> points;
points->SetNumberOfPoints(nx * ny * nz);

// 为每个网格点赋坐标
for (int k = 0; k < nz; ++k)
{
  for (int j = 0; j < ny; ++j)
  {
    for (int i = 0; i < nx; ++i)
    {
      // 将IJK索引映射到[-2, 2]范围的物理坐标
      double u = -2.0 + 4.0 * static_cast<double>(i) / (nx - 1);
      double v = -2.0 + 4.0 * static_cast<double>(j) / (ny - 1);
      double w = static_cast<double>(k) / (nz - 1);  // [0, 1]

      // 抛物面方程: z = w * (u^2 + v^2)
      double x = u;
      double y = v;
      double z = w * (u * u + v * v) * 0.8;

      vtkIdType pointId = (i * ny + j) * nz + k;
      points->SetPoint(pointId, x, y, z);
    }
  }
}

structuredGrid->SetPoints(points);
```

### 9.4.4 单元的有效性

由于`vtkStructuredGrid`允许任意点坐标，可能出现**单元退化或反转**的情况：

- **退化单元（Degenerate Cell）**：某个单元的8个顶点中，有顶点重合或共面，导致单元体积为零。
- **反转单元（Inverted Cell）**：单元内部某处的雅可比行列式（Jacobian）为负，意味着网格折叠（mesh folding）。
- **自交网格（Self-intersecting Mesh）**：不同位置的单元在物理空间中相互穿透。

大多数VTK过滤器在StructuredGrid上工作良好，但某些依赖单元几何属性的操作（如计算梯度、检查网格质量）在退化单元上可能产生不正确的结果。在生成StructuredGrid时，应确保点坐标所定义的六面体单元是有效的。

### 9.4.5 典型应用场景

- **CFD贴体网格（Body-Fitted Grid）**：为了精确模拟流体在复杂几何体（如机翼、涡轮叶片、汽车车身）周围的流动，网格必须贴合物体表面。这些网格通常在计算空间中规则（IxJxK），但在物理空间中弯曲。

- **变形几何（Deformed Geometry）**：在结构力学或流固耦合（Fluid-Structure Interaction, FSI）仿真中，材料的大变形使得初始的规整网格变为任意形状。

- **地形跟随网格（Terrain-Following Grid）**：在大气和海洋模型中，垂向坐标常跟随地表地形（sigma坐标系），使得网格在山区变密、在平原变疏。

- **医学图像分割的几何重构**：从CT/MRI中分割出的器官表面可以通过StructuredGrid来参数化表示。

### 9.4.6 代码示例：波浪面结构化网格

```cpp
// 创建一个波浪面形状的结构化网格
int nx = 40, ny = 40, nz = 8;

auto waveGrid = vtkSmartPointer<vtkStructuredGrid>::New();
waveGrid->SetExtent(0, nx-1, 0, ny-1, 0, nz-1);

vtkNew<vtkPoints> points;
points->SetNumberOfPoints(nx * ny * nz);

for (int k = 0; k < nz; ++k)
{
  for (int j = 0; j < ny; ++j)
  {
    for (int i = 0; i < nx; ++i)
    {
      double u = -3.0 + 6.0 * static_cast<double>(i) / (nx - 1);
      double v = -3.0 + 6.0 * static_cast<double>(j) / (ny - 1);
      double w = static_cast<double>(k) / (nz - 1);   // 拉伸方向 [0, 1]

      // 创建波浪面：z方向位移由sin/cos组合产生
      double x = u;
      double y = v;
      double z = w * (2.0 + std::sin(u * 2.0) * std::cos(v * 1.5) * 1.2);

      vtkIdType pointId = (i * ny + j) * nz + k;
      points->SetPoint(pointId, x, y, z);
    }
  }
}

waveGrid->SetPoints(points);
```

---

## 9.5 数据类型转换

在VTK管道中，经常需要在不同数据类型之间进行转换。结构化数据之间的转换通常比较简单，因为它们的底层拓扑（IJK结构）是兼容的。

### 9.5.1 vtkImageData 转换为 vtkRectilinearGrid

由于`vtkImageData`是`vtkRectilinearGrid`的特例（坐标是等间距的），转换很简单：从ImageData中提取origin和spacing信息，构造对应的三个坐标数组。

```cpp
vtkSmartPointer<vtkRectilinearGrid> ImageDataToRectilinearGrid(
    vtkImageData* input)
{
  auto output = vtkSmartPointer<vtkRectilinearGrid>::New();

  int* extent = input->GetExtent();
  output->SetExtent(extent);

  double origin[3], spacing[3];
  input->GetOrigin(origin);
  input->GetSpacing(spacing);

  int nx = extent[1] - extent[0] + 1;
  int ny = extent[3] - extent[2] + 1;
  int nz = extent[5] - extent[4] + 1;

  // 构建X坐标数组
  vtkNew<vtkDoubleArray> xCoords;
  xCoords->SetNumberOfTuples(nx);
  for (int i = 0; i < nx; ++i)
    xCoords->SetTuple1(i, origin[0] + i * spacing[0]);

  // 构建Y坐标数组
  vtkNew<vtkDoubleArray> yCoords;
  yCoords->SetNumberOfTuples(ny);
  for (int j = 0; j < ny; ++j)
    yCoords->SetTuple1(j, origin[1] + j * spacing[1]);

  // 构建Z坐标数组
  vtkNew<vtkDoubleArray> zCoords;
  zCoords->SetNumberOfTuples(nz);
  for (int k = 0; k < nz; ++k)
    zCoords->SetTuple1(k, origin[2] + k * spacing[2]);

  output->SetXCoordinates(xCoords);
  output->SetYCoordinates(yCoords);
  output->SetZCoordinates(zCoords);

  // 复制属性数据
  output->GetPointData()->ShallowCopy(input->GetPointData());
  output->GetCellData()->ShallowCopy(input->GetCellData());

  return output;
}
```

### 9.5.2 任何结构化数据转换为 vtkStructuredGrid

存在一个现成的过滤器可以完成此任务：`vtkStructuredGridGeometryFilter`可以将输入的任何结构化数据集（ImageData、RectilinearGrid或StructuredGrid本身）输出为`vtkStructuredGrid`。

```cpp
// 使用vtkStructuredGridGeometryFilter将ImageData转为StructuredGrid
vtkNew<vtkStructuredGridGeometryFilter> converter;
converter->SetInputData(imageData);
converter->Update();

vtkStructuredGrid* result = converter->GetOutput();
```

但更常见的是，你只需要StructuredGrid的坐标信息，而不需要通过过滤器创建副本。此时可以直接访问数据集的内置点数组。`vtkDataSet::GetPoints()`对于ImageData和RectilinearGrid返回`nullptr`（因为它们不显式存储点），但对于StructuredGrid返回有效的Points对象。

### 9.5.3 结构化数据转换为 vtkPolyData

从结构化网格提取多边形表面是一个高频操作：

| 过滤器 | 功能 |
|--------|------|
| `vtkDataSetSurfaceFilter` | 提取任何vtkDataSet的外部表面为vtkPolyData |
| `vtkStructuredGridGeometryFilter` | 提取结构化网格的几何表面（功能类似，更专用的版本） |
| `vtkContourFilter` | 从标量场中提取等值面（Isosurface），输出为vtkPolyData |
| `vtkCutter` | 用平面（或其他隐式函数）裁切数据集，产生轮廓线/面 |

使用示例：

```cpp
// 从ImageData提取表面（6个外表面）
vtkNew<vtkDataSetSurfaceFilter> surfaceFilter;
surfaceFilter->SetInputData(imageData);
surfaceFilter->Update();
// surfaceFilter->GetOutput() 是 vtkPolyData

// 从StructuredGrid提取等值面
vtkNew<vtkContourFilter> contourFilter;
contourFilter->SetInputData(structuredGrid);
contourFilter->SetValue(0, 2.5);  // 提取标量值为2.5的等值面
contourFilter->Update();
```

### 9.5.4 vtkPolyData 转换为 vtkStructuredGrid

这个方向不太常见，因为`vtkPolyData`通常不保留结构化拓扑。但如果你知道PolyData的点恰好按IJK顺序排列，可以手动构造StructuredGrid：

```cpp
vtkSmartPointer<vtkStructuredGrid> PolyDataToStructuredGrid(
    vtkPolyData* input, int nx, int ny, int nz)
{
  auto output = vtkSmartPointer<vtkStructuredGrid>::New();
  output->SetExtent(0, nx-1, 0, ny-1, 0, nz-1);

  // 直接共享点数据（浅拷贝）
  vtkNew<vtkPoints> points;
  points->DeepCopy(input->GetPoints());
  output->SetPoints(points);

  // 复制属性数据
  output->GetPointData()->ShallowCopy(input->GetPointData());

  return output;
}
```

注意：只有当PolyData中的点确实是按K-fastest顺序排列并且数量等于`nx*ny*nz`时，这个转换才有意义。

### 9.5.5 双向转换一览

```
              vtkImageData
                   |
                   | ImageDataToRectilinearGrid (手动)
                   v
           vtkRectilinearGrid
                   |
                   | vtkStructuredGridGeometryFilter
                   v
           vtkStructuredGrid
                   |
                   | vtkDataSetSurfaceFilter / vtkContourFilter
                   v
              vtkPolyData
```

---

## 9.6 代码示例：三类数据对比展示

本节的综合示例将创建三种数据结构--一个`vtkImageData`、一个`vtkRectilinearGrid`和一个`vtkStructuredGrid`--并在同一个窗口中以三个并排视口展示它们，使你能够直观地对比它们的几何差异。

### 9.6.1 程序设计

**显示策略：**由于三种结构化的数据集都不能被VTK直接渲染（它们没有显式的面片几何），我们需要一个统一的显示策略。在本例中，我们将：

1. 在每个网格的点上填充相同的标量场（距离中心场）。
2. 使用`vtkDataSetSurfaceFilter`提取每个网格的外部表面，得到`vtkPolyData`。
3. 使用`vtkContourFilter`提取等值面，叠加在表面渲染之上。
4. 以**线框模式（Wireframe）**额外展示网格的线框结构，以突显三类网格在几何分布上的差异。

**三种网格的配置：**

| 网格类型 | 配置描述 | 点分辨率 |
|---------|---------|---------|
| ImageData | 均匀间距（Spacing 0.5），原点(-4, -4, -4) | 20x20x20 |
| RectilinearGrid | X对数间距、Y正弦分布、Z均匀 | 20x20x20 |
| StructuredGrid | 抛物面弯曲形状 | 20x20x20 |

**视口布局：**

```
+---------------------+----------------------+----------------------+
|  ImageData           |  RectilinearGrid     |  StructuredGrid      |
|  (均匀网格)           |  (直线网格)           |  (结构化网格)         |
|  等值面+线框          |  等值面+线框           |  等值面+线框           |
+---------------------+----------------------+----------------------+
```

### 9.6.2 完整代码

```cpp
// ============================================================================
// 第九章综合示例：三种结构化数据类型的对比展示
//
// 在一个窗口中并排展示：
//   左：vtkImageData（均匀网格）
//   中：vtkRectilinearGrid（直线网格，对数间距+X正弦分布）
//   右：vtkStructuredGrid（结构化网格，抛物面弯曲）
//
// 每种网格：
//   - 填充相同的距离场标量
//   - 以线框模式显示，突显网格几何结构的差异
//   - 提取等值面以展示标量场的形状
//
// 需要的VTK模块（CMake COMPONENTS）：
//   CommonCore, CommonDataModel, CommonExecutionModel,
//   FiltersCore, FiltersSources, FiltersGeometry,
//   InteractionStyle, RenderingCore, RenderingOpenGL2, RenderingFreeType
// ============================================================================

#include "vtkActor.h"
#include "vtkCamera.h"
#include "vtkContourFilter.h"
#include "vtkDataSetSurfaceFilter.h"
#include "vtkDoubleArray.h"
#include "vtkExtractEdges.h"
#include "vtkImageData.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPoints.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRectilinearGrid.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkSmartPointer.h"
#include "vtkStructuredGrid.h"
#include "vtkTextActor.h"
#include "vtkTextProperty.h"

#include <cmath>
#include <iostream>
#include <string>

#ifndef M_PI
#define M_PI 3.14159265358979323846
#endif

// ============================================================================
// 辅助函数：为ImageData填充距离中心标量场
// ============================================================================
void FillDistanceScalars(vtkDataSet* dataset)
{
  vtkIdType numPoints = dataset->GetNumberOfPoints();

  // 首先计算数据集的中心
  double bounds[6];
  dataset->GetBounds(bounds);
  double cx = (bounds[0] + bounds[1]) / 2.0;
  double cy = (bounds[2] + bounds[3]) / 2.0;
  double cz = (bounds[4] + bounds[5]) / 2.0;

  // 计算最大可能距离（用于归一化）
  double maxDist = 0.0;
  for (vtkIdType ptId = 0; ptId < numPoints; ++ptId)
  {
    double p[3];
    dataset->GetPoint(ptId, p);
    double dx = p[0] - cx;
    double dy = p[1] - cy;
    double dz = p[2] - cz;
    double dist = std::sqrt(dx*dx + dy*dy + dz*dz);
    if (dist > maxDist) maxDist = dist;
  }

  // 创建标量数组
  vtkNew<vtkDoubleArray> scalars;
  scalars->SetName("Distance");
  scalars->SetNumberOfComponents(1);
  scalars->SetNumberOfTuples(numPoints);

  for (vtkIdType ptId = 0; ptId < numPoints; ++ptId)
  {
    double p[3];
    dataset->GetPoint(ptId, p);
    double dx = p[0] - cx;
    double dy = p[1] - cy;
    double dz = p[2] - cz;
    double dist = std::sqrt(dx*dx + dy*dy + dz*dz);
    // 归一化到[0, 1]
    scalars->SetTuple1(ptId, (maxDist > 0.0) ? dist / maxDist : 0.0);
  }

  dataset->GetPointData()->SetScalars(scalars);
}

// ============================================================================
// 辅助函数：为数据集创建可视化管线
//
// 对于每种结构化数据集，我们创建两个Actor：
//   1. wireframeActor: 线框模式，展示网格本身的几何结构
//   2. contourActor:   等值面，展示标量场的一个截面
//
// 参数:
//   dataset  - 结构化的输入数据集 (ImageData / RectilinearGrid / StructuredGrid)
//   wireframeColor - 线框的颜色
//   contourColor   - 等值面的颜色
//   isoValue       - 等值面提取的标量值 (归一化后的值, 默认 0.5)
// ============================================================================
struct VisualizationActors
{
  vtkSmartPointer<vtkActor> wireframeActor;
  vtkSmartPointer<vtkActor> contourActor;
};

VisualizationActors CreateVisualization(
    vtkDataSet* dataset,
    double wireframeColor[3],
    double contourColor[3],
    double isoValue = 0.5)
{
  VisualizationActors result;

  // ——— 管线1：线框展示 ———
  // 步骤1：提取数据集的外部表面
  vtkNew<vtkDataSetSurfaceFilter> surfaceFilter;
  surfaceFilter->SetInputData(dataset);

  // 步骤2：提取表面的所有边，产生线框效果
  vtkNew<vtkExtractEdges> edgeFilter;
  edgeFilter->SetInputConnection(surfaceFilter->GetOutputPort());

  // 步骤3：Mapper -> Actor
  vtkNew<vtkPolyDataMapper> wireframeMapper;
  wireframeMapper->SetInputConnection(edgeFilter->GetOutputPort());
  wireframeMapper->ScalarVisibilityOff();

  auto wireframeActor = vtkSmartPointer<vtkActor>::New();
  wireframeActor->SetMapper(wireframeMapper);
  wireframeActor->GetProperty()->SetColor(wireframeColor);
  wireframeActor->GetProperty()->SetLineWidth(1.0);
  wireframeActor->GetProperty()->SetOpacity(0.7);

  result.wireframeActor = wireframeActor;

  // ——— 管线2：等值面展示 ———
  // 步骤1：从标量场中提取等值面
  vtkNew<vtkContourFilter> contourFilter;
  contourFilter->SetInputData(dataset);
  contourFilter->SetValue(0, isoValue);
  contourFilter->ComputeNormalsOn();  // 计算法向以保证正确的光照

  // 步骤2：Mapper -> Actor
  vtkNew<vtkPolyDataMapper> contourMapper;
  contourMapper->SetInputConnection(contourFilter->GetOutputPort());
  contourMapper->ScalarVisibilityOff();

  auto contourActor = vtkSmartPointer<vtkActor>::New();
  contourActor->SetMapper(contourMapper);
  contourActor->GetProperty()->SetColor(contourColor);
  contourActor->GetProperty()->SetDiffuse(0.9);
  contourActor->GetProperty()->SetSpecular(0.4);
  contourActor->GetProperty()->SetSpecularPower(20.0);
  contourActor->GetProperty()->SetOpacity(0.85);

  result.contourActor = contourActor;

  return result;
}

// ============================================================================
// 辅助函数：创建并返回一个完全配置好的vtkImageData
//
// 创建一个20x20x20的均匀网格，范围[-4, 4]^3
// ============================================================================
vtkSmartPointer<vtkImageData> CreateImageDataDemo()
{
  auto imageData = vtkSmartPointer<vtkImageData>::New();

  int nx = 20, ny = 20, nz = 20;
  imageData->SetExtent(0, nx - 1, 0, ny - 1, 0, nz - 1);
  imageData->SetOrigin(-4.0, -4.0, -4.0);
  imageData->SetSpacing(8.0 / (nx - 1), 8.0 / (ny - 1), 8.0 / (nz - 1));

  FillDistanceScalars(imageData);

  return imageData;
}

// ============================================================================
// 辅助函数：创建并返回一个完全配置好的vtkRectilinearGrid
//
// X方向：对数间距 (在[-4, 4]之间，中间密集两端稀疏)
// Y方向：正弦分布 (在[-4, 4]之间，两端密集中间稀疏)
// Z方向：均匀间距
// ============================================================================
vtkSmartPointer<vtkRectilinearGrid> CreateRectilinearGridDemo()
{
  auto grid = vtkSmartPointer<vtkRectilinearGrid>::New();

  int nx = 20, ny = 20, nz = 20;
  grid->SetExtent(0, nx - 1, 0, ny - 1, 0, nz - 1);

  // X坐标：对数分布，从-4到4
  vtkNew<vtkDoubleArray> xCoords;
  xCoords->SetNumberOfTuples(nx);
  for (int i = 0; i < nx; ++i)
  {
    // 将均匀的[0, 1]映射为对数空间，再线性映射到[-4, 4]
    double t = static_cast<double>(i) / (nx - 1);
    // 使用指数映射产生中间密两端疏的分布
    // 先映射到[0.01, 1.0]，再取log，最后线性缩放到[-4, 4]
    double logVal = std::log10(0.1 + 9.9 * t);  // 范围[log10(0.1)=-1, log10(10)=1]
    double x = logVal * 4.0;                    // 范围[-4, 4]
    xCoords->SetTuple1(i, x);
  }

  // Y坐标：正弦分布，从-4到4（两端密集，中间稀疏）
  vtkNew<vtkDoubleArray> yCoords;
  yCoords->SetNumberOfTuples(ny);
  for (int j = 0; j < ny; ++j)
  {
    double t = static_cast<double>(j) / (ny - 1);  // [0, 1]
    // sin映射：[-pi/2, pi/2] -> [-4, 4]，导数在端点处最小（网格密集）
    double angle = -M_PI / 2.0 + t * M_PI;
    double y = 4.0 * std::sin(angle);
    yCoords->SetTuple1(j, y);
  }

  // Z坐标：均匀间距
  vtkNew<vtkDoubleArray> zCoords;
  zCoords->SetNumberOfTuples(nz);
  for (int k = 0; k < nz; ++k)
  {
    double z = -4.0 + 8.0 * static_cast<double>(k) / (nz - 1);
    zCoords->SetTuple1(k, z);
  }

  grid->SetXCoordinates(xCoords);
  grid->SetYCoordinates(yCoords);
  grid->SetZCoordinates(zCoords);

  FillDistanceScalars(grid);

  return grid;
}

// ============================================================================
// 辅助函数：创建并返回一个完全配置好的vtkStructuredGrid
//
// 创建一个抛物面形状的结构化网格
// z = 0.2 * (x^2 + y^2)，形成碗状弯曲
// ============================================================================
vtkSmartPointer<vtkStructuredGrid> CreateStructuredGridDemo()
{
  auto grid = vtkSmartPointer<vtkStructuredGrid>::New();

  int nx = 20, ny = 20, nz = 20;
  grid->SetExtent(0, nx - 1, 0, ny - 1, 0, nz - 1);

  vtkNew<vtkPoints> points;
  points->SetNumberOfPoints(nx * ny * nz);

  for (int k = 0; k < nz; ++k)
  {
    for (int j = 0; j < ny; ++j)
    {
      for (int i = 0; i < nx; ++i)
      {
        // IJK -> 参数坐标
        double u = -4.0 + 8.0 * static_cast<double>(i) / (nx - 1);
        double v = -4.0 + 8.0 * static_cast<double>(j) / (ny - 1);
        double w = static_cast<double>(k) / (nz - 1);  // [0, 1]

        // 抛物面弯曲：碗越深(w越大)弯曲越明显
        double x = u;
        double y = v;
        double z = w * (-4.0 + 8.0 * static_cast<double>(k) / (nz - 1))
                 + (1.0 - w) * 0.0
                 + 0.15 * (u * u + v * v);  // 抛物面项

        vtkIdType pointId = (i * ny + j) * nz + k;
        points->SetPoint(pointId, x, y, z);
      }
    }
  }

  grid->SetPoints(points);
  FillDistanceScalars(grid);

  return grid;
}

// ============================================================================
// 辅助函数：为渲染器创建文本标签
// ============================================================================
vtkSmartPointer<vtkTextActor> CreateTitleLabel(
    const std::string& title,
    const std::string& subtitle,
    int fontSize = 14)
{
  auto textActor = vtkSmartPointer<vtkTextActor>::New();
  std::string fullText = title + "\n" + subtitle;
  textActor->SetInput(fullText.c_str());
  textActor->SetPosition(10, 10);
  textActor->GetTextProperty()->SetFontSize(fontSize);
  textActor->GetTextProperty()->SetColor(1.0, 1.0, 1.0);
  textActor->GetTextProperty()->SetFontFamilyToCourier();
  textActor->GetTextProperty()->SetJustificationToLeft();
  textActor->GetTextProperty()->SetVerticalJustificationToBottom();

  return textActor;
}

// ============================================================================
// 辅助函数：为渲染器统一设置相机
// ============================================================================
void SetupCamera(vtkRenderer* renderer)
{
  auto camera = renderer->GetActiveCamera();
  camera->SetPosition(6.0, 6.0, 10.0);
  camera->SetFocalPoint(0.0, 0.0, 0.0);
  camera->SetViewUp(0.0, 1.0, 0.0);
  camera->SetClippingRange(0.1, 100.0);
}

// ============================================================================
// 主程序入口
// ============================================================================
int main(int argc, char* argv[])
{
  (void)argc;
  (void)argv;

  // ==========================================================================
  // 第一步：创建三种数据集
  // ==========================================================================

  std::cout << "Creating vtkImageData..." << std::endl;
  auto imageData = CreateImageDataDemo();

  std::cout << "Creating vtkRectilinearGrid..." << std::endl;
  auto rectilinearGrid = CreateRectilinearGridDemo();

  std::cout << "Creating vtkStructuredGrid..." << std::endl;
  auto structuredGrid = CreateStructuredGridDemo();

  // 输出各数据集的统计信息
  std::cout << "\nDataset Statistics:\n";
  std::cout << "  ImageData:        " << imageData->GetNumberOfPoints()
            << " points, " << imageData->GetNumberOfCells() << " cells\n";
  std::cout << "  RectilinearGrid:  " << rectilinearGrid->GetNumberOfPoints()
            << " points, " << rectilinearGrid->GetNumberOfCells() << " cells\n";
  std::cout << "  StructuredGrid:   " << structuredGrid->GetNumberOfPoints()
            << " points, " << structuredGrid->GetNumberOfCells() << " cells\n";
  std::cout << std::endl;

  // ==========================================================================
  // 第二步：创建渲染基础设施
  // ==========================================================================

  auto renderWindow = vtkSmartPointer<vtkRenderWindow>::New();
  renderWindow->SetMultiSamples(4);
  renderWindow->SetSize(1500, 550);
  renderWindow->SetWindowName("Chapter 9: Structured Data Types Comparison");

  const double bgColor[3] = {0.12, 0.14, 0.20};

  // 三个视口：水平并排
  // 视口0: ImageData (左)
  // 视口1: RectilinearGrid (中)
  // 视口2: StructuredGrid (右)
  const int numViewports = 3;
  vtkSmartPointer<vtkRenderer> renderers[numViewports];

  for (int i = 0; i < numViewports; ++i)
  {
    double xmin = static_cast<double>(i)     / numViewports;
    double xmax = static_cast<double>(i + 1) / numViewports;

    auto renderer = vtkSmartPointer<vtkRenderer>::New();
    renderer->SetViewport(xmin, 0.0, xmax, 1.0);
    renderer->SetBackground(bgColor);
    SetupCamera(renderer);

    // 为每个视口添加分隔边界效果（通过设置不同的背景色带）
    renderers[i] = renderer;
    renderWindow->AddRenderer(renderer);
  }

  // ==========================================================================
  // 第三步：为每种数据集创建可视化
  // ==========================================================================

  // 颜色方案
  double imageWireColor[3]      = {0.40, 0.80, 1.00};   // 浅蓝线框
  double rectWireColor[3]       = {0.40, 1.00, 0.55};   // 浅绿线框
  double structWireColor[3]     = {1.00, 0.60, 0.30};   // 橙色线框

  double imageContourColor[3]   = {0.20, 0.50, 0.90};   // 蓝色等值面
  double rectContourColor[3]    = {0.20, 0.75, 0.35};   // 绿色等值面
  double structContourColor[3]  = {0.90, 0.45, 0.15};   // 橙色等值面

  // 创建ImageData的可视化
  auto imageVis = CreateVisualization(
      imageData, imageWireColor, imageContourColor, 0.45);
  renderers[0]->AddActor(imageVis.wireframeActor);
  renderers[0]->AddActor(imageVis.contourActor);

  // 创建RectilinearGrid的可视化
  auto rectVis = CreateVisualization(
      rectilinearGrid, rectWireColor, rectContourColor, 0.45);
  renderers[1]->AddActor(rectVis.wireframeActor);
  renderers[1]->AddActor(rectVis.contourActor);

  // 创建StructuredGrid的可视化
  auto structVis = CreateVisualization(
      structuredGrid, structWireColor, structContourColor, 0.45);
  renderers[2]->AddActor(structVis.wireframeActor);
  renderers[2]->AddActor(structVis.contourActor);

  // ==========================================================================
  // 第四步：添加文本标签
  // ==========================================================================

  // ImageData标签
  auto labelImage = CreateTitleLabel(
      "vtkImageData",
      "Uniform grid | dx=dy=dz constant",
      13);
  labelImage->GetTextProperty()->SetColor(imageWireColor[0],
      imageWireColor[1], imageWireColor[2]);
  renderers[0]->AddActor2D(labelImage);

  // RectilinearGrid标签
  auto labelRect = CreateTitleLabel(
      "vtkRectilinearGrid",
      "Log X + Sine Y spacing",
      13);
  labelRect->GetTextProperty()->SetColor(rectWireColor[0],
      rectWireColor[1], rectWireColor[2]);
  renderers[1]->AddActor2D(labelRect);

  // StructuredGrid标签
  auto labelStruct = CreateTitleLabel(
      "vtkStructuredGrid",
      "Paraboloid curvature",
      13);
  labelStruct->GetTextProperty()->SetColor(structWireColor[0],
      structWireColor[1], structWireColor[2]);
  renderers[2]->AddActor2D(labelStruct);

  // ==========================================================================
  // 第五步：创建交互器并运行
  // ==========================================================================

  auto interactor = vtkSmartPointer<vtkRenderWindowInteractor>::New();
  interactor->SetRenderWindow(renderWindow);

  auto style = vtkSmartPointer<vtkInteractorStyleTrackballCamera>::New();
  interactor->SetInteractorStyle(style);

  // 首帧渲染
  renderWindow->Render();

  // 输出使用说明
  std::cout << "========================================\n"
            << "  Chapter 9: Structured Data Comparison\n"
            << "========================================\n"
            << "\n"
            << "左视口：vtkImageData（均匀网格）\n"
            << "   - 显示：线框（表面边的提取）+ 等值面\n"
            << "   - 特征：所有网格间距完全均匀\n"
            << "\n"
            << "中视口：vtkRectilinearGrid（直线网格）\n"
            << "   - 显示：线框 + 等值面\n"
            << "   - 特征：X对数间距（中心密两端疏）\n"
            << "            Y正弦分布（两端密中间疏）\n"
            << "            Z均匀间距\n"
            << "\n"
            << "右视口：vtkStructuredGrid（结构化网格）\n"
            << "   - 显示：线框 + 等值面\n"
            << "   - 特征：抛物面弯曲形状\n"
            << "\n"
            << "交互操作：\n"
            << "  - 鼠标左键拖拽：旋转场景\n"
            << "  - 鼠标右键拖拽：缩放\n"
            << "  - 鼠标中键拖拽：平移\n"
            << "  - 滚轮：缩放\n"
            << "  - 按 R 键：重置相机\n"
            << "\n"
            << "观察要点：\n"
            << "  - 左视口：网格线完全正交且等间距\n"
            << "  - 中视口：网格线仍正交但间距变化明显\n"
            << "  - 右视口：网格线弯曲且不再正交\n"
            << "  - 三个等值面的形状因几何变形而不同\n"
            << "========================================\n"
            << std::endl;

  // 进入事件循环
  interactor->Start();

  return 0;
}
```

### 9.6.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
project(StructuredDataComparison VERSION 1.0.0 LANGUAGES CXX)

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
    FiltersCore
    FiltersGeneral
    FiltersGeometry
    FiltersSources
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
    RenderingFreeType
  REQUIRED
)

# 打印VTK信息
message(STATUS "VTK version: ${VTK_VERSION}")
message(STATUS "VTK rendering backend: OpenGL2")

# 创建可执行文件
add_executable(StructuredDataComparison StructuredDataComparison.cxx)

# 链接VTK库
target_link_libraries(StructuredDataComparison
  PRIVATE
    ${VTK_LIBRARIES}
)

# VTK模块自动初始化
vtk_module_autoinit(
  TARGETS StructuredDataComparison
  MODULES ${VTK_LIBRARIES}
)
```

**所需模块说明：**

| 模块 | 用途 |
|------|------|
| `CommonCore` | VTK基础类型、智能指针、引用计数 |
| `CommonDataModel` | `vtkImageData`、`vtkRectilinearGrid`、`vtkStructuredGrid`、`vtkPolyData`、`vtkPoints` |
| `CommonExecutionModel` | 管道执行基础设施 |
| `CommonMath` | 数学常量与工具 |
| `FiltersCore` | `vtkContourFilter`（等值面提取） |
| `FiltersGeneral` | `vtkDataSetSurfaceFilter`（表面提取） |
| `FiltersGeometry` | `vtkExtractEdges`（边提取） |
| `FiltersSources` | 数据源生成器（如果需要） |
| `InteractionStyle` | `vtkInteractorStyleTrackballCamera` |
| `RenderingCore` | `vtkActor`、`vtkRenderer`、`vtkRenderWindow`、`vtkProperty` |
| `RenderingOpenGL2` | OpenGL2渲染后端 |
| `RenderingFreeType` | FreeType字体渲染（`vtkTextActor`） |

---

## 9.7 本章小结

本章系统介绍了VTK中三种结构化数据类型的创建、使用和转换。以下是关键要点：

### 9.7.1 三类数据的递进关系

```
灵活性递增 ──────────────────────────────>
   vtkImageData  →  vtkRectilinearGrid  →  vtkStructuredGrid
内存递增 ────────────────────────────────>
   (最省)              (中等)                (最大)
```

- **vtkImageData**：最节省内存，只需要origin + spacing + extent三个参数。适用于所有维度上采样均匀的场景。
- **vtkRectilinearGrid**：比ImageData稍多些内存（三个一维坐标数组），但允许每个维度独立变化间距。适用于轴对齐但非均匀采样的场景。
- **vtkStructuredGrid**：需要存储所有点的完整坐标，但提供了完全的几何自由度。适用于弯曲/变形的结构化网格。

### 9.7.2 核心API速查

| 操作 | vtkImageData | vtkRectilinearGrid | vtkStructuredGrid |
|------|-------------|-------------------|------------------|
| 设置范围 | `SetExtent()` 或 `SetDimensions()` | `SetExtent()` | `SetExtent()` |
| 设置几何 | `SetOrigin()` + `SetSpacing()` | `SetXCoordinates()` + `SetYCoordinates()` + `SetZCoordinates()` | `SetPoints(vtkPoints*)` |
| 获取点坐标 | `GetPoint(id, p)` | `GetPoint(id, p)` | `GetPoint(id, p)` |
| 获取尺寸 | `GetDimensions()` | `GetDimensions()` | `GetDimensions()` |
| 添加标量 | `GetPointData()->SetScalars()` | `GetPointData()->SetScalars()` | `GetPointData()->SetScalars()` |
| 获取边界 | `GetBounds()` | `GetBounds()` | `GetBounds()` |

### 9.7.3 常用转换路径

```
任意结构化数据
    |
    ├── vtkDataSetSurfaceFilter ──> vtkPolyData (外部表面)
    ├── vtkContourFilter         ──> vtkPolyData (等值面)
    ├── vtkCutter                ──> vtkPolyData (切片)
    ├── vtkExtractEdges          ──> vtkPolyData (边线)
    └── vtkStructuredGridGeometryFilter ──> vtkStructuredGrid
```

### 9.7.4 何时使用哪种类型

| 你的数据特征 | 推荐类型 |
|-------------|---------|
| 所有方向均匀采样，如CT/MRI堆叠、位图 | `vtkImageData` |
| 某方向有变间距但网格仍正交，如CFD边界层拉伸 | `vtkRectilinearGrid` |
| 网格弯曲贴合物面或发生变形 | `vtkStructuredGrid` |
| 拓扑不规则（混合单元类型、任意连接关系） | `vtkUnstructuredGrid`（见第八章） |
| 由点-线-面组成的离散几何模型 | `vtkPolyData`（见第七章） |

### 9.7.5 与后续章节的衔接

掌握了本章的结构化数据，你已经完成了VTK核心数据模型的全部学习（第七章的PolyData、第八章的UnstructuredGrid、本章的三种结构化数据类型）。从第十章开始，你将进入VTK的I/O体系——学习如何从磁盘文件中读取真实的科学数据（如.vtk、.vtu、.vti、.vtr、.vts格式），以及如何将你的计算结果写入这些格式以供ParaView等工具使用。

这些文件格式与数据模型之间有着精确的对应关系：`.vti`对应ImageData，`.vtr`对应RectilinearGrid，`.vts`对应StructuredGrid，`.vtu`对应UnstructuredGrid，`.vtp`对应PolyData。理解本章的数据模型将使你在面对文件I/O时胸有成竹。

---

> **学习建议**：在继续下一章之前，建议你运行9.6节的综合示例，并在交互窗口中从不同角度旋转观察三个视口。特别注意：
> - ImageData视口中网格线的完全规则性（正交+等距）
> - RectilinearGrid视口中X方向中间密两端疏、Y方向两端密中间疏的视觉效果
> - StructuredGrid视口中网格线在抛物面弯曲下的变形
> - 三种网格中等值面的形状差异——同样的标量值0.45，因网格几何不同而产生不同形状的等值面
>
> 这种直观的视觉对比将使你对三种结构化数据类型的理解更加深刻。
