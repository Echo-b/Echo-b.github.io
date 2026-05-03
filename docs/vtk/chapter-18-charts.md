# 第十八章 2D图表系统 (2D Chart System)

## 本章导读

在前面几章中,你已经深入学习了VTK的3D可视化管线--从数据模型、过滤器、Mapper到Renderer和RenderWindow的完整渲染流程。你也掌握了颜色映射、材质属性、光照模型等高级渲染技术。然而，科学可视化的实际应用中，除了三维几何展示之外，还有一个同样重要的需求：**数据图表**（Data Charts）。

想象以下场景：
- 你有一个CFD仿真结果，想在3D模型旁边同时显示某条截线上的压力分布曲线。
- 你正在运行一个参数化研究，需要快速生成多组数据的对比散点图。
- 你在构建一个科学仪表板，需要将折线图、柱状图和3D视图整合在同一个窗口中。

这些场景都需要2D图表的能力。VTK从5.0版本开始引入了**ChartsCore**模块，提供了一套完整的2D图表绘制系统。这个系统基于VTK的Context2D渲染框架，可以独立使用，也可以与3D视图无缝混合。

本章将系统地讲解VTK的2D图表系统，包括：

1. **图表系统架构**--理解vtkChartXY、vtkContextView、vtkChartScene和vtkTable之间的关系，以及图表渲染管线与3D渲染管线的区别。
2. **基本图表类型**--折线图（Line）、散点图（Scatter）、柱状图（Bar）、面积图（Area）和统计袋图（Bag Plot）的用法和关键API。
3. **数据表（vtkTable）的构建**--如何用代码创建和填充数据表，为图表提供结构化的数据源。
4. **图表样式与标注**--坐标轴配置、标题、图例、网格线、颜色和工具提示的完整控制。
5. **与3D视图混合**--使用vtkContextActor将图表嵌入3D场景，实现图表与3D可视化的同窗展示。
6. **一个完整的综合代码示例**--在2x2布局中同时展示折线图、散点图、柱状图和组合图表，所有数据通过代码生成。

本章需要的前置知识包括第5章（渲染管道基础）。如果你对Renderer/RenderWindow的概念还不太清楚，建议先回顾第5章。本章不需要ChartsCore之外的特殊VTK模块。

---

## 18.1 VTK图表系统概览

### 18.1.1 图表系统的架构层次

VTK的2D图表系统与3D渲染管线有着截然不同的架构。理解这个架构是正确使用图表功能的前提。

**3D渲染管线回顾：**

```
Source/Reader --> Filter --> Mapper --> Actor --> Renderer --> RenderWindow
```

**2D图表渲染管线：**

```
vtkTable --> vtkPlot (series) --> vtkChart --> vtkContextScene
                                              --> vtkContextView --> vtkRenderWindow
                                              --> vtkContextActor --> vtkRenderer (可选)
```

图表系统的核心组件及其职责：

| 组件 | 角色 | 关键方法 |
|------|------|---------|
| `vtkTable` | 数据容器，列=变量，行=数据点 | `AddColumn()`, `SetValue()`, `GetNumberOfRows()` |
| `vtkPlot` （及其子类） | 单个数据系列的绘制器 | `SetInputData()`, `SetColor()`, `SetWidth()` |
| `vtkChartXY` | 图表管理器，管理坐标轴和图线集合 | `AddPlot()`, `SetTitle()`, `GetAxis()` |
| `vtkContextScene` | 2D场景图根节点 | `AddItem()`, `SetGeometry()` |
| `vtkContextView` | 将ContextScene连接到RenderWindow | `SetScene()`, `GetRenderer()`, `SetRenderWindow()` |
| `vtkContextActor` | 将ContextScene嵌入3D场景的桥梁 | `SetScene()` |

### 18.1.2 vtkChartXY--2D图表的核心

`vtkChartXY`是整个图表系统的核心类。它负责：

1. **管理绘图**（Plots）：通过`AddPlot()`方法添加各种类型的绘图（折线、散点、柱状等）。一个Chart可以包含多个Plot，实现多系列数据的叠加显示。

2. **管理坐标轴**（Axes）：每个Chart拥有四个坐标轴--底部轴（BOTTOM）、顶部轴（TOP）、左侧轴（LEFT）和右侧轴（RIGHT）。每个轴可以独立配置标题、范围、刻度数等。

3. **管理图表布局**：包括标题、图例、边距、网格线等。

创建图表的最小代码：

```cpp
// 创建图表视图
vtkNew<vtkContextView> view;
view->GetRenderer()->SetBackground(1.0, 1.0, 1.0);
view->GetRenderWindow()->SetSize(800, 600);

// 创建图表
vtkNew<vtkChartXY> chart;

// 将图表添加到场景
view->GetScene()->AddItem(chart);

// 添加一个折线图（需要先有vtkTable作为数据源）
vtkPlot* line = chart->AddPlot(vtkChart::LINE);
line->SetInputData(table, "x_column", "y_column");
```

### 18.1.3 vtkContextView和vtkChartScene

图表系统的"视图"不同于3D的`vtkRenderWindow`+`vtkRenderer`组合。VTK使用`vtkContextView`作为图表的顶层容器：

```
vtkContextView
  ├── vtkRenderWindow         (提供窗口和OpenGL上下文)
  ├── vtkRenderer             (内嵌的3D渲染器，用于背景色等)
  └── vtkContextScene         (2D场景图)
        ├── vtkChartXY        (图表1)
        ├── vtkChartXY        (图表2)
        └── ...               (更多2D元素)
```

`vtkContextScene`是2D场景图的根节点。所有2D绘制项（`vtkContextItem`的子类）都通过`scene->AddItem()`添加到场景中。`vtkChartXY`本身就是一个`vtkContextItem`。

关键API：

```cpp
// 获取场景
vtkContextScene* scene = view->GetScene();

// 添加图表到场景
scene->AddItem(chart);

// 设置场景的几何范围（像素坐标）
scene->SetGeometry(0, 0, 800, 600);

// 获取背景渲染器（用于设置背景颜色/渐变）
vtkRenderer* renderer = view->GetRenderer();
renderer->SetBackground(1.0, 1.0, 1.0);
```

### 18.1.4 vtkTable--图表的数据模型

`vtkTable`是图表系统的数据源，它是一个表格结构的数据容器。与3D可视化中使用的`vtkDataSet`（带拓扑结构的数据集）不同，`vtkTable`是纯粹的表格：

- **列（Column）**：每个列是一个`vtkAbstractArray`（通常是`vtkFloatArray`或`vtkDoubleArray`），代表一个**变量**。列有名称，用于在图表中标识数据系列。
- **行（Row）**：每一行代表一个**数据点**。所有列必须有相同数量的行。

这种结构与电子表格或数据库表完全一致。例如，一个包含三个变量（x, sin(x), cos(x)）的表格：

| x (第0列) | sin(x) (第1列) | cos(x) (第2列) |
|-----------|---------------|---------------|
| 0.0       | 0.000         | 1.000         |
| 0.1       | 0.100         | 0.995         |
| ...       | ...           | ...           |
| 6.2       | -0.083        | 0.997         |

在VTK中表示为：

```cpp
vtkNew<vtkTable> table;

// 创建并填充列
vtkNew<vtkFloatArray> arrX;
arrX->SetName("x");
table->AddColumn(arrX);

vtkNew<vtkFloatArray> arrSin;
arrSin->SetName("sin(x)");
table->AddColumn(arrSin);
```

### 18.1.5 vtkContextActor--嵌入3D场景的桥梁

当需要在3D视图中嵌入2D图表时，使用`vtkContextActor`作为桥梁：

```
vtkContextScene --> vtkContextActor --> vtkRenderer (3D场景中的一员)
```

`vtkContextActor`是一个特殊的`vtkActor`子类（实际上是`vtkProp3D`的子类），它将自己的绘制委托给`vtkContextScene`。这意味着你可以将整个2D图表系统放入3D场景中的任意位置--覆盖在3D几何体上作为HUD，或占据视口的一部分。

```cpp
vtkNew<vtkContextScene> chartScene;
vtkNew<vtkChartXY> chart;
chartScene->AddItem(chart);

vtkNew<vtkContextActor> chartActor;
chartActor->SetScene(chartScene);

// 将图表Actor添加到3D渲染器
renderer->AddActor(chartActor);
```

本节建立了图表系统的架构全景图。接下来，我们将逐一深入学习各种图表类型的具体用法。

---

## 18.2 基本图表类型

VTK提供了五种主要的图表类型，每种都对应一个`vtkPlot`的子类。所有图表类型共享一个通用的工作模式：通过`vtkChartXY::AddPlot()`添加，通过返回的`vtkPlot*`指针配置外观。

### 18.2.1 vtkPlotLine--折线图

**用途：** 展示数据随某个连续变量（通常是x轴）变化的趋势。折线图是科学可视化中最常用的图表类型之一，适用于显示函数曲线、时间序列数据和参数扫描结果。

**关键方法：**

| 方法 | 说明 |
|------|------|
| `SetColor(r, g, b, a)` | 设置线条颜色（RGBA，范围0.0-1.0） |
| `SetWidth(float)` | 设置线条宽度（像素） |
| `SetInputData(table, xCol, yCol)` | 设置数据源和列名 |
| `SetTooltipStringFormat()` | 设置工具提示的格式化字符串 |
| `GetPen()` | 获取底层画笔对象，提供更细粒度的线条样式控制 |

**短期代码示例：**

```cpp
// 创建折线图
vtkPlot* linePlot = chart->AddPlot(vtkChart::LINE);
linePlot->SetInputData(table, "x", "sin(x)");
linePlot->SetColor(0.0, 0.0, 1.0, 1.0);  // 蓝色
linePlot->SetWidth(2.0);                   // 2像素宽
```

### 18.2.2 vtkPlotPoints--散点图

**用途：** 用点标记展示离散数据点的分布。适用于展示采样数据、实验测量值或两个变量之间的相关关系。

**关键方法：**

| 方法 | 说明 |
|------|------|
| `SetColor(r, g, b, a)` | 设置点的颜色 |
| `SetWidth(float)` | 设置点的尺寸（像素） |
| `SetMarkerStyle(int)` | 设置标记样式：`vtkPlotPoints::CIRCLE`, `SQUARE`, `DIAMOND`, `CROSS`, `PLUS`等 |
| `SetInputData(table, xCol, yCol)` | 设置数据源和列名 |

**短期代码示例：**

```cpp
// 创建散点图
vtkPlot* scatterPlot = chart->AddPlot(vtkChart::POINTS);
scatterPlot->SetInputData(table, "x", "cos(x)");
scatterPlot->SetColor(1.0, 0.0, 0.0, 1.0);  // 红色
scatterPlot->SetWidth(8.0);                   // 点的大小
scatterPlot->SetMarkerStyle(vtkPlotPoints::DIAMOND);
```

### 18.2.3 vtkPlotBar--柱状图

**用途：** 使用矩形条的高度（或水平条的长度）表示数据值，适用于展示分类数据的比较、频率分布和直方图。

**关键方法：**

| 方法 | 说明 |
|------|------|
| `SetColor(r, g, b, a)` | 设置柱子的填充颜色 |
| `SetOffset(float)` | 设置组的偏移量，用于并列显示多个柱状系列 |
| `SetInputData(table, xCol, yCol)` | 设置数据源和列名 |
| `SetWidth(float)` | 设置柱子的宽度（在x轴坐标空间中的宽度） |

**短期代码示例：**

```cpp
// 创建柱状图
vtkPlot* barPlot = chart->AddPlot(vtkChart::BAR);
barPlot->SetInputData(table, "category", "value");
barPlot->SetColor(0.2, 0.6, 0.9, 1.0);
barPlot->SetWidth(0.6);

// 组柱状图的第二个系列（使用偏移量避免重叠）
vtkPlot* barPlot2 = chart->AddPlot(vtkChart::BAR);
barPlot2->SetInputData(table2, "category", "value2");
barPlot2->SetColor(0.9, 0.4, 0.2, 1.0);
barPlot2->SetOffset(0.6);  // 偏移以防止重叠
```

### 18.2.4 vtkPlotArea--面积填充图

**用途：** 在折线图的基础上，将曲线下方的区域用颜色填充。适用于强调数据的变化量、表示累积分布或显示数据范围的上下界。

**关键方法：**

| 方法 | 说明 |
|------|------|
| `SetColor(r, g, b, a)` | 设置填充颜色 |
| `SetInputData(table, xCol, yCol)` | 设置数据源和列名 |
| `SetWidth(float)` | 设置边界线宽度 |

**短期代码示例：**

```cpp
// 创建面积图
vtkPlot* areaPlot = chart->AddPlot(vtkChart::AREA);
areaPlot->SetInputData(table, "x", "distribution");
areaPlot->SetColor(0.3, 0.7, 0.3, 0.6);  // 半透明绿色填充
```

### 18.2.5 vtkPlotBag--统计袋图

**用途：** 袋图（Bag Plot）是一种用于二维统计分布的专用可视化方法。它是箱线图（Box Plot）的二维推广，通过绘制"袋"（包含50%数据点的区域）和"围栏"（fence，包含约99%数据点的区域）来展示二维数据的分布、中心和离群值。适用于多变量统计分析、异常值检测和群体比较。

VTK的`vtkPlotBag`使用`vtkTable`中的数据，通过基于深度的算法（如半空间深度或基于凸包的方法）自动计算袋和围栏的边界。

**关键方法：**

| 方法 | 说明 |
|------|------|
| `SetInputData(table, xCol, yCol)` | 设置数据源（两列数值数据） |
| `SetLinePen(vtkPen*)` | 设置袋和围栏边界的画笔 |
| `SetPointPen(vtkPen*)` | 设置离群值点的画笔 |
| `SetBagColor(r, g, b, a)` | 设置袋的填充颜色 |
| `SetGridPen(vtkPen*)` | 设置中位数交叉线的画笔 |

**短期代码示例：**

```cpp
// 创建袋图
vtkPlot* bagPlot = chart->AddPlot(vtkChart::BAG);
bagPlot->SetInputData(table, "feature_x", "feature_y");
bagPlot->SetBagColor(0.4, 0.7, 1.0);
bagPlot->SetColor(0.0, 0.3, 0.7, 1.0);
```

### 18.2.6 在同一图表中叠加多个Plot

`vtkChartXY`支持多个Plot的叠加显示。每次调用`AddPlot()`都会添加一个新的数据系列到同一个图表中：

```cpp
// 在同一个chart上叠加折线图和散点图
vtkPlot* line = chart->AddPlot(vtkChart::LINE);
line->SetInputData(table, "x", "sin(x)");
line->SetColor(0.0, 0.0, 1.0);      // 蓝色折线
line->SetWidth(2.0);

vtkPlot* points = chart->AddPlot(vtkChart::POINTS);
points->SetInputData(table, "x", "cos(x)");
points->SetColor(1.0, 0.0, 0.0);    // 红色散点
points->SetWidth(6.0);

// 继续添加更多系列...
vtkPlot* line2 = chart->AddPlot(vtkChart::LINE);
line2->SetInputData(table, "x", "tanh_x");
line2->SetColor(0.0, 0.6, 0.0);     // 绿色折线
```

这种叠加能力让图表系统非常灵活--你可以在同一个坐标系中自由组合不同类型的图线。

---

## 18.3 创建数据表（vtkTable）

### 18.3.1 vtkTable的基本结构

`vtkTable`是VTK中存储表格数据的核心数据结构。与C++标准库中的表格或Pandas的DataFrame类似，它包含命名的列和等长的数据数组。

**内部结构：**

```
vtkTable
  ├── Column "x"       : vtkFloatArray (N个值)
  ├── Column "sin(x)"  : vtkFloatArray (N个值)
  ├── Column "cos(x)"  : vtkFloatArray (N个值)
  └── Column "label"   : vtkStringArray (N个值)
```

### 18.3.2 通过代码构建vtkTable

构建`vtkTable`的标准流程是：

1. 创建各个列数组并设置名称
2. 向每个数组填充数据
3. 将列添加到表中
4. 设置表的行数

**代码示例：构建包含x、sin(x)和cos(x)列的数据表**

```cpp
#include <vtkTable.h>
#include <vtkFloatArray.h>
#include <vtkNew.h>
#include <cmath>

vtkSmartPointer<vtkTable> CreateFunctionTable(int numPoints)
{
    // 创建列数组
    vtkNew<vtkFloatArray> arrX;
    arrX->SetName("x");

    vtkNew<vtkFloatArray> arrSin;
    arrSin->SetName("sin(x)");

    vtkNew<vtkFloatArray> arrCos;
    arrCos->SetName("cos(x)");

    // 填充数据
    for (int i = 0; i < numPoints; ++i)
    {
        double x = 2.0 * 3.14159265 * i / (numPoints - 1);  // [0, 2*pi]
        arrX->InsertNextValue(static_cast<float>(x));
        arrSin->InsertNextValue(static_cast<float>(std::sin(x)));
        arrCos->InsertNextValue(static_cast<float>(std::cos(x)));
    }

    // 构建表
    vtkNew<vtkTable> table;
    table->AddColumn(arrX);
    table->AddColumn(arrSin);
    table->AddColumn(arrCos);

    return table;
}
```

### 18.3.3 为不同图表类型构建数据

不同类型的图表需要不同结构的数据：

**折线图/散点图的数据结构：**
数据通常按x轴值排序，y列包含对应的函数值。如果x列的值是单调递增的（或至少是排序的），折线图会呈现为一条合理的曲线。

**柱状图的数据结构：**
柱状图的x列通常是字符串标签（使用`vtkStringArray`），y列是数值。如果x列是数值，柱状图会将每个数值视为一个离散类别。

```cpp
// 创建柱状图数据
vtkNew<vtkTable> barTable;

vtkNew<vtkStringArray> labels;
labels->SetName("Category");
labels->InsertNextValue("A");
labels->InsertNextValue("B");
labels->InsertNextValue("C");
labels->InsertNextValue("D");
labels->InsertNextValue("E");

vtkNew<vtkFloatArray> values;
values->SetName("Value");
values->InsertNextValue(23.0f);
values->InsertNextValue(45.0f);
values->InsertNextValue(32.0f);
values->InsertNextValue(56.0f);
values->InsertNextValue(41.0f);

barTable->AddColumn(labels);
barTable->AddColumn(values);
```

**袋图的数据结构：**
袋图需要两列数值数据，分别对应二维空间中的x和y坐标。数据点不需要排序。

```cpp
// 创建袋图数据（从二维正态分布采样）
vtkNew<vtkTable> bagTable;

vtkNew<vtkFloatArray> bx;
bx->SetName("x_coord");

vtkNew<vtkFloatArray> by;
by->SetName("y_coord");

// 生成具有相关性的二维数据
std::default_random_engine generator(42);
std::normal_distribution<double> dist(0.0, 1.0);

for (int i = 0; i < 500; ++i)
{
    double u = dist(generator);
    double v = dist(generator);
    // 引入相关性：y = 0.6*x + 0.8*v
    bx->InsertNextValue(static_cast<float>(u));
    by->InsertNextValue(static_cast<float>(0.6 * u + 0.8 * v));
}

bagTable->AddColumn(bx);
bagTable->AddColumn(by);
```

### 18.3.4 列命名的最佳实践

列的命名直接影响图表的显示：

1. **轴标签的来源**：当图表使用列名作为轴标签时，清晰且有意义的列名能让图表更易于理解。例如，`"Time (s)"`和`"Velocity (m/s)"`比`"col1"`和`"col2"`要好得多。

2. **图例标签**：当在同一个图表中叠加多个Plot时，列名通常会自动成为图例中的条目名称。因此，列名应该简洁且具有描述性。

3. **通过SetInputData指定列**：`SetInputData(table, xColName, yColName)`使用列名（而非索引）来指定数据源，因此你必须在创建列时设置名称。

---

## 18.4 图表样式与标注

一个专业的数据图表不仅需要正确的数据展示，还需要完善的样式配置。VTK提供了丰富的API来控制图表的视觉呈现。

### 18.4.1 坐标轴配置

每个`vtkChartXY`拥有四个独立的坐标轴对象，通过`vtkAxis`类型访问：

```cpp
// 获取底部坐标轴
vtkAxis* axisX = chart->GetAxis(vtkAxis::BOTTOM);
// 获取左侧坐标轴
vtkAxis* axisY = chart->GetAxis(vtkAxis::LEFT);
// 可用参数：vtkAxis::BOTTOM, vtkAxis::TOP, vtkAxis::LEFT, vtkAxis::RIGHT
```

**坐标轴常用配置方法：**

| 方法 | 说明 | 示例 |
|------|------|------|
| `SetTitle(text)` | 设置轴标题 | `axisX->SetTitle("Time (s)")` |
| `SetRange(min, max)` | 设置轴的数据范围 | `axisY->SetRange(-1.5, 1.5)` |
| `SetNumberOfTicks(n)` | 设置刻度数量 | `axisX->SetNumberOfTicks(10)` |
| `SetBehavior(int)` | 设置轴行为：`vtkAxis::AUTO`(自动), `vtkAxis::FIXED`(固定), `vtkAxis::CUSTOM`(自定义) | `axisY->SetBehavior(vtkAxis::FIXED)` |
| `SetLogScale(bool)` | 启用对数刻度 | `axisY->SetLogScale(true)` |
| `SetTitleVisible(bool)` | 显示/隐藏轴标题 | `axisX->SetTitleVisible(true)` |
| `SetAxisVisible(bool)` | 显示/隐藏整个轴 | `axisY->SetAxisVisible(true)` |
| `SetLabelsVisible(bool)` | 显示/隐藏刻度标签 | `axisX->SetLabelsVisible(true)` |
| `SetGridVisible(bool)` | 显示/隐藏该轴的网格线 | `axisY->SetGridVisible(true)` |
| `SetNotation(int)` | 设置数字表示法：0=标准, 1=科学计数法, 2=固定, 3=工程 | `axisY->SetNotation(1)` |
| `SetPrecision(int)` | 设置小数点后的位数 | `axisY->SetPrecision(2)` |
| `SetColor(r, g, b, a)` | 设置轴的颜色 | `axisX->SetColor(0.2, 0.2, 0.2, 1.0)` |
| `SetTitleFontSize(int)` | 设置标题字体大小 | `axisX->SetTitleFontSize(14)` |
| `SetLabelSize(int)` | 设置标签字体大小 | `axisY->SetLabelSize(10)` |
| `SetPen(vtkPen*)` | 设置轴的画笔 | |

**代码示例：配置坐标轴**

```cpp
// 配置底部轴
vtkAxis* bottomAxis = chart->GetAxis(vtkAxis::BOTTOM);
bottomAxis->SetTitle("x [radians]");
bottomAxis->SetTitleVisible(true);
bottomAxis->SetTitleFontSize(12);
bottomAxis->SetNumberOfTicks(10);
bottomAxis->SetBehavior(vtkAxis::FIXED);
bottomAxis->SetRange(0.0, 6.28);
bottomAxis->SetLabelSize(10);

// 配置左侧轴
vtkAxis* leftAxis = chart->GetAxis(vtkAxis::LEFT);
leftAxis->SetTitle("f(x)");
leftAxis->SetTitleVisible(true);
leftAxis->SetTitleFontSize(12);
leftAxis->SetBehavior(vtkAxis::FIXED);
leftAxis->SetRange(-1.5, 1.5);
leftAxis->SetLabelSize(10);

// 可选：配置顶部和右侧轴（通常用于双轴图表）
vtkAxis* rightAxis = chart->GetAxis(vtkAxis::RIGHT);
rightAxis->SetTitle("cos(x)");
// 双轴：为右轴设置独立的数据映射
```

### 18.4.2 图表标题

图表标题通过`vtkChartXY`的方法直接配置：

```cpp
// 设置图表标题
chart->SetTitle("Trigonometric Functions: sin(x) vs cos(x)");

// 设置标题的字体大小
chart->SetTitleSize(16);

// 控制标题的对齐方式
chart->SetTitleAlign(vtkChartXY::ALIGN_CENTER);  // 居中
// 可用值: ALIGN_LEFT, ALIGN_CENTER, ALIGN_RIGHT
```

### 18.4.3 图例

图例（Legend）自动显示所有已添加的Plot的名称。通过以下方法控制图例：

```cpp
// 显示图例
chart->SetShowLegend(true);

// 设置图例位置
chart->GetLegend()->SetHorizontalAlignment(vtkChartLegend::RIGHT);
chart->GetLegend()->SetVerticalAlignment(vtkChartLegend::TOP);
// 水平对齐: LEFT, CENTER, RIGHT, CUSTOM
// 垂直对齐: TOP, CENTER, BOTTOM, CUSTOM

// 设置图例的字体大小
chart->GetLegend()->SetLabelSize(10);

// 设置图例的填充（边距）
chart->GetLegend()->SetPadding(5);

// 设置图例边框可见性
chart->GetLegend()->SetInline(true);       // 图例内嵌在图表区域
chart->GetLegend()->SetDragEnabled(true);  // 允许拖拽图例
```

### 18.4.4 网格线与背景

```cpp
// 隐藏所有网格线
chart->GetAxis(vtkAxis::BOTTOM)->SetGridVisible(false);
chart->GetAxis(vtkAxis::LEFT)->SetGridVisible(false);

// 或根据需要分别控制每个轴的网格线
chart->GetAxis(vtkAxis::BOTTOM)->SetGridVisible(true);
chart->GetAxis(vtkAxis::LEFT)->SetGridVisible(true);

// 设置背景颜色（通过ContextView的Renderer）
view->GetRenderer()->SetBackground(0.98, 0.98, 0.98);  // 浅灰

// 设置网格线的可视样式（通过轴的画笔）
vtkNew<vtkPen> gridPen;
gridPen->SetColor(200, 200, 200, 255);  // 浅灰色网格线
gridPen->SetLineType(vtkPen::DASH_LINE);
chart->GetAxis(vtkAxis::BOTTOM)->SetGridPen(gridPen);
```

### 18.4.5 边距与布局

`vtkChartXY`提供了对图表边距的精细控制：

```cpp
// 设置图表边距（单位：像素）
chart->SetLeftMargin(60);     // 左侧留出空间给轴标题
chart->SetBottomMargin(40);   // 底部留出空间给轴标题
chart->SetRightMargin(20);    // 右侧边距
chart->SetTopMargin(30);      // 顶部留出空间给图表标题

// 设置自动布局（根据轴标签自动计算边距）
chart->SetAutoSize(true);
```

### 18.4.6 工具提示交互

工具提示（Tooltip）是鼠标悬停在数据点上时弹出的信息框。VTK的图表系统内置了工具提示支持：

```cpp
// 设置工具提示显示
chart->SetTooltip(vtkTooltipItem* tooltip);

// 简单启用（VTK会自动创建默认工具提示）
chart->SetTooltip(nullptr);  // 在某些VTK版本中不需要额外配置

// 为单个Plot设置工具提示的格式化字符串
vtkPlotLine* linePlot = vtkPlotLine::SafeDownCast(chart->AddPlot(vtkChart::LINE));
linePlot->SetTooltipStringFormat("x: %x, y: %y");
// 格式化标识符：%x = x值, %y = y值, %l = 列名(标签), %i = 数据点索引
```

工具提示需要在交互式环境中使用（`vtkRenderWindowInteractor`），且只在鼠标位于数据点的有效距离内时显示。

### 18.4.7 颜色和整体视觉风格

```cpp
// 设置图表区域的背景色
chart->SetBackgroundBrush(vtkBrush* brush);

// 设置图表边框
chart->SetBorderVisible(true);
chart->SetBorderColor(150, 150, 150, 255);

// 设置Plot的颜色和宽度（见18.2节中的各Plot方法）
```

### 18.4.8 完整配置示例

以下代码展示了一个完整配置的图表：

```cpp
void ConfigureChart(vtkChartXY* chart)
{
    // ---- 标题 ----
    chart->SetTitle("Sine and Cosine Functions");
    chart->SetTitleSize(16);

    // ---- 图例 ----
    chart->SetShowLegend(true);
    chart->GetLegend()->SetHorizontalAlignment(vtkChartLegend::RIGHT);
    chart->GetLegend()->SetVerticalAlignment(vtkChartLegend::TOP);
    chart->GetLegend()->SetLabelSize(10);

    // ---- 底部轴 ----
    vtkAxis* axisX = chart->GetAxis(vtkAxis::BOTTOM);
    axisX->SetTitle("x [radians]");
    axisX->SetTitleVisible(true);
    axisX->SetTitleFontSize(12);
    axisX->SetBehavior(vtkAxis::FIXED);
    axisX->SetRange(0.0, 6.28);
    axisX->SetNumberOfTicks(7);
    axisX->SetLabelSize(10);
    axisX->SetGridVisible(true);

    // ---- 左侧轴 ----
    vtkAxis* axisY = chart->GetAxis(vtkAxis::LEFT);
    axisY->SetTitle("f(x)");
    axisY->SetTitleVisible(true);
    axisY->SetTitleFontSize(12);
    axisY->SetBehavior(vtkAxis::FIXED);
    axisY->SetRange(-1.2, 1.2);
    axisY->SetNumberOfTicks(5);
    axisY->SetLabelSize(10);
    axisY->SetGridVisible(true);

    // ---- 布局 ----
    chart->SetLeftMargin(65);
    chart->SetBottomMargin(45);
    chart->SetRightMargin(20);
    chart->SetTopMargin(30);
}
```

---

## 18.5 与3D视图混合

### 18.5.1 vtkContextActor--将2D图表嵌入3D世界

`vtkContextActor`是实现2D图表与3D视图混合的关键类。它是`vtkProp3D`（更确切地说是`vtkProp`和`vtkContextActor`本身的子类）的子类，可以像普通的3D Actor一样被添加到`vtkRenderer`中。但它内部的渲染逻辑完全委托给了一个`vtkContextScene`。

工作流程：

```
1. 创建 vtkContextScene
     └── 添加一个或多个 vtkChartXY 作为子项

2. 创建 vtkContextActor
     └── actor->SetScene(scene)

3. 将 vtkContextActor 添加到 vtkRenderer
     └── renderer->AddActor(actor)

4. 正常渲染
     └── Actor在3D场景中的位置、缩放由
         SetPosition()/SetScale() 或 Camera设置控制
```

**关键代码：**

```cpp
// 创建图表场景
vtkNew<vtkContextScene> chartScene;

// 创建图表
vtkNew<vtkChartXY> chart;
chartScene->AddItem(chart);

// ... 配置图表、添加数据 ...

// 创建ContextActor
vtkNew<vtkContextActor> chartActor;
chartActor->SetScene(chartScene);

// 添加到3D渲染器
vtkNew<vtkRenderer> renderer;
renderer->AddActor(chartActor);  // 图表作为Actor加入3D场景
```

### 18.5.2 对比：独立图表 vs 嵌入图表

| 特性 | 独立图表 (vtkContextView) | 嵌入图表 (vtkContextActor) |
|------|--------------------------|---------------------------|
| 使用场景 | 纯粹的2D图表窗口 | 在3D场景中叠加或并列显示图表 |
| 窗口管理 | 拥有自己的RenderWindow | 共享3D场景的RenderWindow |
| 坐标空间 | 2D像素坐标 | 3D世界坐标（但绘制是2D的） |
| 缩放行为 | 窗口缩放时图表按像素保持 | 由Camera控制，图表大小可能与3D对象一起变化 |
| 交互 | 独立的2D交互（平移、缩放） | 共享3D场景的交互器 |
| 适用性 | 数据探索和分析 | 科学仪表板、混合展示 |

### 18.5.3 分割视口：图表在左，3D视图在右

在实践中，最常见的混合模式不是将图表嵌入3D世界坐标，而是使用**视口分割**（Viewport Splitting）：使用同一个RenderWindow，但将不同的Renderer分配到不同的视口区域。

**方案：使用多个Renderer和视口划分**

```cpp
vtkNew<vtkRenderWindow> renderWindow;
renderWindow->SetSize(1400, 600);

// ---- 左侧：2D图表（使用vtkContextView的Renderer） ----
vtkNew<vtkContextView> chartView;
chartView->SetRenderWindow(renderWindow);  // 共享RenderWindow
vtkRenderer* chartRenderer = chartView->GetRenderer();
chartRenderer->SetViewport(0.0, 0.0, 0.45, 1.0);  // 左上角到右下角（归一化坐标）
chartRenderer->SetBackground(1.0, 1.0, 1.0);

// ... 创建图表并添加到chartView->GetScene() ...

// ---- 右侧：3D视图 ----
vtkNew<vtkRenderer> renderer3D;
renderer3D->SetViewport(0.5, 0.0, 1.0, 1.0);  // 右侧一半
renderer3D->SetBackground(0.1, 0.1, 0.15);
renderWindow->AddRenderer(renderer3D);

// ... 添加3D Actor到renderer3D ...
```

### 18.5.4 实际应用场景

**科学仪表板（Scientific Dashboard）**

在同一个窗口中展示CFD仿真结果的3D几何体（如机翼表面的压力分布）和关键测量截线上的数据曲线。这种布局在商业可视化工具（如ParaView、Tecplot）中非常常见。

**参数研究展示**

当进行参数化研究时，使用图表展示扫描参数的响应曲面，同时在3D视图中展示参数对应的几何形状。这种"数据+几何"的联合展示能极大地提高分析效率。

**传感器数据可视化**

对于结构健康监测、风洞测试等应用，可以使用2D图表实时展示传感器的时间序列数据，同时在3D视图中展示传感器的物理位置和被测结构。

---

## 18.6 代码示例：多类型图表展示

### 18.6.1 程序概述

本综合示例创建一个完整的C++程序，在2x2的布局中展示四种不同的图表：
- **左上（折线图）**：正弦和余弦函数的折线图
- **右上（散点图）**：使用随机数据生成的散点图
- **左下（柱状图）**：五类数据的柱状图
- **右下（组合图表）**：折线图+散点图叠加在同一轴上

所有数据通过代码自动生成，不依赖外部文件。

### 18.6.2 完整C++代码

```cpp
// ============================================================================
// Chapter 18: 2D Chart System
// File: ChartDemo.cxx
// Description: A comprehensive demonstration of VTK's 2D chart system.
//              Displays four chart types (line, scatter, bar, combined) in a
//              2x2 grid layout using vtkContextView.
// VTK Version: 9.5.2
// Dependencies: ChartsCore, RenderingContext2D
// ============================================================================

#include <vtkAxis.h>
#include <vtkChart.h>
#include <vtkChartLegend.h>
#include <vtkChartXY.h>
#include <vtkContextScene.h>
#include <vtkContextView.h>
#include <vtkFloatArray.h>
#include <vtkMath.h>
#include <vtkNew.h>
#include <vtkPen.h>
#include <vtkPlot.h>
#include <vtkPlotBar.h>
#include <vtkPlotLine.h>
#include <vtkPlotPoints.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkSmartPointer.h>
#include <vtkStringArray.h>
#include <vtkTable.h>

#include <cmath>
#include <cstdlib>
#include <vector>

// --------------------------------------------------------------------------
// Helper: Create a simple 2D data table with x, sin(x), cos(x) columns
// --------------------------------------------------------------------------
vtkSmartPointer<vtkTable> CreateTrigTable(int numPoints)
{
    vtkNew<vtkFloatArray> arrX;
    arrX->SetName("x");

    vtkNew<vtkFloatArray> arrSin;
    arrSin->SetName("sin(x)");

    vtkNew<vtkFloatArray> arrCos;
    arrCos->SetName("cos(x)");

    for (int i = 0; i < numPoints; ++i)
    {
        double x = 2.0 * vtkMath::Pi() * static_cast<double>(i)
                   / static_cast<double>(numPoints - 1);
        arrX->InsertNextValue(static_cast<float>(x));
        arrSin->InsertNextValue(static_cast<float>(std::sin(x)));
        arrCos->InsertNextValue(static_cast<float>(std::cos(x)));
    }

    vtkNew<vtkTable> table;
    table->AddColumn(arrX);
    table->AddColumn(arrSin);
    table->AddColumn(arrCos);

    return table;
}

// --------------------------------------------------------------------------
// Helper: Create a random scatter data table
// --------------------------------------------------------------------------
vtkSmartPointer<vtkTable> CreateScatterTable(int numPoints)
{
    vtkNew<vtkFloatArray> arrX;
    arrX->SetName("Random X");

    vtkNew<vtkFloatArray> arrY;
    arrY->SetName("Random Y");

    // Seed the random number generator
    std::srand(42);

    for (int i = 0; i < numPoints; ++i)
    {
        // Generate points with a cluster structure: two clusters
        double cx, cy;
        if (i < numPoints / 2)
        {
            // Cluster 1: centered at (0.3, 0.7)
            cx = 0.3; cy = 0.7;
        }
        else
        {
            // Cluster 2: centered at (0.7, 0.3)
            cx = 0.7; cy = 0.3;
        }

        double x = cx + 0.15 * (static_cast<double>(std::rand()) / RAND_MAX - 0.5);
        double y = cy + 0.15 * (static_cast<double>(std::rand()) / RAND_MAX - 0.5);

        arrX->InsertNextValue(static_cast<float>(x));
        arrY->InsertNextValue(static_cast<float>(y));
    }

    vtkNew<vtkTable> table;
    table->AddColumn(arrX);
    table->AddColumn(arrY);

    return table;
}

// --------------------------------------------------------------------------
// Helper: Create a bar chart data table with string categories
// --------------------------------------------------------------------------
vtkSmartPointer<vtkTable> CreateBarTable()
{
    vtkNew<vtkStringArray> labels;
    labels->SetName("Category");

    vtkNew<vtkFloatArray> values1;
    values1->SetName("Series A");

    vtkNew<vtkFloatArray> values2;
    values2->SetName("Series B");

    const char* categories[] = {"Alpha", "Beta", "Gamma", "Delta", "Epsilon"};

    for (int i = 0; i < 5; ++i)
    {
        labels->InsertNextValue(categories[i]);

        // Series A: ascending values
        values1->InsertNextValue(static_cast<float>(20 + i * 15));

        // Series B: different pattern
        float vals[] = {35.0f, 28.0f, 42.0f, 22.0f, 50.0f};
        values2->InsertNextValue(vals[i]);
    }

    vtkNew<vtkTable> table;
    table->AddColumn(labels);
    table->AddColumn(values1);
    table->AddColumn(values2);

    return table;
}

// --------------------------------------------------------------------------
// Helper: Configure axis for a chart
// --------------------------------------------------------------------------
void ConfigureAxis(vtkChartXY* chart,
                   const char* xTitle, double xMin, double xMax, int xTicks,
                   const char* yTitle, double yMin, double yMax, int yTicks)
{
    // Bottom axis
    vtkAxis* axisX = chart->GetAxis(vtkAxis::BOTTOM);
    axisX->SetTitle(xTitle);
    axisX->SetTitleVisible(true);
    axisX->SetTitleFontSize(11);
    axisX->SetBehavior(vtkAxis::FIXED);
    axisX->SetRange(xMin, xMax);
    axisX->SetNumberOfTicks(xTicks);
    axisX->SetLabelSize(9);
    axisX->SetGridVisible(true);

    // Left axis
    vtkAxis* axisY = chart->GetAxis(vtkAxis::LEFT);
    axisY->SetTitle(yTitle);
    axisY->SetTitleVisible(true);
    axisY->SetTitleFontSize(11);
    axisY->SetBehavior(vtkAxis::FIXED);
    axisY->SetRange(yMin, yMax);
    axisY->SetNumberOfTicks(yTicks);
    axisY->SetLabelSize(9);
    axisY->SetGridVisible(true);
}

// --------------------------------------------------------------------------
// Helper: Apply common chart style
// --------------------------------------------------------------------------
void StyleChart(vtkChartXY* chart, const char* title)
{
    chart->SetTitle(title);
    chart->SetTitleSize(14);
    chart->SetShowLegend(true);
    chart->GetLegend()->SetHorizontalAlignment(vtkChartLegend::RIGHT);
    chart->GetLegend()->SetVerticalAlignment(vtkChartLegend::TOP);
    chart->GetLegend()->SetLabelSize(9);
    chart->SetLeftMargin(65);
    chart->SetBottomMargin(45);
    chart->SetRightMargin(15);
    chart->SetTopMargin(30);
}

// ============================================================================
// Main
// ============================================================================
int main(int argc, char* argv[])
{
    // ----------------------------------------------------------------------
    // 1. Create the main render window
    // ----------------------------------------------------------------------
    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(1400, 1000);
    renderWindow->SetWindowName("Chapter 18: 2D Chart System -- Multi-Chart Demo");
    renderWindow->SetMultiSamples(8);

    // ----------------------------------------------------------------------
    // 2. Create data tables
    // ----------------------------------------------------------------------
    auto trigTable    = CreateTrigTable(100);
    auto scatterTable = CreateScatterTable(200);
    auto barTable     = CreateBarTable();

    // ----------------------------------------------------------------------
    // 3. Chart 1 (Top-Left): Line chart -- sin(x) and cos(x)
    // ----------------------------------------------------------------------
    vtkNew<vtkContextView> view1;
    view1->SetRenderWindow(renderWindow);
    view1->GetRenderer()->SetViewport(0.00, 0.50, 0.50, 1.00);
    view1->GetRenderer()->SetBackground(0.98, 0.98, 0.98);

    vtkNew<vtkChartXY> chart1;
    view1->GetScene()->AddItem(chart1);

    // Add sine as a line plot
    auto* lineSin = vtkPlotLine::SafeDownCast(
        chart1->AddPlot(vtkChart::LINE));
    lineSin->SetInputData(trigTable, 0, 1);  // Column 0: x, Column 1: sin(x)
    lineSin->SetColor(0.0, 0.0, 1.0, 1.0);   // Blue
    lineSin->SetWidth(2.5);

    // Add cosine as a line plot
    auto* lineCos = vtkPlotLine::SafeDownCast(
        chart1->AddPlot(vtkChart::LINE));
    lineCos->SetInputData(trigTable, 0, 2);  // Column 0: x, Column 2: cos(x)
    lineCos->SetColor(1.0, 0.0, 0.0, 1.0);   // Red
    lineCos->SetWidth(2.5);

    StyleChart(chart1, "Line Chart: sin(x) and cos(x)");
    ConfigureAxis(chart1,
        "x [radians]", 0.0, 2.0 * vtkMath::Pi(), 7,
        "f(x)",        -1.2, 1.2,                5);

    // ----------------------------------------------------------------------
    // 4. Chart 2 (Top-Right): Scatter plot -- random two-cluster data
    // ----------------------------------------------------------------------
    vtkNew<vtkContextView> view2;
    view2->SetRenderWindow(renderWindow);
    view2->GetRenderer()->SetViewport(0.50, 0.50, 1.00, 1.00);
    view2->GetRenderer()->SetBackground(0.98, 0.98, 0.98);

    vtkNew<vtkChartXY> chart2;
    view2->GetScene()->AddItem(chart2);

    auto* scatterPlot = vtkPlotPoints::SafeDownCast(
        chart2->AddPlot(vtkChart::POINTS));
    scatterPlot->SetInputData(scatterTable, 0, 1);
    scatterPlot->SetColor(0.2, 0.5, 0.8, 0.8);    // Semi-transparent blue
    scatterPlot->SetWidth(7.0);
    scatterPlot->SetMarkerStyle(vtkPlotPoints::CIRCLE);

    StyleChart(chart2, "Scatter Plot: Two-Cluster Random Data");
    ConfigureAxis(chart2,
        "X Coordinate", 0.0, 1.0, 5,
        "Y Coordinate", 0.0, 1.0, 5);

    // ----------------------------------------------------------------------
    // 5. Chart 3 (Bottom-Left): Bar chart -- grouped bar chart
    // ----------------------------------------------------------------------
    vtkNew<vtkContextView> view3;
    view3->SetRenderWindow(renderWindow);
    view3->GetRenderer()->SetViewport(0.00, 0.00, 0.50, 0.50);
    view3->GetRenderer()->SetBackground(0.98, 0.98, 0.98);

    vtkNew<vtkChartXY> chart3;
    view3->GetScene()->AddItem(chart3);

    // Add Series A
    auto* barA = vtkPlotBar::SafeDownCast(
        chart3->AddPlot(vtkChart::BAR));
    barA->SetInputData(barTable, "Category", "Series A");
    barA->SetColor(0.25, 0.55, 0.85, 0.9);  // Blue
    barA->SetWidth(0.35);
    barA->SetOffset(-0.175);                  // Shift left for grouping

    // Add Series B
    auto* barB = vtkPlotBar::SafeDownCast(
        chart3->AddPlot(vtkChart::BAR));
    barB->SetInputData(barTable, "Category", "Series B");
    barB->SetColor(0.85, 0.25, 0.25, 0.9);  // Red
    barB->SetWidth(0.35);
    barB->SetOffset(0.175);                   // Shift right for grouping

    StyleChart(chart3, "Bar Chart: Grouped Categories");

    // Bar chart axes: categories on X, values on Y
    vtkAxis* barAxisX = chart3->GetAxis(vtkAxis::BOTTOM);
    barAxisX->SetTitle("Category");
    barAxisX->SetTitleVisible(true);
    barAxisX->SetTitleFontSize(11);
    barAxisX->SetBehavior(vtkAxis::AUTO);
    barAxisX->SetLabelSize(9);
    barAxisX->SetGridVisible(false);

    vtkAxis* barAxisY = chart3->GetAxis(vtkAxis::LEFT);
    barAxisY->SetTitle("Value");
    barAxisY->SetTitleVisible(true);
    barAxisY->SetTitleFontSize(11);
    barAxisY->SetBehavior(vtkAxis::FIXED);
    barAxisY->SetRange(0.0, 60.0);
    barAxisY->SetNumberOfTicks(6);
    barAxisY->SetLabelSize(9);
    barAxisY->SetGridVisible(true);

    // ----------------------------------------------------------------------
    // 6. Chart 4 (Bottom-Right): Combined chart -- line + scatter overlay
    // ----------------------------------------------------------------------
    vtkNew<vtkContextView> view4;
    view4->SetRenderWindow(renderWindow);
    view4->GetRenderer()->SetViewport(0.50, 0.00, 1.00, 0.50);
    view4->GetRenderer()->SetBackground(0.98, 0.98, 0.98);

    vtkNew<vtkChartXY> chart4;
    view4->GetScene()->AddItem(chart4);

    // Add sine as filled area
    auto* areaSin = chart4->AddPlot(vtkChart::AREA);
    areaSin->SetInputData(trigTable, 0, 1);
    areaSin->SetColor(0.2, 0.4, 1.0, 0.3);  // Semi-transparent blue fill
    areaSin->SetWidth(1.5);

    // Add cosine as scatter points on top
    auto* scatterCos = vtkPlotPoints::SafeDownCast(
        chart4->AddPlot(vtkChart::POINTS));
    scatterCos->SetInputData(trigTable, 0, 2);
    scatterCos->SetColor(1.0, 0.0, 0.0, 1.0);
    scatterCos->SetWidth(5.0);
    scatterCos->SetMarkerStyle(vtkPlotPoints::CROSS);

    StyleChart(chart4, "Combined Chart: Area fill (sin) + Scatter (cos)");
    ConfigureAxis(chart4,
        "x [radians]", 0.0, 2.0 * vtkMath::Pi(), 7,
        "f(x)",        -1.2, 1.2,                5);

    // ----------------------------------------------------------------------
    // 7. Interactor and render loop
    // ----------------------------------------------------------------------
    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    // Initialize and start
    renderWindow->Render();
    interactor->Initialize();
    interactor->Start();

    return 0;
}
```

### 18.6.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

# ============================================================================
# Chapter 18: 2D Chart System
# File: CMakeLists.txt
# ============================================================================

project(Chapter18_ChartDemo)

# --------------------------------------------------------------------------
# Find VTK 9.5
# --------------------------------------------------------------------------
find_package(VTK 9.5 REQUIRED COMPONENTS
    ChartsCore             # vtkChartXY, vtkPlot, vtkAxis, etc.
    CommonCore             # vtkObject, vtkSmartPointer, vtkNew
    CommonDataModel        # vtkTable, vtkFloatArray, vtkStringArray
    CommonMath             # vtkMath
    InteractionStyle       # vtkRenderWindowInteractor
    RenderingContext2D     # vtkContextView, vtkContextScene, vtkContextActor
    RenderingCore          # vtkRenderer, vtkRenderWindow
    RenderingFreeType      # Text rendering for chart labels
    RenderingOpenGL2       # OpenGL2 rendering backend
)

# If VTK_DIR is not set, user must provide it via -DVTK_DIR=/path/to/vtk/build
if(NOT VTK_FOUND)
    message(FATAL_ERROR "VTK 9.5 not found. Set -DVTK_DIR=/path/to/vtk/build")
endif()

message(STATUS "VTK version: ${VTK_VERSION}")

# --------------------------------------------------------------------------
# Executable
# --------------------------------------------------------------------------
add_executable(ChartDemo ChartDemo.cxx)

target_link_libraries(ChartDemo PRIVATE
    VTK::ChartsCore
    VTK::CommonCore
    VTK::CommonDataModel
    VTK::CommonMath
    VTK::InteractionStyle
    VTK::RenderingContext2D
    VTK::RenderingCore
    VTK::RenderingFreeType
    VTK::RenderingOpenGL2
)

# --------------------------------------------------------------------------
# C++ standard
# --------------------------------------------------------------------------
set_target_properties(ChartDemo PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
)

# --------------------------------------------------------------------------
# Platform-specific settings
# --------------------------------------------------------------------------
if(WIN32)
    target_compile_definitions(ChartDemo PRIVATE
        _CRT_SECURE_NO_WARNINGS
        NOMINMAX
    )
endif()
```

### 18.6.4 编译与运行

**Windows (Visual Studio):**

```powershell
# 在源代码目录下
mkdir build
cd build
cmake .. -DVTK_DIR=C:/path/to/vtk-9.5.2/build
cmake --build . --config Release

# 运行
.\Release\ChartDemo.exe
```

**Linux / macOS:**

```bash
mkdir build && cd build
cmake .. -DVTK_DIR=/path/to/vtk-9.5.2/build
cmake --build .
./ChartDemo
```

### 18.6.5 预期结果分析

运行程序后，你将看到一个包含四个子图表的大窗口：

| 位置 | 图表类型 | 预期效果 |
|------|---------|---------|
| 左上 | 折线图 | 蓝色曲线为正弦波，红色曲线为余弦波，两条曲线在0到2pi范围内平滑展示。图例自动出现在右上角。 |
| 右上 | 散点图 | 蓝色半透明圆形散点，明显可见两个聚集的簇，一个在左上区域(约0.3,0.7)，一个在右下区域(约0.7,0.3)。 |
| 左下 | 组柱状图 | 五个类别(Alpha到Epsilon)，每个类别有两个并列的柱子。蓝色柱(Series A)呈递增趋势，红色柱(Series B)高低不齐。 |
| 右下 | 组合图表 | 浅蓝色半透明区域填充为sin(x)曲线下方的面积，红色十字散点标记cos(x)函数值。两种图形在同一坐标轴上展示，证明Plot叠加能力。 |

**交互说明：**
- **鼠标左键拖拽**：在图表区域内平移（Pan）
- **鼠标滚轮**：在图表区域内缩放（Zoom）
- 图表自动支持2D交互，无需额外配置。

---

## 18.7 本章小结

本章系统地介绍了VTK中2D图表系统的架构和使用方法。以下是本章的核心要点回顾：

### 一、架构理解

1. **图表渲染管线**与3D渲染管线不同：`vtkTable`(数据) --> `vtkPlot`(图线) --> `vtkChartXY`(图表管理器) --> `vtkContextScene`(2D场景) --> `vtkContextView`或`vtkContextActor`(显示)。

2. **vtkTable是图表的数据模型**，其结构为：列=变量，行=数据点。所有图表类型都通过`SetInputData(table, xCol, yCol)`方法连接数据。

3. **vtkChartXY是图表的中心管理类**，负责管理坐标轴（BOTTOM/TOP/LEFT/RIGHT四个独立轴）、Plot集合、标题和图例。

4. **两种显示模式**：独立模式（`vtkContextView`）适用于纯图表窗口；嵌入模式（`vtkContextActor`）适用于将图表集成到3D场景中。

### 二、关键类与API速查

| 类 | 主要用途 | 关键方法 |
|----|---------|---------|
| `vtkTable` | 表格数据容器 | `AddColumn()`, `SetValue()`, `GetNumberOfRows()` |
| `vtkChartXY` | 2D图表管理器 | `AddPlot(type)`, `SetTitle()`, `GetAxis(axis)`, `SetShowLegend()`, `GetLegend()` |
| `vtkPlotLine` | 折线图 | `SetColor()`, `SetWidth()`, `SetInputData(table, xCol, yCol)` |
| `vtkPlotPoints` | 散点图 | `SetColor()`, `SetWidth()`, `SetMarkerStyle()`, `SetInputData()` |
| `vtkPlotBar` | 柱状图 | `SetColor()`, `SetWidth()`, `SetOffset()`, `SetInputData()` |
| `vtkPlotArea` | 面积填充图 | `SetColor()`, `SetWidth()`, `SetInputData()` |
| `vtkPlotBag` | 统计袋图 | `SetBagColor()`, `SetInputData()`, `SetLinePen()` |
| `vtkAxis` | 坐标轴 | `SetTitle()`, `SetRange()`, `SetNumberOfTicks()`, `SetBehavior()`, `SetLogScale()` |
| `vtkContextView` | 图表视图容器 | `SetRenderWindow()`, `GetScene()`, `GetRenderer()` |
| `vtkContextActor` | 3D场景中的图表桥梁 | `SetScene(scene)`, `GetScene()` |
| `vtkChartLegend` | 图例 | `SetHorizontalAlignment()`, `SetVerticalAlignment()`, `SetLabelSize()` |

### 三、选择指南

- **折线图（vtkPlotLine）**：适用于连续函数的趋势展示、时间序列数据、参数扫描曲线。
- **散点图（vtkPlotPoints）**：适用于离散测量值、聚类分析、两个变量之间的相关性展示。
- **柱状图（vtkPlotBar）**：适用于分类数据的比较、频率分布、直方图。使用`SetOffset()`可以实现分组柱状图。
- **面积图（vtkPlotArea）**：适用于强调累积量、展示数据范围的上下界、或者作为折线图的视觉增强。
- **袋图（vtkPlotBag）**：适用于二维统计分布的可视化、异常值检测、群体之间的分布比较。
- **组合图表**：当需要在一个坐标轴上同时展示多种关系时（如数据点+拟合曲线），使用多个`AddPlot()`叠加。

### 四、图表样式配置的核心实践

1. **始终设置轴标题**：没有轴标题的图表是不完整的，观察者无法理解数据的物理含义。
2. **合理设置轴范围**：使用`SetBehavior(vtkAxis::FIXED)`和`SetRange()`来控制显示范围，避免自动范围导致的视觉误导。对于函数曲线，给定比数据范围稍大的显示范围（如[-1.2, 1.2]展示[-1, 1]范围的sin函数）。
3. **控制刻度数量**：`SetNumberOfTicks()`设置太少会导致视觉粗糙，太多会导致标签重叠。通常5-10个刻度较为合适。
4. **开启图例**：当图表包含多个数据系列时，图例是必需的。使用`SetShowLegend(true)`并合理配置图例位置。
5. **注意边距**：使用`SetLeftMargin()`和`SetBottomMargin()`为轴标签和标题留出足够空间。

### 五、与实践的联系

你现在掌握的图表能力可以用于：

- 在参数化仿真研究中快速生成响应曲线
- 为CFD/FEM结果创建截面线上的物理量分布图
- 构建科学仪表板，将3D几何展示与2D数据图表整合
- 展示实验数据与仿真结果的对比（如使用散点图+对角线参考线）
- 实时监控可视化：将传感器数据流绘制为动态更新的折线图

### 六、下一步

2D图表系统是VTK中一个独立而强大的子系统。掌握了本章的内容后，你可以进一步学习：

- **多轴图表**：使用RIGHT和TOP轴为不同的数据系列创建第二个y轴，实现不同量纲数据的同图展示。
- **自定义ContextItem**：通过继承`vtkContextItem`创建完全自定义的2D绘图元素。
- **交互式选择**：使用图表系统内置的数据点选择和区域高亮功能。
- **动态数据更新**：在交互循环中动态更新`vtkTable`的数据，实现实时滚动的图表显示。
- **导出为图像**：使用`vtkRenderLargeImage`或`vtkWindowToImageFilter`将图表导出为高分辨率光栅图像（PNG, BMP）或矢量图形（SVG, PDF通过GL2PS导出）。

图表系统将数据分析的表达能力从"定性观察几何形状"扩展到"定量展示数值关系"，是你科学可视化工具箱中的关键组成部分。
