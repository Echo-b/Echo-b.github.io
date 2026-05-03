# 第十章 数据I/O（Data I/O）

## 本章导读

你在前九章中学习了VTK的完整管道架构、五种核心数据类型（PolyData、UnstructuredGrid、ImageData、RectilinearGrid、StructuredGrid）的创建与操作，以及渲染与交互的全部基础知识。但所有这些章节中，数据都是在程序内部创建的——你使用`vtkSphereSource`生成球体，用`vtkPoints`手动构建网格，用数学函数填充标量场。

然而在真实世界的科学可视化工作中，数据几乎从来不是在程序内部产生的。它可能来自：

- **CT/MRI扫描仪**导出的DICOM序列（医学影像）
- **CFD求解器**输出的计算结果（计算流体力学）
- **3D扫描仪**捕获的点云和网格（逆向工程与文物保护）
- **地理信息系统**导出的高程数据（数字地形模型）
- **CAD软件**保存的几何模型（工程设计与制造）
- **合作伙伴或公开数据集**提供的标准格式文件

本章就是连接"自己造数据"与"读取真实数据"之间的桥梁。你将学习VTK的文件I/O体系——如何将内存中的数据集持久化到磁盘，以及如何从磁盘加载数据进入VTK的可视化管线。

本章的核心目标有四个：

1. **理解VTK的两大文件格式体系**——Legacy格式与XML格式的设计理念、文件扩展名与数据集类型的映射关系，以及各自的适用场景。
2. **掌握各类型数据的读写API**——PolyData（.vtp/.stl/.obj/.ply）、UnstructuredGrid（.vtu）、结构化数据（.vti/.vtr/.vts），以及常用图像格式（PNG/JPEG/BMP/DICOM）。
3. **学会在实际工作流中选择合适的文件格式**——根据数据类型、文件大小、可调试性、压缩需求等因素作出判断。
4. **通过一个完整的数据管线示例整合所有知识**——创建数据、写入文件、读回、施加过滤器、双视口对比渲染。

读完本章后，你将能够自信地处理VTK项目中的文件I/O需求——无论是保存计算结果、加载外部数据，还是在不同格式之间进行转换。

---

## 10.1 VTK的文件格式体系

### 10.1.1 两大文件格式家族

VTK拥有两套主要的数据文件格式体系：**Legacy格式**（也称为"VTK Simple格式"或"Old格式"）和**XML格式**（也称为"VTK XML格式"或"New格式"）。理解这两种格式的区别是使用VTK I/O的第一步。

#### Legacy格式

Legacy格式是VTK最古老的文件格式，至今仍被广泛支持。它的特征是：

- **文件扩展名**：统一使用`.vtk`，无论存储的是哪种数据集类型。
- **文本可读性**：默认使用ASCII文本格式，可以用任何文本编辑器打开和查看。
- **简单结构**：文件包含一个头部（版本号和描述字符串），后跟按关键字分节的数据块（`POINTS`、`CELLS`、`CELL_TYPES`、`POINT_DATA`、`CELL_DATA`等）。
- **单数据集**：每个`.vtk`文件只能存储一个数据集。
- **支持的格式变体**：`vtkPolyDataReader/Writer`、`vtkUnstructuredGridReader/Writer`、`vtkStructuredPointsReader/Writer`、`vtkRectilinearGridReader/Writer`、`vtkStructuredGridReader/Writer`。

一个典型Legacy格式文件的简化结构如下：

```
# vtk DataFile Version 3.0
My Data
ASCII
DATASET POLYDATA
POINTS 4 float
0.0 0.0 0.0
1.0 0.0 0.0
...
```

**优点**：人类可读、易于调试、向后兼容性好、几乎所有VTK版本和ParaView版本都支持。

**缺点**：ASCII大文件读写慢（文件体积可以是二进制的5-10倍），不支持随机访问，不支持并行I/O，不支持内联压缩。每个文件只能存储一个数据集——如果要存储多个相关的网格（例如流体仿真的多区域网格），需要使用多个文件。

#### XML格式

XML格式是VTK的现代文件格式，推荐用于所有新项目。它的特征是：

- **文件扩展名多样**：根据数据类型使用不同的扩展名（`.vtp`、`.vtu`、`.vti`、`.vtr`、`.vts`、`.vtm`），从扩展名可以直接判断文件内容的数据类型。
- **XML结构**：基于XML标记语言，支持层级嵌套的数据组织。
- **多编码方式**：支持ASCII、Binary、Appended（二进制数据集中附加在XML末尾，支持随机访问）三种编码模式。
- **内置压缩**：支持zlib压缩（通过`compressor="vtkZLibDataCompressor"`属性），可显著减小文件体积。
- **并行I/O支持**：通过`.pvti`、`.pvtu`等并行格式，每个进程可以读写文件的一个分块。
- **多块支持**：`.vtm`格式可以包含多个不同子数据集的引用。

**优点**：文件体积小（二进制+压缩），支持并行I/O，随机访问能力（Appended模式），扩展名即文档（一眼就知道数据类型），适合大型数据集和生产环境。

**缺点**：ASCII模式下XML标签本身会占据可观的冗余空间（这一点上Legacy格式更简洁）。格式较Legacy更复杂，手工编写XML文件较为繁琐。

### 10.1.2 推荐策略

| 场景 | 推荐格式 |
|------|---------|
| 新项目、大型数据集、生产环境 | XML格式（二进制 + zlib压缩） |
| 快速原型、手工编辑数据、与旧版VTK互操作 | Legacy格式（ASCII） |
| 并行计算、分布式可视化 | XML并行格式（.pvti/.pvtu/.pvtr/.pvts） |
| ParaView数据管线导出/导入 | XML格式（ParaView原生支持） |
| 需要肉眼检查数据内容进行调试 | Legacy ASCII 或 XML ASCII |

**一般原则**：为所有新项目使用XML格式，只有在需要与旧版工具链互操作时才使用Legacy格式。

### 10.1.3 扩展名与数据集类型映射

| XML扩展名 | Legacy格式 | 数据集类型 | 典型应用场景 |
|-----------|-----------|-----------|------------|
| `.vtp` | `DATASET POLYDATA` | `vtkPolyData` | 表面网格、点云、CAD模型、等值面 |
| `.vtu` | `DATASET UNSTRUCTURED_GRID` | `vtkUnstructuredGrid` | 有限元网格、任意拓扑CFD结果 |
| `.vti` | `DATASET STRUCTURED_POINTS` | `vtkImageData` | CT/MRI体数据、气象均匀网格 |
| `.vtr` | `DATASET RECTILINEAR_GRID` | `vtkRectilinearGrid` | CFD边界层网格、海洋分层模型 |
| `.vts` | `DATASET STRUCTURED_GRID` | `vtkStructuredGrid` | CFD贴体网格、变形/弯曲网格 |
| `.vtm` | （无Legacy对应） | `vtkMultiBlockDataSet` | 多区域仿真结果、装配体 |
| `.vtk` | （所有Legacy类型共用） | 任意类型（由文件内DATASET关键字指定） | Legacy互操作 |

**记忆诀窍**：XML扩展名的中间字母表示数据类型——`p`=PolyData, `u`=UnstructuredGrid, `i`=ImageData, `r`=RectilinearGrid, `s`=StructuredGrid, `m`=MultiBlock。

---

## 10.2 多边形数据读写（PolyData I/O）

`vtkPolyData`是最常用的数据类型，也是拥有最多文件格式支持的数据类型。本节涵盖VTK原生XML格式、工业标准STL格式、通用图形学OBJ格式，以及点云常用PLY格式的读写。

### 10.2.1 格式概览

| 格式 | 扩展名 | Reader类 | Writer类 | 编码 | 适用领域 |
|------|--------|---------|---------|------|---------|
| VTK XML PolyData | `.vtp` | `vtkXMLPolyDataReader` | `vtkXMLPolyDataWriter` | ASCII/Binary/Appended | VTK原生互操作 |
| VTK Legacy PolyData | `.vtk` | `vtkPolyDataReader` | `vtkPolyDataWriter` | ASCII/Binary | Legacy互操作 |
| STL（Stereolithography） | `.stl` | `vtkSTLReader` | `vtkSTLWriter` | ASCII/Binary | 3D打印、CAD |
| OBJ（Wavefront） | `.obj` | `vtkOBJReader` | `vtkOBJWriter` | ASCII | 图形学、材质纹理 |
| PLY（Stanford Polygon） | `.ply` | `vtkPLYReader` | `vtkPLYWriter` | ASCII/Binary | 点云、3D扫描 |

### 10.2.2 VTK XML PolyData读写

这是VTK生态中最推荐的PolyData I/O方式：

```cpp
#include "vtkXMLPolyDataReader.h"
#include "vtkXMLPolyDataWriter.h"
#include "vtkPolyData.h"
#include "vtkNew.h"

// === 写入 .vtp 文件 ===
void WritePolyData(vtkPolyData* data, const char* filename)
{
  vtkNew<vtkXMLPolyDataWriter> writer;
  writer->SetFileName(filename);
  writer->SetInputData(data);
  // writer->SetDataModeToBinary();      // 默认二进制，体积最小
  // writer->SetDataModeToAscii();       // ASCII模式，可读
  // writer->SetDataModeToAppended();    // Appended二进制，支持随机访问
  // writer->SetCompressorTypeToZLib();  // 启用zlib压缩（仅Binary和Appended模式）
  writer->Write();
  std::cout << "Written: " << filename << std::endl;
}

// === 读取 .vtp 文件 ===
vtkSmartPointer<vtkPolyData> ReadPolyData(const char* filename)
{
  vtkNew<vtkXMLPolyDataReader> reader;
  reader->SetFileName(filename);
  reader->Update();
  vtkSmartPointer<vtkPolyData> data = reader->GetOutput();
  std::cout << "Read: " << filename
            << " (" << data->GetNumberOfPoints() << " points, "
            << data->GetNumberOfCells() << " cells)" << std::endl;
  return data;
}
```

**Writer关键选项说明**：

- **`SetDataModeToBinary()`**：二进制编码，文件体积最小。默认模式。数据以Base64编码内嵌在XML中（但本质仍是二进制）。
- **`SetDataModeToAscii()`**：ASCII编码，文件可直接用文本编辑器阅读，适合调试和教学场景。
- **`SetDataModeToAppended()`**：将数据以二进制形式集中附加在XML末尾，实现随机数据块的访问能力。适合需要只读取文件中某一部分数据的高级场景。
- **`SetCompressorTypeToZLib()`**：启用zlib压缩（需要VTK_COMMON_COMPUTATIONAL_GEOMETRY模块的编译支持）。配合Binary/Appended模式使用，可显著减小文件体积（通常压缩比可达3-10倍）。

### 10.2.3 STL格式

STL（STereoLithography）是3D打印和快速成型领域最通用的格式。一个STL文件存储一个三角形面片的集合——即一个`vtkPolyData`，其中所有单元都是`VTK_TRIANGLE`。

```cpp
#include "vtkSTLReader.h"
#include "vtkSTLWriter.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkProperty.h"

// === 完整的STL文件读取与渲染示例 ===
int main()
{
  // ---------- 步骤1：读取STL文件 ----------
  vtkNew<vtkSTLReader> reader;
  reader->SetFileName("example.stl");
  // 注意：如果STL文件不使用标准三角形顶点的法向量约定，
  // 可以在读取后通过vtkPolyDataNormals重新计算法向量。
  reader->Update();

  vtkPolyData* mesh = reader->GetOutput();
  std::cout << "STL Model: " << mesh->GetNumberOfPoints() << " points, "
            << mesh->GetNumberOfCells() << " triangles" << std::endl;

  // ---------- 步骤2：渲染管线 ----------
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputData(mesh);
  // 如果STL文件不包含法向量，取消下面这行的注释：
  // mapper->SetInputConnection(reader->GetOutputPort());
  // 并在reader和mapper之间插入vtkPolyDataNormals

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(0.6, 0.6, 0.7);     // 银灰色
  actor->GetProperty()->SetSpecular(0.5);             // 金属高光
  actor->GetProperty()->SetSpecularPower(40);          // 高光集中
  actor->GetProperty()->SetEdgeVisibility(0);

  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->SetBackground(0.15, 0.15, 0.18);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(800, 600);
  renderWindow->SetWindowName("STL Viewer");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  renderWindow->Render();
  std::cout << "Rotate to inspect the model. Close window to exit." << std::endl;
  interactor->Start();

  return 0;
}
```

### 10.2.4 格式转换示例

将一种PolyData格式转换为另一种是常见的操作需求。以下是最小化的格式转换程序——从任意支持的格式读取，写入任意支持的格式。

```cpp
// format_converter.cxx
// 通用PolyData格式转换器：STL <-> OBJ <-> PLY <-> VTP
//
// 用法：FormatConverter input.stl output.obj

#include "vtkNew.h"
#include "vtkPolyData.h"
#include "vtkSTLReader.h"
#include "vtkSTLWriter.h"
#include "vtkOBJReader.h"
#include "vtkOBJWriter.h"
#include "vtkPLYReader.h"
#include "vtkPLYWriter.h"
#include "vtkXMLPolyDataReader.h"
#include "vtkXMLPolyDataWriter.h"

#include <algorithm>
#include <cstring>
#include <iostream>
#include <string>

// 根据扩展名获取字符串的小写版本
std::string GetExtension(const std::string& filename)
{
  auto pos = filename.find_last_of('.');
  if (pos == std::string::npos) return "";
  std::string ext = filename.substr(pos);
  std::transform(ext.begin(), ext.end(), ext.begin(), ::tolower);
  return ext;
}

// 根据扩展名选择合适的Reader
vtkSmartPointer<vtkPolyData> ReadFile(const std::string& filename)
{
  std::string ext = GetExtension(filename);
  if (ext == ".vtp")
  {
    vtkNew<vtkXMLPolyDataReader> reader;
    reader->SetFileName(filename.c_str());
    reader->Update();
    return reader->GetOutput();
  }
  else if (ext == ".stl")
  {
    vtkNew<vtkSTLReader> reader;
    reader->SetFileName(filename.c_str());
    reader->Update();
    return reader->GetOutput();
  }
  else if (ext == ".obj")
  {
    vtkNew<vtkOBJReader> reader;
    reader->SetFileName(filename.c_str());
    reader->Update();
    return reader->GetOutput();
  }
  else if (ext == ".ply")
  {
    vtkNew<vtkPLYReader> reader;
    reader->SetFileName(filename.c_str());
    reader->Update();
    return reader->GetOutput();
  }
  else
  {
    std::cerr << "Unsupported input format: " << ext << std::endl;
    return nullptr;
  }
}

// 根据扩展名选择合适的Writer
bool WriteFile(vtkPolyData* data, const std::string& filename)
{
  std::string ext = GetExtension(filename);
  if (ext == ".vtp")
  {
    vtkNew<vtkXMLPolyDataWriter> writer;
    writer->SetFileName(filename.c_str());
    writer->SetInputData(data);
    writer->Write();
  }
  else if (ext == ".stl")
  {
    vtkNew<vtkSTLWriter> writer;
    writer->SetFileName(filename.c_str());
    writer->SetInputData(data);
    writer->Write();
  }
  else if (ext == ".obj")
  {
    vtkNew<vtkOBJWriter> writer;
    writer->SetFileName(filename.c_str());
    writer->SetInputData(data);
    writer->Write();
  }
  else if (ext == ".ply")
  {
    vtkNew<vtkPLYWriter> writer;
    writer->SetFileName(filename.c_str());
    writer->SetInputData(data);
    writer->Write();
  }
  else
  {
    std::cerr << "Unsupported output format: " << ext << std::endl;
    return false;
  }
  return true;
}

int main(int argc, char* argv[])
{
  if (argc != 3)
  {
    std::cout << "Usage: FormatConverter <input> <output>\n"
              << "  Supports: .vtp, .stl, .obj, .ply\n"
              << "  Example: FormatConverter model.stl model.obj\n";
    return 1;
  }

  std::string inputFile  = argv[1];
  std::string outputFile = argv[2];

  std::cout << "Converting: " << inputFile << " -> " << outputFile << std::endl;

  auto data = ReadFile(inputFile);
  if (!data)
  {
    std::cerr << "Failed to read input file." << std::endl;
    return 1;
  }

  std::cout << "  Input:  " << data->GetNumberOfPoints() << " points, "
            << data->GetNumberOfCells() << " cells" << std::endl;

  if (!WriteFile(data, outputFile))
  {
    std::cerr << "Failed to write output file." << std::endl;
    return 1;
  }

  std::cout << "Conversion complete." << std::endl;
  return 0;
}
```

**说明**：VTK对STL、OBJ、PLY格式的支持非常成熟。`vtkSTLReader`能够自动检测ASCII或Binary编码的STL文件。`vtkOBJReader`读取Wavefront .obj文件（包括顶点、面、纹理坐标和法向量），`vtkOBJWriter`输出时也可保留材质库引用。`vtkPLYReader`和`vtkPLYWriter`支持Stanford PLY格式的ASCII和Binary两种变体，广泛用于3D扫描数据的交换。

---

## 10.3 非结构化网格读写

`vtkUnstructuredGrid`的读写遵循与PolyData相同的模式，但使用专用的类。

### 10.3.1 XML格式（推荐）

```cpp
#include "vtkXMLUnstructuredGridReader.h"
#include "vtkXMLUnstructuredGridWriter.h"
#include "vtkUnstructuredGrid.h"

// === 写入 .vtu 文件 ===
void WriteUnstructuredGrid(vtkUnstructuredGrid* grid, const char* filename)
{
  vtkNew<vtkXMLUnstructuredGridWriter> writer;
  writer->SetFileName(filename);
  writer->SetInputData(grid);
  // writer->SetDataModeToBinary();    // 可选
  // writer->SetCompressorTypeToZLib();
  writer->Write();
}

// === 读取 .vtu 文件 ===
vtkSmartPointer<vtkUnstructuredGrid> ReadUnstructuredGrid(const char* filename)
{
  vtkNew<vtkXMLUnstructuredGridReader> reader;
  reader->SetFileName(filename);
  reader->Update();
  return reader->GetOutput();
}
```

### 10.3.2 Legacy格式（向后兼容）

```cpp
#include "vtkUnstructuredGridReader.h"
#include "vtkUnstructuredGridWriter.h"

// Legacy .vtk 文件读写——API与XML版本一致
vtkNew<vtkUnstructuredGridWriter> writer;
writer->SetFileName("grid_legacy.vtk");
writer->SetInputData(grid);
writer->Write();
```

### 10.3.3 完整示例：创建、写入、读回、验证

```cpp
// ugrid_roundtrip.cxx
// 演示：创建UnstructuredGrid -> 写入.vtu -> 读回 -> 验证数据完整性

#include "vtkNew.h"
#include "vtkSmartPointer.h"
#include "vtkUnstructuredGrid.h"
#include "vtkXMLUnstructuredGridReader.h"
#include "vtkXMLUnstructuredGridWriter.h"
#include "vtkPoints.h"
#include "vtkTetra.h"
#include "vtkHexahedron.h"
#include "vtkDoubleArray.h"
#include "vtkPointData.h"
#include "vtkCellData.h"

#include <iostream>

int main()
{
  // ================================================================
  // 第一步：手工构建一个混合单元的UnstructuredGrid
  // ================================================================
  vtkNew<vtkPoints> points;
  points->InsertNextPoint(0.0, 0.0, 0.0);   // 0
  points->InsertNextPoint(1.0, 0.0, 0.0);   // 1
  points->InsertNextPoint(0.0, 1.0, 0.0);   // 2
  points->InsertNextPoint(0.0, 0.0, 1.0);   // 3
  points->InsertNextPoint(1.0, 1.0, 0.0);   // 4
  points->InsertNextPoint(1.0, 0.0, 1.0);   // 5
  points->InsertNextPoint(0.0, 1.0, 1.0);   // 6
  points->InsertNextPoint(1.0, 1.0, 1.0);   // 7

  vtkNew<vtkUnstructuredGrid> grid;
  grid->SetPoints(points);

  // 添加一个四面体 (nodes 0,1,2,3)
  vtkNew<vtkTetra> tetra;
  tetra->GetPointIds()->SetId(0, 0);
  tetra->GetPointIds()->SetId(1, 1);
  tetra->GetPointIds()->SetId(2, 2);
  tetra->GetPointIds()->SetId(3, 3);
  grid->InsertNextCell(tetra->GetCellType(), tetra->GetPointIds());

  // 添加一个六面体 (nodes 0,1,4,2,3,5,7,6)
  vtkNew<vtkHexahedron> hex;
  hex->GetPointIds()->SetId(0, 0);
  hex->GetPointIds()->SetId(1, 1);
  hex->GetPointIds()->SetId(2, 4);
  hex->GetPointIds()->SetId(3, 2);
  hex->GetPointIds()->SetId(4, 3);
  hex->GetPointIds()->SetId(5, 5);
  hex->GetPointIds()->SetId(6, 7);
  hex->GetPointIds()->SetId(7, 6);
  grid->InsertNextCell(hex->GetCellType(), hex->GetPointIds());

  // 添加点数据
  vtkNew<vtkDoubleArray> pointScalars;
  pointScalars->SetName("Temperature");
  pointScalars->SetNumberOfTuples(8);
  for (int i = 0; i < 8; ++i)
    pointScalars->SetValue(i, 20.0 + i * 5.0);   // 20, 25, 30, ..., 55
  grid->GetPointData()->SetScalars(pointScalars);

  // 添加单元数据
  vtkNew<vtkDoubleArray> cellScalars;
  cellScalars->SetName("MaterialID");
  cellScalars->SetNumberOfTuples(2);
  cellScalars->SetValue(0, 1.0);   // 四面体 = 材料1
  cellScalars->SetValue(1, 2.0);   // 六面体 = 材料2
  grid->GetCellData()->SetScalars(cellScalars);

  std::cout << "Original grid:\n"
            << "  Points: " << grid->GetNumberOfPoints() << "\n"
            << "  Cells:  " << grid->GetNumberOfCells() << "\n"
            << "  Cell 0 type: " << grid->GetCellType(0)
            << " (VTK_TETRA=" << VTK_TETRA << ")\n"
            << "  Cell 1 type: " << grid->GetCellType(1)
            << " (VTK_HEXAHEDRON=" << VTK_HEXAHEDRON << ")\n"
            << "  Point Data arrays: "
            << grid->GetPointData()->GetNumberOfArrays() << "\n"
            << "  Cell Data arrays:  "
            << grid->GetCellData()->GetNumberOfArrays() << std::endl;

  // ================================================================
  // 第二步：写入 .vtu 文件
  // ================================================================
  const char* filename = "test_grid.vtu";
  vtkNew<vtkXMLUnstructuredGridWriter> writer;
  writer->SetFileName(filename);
  writer->SetInputData(grid);
  writer->SetDataModeToBinary();
  writer->Write();
  std::cout << "\nWritten to: " << filename << std::endl;

  // ================================================================
  // 第三步：读回文件
  // ================================================================
  vtkNew<vtkXMLUnstructuredGridReader> reader;
  reader->SetFileName(filename);
  reader->Update();

  vtkUnstructuredGrid* loaded = reader->GetOutput();

  // ================================================================
  // 第四步：验证数据完整性
  // ================================================================
  std::cout << "\nLoaded grid:\n"
            << "  Points: " << loaded->GetNumberOfPoints() << "\n"
            << "  Cells:  " << loaded->GetNumberOfCells() << "\n"
            << "  Cell 0 type: " << loaded->GetCellType(0) << "\n"
            << "  Cell 1 type: " << loaded->GetCellType(1) << "\n"
            << "  Point Data arrays: "
            << loaded->GetPointData()->GetNumberOfArrays() << "\n"
            << "  Cell Data arrays:  "
            << loaded->GetCellData()->GetNumberOfArrays() << std::endl;

  // 验证点数据
  vtkDataArray* loadedPointScalars =
    loaded->GetPointData()->GetScalars();
  if (loadedPointScalars)
  {
    std::cout << "  Point scalar name: " << loadedPointScalars->GetName()
              << " (expected: Temperature)\n";
    std::cout << "  Sample values: ";
    for (int i = 0; i < 3; ++i)
      std::cout << loadedPointScalars->GetTuple1(i) << " ";
    std::cout << "...\n";
  }

  // 验证单元数据
  vtkDataArray* loadedCellScalars =
    loaded->GetCellData()->GetScalars();
  if (loadedCellScalars)
  {
    std::cout << "  Cell scalar name: " << loadedCellScalars->GetName()
              << " (expected: MaterialID)\n";
    std::cout << "  Values: "
              << loadedCellScalars->GetTuple1(0) << ", "
              << loadedCellScalars->GetTuple1(1) << "\n";
  }

  // 验证边界
  double bounds[6];
  loaded->GetBounds(bounds);
  std::cout << "  Bounds: ["
            << bounds[0] << ", " << bounds[1] << "] x ["
            << bounds[2] << ", " << bounds[3] << "] x ["
            << bounds[4] << ", " << bounds[5] << "]\n";

  // 完整性断言
  bool ok = true;
  if (loaded->GetNumberOfPoints() != 8)  { ok = false; std::cerr << "ERROR: point count mismatch\n"; }
  if (loaded->GetNumberOfCells() != 2)   { ok = false; std::cerr << "ERROR: cell count mismatch\n"; }
  if (loaded->GetCellType(0) != VTK_TETRA) { ok = false; std::cerr << "ERROR: cell 0 type mismatch\n"; }
  if (loaded->GetCellType(1) != VTK_HEXAHEDRON) { ok = false; std::cerr << "ERROR: cell 1 type mismatch\n"; }

  if (ok)
    std::cout << "\n*** Round-trip verification PASSED ***" << std::endl;
  else
    std::cout << "\n*** Round-trip verification FAILED ***" << std::endl;

  return ok ? 0 : 1;
}
```

**关键点**：上述示例验证了UnstructuredGrid的完整数据保真度——点坐标、单元类型（四面体+六面体）、单元连接关系、点属性数据和单元属性数据都通过写入-读回循环完整保留。这是VTK XML格式的核心优势之一。

---

## 10.4 结构化数据与图像读写

### 10.4.1 XML格式的结构化数据读写

三种结构化数据类型各有其专用的XML读者和写者：

```cpp
// === vtkImageData (.vti) ===
#include "vtkXMLImageDataReader.h"
#include "vtkXMLImageDataWriter.h"

vtkNew<vtkXMLImageDataWriter> imageWriter;
imageWriter->SetFileName("volume.vti");
imageWriter->SetInputData(imageData);
imageWriter->Write();

vtkNew<vtkXMLImageDataReader> imageReader;
imageReader->SetFileName("volume.vti");
imageReader->Update();

// === vtkRectilinearGrid (.vtr) ===
#include "vtkXMLRectilinearGridReader.h"
#include "vtkXMLRectilinearGridWriter.h"

vtkNew<vtkXMLRectilinearGridWriter> rectWriter;
rectWriter->SetFileName("cfd_grid.vtr");
rectWriter->SetInputData(rectilinearGrid);
rectWriter->Write();

// === vtkStructuredGrid (.vts) ===
#include "vtkXMLStructuredGridReader.h"
#include "vtkXMLStructuredGridWriter.h"

vtkNew<vtkXMLStructuredGridWriter> structWriter;
structWriter->SetFileName("body_fitted.vts");
structWriter->SetInputData(structuredGrid);
structWriter->Write();
```

所有XML格式的Writer都支持`SetDataModeToBinary()`、`SetDataModeToAscii()`和`SetCompressorTypeToZLib()`选项，API一致。

### 10.4.2 图像格式读写

VTK提供了对常见光栅图像格式的读写支持，这些类直接输出或输入`vtkImageData`（对于2D图像，Dimensions为[width, height, 1]）：

| 格式 | Reader | Writer | 通道支持 |
|------|--------|--------|---------|
| PNG | `vtkPNGReader` | `vtkPNGWriter` | 1(Grayscale), 2(Gray+Alpha), 3(RGB), 4(RGBA) |
| JPEG | `vtkJPEGReader` | `vtkJPEGWriter` | 1(Grayscale), 3(RGB) |
| BMP | `vtkBMPReader` | `vtkBMPWriter` | 1, 3, 4 |
| TIFF | `vtkTIFFReader` | `vtkTIFFWriter` | 1, 3, 4 (多页) |

```cpp
#include "vtkPNGReader.h"
#include "vtkPNGWriter.h"
#include "vtkJPEGReader.h"
#include "vtkJPEGWriter.h"
#include "vtkBMPReader.h"
#include "vtkBMPWriter.h"

// === 读取PNG图像 ===
vtkNew<vtkPNGReader> pngReader;
pngReader->SetFileName("input.png");
pngReader->Update();
vtkImageData* image = pngReader->GetOutput();

// === 写入JPEG图像（转换） ===
vtkNew<vtkJPEGWriter> jpgWriter;
jpgWriter->SetFileName("output.jpg");
jpgWriter->SetInputData(image);    // 也可以使用SetInputConnection
jpgWriter->SetQuality(95);         // JPEG质量 0-100，默认95
jpgWriter->Write();

// === 写入PNG图像 ===
vtkNew<vtkPNGWriter> pngWriter;
pngWriter->SetFileName("output.png");
pngWriter->SetInputData(image);
pngWriter->Write();
```

### 10.4.3 DICOM医学影像

DICOM（Digital Imaging and Communications in Medicine）是医学影像的标准格式。VTK提供`vtkDICOMImageReader`用于读取DICOM文件序列：

```cpp
#include "vtkDICOMImageReader.h"

vtkNew<vtkDICOMImageReader> dicomReader;
// 方式1：指定目录（读取目录中的所有DICOM文件）
dicomReader->SetDirectoryName("C:/Data/CT_Scan/");
// 方式2：指定单个文件（读取单张切片）
// dicomReader->SetFileName("C:/Data/CT_Scan/slice001.dcm");
dicomReader->Update();

vtkImageData* volume = dicomReader->GetOutput();
int dims[3];
volume->GetDimensions(dims);
std::cout << "DICOM volume: " << dims[0] << " x " << dims[1]
          << " x " << dims[2] << " (slices)" << std::endl;
std::cout << "Patient Name: " << dicomReader->GetPatientName() << std::endl;
std::cout << "Study UID:    " << dicomReader->GetStudyUID() << std::endl;
```

**特别提示**：DICOM数据通常以`vtkImageData`的形式读入，其中`dims[2]`等于切片数量。关于DICOM体数据的完整渲染管线（包括体渲染和传递函数），将在后续的体渲染章节中详细讲解。

### 10.4.4 从图像到3D表面：等值面提取示例

这是一个常见的可视化工作流——读取2D图像，将其转换为3D高程表面：

```cpp
// image_to_surface.cxx
// 读取PNG灰度图像 -> 将像素亮度映射为Z高度 -> 生成3D表面

#include "vtkNew.h"
#include "vtkSmartPointer.h"
#include "vtkPNGReader.h"
#include "vtkImageData.h"
#include "vtkImageDataGeometryFilter.h"
#include "vtkWarpScalar.h"
#include "vtkPolyDataNormals.h"
#include "vtkPolyDataMapper.h"
#include "vtkActor.h"
#include "vtkRenderer.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkProperty.h"
#include "vtkCamera.h"

int main(int argc, char* argv[])
{
  const char* filename = (argc > 1) ? argv[1] : "heightmap.png";

  // ---------- 步骤1：读取图像 ----------
  vtkNew<vtkPNGReader> reader;
  reader->SetFileName(filename);
  reader->Update();

  vtkImageData* image = reader->GetOutput();
  int dims[3];
  image->GetDimensions(dims);
  double spacing[3];
  image->GetSpacing(spacing);
  std::cout << "Image: " << dims[0] << " x " << dims[1]
            << " (spacing: " << spacing[0] << " x " << spacing[1] << ")" << std::endl;

  // ---------- 步骤2：将图像像素转换为几何网格 ----------
  // vtkImageDataGeometryFilter 将ImageData的每个像素中心转化为结构化网格的顶点
  vtkNew<vtkImageDataGeometryFilter> geometryFilter;
  geometryFilter->SetInputData(image);

  // ---------- 步骤3：将标量值（像素亮度）映射为Z方向位移 ----------
  vtkNew<vtkWarpScalar> warp;
  warp->SetInputConnection(geometryFilter->GetOutputPort());
  warp->SetScaleFactor(0.5);   // 位移放大因子——根据你的数据范围调整
  warp->SetUseNormal(0);       // 使用标量值直接位移，而非沿法向位移
  warp->SetXYPlane(1);         // 在Z方向位移（1表示Z方向）

  // ---------- 步骤4：计算法向量 ----------
  vtkNew<vtkPolyDataNormals> normals;
  normals->SetInputConnection(warp->GetOutputPort());
  normals->SetFeatureAngle(50.0);
  normals->ConsistencyOn();

  // ---------- 步骤5：渲染 ----------
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputConnection(normals->GetOutputPort());
  mapper->ScalarVisibilityOn();

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetAmbient(0.2);
  actor->GetProperty()->SetDiffuse(0.8);

  vtkNew<vtkRenderer> renderer;
  renderer->AddActor(actor);
  renderer->SetBackground(0.1, 0.1, 0.15);
  renderer->GetActiveCamera()->SetPosition(dims[0]/2, -dims[1], dims[1]/2);
  renderer->GetActiveCamera()->SetFocalPoint(dims[0]/2, dims[1]/2, 0);
  renderer->GetActiveCamera()->SetViewUp(0, 0, 1);

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(900, 700);
  renderWindow->SetWindowName("Image Heightmap to 3D Surface");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);
  vtkNew<vtkInteractorStyleTrackballCamera> style;
  interactor->SetInteractorStyle(style);

  renderWindow->Render();
  std::cout << "Heightmap surface rendered." << std::endl;
  interactor->Start();

  return 0;
}
```

这个管线是地理高程可视化的基本模板。任何灰度图像（包括真实的高程图、CT切片或数学纹理）都可以通过此管线被转化为可交互旋转的3D曲面。

---

## 10.5 复合数据集读写

### 10.5.1 MultiBlock数据集概念

在复杂的工程仿真中，计算结果通常包含多个区域（Zone）或部件（Part）——例如飞机的外部气动网格（PolyData）与发动机的内部流道网格（UnstructuredGrid）并存。`vtkMultiBlockDataSet`就是VTK中管理这种"数据集集合"的容器。

### 10.5.2 .vtm格式的读写

`.vtm`（VTK MultiBlock）格式用于读写`vtkMultiBlockDataSet`：

```cpp
#include "vtkXMLMultiBlockDataReader.h"
#include "vtkXMLMultiBlockDataWriter.h"
#include "vtkMultiBlockDataSet.h"
#include "vtkPolyData.h"
#include "vtkUnstructuredGrid.h"
#include "vtkSphereSource.h"
#include "vtkNew.h"

// === 创建并写入MultiBlock数据集 ===
void WriteMultiBlock(const char* filename)
{
  // 创建数据块1：球体PolyData
  vtkNew<vtkSphereSource> sphere;
  sphere->SetThetaResolution(32);
  sphere->SetPhiResolution(32);
  sphere->Update();

  // 创建数据块2：从第10.3.3节的示例中复制或导入的UnstructuredGrid
  // （此处省略具体的网格构建代码，实际使用中替换为你的数据）

  // 组装MultiBlock数据集
  vtkNew<vtkMultiBlockDataSet> multiBlock;
  multiBlock->SetNumberOfBlocks(2);
  multiBlock->SetBlock(0, sphere->GetOutput());     // 块索引从0开始
  multiBlock->SetBlock(1, someUnstructuredGrid);
  // 可选：为每个块设置名称（ParaView会在Pipeline Browser中显示）
  multiBlock->GetMetaData(0)->Set(vtkCompositeDataSet::NAME(), "Sphere Surface");
  multiBlock->GetMetaData(1)->Set(vtkCompositeDataSet::NAME(), "Volume Mesh");

  // 写入
  vtkNew<vtkXMLMultiBlockDataWriter> writer;
  writer->SetFileName(filename);
  writer->SetInputData(multiBlock);
  writer->Write();
  std::cout << "MultiBlock written to: " << filename << std::endl;
}

// === 读取MultiBlock数据集 ===
vtkSmartPointer<vtkMultiBlockDataSet> ReadMultiBlock(const char* filename)
{
  vtkNew<vtkXMLMultiBlockDataReader> reader;
  reader->SetFileName(filename);
  reader->Update();

  auto multiBlock = vtkSmartPointer<vtkMultiBlockDataSet>::New();
  multiBlock->ShallowCopy(reader->GetOutput());

  unsigned int numBlocks = multiBlock->GetNumberOfBlocks();
  std::cout << "Read MultiBlock with " << numBlocks << " blocks:" << std::endl;
  for (unsigned int i = 0; i < numBlocks; ++i)
  {
    vtkDataObject* block = multiBlock->GetBlock(i);
    if (block)
    {
      std::cout << "  Block " << i << ": "
                << block->GetClassName() << std::endl;
    }
  }
  return multiBlock;
}
```

### 10.5.3 关键注意事项

- **块的可混合数据类型**：一个`.vtm`文件的各个块可以是不同的数据类型。块0可以是`vtkPolyData`，块1可以是`vtkUnstructuredGrid`，块2可以是`vtkImageData`。
- **块名称元数据**：使用`GetMetaData(i)->Set(vtkCompositeDataSet::NAME(), "名称")`为每个块设置描述性名称，这在ParaView和VTK的Pipeline Browser中非常有用。
- **实际存储机制**：`.vtm`文件不直接内嵌所有数据。标准的做法是：`.vtm`文件包含对各子数据集文件的引用，每个子数据集（.vtp、.vtu等）作为独立文件存储在同一目录下。这种方式允许多个处理器并行读写各自的子文件。

---

## 10.6 文件格式选择指南

### 10.6.1 决策表

| 你的情况 | 推荐格式 | 理由 |
|---------|---------|------|
| PolyData（表面网格），新项目 | `.vtp` | VTK原生，支持压缩和多编码方式 |
| PolyData，3D打印/制造 | `.stl` | 工业标准，所有切片软件都支持 |
| PolyData，需要材质和纹理信息 | `.obj` | 携带材质库(.mtl)引用，图形学工具链广泛支持 |
| PolyData，3D扫描点云/网格 | `.ply` | 学术界和扫描设备的通用格式 |
| UnstructuredGrid，新项目 | `.vtu` | VTK原生XML格式，推荐 |
| UnstructuredGrid，需要与旧版VTK互操作 | `.vtk` (Legacy) | 最广泛的向后兼容性 |
| ImageData（均匀体数据） | `.vti` | XML格式，支持压缩 |
| RectilinearGrid（直线网格） | `.vtr` | XML格式 |
| StructuredGrid（结构化网格） | `.vts` | XML格式 |
| 多区域/多部件仿真结果 | `.vtm` | VTK MultiBlock XML格式 |
| 2D光栅图像 | `.png` | 无损压缩，支持alpha通道 |
| 照片级输出/材质贴图 | `.jpg` | 有损压缩，文件体积小 |
| 医学影像 | DICOM (`.dcm`) | 医学标准格式，携带患者元数据 |
| 需要人工检查/调试数据内容 | Legacy `.vtk` (ASCII) | 纯文本，人类可直接阅读 |
| 大规模并行计算I/O | `.pvti`/`.pvtu`/`.pvtr`/`.pvts` | VTK XML并行格式 |

### 10.6.2 Binary vs ASCII 权衡

| 特性 | Binary（二进制） | ASCII（文本） |
|------|-----------------|-------------|
| 文件大小 | 最小 | 通常大5-10倍 |
| 读写速度 | 最快 | 较慢 |
| 人类可读性 | 不可读 | 可用文本编辑器直接查看和修改 |
| 调试便利性 | 差 | 好——可以grep、diff、手工修正 |
| 数值精度 | 精确（无文本转换损失） | 受文本表示的精度限制 |
| 适用场景 | 生产环境、大型数据 | 教学、开发、调试 |

**实践建议**：在开发和调试阶段使用ASCII模式，以确保你能验证写入的数据内容。一旦确信管线正确，切换到Binary模式以减小文件体积和提高读写速度。XML格式的Appended模式是折中方案：数据以二进制格式集中存储，但XML头部保留可读的结构描述。

### 10.6.3 XML格式的压缩选项

```cpp
writer->SetDataModeToBinary();
writer->SetCompressorTypeToZLib();
```

zlib压缩的效果取决于数据的特征：

- **规则网格（ImageData/RectilinearGrid）**：压缩效果通常最佳（压缩比可达5-20倍），因为相邻点的标量值高度相关（空间冗余高）。
- **仿真结果（带标量场）**：压缩比通常在3-8倍，取决于标量值的分布。
- **纯几何数据（只有Points和Cells，无标量场）**：压缩比通常在2-4倍，因为坐标值通常比较随机。

**注意**：压缩和解压缩会产生CPU开销。对于交互式可视化应用，如果I/O性能是关键瓶颈，可以考虑跳过压缩（文件更大但读取更快）。对于归档和传输，压缩是最佳选择。

---

## 10.7 代码示例：数据管线

本节是本章的综合实战。我们将构建一个完整的数据管线程序，演示从数据生成到文件持久化、从文件读回到过滤器应用、再到双视口对比渲染的全流程。

### 10.7.1 程序设计

程序将执行以下步骤：

1. **生成数据**：使用`vtkSphereSource`创建一个球体PolyData，带有标量属性（法向量作为颜色源）。
2. **写入文件**：将生成的PolyData写入`.vtp`文件（XML PolyData格式）。
3. **读取文件**：从`.vtp`文件中读取数据，得到Pipeline中第二个独立的数据集。
4. **应用过滤器**：对读取的数据施加`vtkShrinkPolyData`过滤器，使每个面片收缩，展示网格结构。
5. **双视口渲染**：左侧视口显示原始的球体表面，右侧视口显示收缩后的网格结构。
6. **完整CMakeLists.txt**：确保代码可编译运行。

```
数据管线流程图:

  [vtkSphereSource]
         |
         +----> vtkXMLPolyDataWriter --> disk (.vtp)
         |
         +----> vtkPolyDataMapper --> Actor --> Left Viewport (原始球体)
         
  disk (.vtp)
         |
         +----> vtkXMLPolyDataReader
                     |
                     +----> vtkShrinkPolyData
                     |            |
                     |            +----> vtkPolyDataMapper --> Actor --> Right Viewport (收缩网格)
                     |
                     +----> (读取的数据可与更多过滤器连接)
```

### 10.7.2 完整源代码

```cpp
// data_pipeline_demo.cxx
// ============================================================================
// 第十章综合示例：完整的数据I/O管线
//
// 管线流程：
//   1. vtkSphereSource 生成球体PolyData
//   2. 分支A：直接渲染（左侧视口）
//   3. 写入 .vtp 文件（磁盘持久化）
//   4. 从 .vtp 文件读回
//   5. 对读回的数据施加 vtkShrinkPolyData 过滤器
//   6. 分支B：渲染收缩网格（右侧视口）
//
// 演示要点：
//   - 管道分支（一个Source驱动两条渲染路径）
//   - XML PolyData写入与读取
//   - 过滤器连接 (SetInputConnection / SetInputData)
//   - 双视口对比布局
//   - 属性数据在I/O中的保持
//
// 需要的VTK模块（CMake COMPONENTS）：
//   CommonCore, CommonDataModel, CommonExecutionModel,
//   FiltersCore, FiltersSources,
//   IOXML, IOLegacy,
//   InteractionStyle, RenderingCore, RenderingOpenGL2, RenderingFreeType
// ============================================================================

#include "vtkActor.h"
#include "vtkCamera.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPointData.h"
#include "vtkPolyData.h"
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
#include "vtkXMLPolyDataReader.h"
#include "vtkXMLPolyDataWriter.h"

#include <cmath>
#include <iostream>
#include <string>

// ============================================================================
// 辅助函数：为渲染器添加标题标签
// ============================================================================
vtkSmartPointer<vtkTextActor> CreateLabel(const std::string& title,
                                          const std::string& subtitle,
                                          double r, double g, double b)
{
  auto label = vtkSmartPointer<vtkTextActor>::New();
  std::string fullText = title + "\n" + subtitle;
  label->SetInput(fullText.c_str());
  label->SetPosition(15, 15);
  label->GetTextProperty()->SetFontSize(13);
  label->GetTextProperty()->SetColor(r, g, b);
  label->GetTextProperty()->SetFontFamilyToCourier();
  label->GetTextProperty()->SetJustificationToLeft();
  label->GetTextProperty()->SetVerticalJustificationToBottom();
  label->GetTextProperty()->SetLineSpacing(1.3);
  return label;
}

// ============================================================================
// 主程序入口
// ============================================================================
int main(int argc, char* argv[])
{
  (void)argc;
  (void)argv;

  // ========================================================================
  // 第一步：生成数据 (Source)
  // ========================================================================
  std::cout << "[Step 1] Generating sphere source..." << std::endl;

  vtkNew<vtkSphereSource> sphereSource;
  sphereSource->SetThetaResolution(48);
  sphereSource->SetPhiResolution(48);
  sphereSource->SetRadius(1.0);
  sphereSource->SetCenter(0.0, 0.0, 0.0);
  sphereSource->Update();

  vtkPolyData* originalSphere = sphereSource->GetOutput();
  std::cout << "  Generated sphere: "
            << originalSphere->GetNumberOfPoints() << " points, "
            << originalSphere->GetNumberOfCells() << " cells" << std::endl;

  // 检查原始数据是否携带法向量（vtkSphereSource默认生成）
  vtkDataArray* origNormals =
    originalSphere->GetPointData()->GetNormals();
  std::cout << "  Original normals: "
            << (origNormals ? "Present" : "NOT present") << std::endl;

  // ========================================================================
  // 第二步：写入文件（.vtp XML PolyData）
  // ========================================================================
  const char* outputFile = "pipeline_demo_sphere.vtp";

  std::cout << "\n[Step 2] Writing to file: " << outputFile << std::endl;

  vtkNew<vtkXMLPolyDataWriter> writer;
  writer->SetFileName(outputFile);
  writer->SetInputConnection(sphereSource->GetOutputPort());
  writer->SetDataModeToBinary();
  // 可选：转换为ASCII以便查看文件内容（调试时取消注释）
  // writer->SetDataModeToAscii();
  writer->Write();

  std::cout << "  Write complete." << std::endl;

  // ========================================================================
  // 第三步：读取文件
  // ========================================================================
  std::cout << "\n[Step 3] Reading from file: " << outputFile << std::endl;

  vtkNew<vtkXMLPolyDataReader> reader;
  reader->SetFileName(outputFile);
  reader->Update();

  vtkPolyData* loadedSphere = reader->GetOutput();
  std::cout << "  Loaded sphere: "
            << loadedSphere->GetNumberOfPoints() << " points, "
            << loadedSphere->GetNumberOfCells() << " cells" << std::endl;

  vtkDataArray* loadedNormals =
    loadedSphere->GetPointData()->GetNormals();
  std::cout << "  Loaded normals: "
            << (loadedNormals ? "Present" : "NOT present")
            << " (I/O preserved point data)" << std::endl;

  // 验证数据完整性
  double origBounds[6], loadedBounds[6];
  originalSphere->GetBounds(origBounds);
  loadedSphere->GetBounds(loadedBounds);
  std::cout << "  Original bounds: ["
            << origBounds[0] << ", " << origBounds[1] << "] x ["
            << origBounds[2] << ", " << origBounds[3] << "] x ["
            << origBounds[4] << ", " << origBounds[5] << "]\n";
  std::cout << "  Loaded bounds:   ["
            << loadedBounds[0] << ", " << loadedBounds[1] << "] x ["
            << loadedBounds[2] << ", " << loadedBounds[3] << "] x ["
            << loadedBounds[4] << ", " << loadedBounds[5] << "]\n";

  // ========================================================================
  // 第四步：对读取的数据施加过滤器
  // ========================================================================
  std::cout << "\n[Step 4] Applying vtkShrinkPolyData to loaded data..."
            << std::endl;

  vtkNew<vtkShrinkPolyData> shrinkFilter;
  shrinkFilter->SetInputConnection(reader->GetOutputPort());
  shrinkFilter->SetShrinkFactor(0.8);   // 三角形收缩至原始大小的80%

  // vtkShrinkPolyData 的输入需要是三角形（不能是 Triangle Strips）
  // vtkSphereSource 默认生成 Strips，但 XML 读写后保持原格式。
  // 这里无需额外处理——我们的球体在写入前已经是三角形面片了。

  shrinkFilter->Update();
  vtkPolyData* shrunkSphere = shrinkFilter->GetOutput();
  std::cout << "  Shrunk sphere: "
            << shrunkSphere->GetNumberOfPoints() << " points, "
            << shrunkSphere->GetNumberOfCells() << " cells" << std::endl;

  // ========================================================================
  // 第五步：构建渲染场景（双视口对比布局）
  // ========================================================================

  // 创建渲染窗口
  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->SetSize(1100, 550);
  renderWindow->SetMultiSamples(8);
  renderWindow->SetWindowName("Chapter 10: Data I/O Pipeline Demo");

  const double bgColor[3] = {0.08, 0.10, 0.15};

  // ---- 左视口：原始球体（来自Source的直接渲染） ----
  auto renLeft = vtkSmartPointer<vtkRenderer>::New();
  renLeft->SetViewport(0.0, 0.0, 0.5, 1.0);
  renLeft->SetBackground(bgColor);
  renLeft->GetActiveCamera()->SetPosition(0, -3, 1.5);
  renLeft->GetActiveCamera()->SetFocalPoint(0, 0, 0);
  renLeft->GetActiveCamera()->SetViewUp(0, 0, 1);

  // 原始球体的Mapper和Actor（使用Source的输出——管道分支A）
  vtkNew<vtkPolyDataMapper> mapperOriginal;
  mapperOriginal->SetInputConnection(sphereSource->GetOutputPort());
  mapperOriginal->ScalarVisibilityOff();

  vtkNew<vtkActor> actorOriginal;
  actorOriginal->SetMapper(mapperOriginal);
  actorOriginal->GetProperty()->SetColor(0.20, 0.50, 0.90);    // 蓝色
  actorOriginal->GetProperty()->SetSpecular(0.4);
  actorOriginal->GetProperty()->SetSpecularPower(30);
  actorOriginal->GetProperty()->SetAmbient(0.2);
  actorOriginal->GetProperty()->SetDiffuse(0.7);

  renLeft->AddActor(actorOriginal);

  // 左侧标签
  auto labelLeft = CreateLabel("Original Sphere",
                               "From vtkSphereSource\n"
                               "(pipeline branch A)",
                               0.40, 0.70, 1.00);
  renLeft->AddActor2D(labelLeft);

  renderWindow->AddRenderer(renLeft);

  // ---- 右视口：收缩网格（来自Reader -> ShrinkFilter的处理结果） ----
  auto renRight = vtkSmartPointer<vtkRenderer>::New();
  renRight->SetViewport(0.5, 0.0, 1.0, 1.0);
  renRight->SetBackground(bgColor);

  // 使两个视口共享同一相机（旋转时同步）
  renRight->SetActiveCamera(renLeft->GetActiveCamera());

  // 收缩球体的Mapper和Actor（管道分支B：Reader -> Shrink -> Mapper -> Actor）
  vtkNew<vtkPolyDataMapper> mapperShrunk;
  mapperShrunk->SetInputConnection(shrinkFilter->GetOutputPort());
  mapperShrunk->ScalarVisibilityOff();

  vtkNew<vtkActor> actorShrunk;
  actorShrunk->SetMapper(mapperShrunk);
  actorShrunk->GetProperty()->SetColor(0.95, 0.40, 0.25);    // 橙色
  actorShrunk->GetProperty()->SetSpecular(0.3);
  actorShrunk->GetProperty()->SetSpecularPower(20);
  actorShrunk->GetProperty()->SetEdgeVisibility(1);            // 显示三角形边界
  actorShrunk->GetProperty()->SetEdgeColor(0.10, 0.10, 0.12);
  actorShrunk->GetProperty()->SetLineWidth(0.8);

  renRight->AddActor(actorShrunk);

  // 右侧标签
  auto labelRight = CreateLabel("Shrunk Sphere",
                                "Read from .vtp ->\n"
                                "vtkShrinkPolyData (factor=0.8)\n"
                                "(pipeline branch B)",
                                1.00, 0.55, 0.35);
  renRight->AddActor2D(labelRight);

  renderWindow->AddRenderer(renRight);

  // ========================================================================
  // 第六步：交互器与程序启动
  // ========================================================================
  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renLeft);
  interactor->SetInteractorStyle(style);

  // 首次渲染
  renderWindow->Render();

  // ========================================================================
  // 第七步：输出运行信息
  // ========================================================================
  std::cout << "\n"
            << "============================================================\n"
            << "  Chapter 10: Data I/O Pipeline Demo\n"
            << "============================================================\n"
            << "\n"
            << "Pipeline summary:\n"
            << "  Source:    vtkSphereSource (48x48)\n"
            << "  Write:     vtkXMLPolyDataWriter -> " << outputFile << "\n"
            << "  Read:      vtkXMLPolyDataReader\n"
            << "  Filter:    vtkShrinkPolyData (factor=0.8)\n"
            << "  Render:    Dual viewport comparison\n"
            << "\n"
            << "Viewports:\n"
            << "  Left:   Original sphere (direct from Source)\n"
            << "  Right:  Shrunk sphere (Source -> Write -> Read -> Shrink)\n"
            << "\n"
            << "Interaction:\n"
            << "  - Left mouse drag:   Rotate (both viewports sync)\n"
            << "  - Right mouse drag:  Zoom\n"
            << "  - Middle mouse drag: Pan\n"
            << "  - Scroll wheel:      Zoom\n"
            << "  - Press 'r':         Reset camera\n"
            << "\n"
            << "Observe:\n"
            << "  - The sphere on the left is smooth and complete.\n"
            << "  - The sphere on the right shows each triangle shrunk\n"
            << "    to 80% of its original size, revealing the mesh\n"
            << "    structure. This proves the data survived the\n"
            << "    write-read round-trip intact.\n"
            << "============================================================\n"
            << std::endl;

  // 进入事件循环
  interactor->Start();

  std::cout << "\nProgram terminated. The file '" << outputFile
            << "' remains on disk for inspection." << std::endl;

  return 0;
}
```

### 10.7.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16 FATAL_ERROR)
project(DataPipelineDemo VERSION 1.0.0 LANGUAGES CXX)

# ============================================================================
# 设置C++标准
# ============================================================================
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ============================================================================
# 查找VTK 9.5.2
# ============================================================================
find_package(VTK 9.5.2
  COMPONENTS
    CommonCore
    CommonDataModel
    CommonExecutionModel
    CommonMath
    FiltersCore
    FiltersSources
    IOXML
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
    RenderingFreeType
  REQUIRED
)

# ============================================================================
# 打印VTK信息
# ============================================================================
message(STATUS "VTK_VERSION: ${VTK_VERSION}")
message(STATUS "VTK_RENDERING_BACKEND: OpenGL2")

# ============================================================================
# 创建可执行文件
# ============================================================================
add_executable(DataPipelineDemo data_pipeline_demo.cxx)

# ============================================================================
# 链接VTK库
# ============================================================================
target_link_libraries(DataPipelineDemo
  PRIVATE
    ${VTK_LIBRARIES}
)

# ============================================================================
# VTK模块自动初始化（必须！）
# ============================================================================
vtk_module_autoinit(
  TARGETS DataPipelineDemo
  MODULES ${VTK_LIBRARIES}
)

# ============================================================================
# 编译选项
# ============================================================================
if(MSVC)
  target_compile_options(DataPipelineDemo PRIVATE /W3)
else()
  target_compile_options(DataPipelineDemo PRIVATE -Wall -Wextra)
endif()
```

### 10.7.4 所需模块说明

| 模块 | 用途 |
|------|------|
| `CommonCore` | VTK基础类型、`vtkNew`、`vtkSmartPointer` |
| `CommonDataModel` | `vtkPolyData`、属性数据 (`vtkPointData`) |
| `CommonExecutionModel` | 管道执行基础设施 |
| `CommonMath` | 数学常量 |
| `FiltersCore` | `vtkShrinkPolyData`过滤器 |
| `FiltersSources` | `vtkSphereSource` |
| `IOXML` | `vtkXMLPolyDataReader`、`vtkXMLPolyDataWriter` |
| `InteractionStyle` | `vtkInteractorStyleTrackballCamera` |
| `RenderingCore` | `vtkActor`、`vtkRenderer`、`vtkRenderWindow`、`vtkProperty` |
| `RenderingOpenGL2` | OpenGL2渲染后端 |
| `RenderingFreeType` | FreeType字体渲染（`vtkTextActor`） |

### 10.7.5 代码详解

#### 管道分支的概念

这是本示例最重要的设计概念。观察主程序中的两条数据路径：

```
路径A (左视口):
  sphereSource -> mapperOriginal -> actorOriginal -> renLeft

路径B (右视口):
  sphereSource -> writer -> disk -> reader -> shrinkFilter -> mapperShrunk -> actorShrunk -> renRight
```

**SetInputConnection vs SetInputData**：注意在路径A中，`mapperOriginal`通过`SetInputConnection(sphereSource->GetOutputPort())`直接连接到源。而在路径B中，由于Reader已经调用了`Update()`，数据已经被实际读取，`shrinkFilter`通过`SetInputConnection(reader->GetOutputPort())`保持管道式连接。两种连接方式在VTK中都是合法的。

**数据保真度验证**：程序输出了写入前后的点数、单元数和边界范围。`loadedNormals`的检查证实了属性数据（法向量）在I/O过程中被完整保留。这对于需要保留仿真结果物理属性的科学可视化应用至关重要。

#### 双视口与共享相机

```cpp
renRight->SetActiveCamera(renLeft->GetActiveCamera());
```

这行使右侧渲染器共享左侧渲染器的相机对象。因此当用户旋转其中一个视口时，另一个视口会自动同步旋转——确保两个球体始终处于相同的视角下，便于对比观察。

### 10.7.6 扩展方向

这个数据管线模板可以轻松扩展为更实用的工具：

1. **批量格式转换**：遍历目录中的所有`.stl`文件，读取并写入`.vtp`。
2. **自动验证管道**：在写入后自动读取并验证每一块数据的完整性（点数、单元类型、边界范围、属性数组名和范围）。
3. **多步骤过滤链**：Reader -> Smooth -> Clip -> Contour -> Writer，实现文件到文件的复杂处理管线。
4. **属性驱动的处理分支**：根据读入数据的属性数组决定施加什么过滤器（例如，如果读到"Temperature"数组，自动添加等高线）。
5. **ParaView Python脚本导出**：使用VTK Python API将上述逻辑导出为ParaView自动化脚本。

---

## 10.8 本章小结

本章系统介绍了VTK 9.5.2的数据文件I/O体系，从格式体系到具体API，从单数据集读写到复合数据集管理，从理论到完整的代码示例。

### 要点速览

1. **VTK拥有两套格式体系**：Legacy格式（统一`.vtk`扩展名，ASCII，简单但功能有限）和XML格式（类型特定扩展名 .vtp/.vtu/.vti/.vtr/.vts/.vtm，支持二进制、压缩、并行I/O）。推荐所有新项目使用XML格式。

2. **扩展名直接指示数据类型**：中间字母记忆法——p=PolyData, u=Unstructured, i=Image, r=Rectilinear, s=Structured, m=MultiBlock。

3. **PolyData是最通用的可交换数据类型**：VTK原生支持STL（3D打印）、OBJ（图形学）、PLY（3D扫描）等多种工业标准格式，`vtkSTLReader/Writer`、`vtkOBJReader/Writer`、`vtkPLYReader/Writer`的API高度一致。

4. **所有的Writer都支持三种数据编码模式**：`SetDataModeToAscii()`（可读）、`SetDataModeToBinary()`（高效，默认）、`SetDataModeToAppended()`（随机访问）。`SetCompressorTypeToZLib()`提供透明的数据压缩。

5. **属性数据在I/O中自动保持**：点数据和单元数据（标量、向量、法向量、纹理坐标、张量等）都在写入-读取往返中完整保留，无需手动管理。

6. **vtkDICOMImageReader**是通向医学影像可视化的入口，将在体渲染章节中深入展开。

7. **文件格式选型有明确决策路径**：从数据类型出发，考虑压缩需求、可调试性、互操作性三个维度，参考10.6节的决策表即可作出正确选择。

### 从I/O到全书

本章是连接"程序内创造数据"与"处理真实世界数据"之间的桥梁。掌握了本章的内容后：

- **你在前九章中学到的所有数据类型操作技能**现在可以应用到从磁盘读取的真实数据上。
- **10.7节的数据管线模板**可以作为你所有VTK项目的基础骨架——从文件读取，施加过滤器，渲染结果。
- **ParaView是一个VTK应用**：ParaView使用的文件格式与本章介绍的完全相同。你现在用VTK C++ API读写的数据，可以直接在ParaView中打开、检查和进一步处理。

### 与后续章节的衔接

在下一章（第十一章）中，你将学习VTK的**高级过滤器与算法**——`vtkContourFilter`（等值面提取）、`vtkClipDataSet`（数据裁剪）、`vtkStreamTracer`（流线可视化）、`vtkGlyph3D`（张量/向量场的图标化表示）等。这些过滤器需要输入数据，而现在你已经知道如何从文件中获取这些数据了。

> **本章关键概念口诀**："XML是新格式，Legacy是旧格式；扩展名识类型，二进制压缩小；数据往返保真，管线分支灵活。"

---

*本章完成。第十章为你打开了从磁盘到内存的数据通道——从此以后，你不再仅仅使用`vtkSphereSource`和`vtkCubeSource`，而是能够处理来自CT扫描仪、CFD求解器、3D扫描仪或公共数据集的真实科学数据。*
