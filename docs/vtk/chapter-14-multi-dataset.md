# 第十四章 多个数据集处理 (Multi-Dataset Processing)

## 本章导读

在前面的章节中，我们一直是"一个Actor显示一个数据集"的模式——你创建一个球体、一个方块、或一个曲面，然后将其交给Mapper和Actor渲染到窗口中。这种模式对于简单的演示场景足够了，但现实世界的科学可视化往往涉及远比这更复杂的数据组织需求。

考虑以下几个典型场景：

- 你有一个CAD装配体，它由数十个独立的零件组成——每个零件是一个单独的几何文件。你希望将它们全部导入并显示在同一场景中，同时保留每个零件作为独立可操作单元的能力。
- 你的仿真结果包含多种数据类型——例如一个结构分析同时产生表面网格（PolyData）、体网格（UnstructuredGrid）、以及采样点云（也是PolyData）。你需要将它们组织在一个层次化的数据结构中。
- 你正在构建一个多部件可视化，希望能够独立地切换每个部件的可见性、改变每个部件的颜色和透明度，而不需要为每个部件创建和维护数十个独立的Actor。

VTK为这些需求提供了两套互补的工具：

1. **合并数据集**（Merging）：使用`vtkAppendPolyData`、`vtkAppendFilter`或`vtkAppendDatasets`将多个相同或不同类型的数据集合成为一个单一的数据集。这适合"焊接"多个几何部件或合并多个文件到同一渲染管线。

2. **复合数据集**（Composite Datasets）：使用`vtkMultiBlockDataSet`和`vtkPartitionedDataSetCollection`将多个数据集组织成层次化的、带命名的块结构。每个块保持独立——你可以按块控制颜色、可见性、不透明度。这适合多部件装配、多物理场结果显示等场景。

本章将系统性地讲解这两种数据组织方式，从最简单的合并操作开始，逐步深入到复合数据结构、复合渲染管线，最后以一个完整的多部件装配程序结束。

---

## 14.1 合并数据集

合并（Appending）是将多个数据集组合为一个单一数据集的机制。在VTK中，有三个主要的合并过滤器，它们分别适用于不同的场景。

### 14.1.1 vtkAppendPolyData -- 合并多个PolyData

`vtkAppendPolyData`是最常用的合并过滤器之一，专门用于将多个`vtkPolyData`对象合并为一个。这对于将从不同文件加载的几何体部件合并为单一模型特别有用。

**工作原理**：`vtkAppendPolyData`的工作方式是"追加"——它将所有输入PolyData中的点和单元依次复制到输出中。各个输入之间不进行连接或焊接操作（除非你显式地指定公差）。

关键API：

```cpp
#include <vtkAppendPolyData.h>

vtkNew<vtkAppendPolyData> appendFilter;

// 方法1：使用AddInputData逐个添加
appendFilter->AddInputData(polyData1);
appendFilter->AddInputData(polyData2);
appendFilter->AddInputData(polyData3);

// 方法2（旧式接口，在VTK 9.x中仍可用但不推荐）：
// appendFilter->AddInputConnection(source1->GetOutputPort());

// 执行合并
appendFilter->Update();

// 获取合并结果
vtkPolyData* merged = appendFilter->GetOutput();

// 可选：控制点合并（焊接相邻点）
// 当两个输入的点在指定容差内时，它们可能被合并为一个点
appendFilter->SetTolerance(1e-6);  // 默认值
// appendFilter->SetToleranceIsAbsolute(true);  // 是否使用绝对容差

// 获取合并后的统计信息
std::cout << "Total points: " << merged->GetNumberOfPoints() << std::endl;
std::cout << "Total cells:  " << merged->GetNumberOfCells() << std::endl;
```

**关键点**：
- 点数据（Point Data）和单元数据（Cell Data）也会被合并。如果某个输入缺少某个标量数组，对应的点的该数据属性会被设为NaN或零值。
- 如果你希望对合并后的整体进行统一着色（如按高度着色），可以在合并后使用`vtkElevationFilter`等过滤器来计算标量值。
- `SetTolerance()`控制是否合并邻近的重复点——这在需要真正"焊接"部件边缘时很有用。

### 14.1.2 vtkAppendFilter -- 合并相同类型的数据集

`vtkAppendFilter`是`vtkAppendPolyData`的泛化版本。它可以合并任何类型的`vtkDataSet`，但**所有输入必须是同一种具体类型**——要么全是`vtkPolyData`，要么全是`vtkUnstructuredGrid`，不能混合。

内部工作原理：`vtkAppendFilter`在收到第一个输入时，自动检测其类型；后续所有输入都会被检查是否为同一类型。如果类型不匹配，过滤器会报告错误。

```cpp
#include <vtkAppendFilter.h>

vtkNew<vtkAppendFilter> appendFilter;

// 可以合并PolyData
appendFilter->AddInputData(polyDataA);
appendFilter->AddInputData(polyDataB);

// 也可以合并UnstructuredGrid（但不能和PolyData混合）
appendFilter->AddInputData(unstructuredGridA);
appendFilter->AddInputData(unstructuredGridB);

appendFilter->Update();

// 注意：输出与输入类型相同
vtkDataSet* merged = appendFilter->GetOutput();
// 如果输入全是PolyData，merged可以被安全地向下转型为vtkPolyData
```

**适用场景**：
- 合并同一个CFD仿真输出的多个分区块（partitions），每个块都是UnstructuredGrid。
- 合并从文件读取的多个结构化网格块。
- 将分散的PolyData片断组装为一个整体网格。

**限制**：
- 不能合并不同类型的数据集。
- 合并后的数据集失去了各原始数据块的独立性（无法再单独操控某个原始块的颜色或可见性）。

### 14.1.3 vtkAppendDatasets -- 合并不同类型的数据集

`vtkAppendDatasets`（在VTK 9.x中引入）解决了`vtkAppendFilter`的类型限制问题。它可以接受不同类型的`vtkDataSet`输入——例如将PolyData和UnstructuredGrid合并到同一个输出中。

```cpp
#include <vtkAppendDatasets.h>

vtkNew<vtkAppendDatasets> append;

// 可以混合不同的数据类型
append->AddInputData(polyDataBlock);        // vtkPolyData
append->AddInputData(unstructuredBlock);    // vtkUnstructuredGrid

append->Update();

// 输出类型为vtkPartitionedDataSetCollection
// 这是一种集合类型，内部的每个原始输入成为一个独立的"分区"
auto* output = append->GetOutput();
std::cout << "Number of partitions: " << output->GetNumberOfPartitionedDataSets()
          << std::endl;
```

注意：`vtkAppendDatasets`的输出类型是`vtkPartitionedDataSetCollection`，这是一个复合数据结构（将在14.2节详细讨论）。这与`vtkAppendFilter`直接将点数据合并到一个统一类型的数据集中不同。因此，当你的目标不是"焊接为一个整体网格"而是"保持各部件独立但组织在一起"时，`vtkAppendDatasets`是更合适的选择。

### 14.1.4 三种合并方式的对比

| 特性 | vtkAppendPolyData | vtkAppendFilter | vtkAppendDatasets |
|------|-------------------|-----------------|-------------------|
| **输入类型** | 仅vtkPolyData | 同一种vtkDataSet类型 | 任意vtkDataSet类型 |
| **输出类型** | vtkPolyData | 与输入相同 | vtkPartitionedDataSetCollection |
| **点合并/焊接** | 支持（SetTolerance） | 不支持 | 分开存储（不合并点） |
| **保持部件独立性** | 否（完全合并） | 否（完全合并） | 是（保持分区） |
| **VTK版本** | 经典（所有版本） | 经典（所有版本） | VTK 9.x |
| **适用场景** | 合并几何部件、多文件加载 | 合并同类型的网格分区 | 组织多类型的仿真结果 |

### 14.1.5 使用场景与代码示例：合并三个球体

以下示例展示如何使用`vtkAppendPolyData`将三个位置不同、颜色不同的球体合并为一个PolyData：

```cpp
// ============================================================================
// 示例 14.1: 使用 vtkAppendPolyData 合并三个球体
// ============================================================================
#include <vtkActor.h>
#include <vtkAppendPolyData.h>
#include <vtkCamera.h>
#include <vtkNamedColors.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkSmartPointer.h>
#include <vtkSphereSource.h>
#include <vtkPolyData.h>
#include <vtkFloatArray.h>
#include <vtkPointData.h>
#include <vtkLookupTable.h>

int main()
{
    // 1. 创建三个球体源
    vtkNew<vtkSphereSource> sphere1;
    sphere1->SetCenter(0.0, 0.0, 0.0);
    sphere1->SetRadius(1.0);
    sphere1->SetThetaResolution(32);
    sphere1->SetPhiResolution(32);
    sphere1->Update();

    vtkNew<vtkSphereSource> sphere2;
    sphere2->SetCenter(2.5, 0.0, 0.0);
    sphere2->SetRadius(0.8);
    sphere2->SetThetaResolution(32);
    sphere2->SetPhiResolution(32);
    sphere2->Update();

    vtkNew<vtkSphereSource> sphere3;
    sphere3->SetCenter(1.25, 2.0, 0.0);
    sphere3->SetRadius(0.6);
    sphere3->SetThetaResolution(32);
    sphere3->SetPhiResolution(32);
    sphere3->Update();

    // 2. 为每个球体添加标量数据（用于区分来源）
    vtkNew<vtkFloatArray> scalars1;
    scalars1->SetName("SourceID");
    for (vtkIdType i = 0; i < sphere1->GetOutput()->GetNumberOfPoints(); ++i)
        scalars1->InsertNextValue(1.0);
    sphere1->GetOutput()->GetPointData()->AddArray(scalars1);

    vtkNew<vtkFloatArray> scalars2;
    scalars2->SetName("SourceID");
    for (vtkIdType i = 0; i < sphere2->GetOutput()->GetNumberOfPoints(); ++i)
        scalars2->InsertNextValue(2.0);
    sphere2->GetOutput()->GetPointData()->AddArray(scalars2);

    vtkNew<vtkFloatArray> scalars3;
    scalars3->SetName("SourceID");
    for (vtkIdType i = 0; i < sphere3->GetOutput()->GetNumberOfPoints(); ++i)
        scalars3->InsertNextValue(3.0);
    sphere3->GetOutput()->GetPointData()->AddArray(scalars3);

    // 3. 使用 vtkAppendPolyData 合并三个球体
    vtkNew<vtkAppendPolyData> appendFilter;
    appendFilter->AddInputData(sphere1->GetOutput());
    appendFilter->AddInputData(sphere2->GetOutput());
    appendFilter->AddInputData(sphere3->GetOutput());
    appendFilter->Update();

    // 4. 查找表：为每个来源分配不同颜色
    vtkNew<vtkLookupTable> lut;
    lut->SetNumberOfTableValues(3);
    lut->SetTableRange(1.0, 3.0);
    lut->SetTableValue(0, 1.0, 0.2, 0.2);  // 红色 -- 球体1
    lut->SetTableValue(1, 0.2, 0.8, 0.2);  // 绿色 -- 球体2
    lut->SetTableValue(2, 0.2, 0.2, 1.0);  // 蓝色 -- 球体3
    lut->Build();

    // 5. 渲染管线
    vtkNew<vtkPolyDataMapper> mapper;
    mapper->SetInputConnection(appendFilter->GetOutputPort());
    mapper->SetScalarRange(1.0, 3.0);
    mapper->SetLookupTable(lut);
    mapper->SelectColorArray("SourceID");

    vtkNew<vtkActor> actor;
    actor->SetMapper(mapper);

    vtkNew<vtkRenderer> renderer;
    renderer->AddActor(actor);
    renderer->SetBackground(0.1, 0.1, 0.12);

    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(800, 600);
    renderWindow->AddRenderer(renderer);
    renderWindow->SetWindowName("vtkAppendPolyData -- Merge 3 Spheres");

    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    renderer->ResetCamera();
    renderWindow->Render();
    interactor->Start();

    return 0;
}
```

运行此程序后，你将看到三个不同大小和颜色的球体已经被合并为一个单一的`vtkPolyData`——它们在一个Actor中被渲染，`SourceID`标量数组记录了每个面片来自哪个原始球体。

---

## 14.2 复合数据集（Composite Datasets）

上一节介绍的"合并"方法将所有输入数据融合为一个单一网格。当你需要**保持各数据块的独立性**、**层次化地组织数据**、或**按块控制渲染属性**时，你需要的是复合数据集（Composite Dataset）。

### 14.2.1 vtkMultiBlockDataSet -- 层次化数据组织

`vtkMultiBlockDataSet`是VTK中最重要和最常用的复合数据集类型。它将多个数据集组织成块（Blocks），每个块可以是任何`vtkDataObject`类型（包括`vtkPolyData`、`vtkUnstructuredGrid`、`vtkStructuredGrid`、`vtkImageData`，甚至嵌套的另一个`vtkMultiBlockDataSet`）。

**层次化结构**：`vtkMultiBlockDataSet`支持嵌套——一个块的内部也可以是一个`vtkMultiBlockDataSet`，从而形成树状的数据层次。这与你电脑上的文件夹结构非常相似：

```
vtkMultiBlockDataSet (根)
  ├── Block 0: vtkPolyData (引擎盖)
  ├── Block 1: vtkMultiBlockDataSet (底盘组件)
  │   ├── Block 0: vtkPolyData (左前轮)
  │   ├── Block 1: vtkPolyData (右前轮)
  │   ├── Block 2: vtkPolyData (左后轮)
  │   └── Block 3: vtkPolyData (右后轮)
  ├── Block 2: vtkPolyData (车身外壳)
  └── Block 3: vtkUnstructuredGrid (发动机CFD网格)
```

**核心API**：

```cpp
#include <vtkMultiBlockDataSet.h>

vtkNew<vtkMultiBlockDataSet> mb;

// 设置块的数量 -- 只在创建时需要用一次
mb->SetNumberOfBlocks(4);

// 设置第i个块的内容（块索引从0开始）
mb->SetBlock(0, polyDataEngine);     // 块0: 引擎盖
mb->SetBlock(1, chassisMB);          // 块1: 嵌套的MultiBlockDataSet（底盘）
mb->SetBlock(2, polyDataBody);       // 块2: 车身外壳
mb->SetBlock(3, unstructuredMesh);   // 块3: CFD体网格（注意：类型可以不同）

// 获取块的数量
unsigned int numBlocks = mb->GetNumberOfBlocks();

// 获取第i个块
vtkDataObject* block = mb->GetBlock(i);

// 判断第i个块是否为指定类型
if (mb->GetBlock(i)->IsA("vtkMultiBlockDataSet"))
{
    // 这是一个嵌套的复合数据集
    vtkMultiBlockDataSet* nested = vtkMultiBlockDataSet::SafeDownCast(block);
    // ... 递归处理
}

// 复制整个块结构（包括所有数据和层次嵌套）
vtkNew<vtkMultiBlockDataSet> copy;
copy->DeepCopy(mb);

// 将序号与块索引不同 -- 不必要，但可以设置
mb->SetBlock(2, someData);
// 注意：SetBlock(2, ...) 等价于 SetBlock(2, newData)，
// 会在内部储存数据的浅拷贝（引用计数增加）。
```

**块名称（Meta-Data）**：为了方便组织和查找，你可以为每个块设置名称。名称存储在块的field data中：

```cpp
// 方法1：通过vtkMultiBlockDataSet的便利方法
mb->GetMetaData(0)->Set(vtkCompositeDataSet::NAME(), "Engine Hood");
mb->GetMetaData(1)->Set(vtkCompositeDataSet::NAME(), "Chassis Assembly");
mb->GetMetaData(2)->Set(vtkCompositeDataSet::NAME(), "Body Shell");
mb->GetMetaData(3)->Set(vtkCompositeDataSet::NAME(), "CFD Volume Mesh");

// 方法2：手动创建元数据对象
vtkNew<vtkInformation> info;
info->Set(vtkCompositeDataSet::NAME(), "Engine Hood");
mb->SetMetaData(0, info);

// 读取块名称
const char* name = mb->GetMetaData(0)->Get(vtkCompositeDataSet::NAME());
if (name)
    std::cout << "Block 0 name: " << name << std::endl;
```

有了块名称之后，你可以遍历所有块并找到特定的块：

```cpp
// 通过名称查找块
for (unsigned int i = 0; i < mb->GetNumberOfBlocks(); ++i)
{
    const char* blockName = mb->GetMetaData(i)->Get(vtkCompositeDataSet::NAME());
    if (blockName && std::string(blockName) == "Body Shell")
    {
        vtkPolyData* body = vtkPolyData::SafeDownCast(mb->GetBlock(i));
        // 处理车身数据...
        break;
    }
}
```

**嵌套遍历**：当MultiBlockDataSet包含嵌套结构时，通常使用递归遍历：

```cpp
void TraverseMultiBlock(vtkMultiBlockDataSet* mb, int depth = 0)
{
    for (unsigned int i = 0; i < mb->GetNumberOfBlocks(); ++i)
    {
        // 打印缩进
        for (int d = 0; d < depth; ++d) std::cout << "  ";

        // 获取块名称
        const char* name = mb->GetMetaData(i)
            ? mb->GetMetaData(i)->Get(vtkCompositeDataSet::NAME()) : nullptr;

        vtkDataObject* block = mb->GetBlock(i);
        if (block->IsA("vtkMultiBlockDataSet"))
        {
            std::cout << "[" << i << "] "
                      << (name ? name : "(unnamed)")
                      << " -> MultiBlockDataSet" << std::endl;
            TraverseMultiBlock(
                vtkMultiBlockDataSet::SafeDownCast(block), depth + 1);
        }
        else
        {
            std::cout << "[" << i << "] "
                      << (name ? name : "(unnamed)")
                      << " -> " << block->GetClassName() << std::endl;
        }
    }
}
```

### 14.2.2 vtkPartitionedDataSetCollection -- 分区+集合模型

`vtkPartitionedDataSetCollection`是VTK 9.x中引入的另一种复合数据组织方式。它采用"分区集合"模型：

- 一个Collection包含多个**PartitionedDataSet**。
- 每个PartitionedDataSet包含多个**Partition**（分区）。
- 每个Partition是一个`vtkDataSet`。

这种模型的典型应用场景是**多材料/多相仿真**：每种材料对应一个PartitionedDataSet，该材料在不同处理器上的分区对应不同的Partition。

```cpp
#include <vtkPartitionedDataSet.h>
#include <vtkPartitionedDataSetCollection.h>

// 创建一个PartitionedDataSetCollection
vtkNew<vtkPartitionedDataSetCollection> collection;

// 第一种材料的分区数据
vtkNew<vtkPartitionedDataSet> material1;
material1->SetNumberOfPartitions(2);
material1->SetPartition(0, polyData_steel_part1);
material1->SetPartition(1, polyData_steel_part2);
material1->GetMetaData()->Set(vtkCompositeDataSet::NAME(), "Steel");

// 第二种材料的分区数据
vtkNew<vtkPartitionedDataSet> material2;
material2->SetNumberOfPartitions(3);
material2->SetPartition(0, polyData_aluminum_part1);
material2->SetPartition(1, polyData_aluminum_part2);
material2->SetPartition(2, polyData_aluminum_part3);
material2->GetMetaData()->Set(vtkCompositeDataSet::NAME(), "Aluminum");

// 将分区数据加入集合
collection->SetNumberOfPartitionedDataSets(2);
collection->SetPartitionedDataSet(0, material1);
collection->SetPartitionedDataSet(1, material2);

// 访问数据
vtkPartitionedDataSet* steel = collection->GetPartitionedDataSet(0);
vtkDataSet* steelPart0 = steel->GetPartition(0);
```

### 14.2.3 平铺合并 vs 层次化复合 -- 选择指南

| 需求 | 推荐方案 | 原因 |
|------|---------|------|
| 将所有部件焊接到单一网格 | `vtkAppendPolyData` / `vtkAppendFilter` | 产生一个连续的几何体，点和面片统一存储 |
| 保持部件独立但统一渲染 | `vtkMultiBlockDataSet` + `vtkCompositePolyDataMapper` | 每个块保持独立，可按块控制颜色/可见性 |
| 组织不同类型的仿真结果 | `vtkMultiBlockDataSet` | 支持在同一个结构中混合PolyData、UnstructuredGrid等 |
| 多材料多分区仿真 | `vtkPartitionedDataSetCollection` | 自然映射材料-分区的关系 |
| 多阶段合并处理（先合并再处理） | `vtkAppendDatasets` | 输出到PartitionedDataSetCollection，保留分区信息 |

简单原则：**需要"焊接"用Append，需要"保持独立"用MultiBlock。**

---

## 14.3 复合数据渲染

有了复合数据集，接下来的问题是：如何将它们渲染出来？VTK提供了一套专门的复合渲染管线来完成这个任务。

### 14.3.1 vtkCompositePolyDataMapper -- 复合数据的专用Mapper

标准的`vtkPolyDataMapper`只能处理一个`vtkPolyData`作为输入。对于`vtkMultiBlockDataSet`，你需要使用`vtkCompositePolyDataMapper`。

`vtkCompositePolyDataMapper`能够：
- 接受`vtkMultiBlockDataSet`作为输入（以及`vtkPartitionedDataSetCollection`等复合类型）。
- 自动递归遍历所有块，提取出其中的`vtkPolyData`块进行渲染。
- 支持**按块控制**——允许为每个块设置不同的颜色、可见性和不透明度。

**基本用法**：

```cpp
#include <vtkCompositePolyDataMapper.h>
#include <vtkMultiBlockDataSet.h>

// 创建一个MultiBlockDataSet（假设已经填充了各个块）
vtkNew<vtkMultiBlockDataSet> mb;
mb->SetNumberOfBlocks(2);
mb->SetBlock(0, polyDataPart1);
mb->SetBlock(1, polyDataPart2);

// 使用CompositePolyDataMapper
vtkNew<vtkCompositePolyDataMapper> mapper;
mapper->SetInputData(mb);  // 直接接受复合数据集

vtkNew<vtkActor> actor;
actor->SetMapper(mapper);

// 渲染照常进行
vtkNew<vtkRenderer> renderer;
renderer->AddActor(actor);
```

**重要提示**：`vtkCompositePolyDataMapper`只能渲染复合数据集中的`vtkPolyData`类型的块。如果某个块是`vtkUnstructuredGrid`或其他类型，该块将被跳过（不会报错，只是不渲染）。如果你需要渲染非PolyData的块，需要先将它们通过`vtkCompositeDataGeometryFilter`（见14.3.4节）转换为PolyData。

### 14.3.2 按块可见性控制

`vtkCompositePolyDataMapper`的一个核心功能是**按块可见性控制**——你可以独立地显示或隐藏`vtkMultiBlockDataSet`中的特定块，而不需要为每个块创建独立的Actor。

```cpp
vtkNew<vtkCompositePolyDataMapper> mapper;
mapper->SetInputData(mb);

// 获取所有可见块ID的列表（默认所有块可见）
// 使用SetBlockVisibility()设置特定块的可见性
mapper->SetBlockVisibility(0, true);   // 块0可见
mapper->SetBlockVisibility(1, false);  // 块1隐藏
mapper->SetBlockVisibility(2, true);   // 块2可见

// 查询块的可见性
bool vis0 = mapper->GetBlockVisibility(0);  // true
bool vis1 = mapper->GetBlockVisibility(1);  // false

// 递归移除所有可见性覆盖设置（所有块恢复可见）
mapper->RemoveBlockVisibilities();
```

**实现原理**：当某个块被设为不可见时，CompositePolyDataMapper在渲染过程中跳过该块的图元（不生成对应的OpenGL绘制命令）。数据本身保持不变——如果你后来将块设为可见，它会重新出现。

### 14.3.3 按块颜色和不透明度覆盖

除了可见性，`vtkCompositePolyDataMapper`还支持为每个块单独设置颜色和不透明度——即使这些块没有自己附带标量数据。

```cpp
vtkNew<vtkCompositePolyDataMapper> mapper;
mapper->SetInputData(mb);

// 设置按块颜色覆盖
// 颜色值范围 [0.0, 1.0]，不透明度范围 [0.0, 1.0]
mapper->SetBlockColor(0, 1.0, 0.2, 0.2);  // 块0: 红色
mapper->SetBlockColor(1, 0.2, 0.8, 0.2);  // 块1: 绿色
mapper->SetBlockColor(2, 0.2, 0.2, 1.0);  // 块2: 蓝色
mapper->SetBlockColor(3, 0.8, 0.8, 0.2);  // 块3: 黄色

// 设置按块不透明度覆盖
mapper->SetBlockOpacity(0, 1.0);   // 块0: 完全不透明
mapper->SetBlockOpacity(1, 0.5);   // 块1: 半透明
mapper->SetBlockOpacity(2, 0.8);   // 块2: 80%不透明
mapper->SetBlockOpacity(3, 0.3);   // 块3: 30%不透明（较为透明）

// 清除所有颜色覆盖
mapper->RemoveBlockColors();

// 清除所有不透明度覆盖
mapper->RemoveBlockOpacities();
```

**优先级规则**：
1. 如果为某块设置了块颜色覆盖，该块将使用覆盖颜色——忽略该块数据自带的标量数据（如果有）。
2. 如果没有设置块颜色覆盖，Mapper会检查块的标量数据，如果存在，就使用标量通过查找表映射颜色。
3. 如果没有标量数据也没有块颜色覆盖，则使用Actor的通用`GetProperty()->SetColor()`设置的颜色。
4. 块不透明度覆盖优先于Actor属性中的不透明度。

### 14.3.4 vtkCompositeDataGeometryFilter -- 展平复合数据

有时候你需要将复合数据集中所有的PolyData块"展平"（flatten）为一个单一的PolyData——例如为了保存为文件、传递给不支持复合数据集的过滤器，或者进行统一的后处理。`vtkCompositeDataGeometryFilter`正是为这个目的设计的。

```cpp
#include <vtkCompositeDataGeometryFilter.h>

vtkNew<vtkCompositeDataGeometryFilter> flatten;
flatten->SetInputData(mb);  // 输入可以是vtkMultiBlockDataSet等
flatten->Update();

vtkPolyData* allInOne = flatten->GetOutput();
std::cout << "Flattened points: " << allInOne->GetNumberOfPoints()
          << std::endl;
std::cout << "Flattened cells:  " << allInOne->GetNumberOfCells()
          << std::endl;
```

该过滤器会：
- 递归遍历所有块。
- 提取每个块中的PolyData几何。
- 将所有几何合并（append）到一个输出PolyData中。

与直接使用`vtkAppendPolyData`的区别在于：`vtkCompositeDataGeometryFilter`是专门为复合数据集设计的，它自动处理嵌套的层次结构和混合类型（跳过非PolyData的块），而不需要你手动展开每个块。

**一个典型的用例**：将渲染结果保存为STL文件：

```cpp
vtkNew<vtkCompositeDataGeometryFilter> flatten;
flatten->SetInputData(mb);
flatten->Update();

vtkNew<vtkSTLWriter> writer;
writer->SetFileName("assembly.stl");
writer->SetInputConnection(flatten->GetOutputPort());
writer->Write();
```

---

## 14.4 代码示例：多部件装配

本节提供一个完整的C++程序，演示本章涵盖的所有核心概念。程序构建一个"桌子"的多部件装配模型，使用`vtkMultiBlockDataSet`组织部件，通过`vtkCompositePolyDataMapper`渲染，并支持键盘交互切换各部件可见性。

### 14.4.1 程序概述

**模型结构**：一个"桌面+四条桌腿"的桌子模型。

```
vtkMultiBlockDataSet (根)
  ├── Block 0: "Table Top"    -- 桌面（立方体，缩放为扁平板状）
  ├── Block 1: "Leg FL"       -- 前左桌腿（圆柱体）
  ├── Block 2: "Leg FR"       -- 前右桌腿（圆柱体）
  ├── Block 3: "Leg BL"       -- 后左桌腿（圆柱体）
  └── Block 4: "Leg BR"       -- 后右桌腿（圆柱体）
```

**交互功能**：
- 按数字键 `1`-`5`：切换对应块的可见性（桌面和四条腿）。
- 按 `a` 键：切换所有块可见/隐藏。
- 按 `r` 键：将所有块重置为可见。
- 按 `c` 键：随机改变各块颜色。
- 按 `w` / `s` 键：旋转视角。

**渲染配置**：
- 使用`vtkCompositePolyDataMapper`进行复合渲染。
- 为每个块设置不同的固定颜色（通过按块颜色覆盖）。
- 显示块名称提示信息。

### 14.4.2 完整C++代码

```cpp
// ============================================================================
// Chapter 14: Multi-Dataset Processing
// File: TableAssemblyDemo.cxx
// Description: Comprehensive demonstration of VTK multi-dataset processing.
//              Builds a "table" assembly model with 5 named parts organized
//              in a vtkMultiBlockDataSet, renders with
//              vtkCompositePolyDataMapper, supports per-block visibility
//              toggle and color control via keyboard callbacks.
// VTK Version: 9.5.2
// ============================================================================

#include <vtkActor.h>
#include <vtkCamera.h>
#include <vtkCommand.h>
#include <vtkCompositePolyDataMapper.h>
#include <vtkCubeSource.h>
#include <vtkCylinderSource.h>
#include <vtkInteractorStyleTrackballCamera.h>
#include <vtkMinimalStandardRandomSequence.h>
#include <vtkMultiBlockDataSet.h>
#include <vtkNamedColors.h>
#include <vtkPolyData.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkSmartPointer.h>
#include <vtkTextActor.h>
#include <vtkTextProperty.h>
#include <vtkTransform.h>
#include <vtkTransformPolyDataFilter.h>
#include <vtkVersion.h>

#include <array>
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

// ============================================================================
// Global data shared between main() and the keyboard callback
// ============================================================================
struct AppContext
{
    vtkSmartPointer<vtkCompositePolyDataMapper> mapper;
    vtkSmartPointer<vtkRenderer>                renderer;
    vtkSmartPointer<vtkRenderWindow>            renderWindow;
    vtkSmartPointer<vtkMultiBlockDataSet>       multiBlock;
    vtkSmartPointer<vtkTextActor>               hudText;
    unsigned int                                numBlocks = 0;
    std::vector<double[3]>                      blockColors; // per-block colors
};

// ============================================================================
// Helper: Create a table-top slab from a cube source
// ============================================================================
vtkSmartPointer<vtkPolyData> CreateTableTop(
    double width, double depth, double height, double zPos)
{
    vtkNew<vtkCubeSource> cube;
    cube->SetXLength(width);
    cube->SetYLength(depth);
    cube->SetZLength(height);
    cube->SetCenter(0.0, 0.0, zPos);
    cube->Update();
    return cube->GetOutput();
}

// ============================================================================
// Helper: Create a cylindrical table leg
// ============================================================================
vtkSmartPointer<vtkPolyData> CreateLeg(
    double radius, double legHeight, double xPos, double yPos, double zPos)
{
    vtkNew<vtkCylinderSource> cyl;
    cyl->SetRadius(radius);
    cyl->SetHeight(legHeight);
    cyl->SetResolution(24);
    cyl->SetCenter(xPos, yPos, zPos);
    cyl->Update();
    return cyl->GetOutput();
}

// ============================================================================
// Helper: Update the HUD text showing block visibility status
// ============================================================================
void UpdateHUD(AppContext* ctx)
{
    std::ostringstream oss;
    oss << "Table Assembly -- MultiBlock Demo\n";
    oss << "================================\n";

    const char* blockNames[] = {
        "Table Top", "Leg FL", "Leg FR", "Leg BL", "Leg BR"
    };

    for (unsigned int i = 0; i < ctx->numBlocks; ++i)
    {
        bool visible = ctx->mapper->GetBlockVisibility(i);
        oss << "[" << (i + 1) << "] " << blockNames[i]
            << ": " << (visible ? "ON " : "OFF") << "\n";
    }

    oss << "--------------------------------\n";
    oss << "Keys: 1-5=toggle block | a=all | r=reset | c=colors";

    ctx->hudText->SetInput(oss.str().c_str());
}

// ============================================================================
// Keyboard callback: handles per-block visibility and color control
// ============================================================================
class AssemblyKeyboardCallback : public vtkCommand
{
public:
    static AssemblyKeyboardCallback* New()
    {
        return new AssemblyKeyboardCallback;
    }

    void SetContext(AppContext* ctx) { m_Context = ctx; }

    void Execute(vtkObject* caller, unsigned long eventId,
                 void* vtkNotUsed(callData)) override
    {
        vtkRenderWindowInteractor* interactor =
            static_cast<vtkRenderWindowInteractor*>(caller);

        std::string key = interactor->GetKeySym();

        bool toggleAll   = false;
        bool setAllOn    = false;
        bool randomColor = false;

        // ------------------------------------------------------------------
        // Process key input
        // ------------------------------------------------------------------
        if (key == "1" || key == "2" || key == "3" ||
            key == "4" || key == "5")
        {
            int idx = std::stoi(key) - 1; // zero-based block index
            if (static_cast<unsigned int>(idx) < m_Context->numBlocks)
            {
                bool cur = m_Context->mapper->GetBlockVisibility(idx);
                m_Context->mapper->SetBlockVisibility(idx, !cur);
            }
        }
        else if (key == "a" || key == "A")
        {
            // Toggle all: if any block is visible, hide all; else show all
            bool anyVisible = false;
            for (unsigned int i = 0; i < m_Context->numBlocks; ++i)
            {
                if (m_Context->mapper->GetBlockVisibility(i))
                {
                    anyVisible = true;
                    break;
                }
            }
            for (unsigned int i = 0; i < m_Context->numBlocks; ++i)
                m_Context->mapper->SetBlockVisibility(i, !anyVisible);
        }
        else if (key == "r" || key == "R")
        {
            // Reset: show all blocks
            for (unsigned int i = 0; i < m_Context->numBlocks; ++i)
                m_Context->mapper->SetBlockVisibility(i, true);
        }
        else if (key == "c" || key == "C")
        {
            // Randomize per-block colors
            vtkNew<vtkMinimalStandardRandomSequence> rng;
            rng->SetSeed(static_cast<int>(m_Context->renderWindow->GetNumberOfLayers() + 42));

            for (unsigned int i = 0; i < m_Context->numBlocks; ++i)
            {
                double r, g, b;
                rng->Next(); r = rng->GetValue();
                rng->Next(); g = rng->GetValue();
                rng->Next(); b = rng->GetValue();

                m_Context->blockColors[i][0] = r;
                m_Context->blockColors[i][1] = g;
                m_Context->blockColors[i][2] = b;

                m_Context->mapper->SetBlockColor(i, r, g, b);
            }
        }

        // ------------------------------------------------------------------
        // Update HUD and re-render
        // ------------------------------------------------------------------
        UpdateHUD(m_Context);
        m_Context->renderWindow->Render();

        // Print status to console
        const char* blk[] = {"Top", "FL", "FR", "BL", "BR"};
        std::cout << "[Assembly] Visibility: ";
        for (unsigned int i = 0; i < m_Context->numBlocks; ++i)
        {
            std::cout << blk[i] << ":"
                      << (m_Context->mapper->GetBlockVisibility(i) ? "+" : "-")
                      << " ";
        }
        std::cout << std::endl;
    }

private:
    AppContext* m_Context = nullptr;
};

// ============================================================================
// Main
// ============================================================================
int main()
{
    std::cout << "VTK Version: " << VTK_VERSION << std::endl;
    std::cout << "=== Chapter 14: Table Assembly Demo ===" << std::endl;
    std::cout << "Multi-Block DataSet + CompositePolyDataMapper"
              << std::endl;
    std::cout << std::endl;

    // ========================================================================
    // 1. Build the table model: 1 top + 4 legs
    // ========================================================================

    // --- Table Top ---
    // A flat slab: width 4.0, depth 2.5, thickness 0.3
    vtkSmartPointer<vtkPolyData> top =
        CreateTableTop(4.0, 2.5, 0.3, 2.0);

    // --- Table Legs ---
    // Four cylindrical legs positioned at corners of the table
    // Table center is at (0,0,2.0), top thickness is 0.3, half=0.15
    // Leg top is at z = 2.0 - 0.15 = 1.85
    // Leg height = 1.7, so leg center z = 1.85 - 0.85 = 1.0
    double legRadius  = 0.15;
    double legHeight  = 1.7;
    double topHalfX   = 1.7;  // half of width minus leg radius margin
    double topHalfY   = 1.0;  // half of depth minus leg radius margin
    double legCenterZ = 1.0;

    vtkSmartPointer<vtkPolyData> legFL = CreateLeg(
        legRadius, legHeight, -topHalfX, -topHalfY, legCenterZ);
    vtkSmartPointer<vtkPolyData> legFR = CreateLeg(
        legRadius, legHeight,  topHalfX, -topHalfY, legCenterZ);
    vtkSmartPointer<vtkPolyData> legBL = CreateLeg(
        legRadius, legHeight, -topHalfX,  topHalfY, legCenterZ);
    vtkSmartPointer<vtkPolyData> legBR = CreateLeg(
        legRadius, legHeight,  topHalfX,  topHalfY, legCenterZ);

    // ========================================================================
    // 2. Assemble into vtkMultiBlockDataSet with named blocks
    // ========================================================================
    vtkNew<vtkMultiBlockDataSet> multiBlock;
    multiBlock->SetNumberOfBlocks(5);

    multiBlock->SetBlock(0, top);
    multiBlock->SetBlock(1, legFL);
    multiBlock->SetBlock(2, legFR);
    multiBlock->SetBlock(3, legBL);
    multiBlock->SetBlock(4, legBR);

    // Name all blocks for reference
    multiBlock->GetMetaData(0)->Set(vtkCompositeDataSet::NAME(), "Table Top");
    multiBlock->GetMetaData(1)->Set(vtkCompositeDataSet::NAME(), "Leg FL");
    multiBlock->GetMetaData(2)->Set(vtkCompositeDataSet::NAME(), "Leg FR");
    multiBlock->GetMetaData(3)->Set(vtkCompositeDataSet::NAME(), "Leg BL");
    multiBlock->GetMetaData(4)->Set(vtkCompositeDataSet::NAME(), "Leg BR");

    // Print the hierarchy
    std::cout << "MultiBlock structure:" << std::endl;
    for (unsigned int i = 0; i < multiBlock->GetNumberOfBlocks(); ++i)
    {
        const char* name =
            multiBlock->GetMetaData(i)->Get(vtkCompositeDataSet::NAME());
        vtkPolyData* pd =
            vtkPolyData::SafeDownCast(multiBlock->GetBlock(i));
        std::cout << "  Block " << i << " (" << (name ? name : "?") << "): "
                  << pd->GetNumberOfPoints() << " points, "
                  << pd->GetNumberOfCells()  << " cells" << std::endl;
    }

    // ========================================================================
    // 3. Set up per-block colors
    // ========================================================================
    // We use fixed colors for the table parts:
    //   Top:  warm wood brown
    //   Legs: slightly varied browns for realism
    std::array<std::array<double, 3>, 5> colors = {{
        {{0.545, 0.271, 0.075}},  // Table Top: saddle brown
        {{0.627, 0.322, 0.176}},  // Leg FL
        {{0.627, 0.322, 0.176}},  // Leg FR
        {{0.627, 0.322, 0.176}},  // Leg BL
        {{0.627, 0.322, 0.176}}   // Leg BR
    }};

    // ========================================================================
    // 4. Render with vtkCompositePolyDataMapper
    // ========================================================================
    vtkNew<vtkCompositePolyDataMapper> mapper;
    mapper->SetInputData(multiBlock);

    // Apply per-block colors
    for (unsigned int i = 0; i < 5; ++i)
    {
        mapper->SetBlockColor(i,
            colors[i][0], colors[i][1], colors[i][2]);
    }

    vtkNew<vtkActor> actor;
    actor->SetMapper(mapper);
    actor->GetProperty()->SetSpecular(0.3);
    actor->GetProperty()->SetSpecularPower(30.0);

    // ========================================================================
    // 5. Set up renderer and render window
    // ========================================================================
    vtkNew<vtkRenderer> renderer;
    renderer->AddActor(actor);
    renderer->SetBackground(0.85, 0.85, 0.88); // light gray-blue background

    // Add a subtle gradient background
    renderer->SetBackground2(0.35, 0.35, 0.45);
    renderer->SetGradientBackground(true);

    // Set up camera
    renderer->GetActiveCamera()->SetPosition(6.0, 5.0, 4.5);
    renderer->GetActiveCamera()->SetFocalPoint(0.0, 0.0, 1.0);
    renderer->GetActiveCamera()->SetViewUp(0.0, 0.0, 1.0);
    renderer->ResetCamera();

    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(1024, 768);
    renderWindow->AddRenderer(renderer);
    renderWindow->SetWindowName("Chapter 14: Table Assembly -- "
                                "MultiBlock + CompositePolyDataMapper");

    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    // Use trackball camera style for easy rotation
    vtkNew<vtkInteractorStyleTrackballCamera> style;
    interactor->SetInteractorStyle(style);

    // ========================================================================
    // 6. Set up HUD text overlay
    // ========================================================================
    vtkNew<vtkTextActor> hudText;
    hudText->SetDisplayPosition(20, 20);
    hudText->GetTextProperty()->SetFontSize(14);
    hudText->GetTextProperty()->SetFontFamilyToCourier();
    hudText->GetTextProperty()->SetColor(0.1, 0.1, 0.15);
    hudText->GetTextProperty()->SetOpacity(0.8);
    renderer->AddActor2D(hudText);

    // ========================================================================
    // 7. Set up keyboard callbacks
    // ========================================================================
    AppContext ctx;
    ctx.mapper       = mapper;
    ctx.renderer     = renderer;
    ctx.renderWindow = renderWindow;
    ctx.multiBlock   = multiBlock;
    ctx.hudText      = hudText;
    ctx.numBlocks    = 5;

    // Store initial colors
    for (int i = 0; i < 5; ++i)
    {
        ctx.blockColors.push_back({colors[i][0],
                                   colors[i][1],
                                   colors[i][2]});
    }

    vtkNew<AssemblyKeyboardCallback> keyCallback;
    keyCallback->SetContext(&ctx);
    interactor->AddObserver(vtkCommand::KeyPressEvent, keyCallback);

    // Initialize HUD
    UpdateHUD(&ctx);

    // ========================================================================
    // 8. Display usage guide
    // ========================================================================
    std::cout << std::endl;
    std::cout << "Interactive Controls:" << std::endl;
    std::cout << "  1-5 : Toggle visibility of block 1-5" << std::endl;
    std::cout << "  a   : Toggle all blocks on/off" << std::endl;
    std::cout << "  r   : Reset all blocks to visible" << std::endl;
    std::cout << "  c   : Randomize per-block colors" << std::endl;
    std::cout << "  Mouse drag : Rotate view" << std::endl;
    std::cout << "  Mouse wheel: Zoom in/out" << std::endl;
    std::cout << std::endl;

    // ========================================================================
    // 9. Start the render loop
    // ========================================================================
    renderWindow->Render();
    interactor->Initialize();
    interactor->Start();

    return 0;
}
```

### 14.4.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.12)
project(Chapter14_TableAssemblyDemo)

# Find VTK (required components for this chapter)
find_package(VTK REQUIRED COMPONENTS
    CommonCore
    CommonDataModel
    CommonExecutionModel
    CommonMath
    CommonTransforms
    FiltersCore
    FiltersSources
    InteractionStyle
    RenderingAnnotation
    RenderingContextOpenGL2
    RenderingCore
    RenderingFreeType
    RenderingOpenGL2
)

# Print VTK information
message(STATUS "VTK_VERSION: ${VTK_VERSION}")
message(STATUS "VTK_DIR: ${VTK_DIR}")

# Create executable
add_executable(TableAssemblyDemo TableAssemblyDemo.cxx)

# Link VTK libraries
target_link_libraries(TableAssemblyDemo
    VTK::CommonCore
    VTK::CommonDataModel
    VTK::CommonExecutionModel
    VTK::CommonMath
    VTK::CommonTransforms
    VTK::FiltersCore
    VTK::FiltersSources
    VTK::InteractionStyle
    VTK::RenderingAnnotation
    VTK::RenderingContextOpenGL2
    VTK::RenderingCore
    VTK::RenderingFreeType
    VTK::RenderingOpenGL2
)

# Set C++ standard
set_target_properties(TableAssemblyDemo PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
)
```

### 14.4.4 编译与运行

**Windows (Visual Studio):**

```powershell
# 在源代码目录下
mkdir build
cd build
cmake .. -G "Visual Studio 17 2022"
cmake --build . --config Release

# 运行
.\Release\TableAssemblyDemo.exe
```

**Linux / macOS:**

```bash
mkdir build && cd build
cmake ..
cmake --build .
./TableAssemblyDemo
```

### 14.4.5 预期结果分析

运行程序后，你将看到一个木色桌子模型位于渲染窗口中央，左上角有黑色的HUD文本显示各块的可见性状态：

**视觉呈现**：
- 一块扁平的矩形桌面（由`vtkCubeSource`缩放而成），呈现温暖的马鞍棕色。
- 四条圆柱形桌腿（由`vtkCylinderSource`生成）均匀分布在桌面四角下方。
- 所有部件合并渲染在一个Actor中，但每个部件保持独立的可见性和颜色控制。

**交互体验**：

| 操作 | 预期效果 |
|------|---------|
| 按 `1` | 桌面消失/重新出现。桌腿仍然可见。 |
| 按 `2`-`5` | 对应桌腿独立地消失/重新出现。 |
| 按 `a` | 如果所有块可见，全部隐藏；如果任意块隐藏，全部显示。 |
| 按 `r` | 所有5个块恢复为可见状态，无论之前的状态如何。 |
| 按 `c` | 每个块获得一个随机的RGB颜色，颜色覆盖立即生效。 |
| 鼠标拖拽 | 围绕模型旋转视角（Trackball Camera）。 |
| 鼠标滚轮 | 放大/缩小视图。 |

**关键观察**：

1. **CompositePolyDataMapper的透明切换**——隐藏和显示一个块不需要重新构建渲染数据，切换瞬时完成。
2. **按块颜色覆盖优先于标量着色**——虽然各数据块没有附带标量数据，但块颜色覆盖使其正常运行。
3. **一个Actor管理多个部件**——所有5个部件共享同一个Actor，这意味着共享同一个变换（如果要移动桌子，只需操作一个Actor）。
4. **嵌套能力**——虽然本示例的MultiBlockDataSet是扁平的（只有一层），但`vtkCompositePolyDataMapper`完全支持递归嵌套的MultiBlockDataset。

---

## 14.5 本章小结

本章系统性地介绍了VTK中多数据集处理的两大机制：合并（Append）和复合（Composite）。以下是本章的核心要点回顾：

### 核心概念

1. **合并（Append）**是将多个数据集的内容拼接为一个连续网格的操作。适用于将所有部件"焊接"成单一几何体的场景，或需要统一后处理的场景。`vtkAppendPolyData`是最常用的PolyData合并工具，`vtkAppendFilter`适用于同类型数据集，`vtkAppendDatasets`可混合不同类型但输出为分区集合。

2. **复合数据集（Composite Dataset）**是保持各数据块独立性的层次化数据组织形式。`vtkMultiBlockDataSet`是最核心的复合数据集类，支持嵌套、混合类型，并为每个块提供元数据（名称等）。

3. **复合渲染管线**通过`vtkCompositePolyDataMapper`实现。该Mapper能够遍历复合数据集中的所有PolyData块进行渲染，并支持按块控制可见性、颜色和不透明度。

4. **按块控制**是复合数据渲染的核心优势——你可以在单个Actor中独立地控制每个部件的视觉属性，而不需要创建和维护多个Actor。

### 关键类与API

| 类 | 主要用途 | 关键方法 |
|----|---------|---------|
| `vtkAppendPolyData` | 合并多个PolyData | `AddInputData()`, `SetTolerance()`, `Update()`, `GetOutput()` |
| `vtkAppendFilter` | 合并同类型DataSet | `AddInputData()`, `Update()`, `GetOutput()` |
| `vtkAppendDatasets` | 合并不同类型DataSet | `AddInputData()`, `Update()`, `GetOutput()` |
| `vtkMultiBlockDataSet` | 层次化数据组织 | `SetNumberOfBlocks()`, `SetBlock()`, `GetBlock()`, `GetNumberOfBlocks()`, `GetMetaData()` |
| `vtkPartitionedDataSetCollection` | 分区集合模型 | `SetNumberOfPartitionedDataSets()`, `SetPartitionedDataSet()`, `GetPartitionedDataSet()` |
| `vtkCompositePolyDataMapper` | 复合数据渲染 | `SetInputData()`, `SetBlockVisibility()`, `SetBlockColor()`, `SetBlockOpacity()`, `RemoveBlockVisibilities()`, `RemoveBlockColors()` |
| `vtkCompositeDataGeometryFilter` | 展平复合数据 | `SetInputData()`, `Update()`, `GetOutput()` |
| `vtkCompositeDataSet` (基类) | 复合数据基类 | `NAME()` (元数据键), `GetNumberOfChildren()` |

### 选择指南

- **何时使用Append（合并）**：当所有部件构成一个单一连续的几何体，且你不需要在渲染后独立操控单个部件时。例如：将多个扫描文件合并为一个点云、合并CFD域分解的各块为一个网格、合并拆分保存的CAD模型。

- **何时使用MultiBlockDataSet（复合）**：当各部件需要保持独立性时。例如：多部件装配体（各零件独立操控）、多物理场结果（各物理场使用不同类型的数据集）、多时间步数据（每个时间步作为一个块）、嵌套的多级装配结构。

- **按块控制 vs 多Actor方案**：
  - 按块控制（CompositePolyDataMapper）：所有部件共享同一个Actor和变换。适合部件之间有固定空间关系的装配体。
  - 多Actor方案：每个部件独立的Actor。适合部件需要独立移动/旋转/缩放，或需要不同的渲染属性（如不同的光照参数）的场景。

### 与实践的联系

你现在掌握的技能可以用于：

- 构建CAD装配体的交互式可视化工具，支持独立切换零件可见性
- 组织仿真结果——流场（UnstructuredGrid）和几何表面（PolyData）共存于同一层次结构
- 处理分块保存的大型数据集——先独立加载每个块，再组织为MultiBlock
- 创建"分解视图"（爆炸视图）——先按块组织，再通过块变换实现分解
- 实现层次化的场景图管理，支持嵌套的子装配体

### 下一步

在掌握了多数据集处理后，你已经具备了组织复杂可视化场景的能力。下一章将介绍更高级的数据处理技术——包括自定义过滤器、流线可视化、以及高级交互技术，帮助你构建更专业的科学可视化应用程序。
