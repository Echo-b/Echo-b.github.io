# 第十二章 颜色映射与标量可视化 (Color Mapping and Scalar Visualization)

## 本章导读

在前面几章中，你已经学会了如何构建各种数据类型--从`vtkPolyData`到`vtkUnstructuredGrid`再到结构化数据。你也已经知道如何将数据通过Mapper传递给Actor，最终在渲染窗口中显示出来。但你或许已经注意到了一个关键问题：**颜色从何而来？**

当你调用`actor->GetProperty()->SetColor(1.0, 0.0, 0.0)`时，你是在为整个Actor设置一个单一颜色。但当你想要根据某个数据场的值来着色时--例如，根据温度高低将表面染成蓝到红的渐变、根据海拔高度给地形着色、根据应力大小给有限元网格染色--你需要的是**颜色映射**（Color Mapping）。

颜色映射是科学可视化中最核心的概念之一。它回答了"如何将数据值翻译为屏幕上的颜色"这个问题。本章将系统地讲解VTK中的颜色映射机制，包括：

1. **颜色映射的基本原理**--标量值如何通过查找表（LUT）转换为一组RGBA颜色，以及Mapper在这个过程中扮演什么角色。
2. **vtkLookupTable（离散颜色表）**--VTK中最经典的离散颜色映射工具。你将学习如何手动设置颜色、使用HSV颜色空间生成连续色阶、以及如何控制颜色映射的缩放类型。
3. **vtkColorTransferFunction（连续颜色映射）**--用于生成平滑渐变的颜色传递函数。你将理解它与LUT的核心区别，以及何时应该使用哪一种。
4. **vtkScalarBarActor（标量条）**--为可视化添加颜色图例。学习如何配置标量条的标题、标签数量、方向和位置。
5. **标量范围控制**--标量范围（Scalar Range）如何影响颜色映射的视觉效果，夹持（Clamping）行为是什么，以及如何动态发现数据的实际范围。
6. **一个完整的综合代码示例**--构建一个参数化曲面，使用2x3的视口网格同时展示三种不同的颜色表方案和两种不同的标量范围设置，每个子窗口都配有独立的标量条。

本章是"可视化表达"主题的第一章。掌握颜色映射之后，你将能够为数据赋予有意义的视觉编码，而不仅仅是"显示几何形状"。

---

## 12.1 颜色映射基本概念

### 12.1.1 什么是标量到颜色的映射

在科学可视化中，**颜色映射**（Color Mapping）指的是将数据场中的标量值（Scalar Value）转换为屏幕上的显示颜色（Display Color）的过程。这个过程的输入是一个或多个标量值，输出是一组RGBA颜色分量。

举一个具体的例子：假设你有一个表示地表温度的数据集，温度范围从-10摄氏度到40摄氏度。你希望用蓝色表示低温区域，用红色表示高温区域，中间使用绿色和黄色过渡。这个"温度值-->颜色"的对应关系，就是颜色映射。

在VTK中，颜色映射的核心数据结构是**查找表**（Lookup Table，简称LUT）。查找表本质上是一个映射函数：

```
f: Scalar_Value --> (R, G, B, Alpha)
```

当你给Mapper设置了一个查找表后，Mapper在渲染每个图元（点、线、三角形面片）时，会查询该图元对应的标量值，然后通过查找表将其转换为颜色，最终传递给OpenGL进行绘制。

### 12.1.2 Mapper在颜色映射中的角色

Mapper是颜色映射的执行者。回顾第二章中学习的可视化管线：

```
Source/Reader --> Filter --> Mapper --> Actor --> Renderer --> RenderWindow
```

标量数据附着在数据集（DataSet）上，以**点数据**（Point Data）或**单元数据**（Cell Data）的形式存在。当Mapper收到数据集后，它会执行以下步骤：

1. **选择活动标量**：通过`SetScalarMode()`和`SetArrayName()`指定使用哪一组标量数据（点数据还是单元数据，以及具体的数组名称）。
2. **查询标量值**：对于需要绘制的每个图元，从数据集中获取对应的标量值。
3. **应用标量范围**：使用`ScalarRange`将标量值映射到查找表的索引空间。
4. **查找颜色**：在查找表中查找到对应的RGBA颜色。
5. **传递给渲染后端**：将颜色信息传递给底层图形API（OpenGL/Vulkan）。

关键API：

```cpp
// 设置颜色映射模式 -- 决定是否使用标量值来控制颜色
mapper->SetScalarVisibility(true);    // 默认为true：使用标量值着色
// 设为false时，actor使用GetProperty()->SetColor()设置的单一颜色

// 设置标量范围 -- 定义标量值的最小值和最大值
mapper->SetScalarRange(minVal, maxVal);

// 设置查找表
mapper->SetLookupTable(lut);

// 设置标量模式 -- 选择使用点数据还是单元数据
mapper->SetScalarMode(VTK_SCALAR_MODE_USE_POINT_DATA);
// 可选值: VTK_SCALAR_MODE_DEFAULT,
//         VTK_SCALAR_MODE_USE_POINT_DATA,
//         VTK_SCALAR_MODE_USE_CELL_DATA,
//         VTK_SCALAR_MODE_USE_POINT_FIELD_DATA,
//         VTK_SCALAR_MODE_USE_CELL_FIELD_DATA,
//         VTK_SCALAR_MODE_USE_FIELD_DATA

// 指定标量数组名称 -- 当数据集有多个数组时
mapper->SelectColorArray("Temperature");
```

### 12.1.3 活动标量（Active Scalars）的选择

一个VTK数据集可以同时携带多组标量数据。例如，一个CFD计算结果可能同时包含温度、压力、速度大小、湍流动能等多个标量场。Mapper需要知道应该使用哪一个来着色。

VTK使用**活动标量**（Active Scalars）的概念来解决这个问题。每个数据集都有一个"当前活动的"标量数组，通过以下API管理：

```cpp
// 设置活动标量数组（点数据）
data->GetPointData()->SetActiveScalars("Temperature");

// 设置活动标量数组（单元数据）
data->GetCellData()->SetActiveScalars("Pressure");

// 获取当前活动标量的名称
const char* name = data->GetPointData()->GetScalars()->GetName();
```

当Mapper的`ScalarMode`设为默认（`VTK_SCALAR_MODE_DEFAULT`）时，它会根据以下优先级自动选择：点数据中的活动标量 > 单元数据中的活动标量 > 不使用标量着色。

### 12.1.4 标量范围与归一化

标量范围（Scalar Range）是颜色映射中至关重要的概念。它定义了标量值的"有效区间"--最小值和最大值。查找表的颜色分布在0到1的归一化坐标上，Mapper负责将实际标量值映射到这个归一化区间：

```
normalized_value = (scalar_value - scalar_min) / (scalar_max - scalar_min)
```

例如，如果标量范围设为`(0, 100)`，则：
- 标量值0对应查找表的起始位置（归一化坐标0.0），获得第一个颜色
- 标量值50对应查找表的中间位置（归一化坐标0.5），获得中间颜色
- 标量值100对应查找表的结束位置（归一化坐标1.0），获得最后一个颜色

**夹持行为（Clamping）**：当标量值超出范围时如何处理？VTK默认会进行夹持--小于`scalar_min`的值获得查找表的第一个颜色，大于`scalar_max`的值获得查找表的最后一个颜色。这个行为确保了即使数据中存在异常值，渲染也不会崩溃。

**动态范围发现**：你可以通过数据集本身的API获取标量的实际范围：

```cpp
double range[2];
data->GetScalarRange(range);
// 现在 range[0] 是最小值，range[1] 是最大值
mapper->SetScalarRange(range);
```

---

## 12.2 vtkLookupTable（离散颜色表）

`vtkLookupTable`是VTK中最经典的查找表实现。它将颜色表离散化为固定数量的**表条目**（Table Entries），每个条目在查找表中占据一个确定的位置。当查询一个介于两个条目之间的标量值时，它使用线性插值来计算最终的颜色。

### 12.2.1 基本设置

创建一个查找表并设置其基本属性的标准流程如下：

```cpp
#include <vtkLookupTable.h>

// 创建查找表
vtkNew<vtkLookupTable> lut;

// 设置颜色表的大小（离散条目的数量）
lut->SetNumberOfTableValues(256);
// 256是常用的默认值，提供足够的颜色分辨率
// 更小的值（如8-16）会产生明显的色带效果
// 更大的值（如1024）颜色过渡更平滑，但消耗更多内存

// 设置色调范围 -- 控制从起始到结束的色调变化
lut->SetHueRange(0.0, 0.6667);  // 红(0°)到蓝(240°)

// 设置饱和度范围
lut->SetSaturationRange(1.0, 1.0);  // 全饱和度

// 设置明度范围
lut->SetValueRange(0.5, 1.0);  // 明度从50%到100%

// 设置渐变方式
lut->SetRamp("linear");  // 线性渐变
// lut->SetRamp("sqrt");   // 平方根渐变 -- 更快的过渡
// lut->SetRamp("linear"); // 线性渐变 -- 均匀过渡

// 设置数值缩放类型
lut->SetScaleToLinear();  // 线性缩放
// lut->SetScaleToLog10(); // 对数缩放 -- 适用于跨数量级的数据

// 构建颜色表（必须调用此方法使设置生效）
lut->Build();
```

### 12.2.2 HSV颜色空间与色阶生成

`vtkLookupTable`使用**HSV颜色空间**（Hue色调、Saturation饱和度、Value明度）来生成颜色表。理解HSV空间对于创建有效的颜色映射至关重要：

- **Hue（色调，0.0-1.0）**：对应颜色在色环上的位置。0.0=红色、0.1667=黄色、0.333=绿色、0.5=青色、0.6667=蓝色、0.833=品红、1.0回到红色。
- **Saturation（饱和度，0.0-1.0）**：颜色的纯度。0.0=灰色（无色），1.0=完全饱和的纯色。
- **Value（明度，0.0-1.0）**：颜色的亮度。0.0=黑色，1.0=最亮。

通过设置这些范围，你可以生成各种预设的色阶：

```cpp
// 经典彩虹色阶（红-橙-黄-绿-蓝-紫）
lut->SetHueRange(0.0, 0.8);        // 从红到紫
lut->SetSaturationRange(1.0, 1.0);  // 全饱和度
lut->SetValueRange(1.0, 1.0);       // 全亮度

// 灰度色阶
lut->SetHueRange(0.0, 0.0);          // 色调不变
lut->SetSaturationRange(0.0, 0.0);   // 零饱和度 = 灰
lut->SetValueRange(0.0, 1.0);        // 亮度从黑到白

// 暖金属色阶（黑-红-橙-黄-白）
lut->SetHueRange(0.0, 0.1667);      // 红到黄
lut->SetSaturationRange(1.0, 1.0);
lut->SetValueRange(0.0, 1.0);       // 黑到亮
```

### 12.2.3 手动设置表值

除了使用HSV自动生成外，你还可以通过`SetTableValue()`精确控制每个颜色条目。这种方法适用于需要特定颜色方案的场景：

```cpp
vtkNew<vtkLookupTable> lut;
lut->SetNumberOfTableValues(5);

// 设置5个离散颜色：蓝 -> 青 -> 绿 -> 黄 -> 红
// SetTableValue(index, R, G, B, Alpha)
lut->SetTableValue(0, 0.0, 0.0, 1.0, 1.0);  // 蓝色（最小值）
lut->SetTableValue(1, 0.0, 1.0, 1.0, 1.0);  // 青色
lut->SetTableValue(2, 0.0, 1.0, 0.0, 1.0);  // 绿色（中间值）
lut->SetTableValue(3, 1.0, 1.0, 0.0, 1.0);  // 黄色
lut->SetTableValue(4, 1.0, 0.0, 0.0, 1.0);  // 红色（最大值）

lut->Build();
```

你还可以使用`SetTableRange()`来设置查找表覆盖的标量范围（与Mapper的`SetScalarRange()`不同，这是查找表自身的范围设置）：

```cpp
lut->SetTableRange(minScalar, maxScalar);
```

注意：在VTK 9.5.2中，Mapper的`SetScalarRange()`通常优先于查找表的`SetTableRange()`，推荐的做法是在Mapper上设置标量范围，查找表仅负责定义颜色分布。

### 12.2.4 SetRamp()--线性渐变与平方根渐变

`SetRamp()`控制从查找表开头到末尾的颜色过渡方式：

- **"linear"（线性渐变）**：归一化标量值直接用于插值。在标量值线性变化时，颜色也线性变化。这是最常用的设置。
- **"sqrt"（平方根渐变）**：使用`sqrt(normalized_value)`进行插值。这意味着低端值的颜色变化更快，高端值的颜色变化更慢。适用于数据在低值区域集中分布的场景。

```cpp
// 线性渐变 -- 颜色过渡与标量值成正比
lut->SetRamp("linear");
lut->Build();

// 平方根渐变 -- 低值区域颜色差异更大
lut->SetRamp("sqrt");
lut->Build();
```

### 12.2.5 SetScaleToLinear()与SetScaleToLog10()

`SetScaleToLinear()`和`SetScaleToLog10()`控制标量值到查找表索引的映射方式：

- **Linear（线性缩放）**：标量值与查找表位置呈线性关系。适合大多数工程数据。
- **Log10（对数缩放）**：标量值先取对数再映射。适合数据跨度多个数量级的场景，如地震波振幅、声学强度、浓度数据等。

```cpp
// 线性缩放（默认）
lut->SetScaleToLinear();

// 对数缩放
lut->SetScaleToLog10();
// 注意：使用对数缩放时，标量范围的最小值必须大于0
```

### 12.2.6 使用vtkNamedColors获取内置颜色方案

VTK内置了丰富的命名颜色集合，通过`vtkNamedColors`可以方便地使用这些颜色来构建自定义查找表：

```cpp
#include <vtkNamedColors.h>

vtkNew<vtkNamedColors> namedColors;
// namedColors包含了超过600种预定义颜色

// 获取颜色值（返回值为包含4个double的数组）
double* red = namedColors->GetColor3d("Red");
double* blue = namedColors->GetColor3d("Blue");
double* green = namedColors->GetColor3d("Green");
double* cyan = namedColors->GetColor3d("Cyan");
double* yellow = namedColors->GetColor3d("Yellow");

// 使用命名颜色构建查找表
vtkNew<vtkLookupTable> lut;
lut->SetNumberOfTableValues(256);
lut->SetTableValue(0,   blue[0],  blue[1],  blue[2]);
lut->SetTableValue(63,  cyan[0],  cyan[1],  cyan[2]);
lut->SetTableValue(127, green[0], green[1], green[2]);
lut->SetTableValue(191, yellow[0],yellow[1],yellow[2]);
lut->SetTableValue(255, red[0],   red[1],   red[2]);
lut->Build();
```

`vtkNamedColors`也提供了便捷的方法来获取颜色字符串：

```cpp
// 使用HTML颜色名称
double* tomato = namedColors->GetColor3d("Tomato");
double* dodgerBlue = namedColors->GetColor3d("DodgerBlue");
double* limeGreen = namedColors->GetColor3d("LimeGreen");

// 使用VTK颜色名称（带前缀）
double* vtkRed = namedColors->GetColor3d("VTK_RED");
double* vtkBlue = namedColors->GetColor3d("VTK_BLUE");
```

### 12.2.7 示例：温度色阶（蓝-青-绿-黄-红）

下面是一个完整的温度色阶查找表示例：

```cpp
vtkNew<vtkLookupTable> temperatureLUT;
temperatureLUT->SetNumberOfTableValues(256);

// 使用HSV生成从蓝色(240°)到红色(0°)的完整色阶
// 经过青色(180°), 绿色(120°), 黄色(60°)
temperatureLUT->SetHueRange(0.6667, 0.0);  // 从蓝到红（逆时针）
temperatureLUT->SetSaturationRange(1.0, 1.0);
temperatureLUT->SetValueRange(1.0, 1.0);
temperatureLUT->SetRamp("linear");
temperatureLUT->SetScaleToLinear();
temperatureLUT->Build();

// 或者使用Alpha通道增强效果
temperatureLUT->SetAlphaRange(1.0, 0.8);  // 高端值略微透明
temperatureLUT->Build();
```

---

## 12.3 vtkColorTransferFunction（连续颜色映射）

`vtkColorTransferFunction`（CTF）是VTK中用于连续颜色映射的工具。与`vtkLookupTable`离散化为固定数量的表条目不同，CTF通过定义的**控制点**（Control Points / Color Nodes）之间的连续插值来生成任意标量值对应的颜色。

### 12.3.1 基本概念

CTF的工作方式类似于在二维平面上定义一条函数曲线：X轴是标量值，Y轴是颜色分量（R、G、B分别独立）。你在一些特定的标量值处定义颜色控制点，CTF在这些点之间进行平滑（线性或样条）插值。

这种方法的优势在于：
- **无离散化伪影**：颜色过渡完全平滑，不会出现色带效应。
- **灵活的色阶设计**：你可以在任意标量值位置精确地定义颜色，而不受固定条目数量的限制。
- **独立通道控制**：R、G、B和Alpha分别独立插值，你可以为不同通道设置不同位置的节点。

### 12.3.2 AddRGBPoint()--添加RGB控制点

CTF的核心API是`AddRGBPoint()`，它在指定的标量值位置定义一个RGB颜色：

```cpp
#include <vtkColorTransferFunction.h>

vtkNew<vtkColorTransferFunction> ctf;

// AddRGBPoint(scalar_value, R, G, B)
// 在标量值位置定义颜色，中间会自动插值
ctf->AddRGBPoint(0.0,   0.0, 0.0, 1.0);   // 最小值 --> 蓝色
ctf->AddRGBPoint(0.25,  0.0, 1.0, 1.0);   // 25%位置 --> 青色
ctf->AddRGBPoint(0.5,   0.0, 1.0, 0.0);   // 50%位置 --> 绿色
ctf->AddRGBPoint(0.75,  1.0, 1.0, 0.0);   // 75%位置 --> 黄色
ctf->AddRGBPoint(1.0,   1.0, 0.0, 0.0);   // 最大值 --> 红色
```

每个RGB分量在相邻控制点之间独立进行线性插值。例如，在标量值0.0到0.25之间：
- R从0.0平滑过渡到0.0（保持不变）
- G从0.0平滑过渡到1.0
- B从1.0平滑过渡到1.0（保持不变）

### 12.3.3 AddHSVPoint()--使用HSV颜色空间

如果你更喜欢在HSV空间中定义颜色，可以使用`AddHSVPoint()`：

```cpp
// AddHSVPoint(scalar_value, H, S, V)
ctf->AddHSVPoint(0.0,   0.6667, 1.0, 1.0);  // 蓝色
ctf->AddHSVPoint(0.5,   0.3333, 1.0, 1.0);  // 绿色
ctf->AddHSVPoint(1.0,   0.0,    1.0, 1.0);  // 红色
```

### 12.3.4 添加Alpha透明通道

CTF支持添加包含Alpha（透明度）的控制点：

```cpp
// AddRGBAPoint(scalar_value, R, G, B, Alpha)
ctf->AddRGBAPoint(0.0,  0.0, 0.0, 1.0, 1.0);   // 完全不透明
ctf->AddRGBAPoint(0.5,  0.0, 1.0, 0.0, 0.8);
ctf->AddRGBAPoint(1.0,  1.0, 0.0, 0.0, 0.3);   // 70%透明
```

### 12.3.5 Build()与查询颜色

与`vtkLookupTable`类似，CTF在添加完所有控制点后通常不需要显式调用`Build()`（CTF会自动更新）。你可以通过`GetColor()`方法查询任意标量值对应的颜色：

```cpp
double color[3];
ctf->GetColor(0.3, color);  // 查询标量值0.3对应的颜色
// color = {R, G, B}
```

### 12.3.6 LUT vs CTF：何时使用哪种

这是初学者经常会问的问题。下面是一张对比表，帮助你在两者之间做出选择：

| 特性 | vtkLookupTable | vtkColorTransferFunction |
|------|---------------|-------------------------|
| **本质** | 离散颜色表（固定条目数） | 连续颜色函数（控制点插值） |
| **颜色过渡** | 介于离散条目之间线性插值 | 控制点之间完全连续插值 |
| **色带效果** | 条目少时可见色带 | 完全平滑，无色带 |
| **定义方式** | 条目数量 + HSV范围 或 SetTableValue() | AddRGBPoint() 控制点 |
| **典型使用场景** | 等值线、分段着色、带明显边界的气象图 | 地形高程着色、连续温度场、任何需要平滑过度的场景 |
| **体积渲染** | 不推荐（会产生伪影） | 推荐使用 |
| **性能** | 略快（预计算表） | 基本相同（现代硬件上差异可忽略） |

简单总结：如果你需要**离散的、分段的颜色带**（如等温线图），使用`vtkLookupTable`。如果你需要**平滑连续的颜色渐变**（如地形图、热力分布图），使用`vtkColorTransferFunction`。

### 12.3.7 示例：地形高程着色

下面是一个为地形可视化准备的高程色阶CTF：

```cpp
vtkNew<vtkColorTransferFunction> terrainCTF;

// 从海平面以下的深海蓝到山顶的白色
// 模拟真实地形图的颜色方案
terrainCTF->AddRGBPoint(-100.0, 0.0, 0.0, 0.3);     // 深海：暗蓝
terrainCTF->AddRGBPoint(0.0,    0.0, 0.5, 0.8);     // 海平面：浅蓝
terrainCTF->AddRGBPoint(10.0,   0.2, 0.6, 0.2);     // 低地：青绿
terrainCTF->AddRGBPoint(50.0,   0.4, 0.8, 0.2);     // 低地：绿色
terrainCTF->AddRGBPoint(200.0,  0.6, 0.7, 0.3);     // 丘陵：黄绿
terrainCTF->AddRGBPoint(500.0,  0.7, 0.5, 0.2);     // 山地：棕色
terrainCTF->AddRGBPoint(1000.0, 0.6, 0.45, 0.3);    // 高山：深棕
terrainCTF->AddRGBPoint(2000.0, 0.8, 0.8, 0.8);     // 雪线以上：浅灰
terrainCTF->AddRGBPoint(4000.0, 1.0, 1.0, 1.0);     // 积雪：白色
```

这个方案模拟了真实地形图从深海到雪山的颜色过渡，包含了多个控制点以实现细致的颜色变化。

---

## 12.4 标量条（Scalar Bar）

**标量条**（Scalar Bar），也称为颜色图例（Color Legend），是颜色映射可视化中不可或缺的组件。它告诉观察者："这个颜色对应的是什么数值"。在VTK中，标量条由`vtkScalarBarActor`实现。

### 12.4.1 vtkScalarBarActor的基本用法

创建一个标量条并与查找表关联：

```cpp
#include <vtkScalarBarActor.h>

vtkNew<vtkScalarBarActor> scalarBar;

// 将标量条与查找表关联 -- 这是关键步骤
scalarBar->SetLookupTable(lut);
// 或者对于vtkColorTransferFunction，需要使用SetLookupTable()
// 注意：vtkColorTransferFunction继承自vtkScalarsToColors，
// 可以直接作为参数传递
scalarBar->SetLookupTable(ctf);

// 设置标题
scalarBar->SetTitle("Temperature (C)");

// 设置标签数量
scalarBar->SetNumberOfLabels(5);  // 显示5个数值标签

// 设置方向
scalarBar->SetOrientationToVertical();    // 垂直方向
// scalarBar->SetOrientationToHorizontal(); // 水平方向
```

### 12.4.2 标题与标签的格式化

对于数值标签的显示格式，可以使用`SetLabelFormat()`进行控制：

```cpp
// 使用printf风格的格式字符串
scalarBar->SetLabelFormat("%.1f");   // 保留一位小数
scalarBar->SetLabelFormat("%.2e");   // 科学计数法，两位小数
scalarBar->SetLabelFormat("%4.0f");  // 至少4位宽度，整数显示
```

标题文字的样式可以通过`GetTitleTextProperty()`获取的文本属性对象来设置：

```cpp
scalarBar->GetTitleTextProperty()->SetFontSize(14);
scalarBar->GetTitleTextProperty()->SetColor(0.0, 0.0, 0.0);  // 黑色
scalarBar->GetTitleTextProperty()->SetBold(true);
scalarBar->GetTitleTextProperty()->SetItalic(false);
```

同样，标签文字的样式通过`GetLabelTextProperty()`设置：

```cpp
scalarBar->GetLabelTextProperty()->SetFontSize(10);
scalarBar->GetLabelTextProperty()->SetColor(0.2, 0.2, 0.2);
```

### 12.4.3 位置与尺寸控制

标量条在渲染窗口中的位置和尺寸由以下几个属性控制：

```cpp
// 设置标量条在视口中的位置（归一化坐标，0.0到1.0）
scalarBar->SetPosition(0.85, 0.1);   // 左下角位置
// 注意：实际可用的位置还取决于SetMaximumWidthInPixels()和SetMaximumHeightInPixels()

// 设置标量条的尺寸（归一化坐标）
scalarBar->SetWidth(0.1);            // 宽度（归一化）
scalarBar->SetHeight(0.5);           // 高度（归一化）

// 设置最大尺寸（像素单位）
scalarBar->SetMaximumWidthInPixels(100);
scalarBar->SetMaximumHeightInPixels(500);
```

位置使用**归一化视口坐标**（Normalized Viewport Coordinates），即整个渲染窗口左下角为(0, 0)，右上角为(1, 1)。这意味着位置的设置与窗口的实际像素大小无关。

### 12.4.4 在Renderer中布置Scalar Bar

将标量条添加到场景中有两种方式：

**方式一：作为普通Actor添加**

```cpp
renderer->AddActor2D(scalarBar);
```

`AddActor2D()`将ScalarBar作为2D叠加层添加，它的位置和尺寸始终以屏幕坐标（归一化视口坐标）为准，不受3D相机变换的影响。这是最常用的方式。

**方式二：使用vtkScalarBarWidget（可交互标量条）**

```cpp
#include <vtkScalarBarWidget.h>

vtkNew<vtkScalarBarWidget> scalarBarWidget;
scalarBarWidget->SetScalarBarActor(scalarBar);
scalarBarWidget->SetInteractor(renderWindowInteractor);
scalarBarWidget->On();  // 启用交互
```

`vtkScalarBarWidget`提供了可交互的标量条，用户可以用鼠标移动和调整标量条的位置。但对于大多数静态可视化，使用`AddActor2D()`就足够了。

### 12.4.5 多视口下标量条的管理

当你在一个窗口中放置多个Renderer（多视口布局）时，每个Renderer可以有自己独立的标量条。标量条的位置是相对于其所属Renderer的视口而言的。这是一种良好的设计，因为每个子视图可以有独立的颜色表和数据范围。

---

## 12.5 标量范围控制

标量范围（Scalar Range）是决定颜色映射视觉效果的关键参数。即使使用完全相同的颜色表和完全相同的数据，不同的标量范围设置也会产生截然不同的可视化结果。

### 12.5.1 SetScalarRange()的工作原理

Mapper上的`SetScalarRange(min, max)`定义了一个映射区间。这个区间告诉Mapper："将标量值min映射到颜色表的起始端，将标量值max映射到颜色表的终止端"。

```cpp
// 在Mapper上设置标量范围
mapper->SetScalarRange(0.0, 100.0);
// 标量值0.0 --> 查找表起始颜色（蓝色）
// 标量值50.0 --> 查找表中间颜色（绿色）
// 标量值100.0 --> 查找表终止颜色（红色）
```

内部执行的映射计算为：

```
t = (scalar_value - min) / (max - min)   // t的范围是[0, 1]
color = lut->MapValue(t)                   // 在查找表中查找
```

### 12.5.2 夹持（Clamping）行为

VTK默认启用夹持（Clamping）行为。这意味着超出范围的标量值不会导致错误，而是被"夹持"到最近的范围边界：

```cpp
// 假设设置了 SetScalarRange(0.0, 100.0)
double t;

// 正常情况：标量值在范围内
t = (50.0 - 0.0) / (100.0 - 0.0) = 0.5   // 获得中间颜色

// 下溢：标量值低于最小值
// 实际计算: ( -10.0 - 0.0 ) / 100.0 = -0.1 --> 夹持为 0.0
// 获得起始颜色（最小值对应的颜色）

// 上溢：标量值高于最大值
// 实际计算: (150.0 - 0.0) / 100.0 = 1.5 --> 夹持为 1.0
// 获得终止颜色（最大值对应的颜色）
```

夹持行为确保渲染不会因为数据中的极端异常值而出现错误颜色或崩溃。但这也意味着你需要合理设置标量范围，以确保感兴趣的数据区域获得充分的颜色区分度。

你可以通过以下方式启用或禁用夹持：

```cpp
// 禁用查找表夹持（不推荐用于大多数场景）
lut->SetClamping(false);
```

一般而言，保持默认的夹持行为是明智的选择。

### 12.5.3 动态范围发现：data->GetScalarRange()

在实际应用中，你通常不需要手动猜测数据的范围。VTK数据集提供了`GetScalarRange()`方法来获取当前活动标量数组的实际最小值和最大值：

```cpp
double range[2];
data->GetScalarRange(range);
std::cout << "Scalar range: [" << range[0] << ", " << range[1] << "]" << std::endl;

// 直接使用数据的实际范围
mapper->SetScalarRange(range);
```

`GetScalarRange()`会遍历整个活动标量数组来找到最小值和最大值，这是一个O(N)的操作。

如果你需要获取特定数组的范围而不是活动标量数组：

```cpp
vtkDataArray* arr = data->GetPointData()->GetArray("Temperature");
if (arr) {
    double range[2];
    arr->GetRange(range);
    // range[0] = 最小值, range[1] = 最大值
}
```

### 12.5.4 标量范围对视觉效果的影响

相同的颜色表和相同的数据，不同的标量范围会产生显著不同的视觉效果：

- **完整范围**（使用数据的实际最小值和最大值）：所有颜色被"拉伸"覆盖整个数据范围，数据中的每一个值都获得唯一的颜色。这提供了最大的颜色区分度。

- **窄范围**（人工设置更紧的范围，如只关注中间值）：颜色集中在感兴趣的区域，该区域内的微小变化获得较大的颜色差异。范围外的值被夹持为端点颜色。

- **宽范围**（人工设置更宽松的范围）：颜色分布更加"稀释"，数据中的变化对应的颜色差异更小。这适用于需要为未来的数据预留空间的场景，或者需要将多个数据集放在统一色阶下比较的场景。

这正是12.6节中多方案对比示例要展示的核心概念--你将看到完全相同的数据在三种不同标量范围下的着色效果。

---

## 12.6 代码示例：多方案对比展示

本节提供了一个完整的C++程序，演示本章涵盖的所有核心概念。程序创建一个具有高度变化的参数化曲面，然后在2x3的视口网格中展示不同的颜色映射方案。

### 12.6.1 程序概述

**第一行（Row 0）**：使用相同的彩虹色阶，但应用三种不同的标量范围。
- (0,0)：完整数据范围 `[0, 6]`
- (0,1)：缩小的范围 `[2, 4]` -- 放大中间区域
- (0,2)：扩展的范围 `[-3, 9]` -- 大部分数据被压入中间颜色

**第二行（Row 1）**：使用相同的完整数据范围，但应用五种不同的颜色映射方案。
- (1,0)：彩虹色阶（Rainbow） -- 使用`vtkColorTransferFunction`
- (1,1)：灰度色阶（Grayscale） -- 使用`vtkLookupTable`
- (1,2)：热金属色阶（Hot Metal） -- 黑-红-黄-白
- (1,3)：冷暖发散色阶（Cool-Warm） -- 蓝-白-红（发散型）
- (1,4)：地形色阶（Terrain） -- 蓝-绿-黄-棕-白

每个子窗口都配有独立的标量条。注意：第一行的三个子窗口共享同一个颜色表但标量范围不同，第二行的五个子窗口共享同一个标量范围但颜色表不同。

### 12.6.2 完整C++代码

```cpp
// ============================================================================
// Chapter 12: Color Mapping and Scalar Visualization
// File: ColorMappingDemo.cxx
// Description: A comprehensive demonstration of VTK color mapping techniques,
//              including vtkLookupTable, vtkColorTransferFunction, scalar bar,
//              and scalar range control. Uses a 2-row viewport grid:
//              Row 0: Same colormap, different scalar ranges
//              Row 1: Same scalar range, different colormaps
// VTK Version: 9.5.2
// ============================================================================

#include <vtkActor.h>
#include <vtkCamera.h>
#include <vtkCleanPolyData.h>
#include <vtkColorTransferFunction.h>
#include <vtkElevationFilter.h>
#include <vtkLookupTable.h>
#include <vtkNamedColors.h>
#include <vtkParametricFunctionSource.h>
#include <vtkParametricRandomHills.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkScalarBarActor.h>
#include <vtkSmartPointer.h>
#include <vtkTextProperty.h>

#include <array>
#include <vector>

// --------------------------------------------------------------------------
// Helper: Create a complex parametric surface with varying height
// Uses vtkParametricRandomHills to generate a terrain-like surface
// --------------------------------------------------------------------------
vtkSmartPointer<vtkPolyData> CreateSurface()
{
    // Create a random hills parametric surface -- generates smooth terrain
    vtkNew<vtkParametricRandomHills> hillsFunc;

    // Configure the random hills parameters
    hillsFunc->SetHillXVariance(2.5);       // Variance of hills in X direction
    hillsFunc->SetHillYVariance(2.5);       // Variance of hills in Y direction
    hillsFunc->SetAmplitude(3.0);           // Height amplitude
    hillsFunc->SetXVarianceScaleFactor(1.0);
    hillsFunc->SetYVarianceScaleFactor(1.0);
    hillsFunc->SetAllowRandomGeneration(true);
    hillsFunc->SetRandomSeed(42);           // Fixed seed for reproducibility
    hillsFunc->SetNumberOfHills(8);         // Number of hills

    // Generate the surface as a polydata
    vtkNew<vtkParametricFunctionSource> source;
    source->SetParametricFunction(hillsFunc);
    source->SetUResolution(80);             // Resolution in U direction
    source->SetVResolution(80);             // Resolution in V direction
    source->SetWResolution(80);
    source->SetScalarModeToZ();             // Use Z (height) as scalar value
    source->Update();

    // Clean the polydata to merge duplicate points
    vtkNew<vtkCleanPolyData> cleaner;
    cleaner->SetInputConnection(source->GetOutputPort());
    cleaner->Update();

    return cleaner->GetOutput();
}

// --------------------------------------------------------------------------
// Helper: Configure a renderer with a mapper, actor, and scalar bar
// Returns the renderer so it can be added to a viewport
// --------------------------------------------------------------------------
vtkSmartPointer<vtkRenderer> SetupViewport(
    vtkPolyData*                      surface,
    vtkScalarsToColors*               colorMap,      // Lookup table or CTF
    double                            scalarMin,
    double                            scalarMax,
    const std::string&                title,
    bool                              showScalarBar)
{
    // --- Mapper ---
    vtkNew<vtkPolyDataMapper> mapper;
    mapper->SetInputData(surface);
    mapper->SetLookupTable(colorMap);
    mapper->SetScalarRange(scalarMin, scalarMax);
    mapper->SetScalarVisibility(true);

    // --- Actor ---
    vtkNew<vtkActor> actor;
    actor->SetMapper(mapper);
    actor->GetProperty()->SetSpecular(0.3);
    actor->GetProperty()->SetSpecularPower(20);

    // --- Renderer ---
    vtkNew<vtkRenderer> renderer;
    renderer->AddActor(actor);
    renderer->SetBackground(0.15, 0.15, 0.15);
    renderer->SetViewport(0.0, 0.0, 1.0, 1.0);

    // --- Scalar Bar ---
    if (showScalarBar)
    {
        vtkNew<vtkScalarBarActor> scalarBar;
        scalarBar->SetLookupTable(colorMap);
        scalarBar->SetTitle(title.c_str());
        scalarBar->SetNumberOfLabels(6);
        scalarBar->SetLabelFormat("%.1f");
        scalarBar->SetOrientationToVertical();
        scalarBar->SetPosition(0.82, 0.1);
        scalarBar->SetWidth(0.12);
        scalarBar->SetHeight(0.7);
        scalarBar->GetTitleTextProperty()->SetFontSize(12);
        scalarBar->GetTitleTextProperty()->SetColor(1.0, 1.0, 1.0);
        scalarBar->GetLabelTextProperty()->SetFontSize(10);
        scalarBar->GetLabelTextProperty()->SetColor(1.0, 1.0, 1.0);
        scalarBar->SetTextPositionToPrecedeScalarBar();

        renderer->AddActor2D(scalarBar);
    }

    // Set camera to an overhead-ish view for terrain visualization
    renderer->GetActiveCamera()->SetPosition(0, -12, 6);
    renderer->GetActiveCamera()->SetFocalPoint(0, 0, 1.5);
    renderer->GetActiveCamera()->SetViewUp(0, 0, 1);
    renderer->GetActiveCamera()->SetClippingRange(0.1, 50);
    renderer->ResetCamera();

    return renderer;
}

// ============================================================================
// Color Map Factories
// ============================================================================

// --- Rainbow CTF (continuous) ---
vtkSmartPointer<vtkColorTransferFunction> CreateRainbowCTF()
{
    vtkNew<vtkColorTransferFunction> ctf;
    ctf->AddRGBPoint(0.0,   0.231, 0.298, 0.753);  // Blue
    ctf->AddRGBPoint(0.25,  0.0,   0.706, 0.776);  // Cyan
    ctf->AddRGBPoint(0.5,   0.302, 0.686, 0.290);  // Green
    ctf->AddRGBPoint(0.75,  1.0,   0.925, 0.157);  // Yellow
    ctf->AddRGBPoint(1.0,   0.851, 0.110, 0.110);  // Red
    return ctf;
}

// --- Grayscale LUT ---
vtkSmartPointer<vtkLookupTable> CreateGrayscaleLUT()
{
    vtkNew<vtkLookupTable> lut;
    lut->SetNumberOfTableValues(256);
    lut->SetHueRange(0.0, 0.0);
    lut->SetSaturationRange(0.0, 0.0);
    lut->SetValueRange(0.1, 1.0);       // Dark gray to white
    lut->SetRamp("linear");
    lut->Build();
    return lut;
}

// --- Hot Metal LUT ---
vtkSmartPointer<vtkLookupTable> CreateHotMetalLUT()
{
    vtkNew<vtkLookupTable> lut;
    lut->SetNumberOfTableValues(256);
    lut->SetHueRange(0.0, 0.15);        // Red through orange to yellow
    lut->SetSaturationRange(1.0, 1.0);
    lut->SetValueRange(0.1, 1.0);       // Dark (black-red) to bright (yellow-white)
    lut->SetRamp("sqrt");               // Faster transition at the low end
    lut->Build();
    return lut;
}

// --- Cool-Warm Diverging CTF ---
vtkSmartPointer<vtkColorTransferFunction> CreateCoolWarmCTF()
{
    vtkNew<vtkColorTransferFunction> ctf;
    // Cool (blue) at low end, neutral (white/gray) in middle, warm (red) at high end
    ctf->AddRGBPoint(0.0,  0.231, 0.298, 0.753);  // Blue
    ctf->AddRGBPoint(0.25, 0.553, 0.690, 0.996);  // Light blue
    ctf->AddRGBPoint(0.5,  1.0,   1.0,   1.0);    // White
    ctf->AddRGBPoint(0.75, 0.988, 0.710, 0.561);  // Light red
    ctf->AddRGBPoint(1.0,  0.851, 0.110, 0.110);  // Red
    return ctf;
}

// --- Terrain Elevation CTF ---
vtkSmartPointer<vtkColorTransferFunction> CreateTerrainCTF()
{
    vtkNew<vtkColorTransferFunction> ctf;
    // Simulating a topographic map: deep blue -> shallow blue -> green ->
    // yellow green -> tan -> brown -> light gray -> white
    ctf->AddRGBPoint(0.0,   0.2,  0.3,  0.7);   // Deep water: dark blue
    ctf->AddRGBPoint(0.1,   0.3,  0.5,  0.9);   // Shallow water: medium blue
    ctf->AddRGBPoint(0.2,   0.2,  0.7,  0.3);   // Lowland: green
    ctf->AddRGBPoint(0.4,   0.5,  0.8,  0.2);   // Forest: yellow-green
    ctf->AddRGBPoint(0.6,   0.7,  0.7,  0.3);   // Hills: tan-gold
    ctf->AddRGBPoint(0.8,   0.6,  0.4,  0.2);   // Mountains: brown
    ctf->AddRGBPoint(0.9,   0.8,  0.75, 0.7);   // Rocky: light gray
    ctf->AddRGBPoint(1.0,   1.0,  1.0,  1.0);   // Snow: white
    return ctf;
}

// ============================================================================
// Main
// ============================================================================
int main(int argc, char* argv[])
{
    // ----------------------------------------------------------------------
    // 1. Create the surface geometry
    // ----------------------------------------------------------------------
    auto surface = CreateSurface();

    // Get the actual data range
    double dataRange[2];
    surface->GetScalarRange(dataRange);
    std::cout << "Surface scalar range: [" << dataRange[0] << ", "
              << dataRange[1] << "]" << std::endl;

    // ----------------------------------------------------------------------
    // 2. Pre-create all color maps
    // ----------------------------------------------------------------------
    auto rainbowCTF   = CreateRainbowCTF();
    auto grayscaleLUT = CreateGrayscaleLUT();
    auto hotMetalLUT  = CreateHotMetalLUT();
    auto coolWarmCTF  = CreateCoolWarmCTF();
    auto terrainCTF   = CreateTerrainCTF();

    // Also create a discrete rainbow LUT for Row 0 (same color, different ranges)
    vtkNew<vtkLookupTable> rainbowLUT;
    rainbowLUT->SetNumberOfTableValues(256);
    rainbowLUT->SetHueRange(0.6667, 0.0);
    rainbowLUT->SetSaturationRange(1.0, 1.0);
    rainbowLUT->SetValueRange(0.6, 1.0);
    rainbowLUT->SetRamp("linear");
    rainbowLUT->Build();

    // ----------------------------------------------------------------------
    // 3. Build the 2-row viewport layout
    //    Row 0: 3 viewports (different scalar ranges, same LUT)
    //    Row 1: 5 viewports (different colormaps, same scalar range)
    //
    //    We'll use a 5-column grid, with Row 0 viewports spanning adjusted
    //    widths and Row 1 using all 5 columns evenly.
    //
    //    Layout:
    //    +-------------+-------------+-------------+
    //    |   (0,0)     |   (0,1)     |   (0,2)     |   Row 0: Range demo
    //    +-------------+-------------+-------------+
    //    | (1,0)| (1,1)| (1,2)| (1,3)| (1,4)|       Row 1: Colormap demo
    //    +------+------+------+------+------+
    // ----------------------------------------------------------------------

    // We actually build a 5-column layout for Row 1, and merge columns for Row 0.
    // Row 0 uses 3 equal columns: each spanning approximately 5/3 columns.
    // For simplicity, we'll use a full 2x5 grid and only use the needed cells.

    const int nCols = 5;
    const int nRows = 2;

    // Row 0: 3 viewports (col 0-1, col 2-3, col 4)
    // Row 1: 5 viewports (each col)

    // Define viewport coordinates for each sub-window
    // Format: {xmin, ymin, xmax, ymax}
    struct ViewportCoords {
        double xmin, ymin, xmax, ymax;
    };

    // Row 0 (y from 0.5 to 1.0) -- 3 equal-width viewports
    double row0YMin = 0.5;
    double row0YMax = 1.0;
    double row0Widths[3] = {0.0, 1.0/3.0, 2.0/3.0};
    double row0Ends[3]   = {1.0/3.0, 2.0/3.0, 1.0};

    // Row 1 (y from 0.0 to 0.5) -- 5 equal-width viewports
    double row1YMin = 0.0;
    double row1YMax = 0.5;

    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(1800, 800);
    renderWindow->SetWindowName("Chapter 12: Color Mapping and Scalar Visualization");

    // Storage for renderers
    std::vector<vtkSmartPointer<vtkRenderer>> allRenderers;

    // ----------------------------------------------------------------------
    // Row 0: Same colormap (rainbowLUT), different scalar ranges
    // ----------------------------------------------------------------------

    // Viewport (0,0): Full data range
    {
        double r0 = dataRange[0];
        double r1 = dataRange[1];
        auto renderer = SetupViewport(surface, rainbowLUT, r0, r1,
            "Full Range", true);
        renderWindow->AddRenderer(renderer);
        renderer->SetViewport(row0Widths[0], row0YMin, row0Ends[0], row0YMax);
        allRenderers.push_back(renderer);
    }

    // Viewport (0,1): Narrow range [2, 4] -- zoom into middle values
    {
        auto renderer = SetupViewport(surface, rainbowLUT, 2.0, 4.0,
            "Range: [2, 4] (Narrow)", true);
        renderWindow->AddRenderer(renderer);
        renderer->SetViewport(row0Widths[1], row0YMin, row0Ends[1], row0YMax);
        allRenderers.push_back(renderer);
    }

    // Viewport (0,2): Wide range [-3, 9] -- extended beyond data
    {
        auto renderer = SetupViewport(surface, rainbowLUT, -3.0, 9.0,
            "Range: [-3, 9] (Wide)", true);
        renderWindow->AddRenderer(renderer);
        renderer->SetViewport(row0Widths[2], row0YMin, row0Ends[2], row0YMax);
        allRenderers.push_back(renderer);
    }

    // ----------------------------------------------------------------------
    // Row 1: Same scalar range (full data range), different colormaps
    // ----------------------------------------------------------------------
    double rMin = dataRange[0];
    double rMax = dataRange[1];

    struct ColormapEntry {
        vtkSmartPointer<vtkScalarsToColors> cmap;
        std::string                          label;
    };

    ColormapEntry colormaps[5] = {
        { rainbowCTF,   "Rainbow (CTF)" },
        { grayscaleLUT, "Grayscale (LUT)" },
        { hotMetalLUT,  "Hot Metal (LUT)" },
        { coolWarmCTF,  "Cool-Warm (CTF)" },
        { terrainCTF,   "Terrain (CTF)" }
    };

    for (int col = 0; col < 5; ++col)
    {
        double xMin = static_cast<double>(col) / nCols;
        double xMax = static_cast<double>(col + 1) / nCols;

        auto renderer = SetupViewport(surface,
            colormaps[col].cmap, rMin, rMax,
            colormaps[col].label, true);
        renderWindow->AddRenderer(renderer);
        renderer->SetViewport(xMin, row1YMin, xMax, row1YMax);
        allRenderers.push_back(renderer);
    }

    // ----------------------------------------------------------------------
    // 4. Synchronize cameras across all renderers for consistent views
    // ----------------------------------------------------------------------
    // We'll copy the camera from the first renderer to all others after
    // the first render, using a callback or by sharing the camera.
    // For simplicity, we set each camera to the same position.
    for (auto& ren : allRenderers)
    {
        ren->GetActiveCamera()->SetPosition(0, -12, 6);
        ren->GetActiveCamera()->SetFocalPoint(0, 0, 1.5);
        ren->GetActiveCamera()->SetViewUp(0, 0, 1);
        ren->ResetCamera();
    }

    // ----------------------------------------------------------------------
    // 5. Start the interactive window
    // ----------------------------------------------------------------------
    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    renderWindow->Render();
    interactor->Initialize();
    interactor->Start();

    return 0;
}
```

### 12.6.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.12)
project(Chapter12_ColorMappingDemo)

# Find VTK
find_package(VTK REQUIRED COMPONENTS
    CommonCore
    CommonColor
    CommonDataModel
    CommonExecutionModel
    CommonMath
    CommonTransforms
    FiltersCore
    FiltersSources
    InteractionStyle
    InteractionWidgets
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
add_executable(ColorMappingDemo ColorMappingDemo.cxx)

# Link VTK libraries
target_link_libraries(ColorMappingDemo
    VTK::CommonCore
    VTK::CommonColor
    VTK::CommonDataModel
    VTK::CommonExecutionModel
    VTK::CommonMath
    VTK::CommonTransforms
    VTK::FiltersCore
    VTK::FiltersSources
    VTK::InteractionStyle
    VTK::InteractionWidgets
    VTK::RenderingAnnotation
    VTK::RenderingContextOpenGL2
    VTK::RenderingCore
    VTK::RenderingFreeType
    VTK::RenderingOpenGL2
)

# Set C++ standard
set_target_properties(ColorMappingDemo PROPERTIES
    CXX_STANDARD 17
    CXX_STANDARD_REQUIRED ON
)
```

### 12.6.4 编译与运行

**Windows (Visual Studio):**

```powershell
# 在源代码目录下
mkdir build
cd build
cmake .. -G "Visual Studio 17 2022"
cmake --build . --config Release

# 运行
.\Release\ColorMappingDemo.exe
```

**Linux / macOS:**

```bash
mkdir build && cd build
cmake ..
cmake --build .
./ColorMappingDemo
```

### 12.6.5 预期结果分析

运行程序后，你将看到一个包含8个子窗口的大窗口：

**第一行（标量范围对比）**：

| 位置 | 标量范围 | 预期效果 |
|------|---------|---------|
| (0,0) | 完整范围`[~0, ~6]` | 颜色均匀分布在整个曲面上。低洼区域呈现蓝色，高峰区域呈现红色，中间过渡清晰可见。这是"标准"的显示效果。 |
| (0,1) | 窄范围`[2, 4]` | 高度在2到4之间的区域获得了最大的颜色区分度。低于2的区域全部呈现蓝色（被夹持），高于4的区域全部呈现红色（被夹持）。曲面的细节在中间高度区域被"放大"了。 |
| (0,2) | 宽范围`[-3, 9]` | 整个曲面的高度值只占据了颜色表中间的一部分。颜色对比度降低，曲面看起来颜色变化较为"平淡"，因为所有数据值都被映射到了颜色表的一个较小区间。 |

**第二行（颜色表对比）**：

| 位置 | 颜色表 | 预期效果 |
|------|--------|---------|
| (1,0) | Rainbow CTF | 经典的彩虹色阶：蓝-青-绿-黄-红。色彩丰富但可能引起视觉误导（不同颜色饱和度给人的"重要性"感不同）。 |
| (1,1) | Grayscale LUT | 纯灰度显示：从深灰到白色。强调的是数值的大小关系而非绝对的视觉分类。对于色盲友好，但不适合展示细微差异。 |
| (1,2) | Hot Metal LUT | 黑-红-橙-黄-白的过渡。模仿金属加热时的颜色变化。低值区域快速从黑色过渡到红色（因为使用了sqrt渐变），高值区域呈现明亮的黄色和白色。 |
| (1,3) | Cool-Warm CTF | 发散型色阶：低值蓝色，中间白色，高值红色。白色作为中性分界点，非常适合展示与某个参考值（如平均值）的偏差。 |
| (1,4) | Terrain CTF | 模拟地形图：深海蓝到草地绿到山地棕到雪顶白。这是地理信息系统(GIS)中常用的配色方案，让观察者直观地联想到真实地形。 |

通过这个综合示例，你可以直观地理解：**颜色映射方案的选择和标量范围的设定，对数据可视化的传达效果有着截然不同的影响**。在实际应用中，你需要根据数据的性质和你想传达的信息来做出选择。

---

## 12.7 本章小结

本章系统性地介绍了VTK中的颜色映射与标量可视化技术。以下是本章的核心要点回顾：

### 核心概念

1. **颜色映射**是将数据场中的标量值转换为显示颜色的过程。它由Mapper执行，通过查找表（LUT）或颜色传递函数（CTF）实现。

2. **活动标量**（Active Scalars）是Mapper自动选择用于着色的标量数组。一个数据集可以包含多个标量场，通过`SetActiveScalars()`指定当前使用哪一个。

3. **标量范围**（Scalar Range）定义了标量值的有效区间。它直接影响颜色映射的视觉效果：窄范围"放大"感兴趣区域的细节，宽范围"压缩"颜色变化。

### 关键类与API

| 类 | 主要用途 | 关键方法 |
|----|---------|---------|
| `vtkLookupTable` | 离散颜色表 | `SetNumberOfTableValues()`, `SetTableValue()`, `SetHueRange()`, `SetRamp()`, `SetScaleToLinear()`, `Build()` |
| `vtkColorTransferFunction` | 连续颜色映射 | `AddRGBPoint()`, `AddHSVPoint()`, `AddRGBAPoint()`, `GetColor()` |
| `vtkNamedColors` | 内置颜色库 | `GetColor3d(name)` |
| `vtkScalarBarActor` | 颜色图例/标量条 | `SetLookupTable()`, `SetTitle()`, `SetNumberOfLabels()`, `SetOrientationToVertical()` |
| `vtkMapper` (基类) | 颜色映射执行者 | `SetScalarVisibility()`, `SetScalarRange()`, `SetLookupTable()`, `SetScalarMode()`, `SelectColorArray()` |
| `vtkDataSet` (基类) | 标量数据容器 | `GetScalarRange()`, `GetPointData()->SetActiveScalars()` |

### 选择指南

- **何时使用vtkLookupTable**：需要离散颜色带（如等温线图）、需要精确控制条目数量、或者需要HSV自动生成色阶。
- **何时使用vtkColorTransferFunction**：需要完全平滑的颜色过渡（如地形图、热力分布图）、需要在任意标量位置定义精确颜色、或者需要用于体渲染。
- **标量范围选择**：通常使用数据的完整范围（`GetScalarRange()`）作为默认设置。当需要突出特定数值区域时，可以缩小范围；当需要为未来数据预留空间或统一多个数据集的色阶时，可以扩大范围。

### 与实践的联系

你现在掌握的技能可以用于：

- 将FEM仿真结果（应力、位移、温度）映射为有意义的颜色
- 为医学影像数据（CT值、MRI信号强度）创建诊断友好的色阶
- 在气象可视化中使用发散型色阶展示温度偏差
- 为地形数据选择符合制图学惯例的高程色阶
- 通过标量条为观察者提供数值参考

### 下一步

颜色映射是"可视化表达"的起点。在掌握了如何将数据值映射为颜色之后，你需要进一步学习：

- **第十三章：等值线与等值面**--将颜色映射扩展到三维标量场的等值面提取
- **第十四章：体渲染入门**--使用颜色传递函数和不透明度传递函数直接渲染三维体数据
- **第十五章：复合可视化技术**--在同一场景中组合多种可视化技术（如流线 + 等值面 + 半透明体渲染）

颜色映射是你可视化工具箱中最基础也最常用的工具。花时间将本章的代码示例运行一遍，尝试修改颜色方案和标量范围，观察视觉效果的变化--这将帮助你形成对颜色映射的直观理解。
