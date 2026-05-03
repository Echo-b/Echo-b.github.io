# 第二章 可视化管线（Visualization Pipeline）

## 本章导读

如果你只打算认真读一章VTK教程，那就应该是这一章。

第一章让你看到了VTK的"是什么"（What）——你安装好了环境，写出了一个Cone示例，知道VTK由Source、Mapper、Actor这些东西组成。但你不一定理解它们"为什么"（Why）要这么设计，以及"如何"（How）正确地组合它们。

本章的目标就是回答这两个问题。我们将深入VTK最核心的设计概念——**管道（Pipeline）**。管道不是VTK的一个可选功能，也不是一种使用技巧；管道是VTK本身。VTK的每一个类、每一个方法、每一个设计决策，背后都贯穿着管道架构的逻辑。

本章覆盖以下内容：

1. **管道的整体模型**——Source、Filter、Mapper、Actor、Renderer、RenderWindow各自扮演什么角色，以及可视化管道与渲染管道的分界线在哪。
2. **数据如何沿着管道流动**——`vtkAlgorithm`基类、`vtkAlgorithmOutput`输出端口、`SetInputConnection()`与`SetInputData()`的致命区别。
3. **管道何时执行计算**——按需执行（Demand-Driven Execution）机制、`Update()`的调用时机、修改时间戳（MTime）如何驱动缓存失效。
4. **管道的分支与合并**——一个数据源如何同时驱动多个可视化分支，以及如何将分叉的管道重新合并。
5. **渲染触发链**——从你调用`Render()`开始，VTK内部到底发生了什么。
6. **一个贯穿全章的完整示例**——用ConeSource + ShrinkFilter搭建一条真正有过滤器的管道，并观察参数修改后的行为。

读完本章后，你应该能够自信地回答："我理解VTK是如何工作的。"从第三章开始，我们将在这个基础上逐一攻克具体的数据类型和过滤器——那时你会感谢自己在这一章花的每一个小时。

---

## 2.1 管道模型概述

### 2.1.1 什么是管道

在计算机科学中，"管道"（Pipeline）是一种将复杂任务分解为一系列有序处理阶段的设计模式。每个阶段从一个方向接收输入，对数据执行某种变换，然后将结果传递给下一个阶段。这个模型在Unix命令行中广为人知：

```
cat data.txt | grep "error" | sort | uniq -c
```

上面的命令就是一个四阶段管道：读取文件、过滤包含"error"的行、排序、去重计数。每个命令只做一件事，数据像流水线一样从左向右流过。

VTK的管道在概念上完全一致，只不过管道的节点不是命令行程序，而是C++对象：

```
数据源(Source) --> 过滤器(Filter) --> 映射器(Mapper) --> 演员(Actor) --> 渲染器(Renderer) --> 渲染窗口(RenderWindow)
```

但这里有一个在Unix管道中看不到的关键特性：**VTK管道是按需执行（Demand-Driven）的**。Unix管道是"推"（Push）模型——上游产生数据，立即推给下游；VTK管道是"拉"（Pull）模型——下游请求数据，上游才去计算。这两个字的差别，决定了VTK整个执行引擎的设计。

### 2.1.2 管道的六个阶段

让我们——认识管道中每个阶段的角色。下面的描述中，我们会用到很多术语，但请不必记忆；把它们当作初次见面的朋友，在本章后续的深入讨论中你们会反复相遇。

#### 阶段一：数据源（Source）

**Source是管道的起点——它是数据的生产者，不依赖任何上游输入。**

Source的职责是"创造"数据：从文件读取、从算法生成、从外部设备采集。常见的Source包括：

| 类别 | 示例 | 说明 |
|------|------|------|
| 几何生成器 | `vtkConeSource`、`vtkSphereSource`、`vtkCylinderSource` | 用数学公式生成简单几何体 |
| 文件读取器 | `vtkSTLReader`、`vtkOBJReader`、`vtkXMLPolyDataReader` | 从磁盘文件加载数据 |
| 程序化数据源 | `vtkProgrammableSource` | 用自定义回调函数生成数据 |

在VTK的类继承体系中，所有Source都继承自`vtkAlgorithm`。Source的特殊之处在于：它没有输入端口（Input Port），只有输出端口（Output Port）。这是它区别于Filter的判定标准。

#### 阶段二：过滤器（Filter）

**Filter是管道的加工车间——它接收数据，执行某种变换，输出变换后的数据。**

Filter是VTK中最丰富、最灵活的部分。1000多个Filter类涵盖了科学可视化中几乎所有的数据处理需求：

| 类别 | 示例 | 说明 |
|------|------|------|
| 几何操作 | `vtkShrinkPolyData`、`vtkSmoothPolyDataFilter`、`vtkDecimatePro` | 收缩、平滑、简化多边形数据 |
| 拓扑分析 | `vtkContourFilter`、`vtkSliceFilter`、`vtkClipDataSet` | 等值面提取、切片、裁剪 |
| 属性计算 | `vtkGradientFilter`、`vtkCurvatures`、`vtkMeshQuality` | 梯度计算、曲率估计、网格质量评估 |
| 类型转换 | `vtkTriangleFilter`、`vtkDataSetTriangleFilter` | 将多边形转换为三角形 |
| 数据抽取 | `vtkProbeFilter`、`vtkPointInterpolator` | 在指定位置采样数据 |

Filter同时拥有输入端口和输出端口。一个Filter可以将上一级的输出作为自己的输入——这正是管道可组合性的来源。

#### 阶段三：映射器（Mapper）

**Mapper是可视化管道与渲染管道的分界桥——它将数据对象转换为图形基元（Graphics Primitives）。**

在Mapper之前，管道处理的是"数据"（Points、Cells、Arrays）；在Mapper之后，管道处理的是"图形"（Triangles、Lines、Textures、Shaders）。可以说，Mapper是"数据世界"和"图形世界"之间的翻译官。

Mapper的主要职责包括：

1. **几何转换**：将vtkDataSet中的点（Points）和单元（Cells）转换为OpenGL可渲染的顶点和三角形列表。
2. **属性映射**：将数据的标量场（Scalars）、矢量场（Vectors）映射为颜色、大小、纹理坐标等可视化属性。
3. **LOD管理**：一些Mapper支持多级细节（Level of Detail），在交互时降低几何复杂度以保持帧率。

常用的Mapper包括：

| Mapper | 适用数据类型 | 说明 |
|--------|-------------|------|
| `vtkPolyDataMapper` | vtkPolyData | 最常用的多边形数据映射器 |
| `vtkDataSetMapper` | vtkDataSet | 通用数据集映射器（内部根据具体类型委派） |
| `vtkGlyph3DMapper` | vtkPolyData + vtkDataSet | 将点数据渲染为三维符号（箭头、球体等） |
| `vtkVolumeRayCastMapper` | vtkImageData | 光线投射体渲染映射器 |

#### 阶段四：演员（Actor）

**Actor是场景中的可视化实体——它不直接处理几何数据，而是持有Mapper和Property，描述"画什么"和"怎么画"。**

Actor是你操作场景中物体的直接接口。一个Actor包含两个核心组件：

- **Mapper（映射器）**：决定这个物体从哪种数据而来。
- **Property（属性）**：决定这个物体的外观——颜色、材质、不透明度、边界线样式、着色模式等。

Actor还负责管理物体的空间位置：你可以通过`actor->SetPosition(x, y, z)`、`actor->SetOrientation(rx, ry, rz)`、`actor->SetScale(sx, sy, sz)`来变换物体在场景中的位姿。

重要概念：Actor与Mapper是一对一的关系（一个Actor只有一个Mapper），但Mapper与Actor是一对多的关系（多个Actor可以共享同一个Mapper，从而在不同的位置显示相同的数据）。

#### 阶段五：渲染器（Renderer）

**Renderer是场景的舞台——它管理一组Actor、一个或多个光源（Light）、以及一台相机（Camera）。**

Renderer的职责包括：

1. **Actor管理**：维护场景中的Actor列表，管理它们的添加、移除和可见性。
2. **相机控制**：持有`vtkCamera`对象，定义观察者的位置、朝向、投影方式（正交/透视）。
3. **光照管理**：持有光源列表，控制场景的照明环境。
4. **背景设置**：设置背景颜色、渐变、或背景纹理。
5. **渲染执行**：当收到渲染请求时，遍历Actor列表，调度Mapper执行数据到图形的转换，然后将结果绘制到RenderWindow。

一个RenderWindow可以包含多个Renderer（每个占据窗口的一个矩形区域），这使得你可以在同一个窗口中创建多个视图——例如科学可视化的"前视图+顶视图+侧视图+三维透视图"四视图布局。

#### 阶段六：渲染窗口（RenderWindow）

**RenderWindow是管道的终点——它代表屏幕上的一个窗口，负责将Renderer的渲染结果呈现给用户。**

RenderWindow的职责是封装操作系统相关的窗口操作：窗口创建、大小调整、像素显示、用户输入事件路由。在VTK中，这些底层细节由具体的窗口后端实现（Windows上用Win32 API、Linux上用X11或Wayland、macOS上用Cocoa），而`vtkRenderWindow`提供了统一的抽象接口。

RenderWindow本身不执行渲染计算——它将这个任务委派给Renderer，Renderer再委派给Mapper，逐级向上游请求，最终触发整条管道的执行。

### 2.1.3 两条管道的分界

现在我们来理清一个经常让初学者混淆的概念：VTK管道实际上由**两层管道**叠加而成。

```
+------------------------- 可视化管道 --------------------------+
|                                                                |
| Source --> Filter --> Filter --> ... --> Mapper                |
|                                         |                      |
+-----------------------------------------+----------------------+
                                          |
                  +-----------------------+
                  |
+-----------------+------------ 渲染管道 -------------------------+
|                 |                                                |
|                 v                                                |
|              Actor --> Renderer --> RenderWindow                 |
|                                                                |
+----------------------------------------------------------------+
```

**可视化管道（Visualization Pipeline）** 处理的是数据：

- 输入：原始数据（文件、算法生成的几何体、标量场、矢量场等）
- 处理：几何变换、拓扑分析、属性计算、类型转换
- 输出：经过Mapper转换后的图形基元
- 核心类：`vtkAlgorithm`及其子类（XxxSource、XxxFilter、XxxMapper）

**渲染管道（Rendering Pipeline）** 处理的是显示：

- 输入：Mapper产生的图形基元
- 处理：空间变换、光照计算、相机投影、光栅化
- 输出：屏幕上的像素
- 核心类：`vtkProp`及其子类（`vtkActor`、`vtkVolume`等）、`vtkRenderer`、`vtkRenderWindow`

两条管道通过**Mapper与Actor的绑定**（`actor->SetMapper(mapper)`）连接在一起。这个绑定就像电源插座——可视化管道"供电"，渲染管道"用电"。

理解这个分层对于调试问题至关重要。当你的程序"什么都不显示"时，你需要判断：是数据没有正确生成（可视化管道的问题），还是数据生成了但没有被正确渲染（渲染管道的问题）？两个层面的排查方法完全不同。

### 2.1.4 ASCII全景图

下面是管道数据流的完整ASCII示意图。建议你在继续阅读之前，花两分钟时间仔细看一下这幅图，在大脑中建立管道的空间结构。

```
                           可视化管道
    +-----------------------------------------------------------+
    |                                                           |
    |  +----------+    +----------+    +----------+    +--------+-+
    |  |  Source  |    |  Filter  |    |  Filter  |    | Mapper  |
    |  |          |    |          |    |          |    |         |
    |  | 生成数据  |--->| 变换数据  |--->| 再变换   |--->| 数据->图形|
    |  |          |    |          |    |          |    |         |
    |  +----------+    +----------+    +----------+    +--------+-+
    |                              ^                          |
    |                              |                          |
    |                    SetInputConnection()          SetInputConnection()
    |                        连接上下游                    连接Mapper
    +-----------------------------------------------------------+
                                                           |
                                       actor->SetMapper(mapper)
                                                           |
                           渲染管道                         |
    +-----------------------------------------------------------+
    |                                                      |    |
    |  +--------+-+    +-----------+    +---------------+  |    |
    |  |  Actor   |    | Renderer  |    | RenderWindow  |  |    |
    |  |          |    |           |    |               |  |    |
    |  | 持有Mapper|<---| 管理Actor,|--->| 屏幕上的窗口   |  |    |
    |  | 持有属性  |    | 光源,相机  |    |               |  |    |
    |  |          |    |           |    |               |  |    |
    |  +----------+    +-----------+    +---------------+  |    |
    |        ^                                              |    |
    |        |                                              |    |
    |  renderer->AddActor(actor)                            |    |
    +-----------------------------------------------------------+
```

### 2.1.5 为什么管道架构如此重要

如果你以前使用过Matplotlib或MATLAB来画图，你可能会觉得VTK的管道模型太"重"了。在Matplotlib中，画一个图形只需要`plt.plot(x, y)`——为什么VTK需要这么多步骤？

答案是：**管道架构为灵活性付出的代价，在复杂场景中会以百倍的形式回馈给你。**

考虑以下实际场景：

1. **多视图可视化**：同一个数据集，你想同时看到它的三维模型、二维切片、数值统计图表。在管道架构中，你只需将一个Source连接到三个不同的Filter分支，每个分支产生不同的可视化结果——数据只计算一次，但可以以多种方式观看。

2. **交互式参数探索**：你拖动一个滑块改变等值面的提取阈值。在管道架构中，当你改变阈值时，只有等值面提取及其下游节点需要重新计算——Source的数据一旦生成就被复用，不需要重新从文件读取。

3. **计算缓存**：你有一个昂贵的科学计算Source，生成数十GB的数据。在管道架构中，这个Source可以连接到多个Filter，每个Filter都复用同一份上游数据——而不是为每个下游处理都重新算一次。

4. **管道组合**：你可以将一个复杂的处理流程封装成一个自定义Filter，然后像搭乐高积木一样将它插入到任意管道中——因为所有Filter都遵循相同的输入/输出接口。

这些能力在简单的`plt.plot()`式API中是不可能实现的。VTK选择了前期学习成本更高但后期表达能力更强的设计——这就是管道架构存在的根本原因。

---

## 2.2 数据流与连接

### 2.2.1 vtkAlgorithm——管道世界的基石

在VTK中，**所有参与管道的对象有一个共同的基类：`vtkAlgorithm`**。这个类定义了管道对象之间如何连接、如何传递数据、如何触发执行的统一接口。无论你是在处理一个ConeSource、一个ShrinkFilter、还是一个PolyDataMapper，它们都共享着从`vtkAlgorithm`继承来的管道行为。

`vtkAlgorithm`的核心职责包括：

1. **端口管理（Port Management）**：定义算法有多少个输入端口（Input Port）和多少个输出端口（Output Port）。
2. **连接管理（Connection Management）**：提供`SetInputConnection()`方法，使得一个算法的输出端口可以连接到另一个算法的输入端口。
3. **执行调度（Execution Scheduling）**：提供`Update()`方法，触发从当前节点往上游追溯、直到所有上游都完成计算。
4. **信息查询（Information Query）**：提供`GetOutput()`方法，让下游获取上游产生的数据结果。

以下是一些关键方法的职责速览：

```cpp
// 获取指定输出端口（默认端口索引为0）
vtkAlgorithmOutput* GetOutputPort(int port = 0);

// 获取指定输出端口的数据（必须先Update()）
vtkDataObject* GetOutputDataObject(int port = 0);

// 设置第port个输入端口的连接
void SetInputConnection(int port, vtkAlgorithmOutput* input);
void SetInputConnection(vtkAlgorithmOutput* input);  // port=0

// 直接设置第port个输入端口的输入数据（绕过管道连接）
void SetInputDataObject(int port, vtkDataObject* data);
void SetInputDataObject(vtkDataObject* data);        // port=0

// 触发管道执行——从当前节点往上游追溯并计算
virtual void Update(int port = 0);

// 获取管道信息（元数据：数据范围、类型等，不触发执行）
vtkInformation* GetOutputInformation(int port = 0);
```

`GetOutputPort()`的返回值类型是`vtkAlgorithmOutput*`。这引出了管道中另一个关键概念——输出端口对象。

### 2.2.2 vtkAlgorithmOutput——输出端口

`vtkAlgorithmOutput`是一个小而重要的类。它的作用很简单：**代表一个算法的某个输出端口，作为下游算法SetInputConnection()的参数**。

你可以把它想象成一个"管道接头"：一个算法的输出是一根管子的出口端，另一个算法的输入是管子的入口端。`vtkAlgorithmOutput`就是这个接头——它让两根管子能够严丝合缝地对接。

```cpp
// Source产生输出端口
vtkNew<vtkConeSource> cone;
vtkAlgorithmOutput* sourceOutput = cone->GetOutputPort();  // 获取接头

// Filter连接：将Source的输出接头插入Filter的输入端口
vtkNew<vtkShrinkPolyData> shrink;
shrink->SetInputConnection(sourceOutput);                  // 插入接头

// 继续连接下游
vtkNew<vtkPolyDataMapper> mapper;
vtkAlgorithmOutput* filterOutput = shrink->GetOutputPort();
mapper->SetInputConnection(filterOutput);
```

为什么VTK要设计一个专门的`vtkAlgorithmOutput`类，而不是让`GetOutputPort()`直接返回算法对象的指针？这是为了支持**多输出端口**。有些算法拥有不止一个输出端口（例如提取表面的Filter可能同时输出表面和内部数据），用专门的端口对象可以精确指定"到底是哪个输出"。此外，`vtkAlgorithmOutput`在内部维护了连接信息，使得管道能够高效地向上游追溯执行。

### 2.2.3 SetInputConnection() vs SetInputData()——本章最重要的概念

> **如果你读到这里只能记住一件事，那就是这一节的内容。**

VTK提供了两种将数据传递给下游算法的方式。它们在名称上只有几个字母的区别，但在行为上是天壤之别：

- **`SetInputConnection()`**：连接管道，保留按需执行
- **`SetInputData()`**：传递快照，打断管道

#### SetInputConnection()——管道式连接

```cpp
vtkNew<vtkConeSource> cone;
vtkNew<vtkShrinkPolyData> shrink;

// 【正确做法】用SetInputConnection建立管道连接
shrink->SetInputConnection(cone->GetOutputPort());
```

当使用`SetInputConnection()`时，你做的事情是：告诉Shrink，"你的输入数据将从Cone的输出端口获取，但不必现在就问Cone要数据——等你被要求输出结果时再说。"

此时，**Cone和Shrink之间建立了一条"活的"连接**。这条连接的生命周期行为如下：

1. Shrink在初次被要求输出时，会沿着这条连接向上游追溯，找到Cone。
2. Shrink请求Cone执行计算（如果Cone还没有计算过的话）。
3. Shrink拿到Cone的输出数据，执行自己的处理，返回结果。
4. 此后，如果你修改了Cone的参数（例如调用`cone->SetResolution(100)`），Cone的MTime会更新。
5. 下次Shrink被要求输出时，它会发现Cone的MTime比它上次缓存的更新鲜——于是它再次请求Cone重新计算，然后基于新数据重新执行自己的处理。

**这就是管道的精髓：下游始终能从上游获取最新的数据，不需要你手动管理数据一致性。** 数据"按需流动"，计算"按需执行"。

#### SetInputData()——快照式赋值

```cpp
vtkNew<vtkConeSource> cone;
cone->Update();                    // 强制Cone立即计算
vtkPolyData* snapshot = cone->GetOutput();  // 拿走结果

vtkNew<vtkShrinkPolyData> shrink;

// 【注意】这里打断了管道
shrink->SetInputData(snapshot);    // 传递的是一个快照
```

当使用`SetInputData()`时，你做的事情是：直接把自己的数据塞给Shrink，说："这就是你的输入，拿去用，不用管数据从哪来的。"

此时，**Shrink的上游管道被切断了**。Shrink不再知道自己处理的数据来自于Cone——它只知道手里有一份数据快照。这条连接的行为如下：

1. Shrink被要求输出时，直接使用你传入的这份数据。
2. 如果你修改了Cone的参数（例如调用`cone->SetResolution(100)`），Shrink完全不知情——它手里还是那旧快照。
3. 要让Shrink使用新数据，你必须手动再次调用`cone->Update()`和`shrink->SetInputData(cone->GetOutput())`。

**SetInputData()打断了管道的按需执行链路，把自动化的数据流变回了手动管理。**

#### 直观对比表

| 特性 | SetInputConnection() | SetInputData() |
|------|---------------------|----------------|
| **管道连续性** | 保持管道连接 | 断开管道连接 |
| **参数变更传播** | 自动（通过MTime检测） | 不自动（需手动重新赋值） |
| **执行时机** | 按需（下游请求时才计算） | 上游需手动Update() |
| **内存使用** | 按需分配和释放 | 数据一直保持在内存中 |
| **适用场景** | 标准管道，数据需要随参数变化 | 静态快照，或需要脱离管道独立操作数据 |
| **执行效率** | 高（只计算需要的部分） | 低（可能做不必要的计算） |

#### 一个典型的错误用法

以下是初学者经常写的代码——它表面看起来正确，但存在隐患：

```cpp
vtkNew<vtkConeSource> cone;
cone->Update();  // 强制计算——但这破坏了按需执行的优雅性

vtkNew<vtkPolyDataMapper> mapper;
mapper->SetInputConnection(cone->GetOutputPort());  // 这里是Connection，管道还在
```

这段代码虽然能运行，但`cone->Update()`是完全多余的。因为`mapper->SetInputConnection(cone->GetOutputPort())`已经建立了管道连接——当渲染发生时，Mapper会自动向上游请求数据，Cone会自动计算。你手动调用的这个`Update()`破坏了管道的按需特性，使得Cone在真正需要它之前就执行了计算。

**一条经验法则：在使用`SetInputConnection()`的管道中，你几乎永远不需要手动调用`Update()`。** 渲染循环会帮你在正确的时机触发一切。

#### 什么时候该用SetInputData()

当然，`SetInputData()`并非一无是处。以下场景中，使用`SetInputData()`是合理的：

1. **手动构造的数据集**：你自己用代码创建了一个`vtkPolyData`，手动填充了点和单元——它没有上游Source，自然也无管道可连。

```cpp
vtkNew<vtkPoints> points;
// ... 手动添加点 ...
vtkNew<vtkPolyData> manualData;
manualData->SetPoints(points);
// 没有Source——只能SetInputData
mapper->SetInputData(manualData);
```

2. **脱离管道的后处理**：你想对管道中的某个中间结果进行复杂的自定义分析，而这些分析不需要VTK的Filter机制。

3. **数据快照/对比场景**：你想在管道中保留某时刻的数据"快照"，与后续的结果做对比。

**原则是：默认使用`SetInputConnection()`；只有在明确知道自己需要独立快照，或者数据来源不是VTK算法对象时，才使用`SetInputData()`。**

### 2.2.4 多输入端口

一些高级Filter拥有多个输入端口。例如，`vtkProbeFilter`需要两个输入：一个是被采样的数据集（输入端口0），另一个是采样点的数据集（输入端口1）。`vtkAppendPolyData`可以将多个PolyData合并为一个，它支持无限多个输入端口。

多端口的连接方式是：

```cpp
vtkNew<vtkProbeFilter> probe;
probe->SetInputConnection(0, sourceData->GetOutputPort());     // 端口0：被采样的数据
probe->SetInputConnection(1, probePoints->GetOutputPort());    // 端口1：采样点位置
```

每个算法通过`GetNumberOfInputPorts()`声明它需要多少个输入端口。这是由算法类本身决定的，你不需要也不能改变。

### 2.2.5 管道连接的完整代码模式

将以上知识综合起来，以下是VTK管道连接的标准代码模式：

```cpp
// 模式1：直接串联管道（最常用）
vtkNew<SourceType> source;
vtkNew<FilterType> filter;
filter->SetInputConnection(source->GetOutputPort());

vtkNew<MapperType> mapper;
mapper->SetInputConnection(filter->GetOutputPort());

// 模式2：多Filter串联
vtkNew<SourceType> source;
vtkNew<FilterA> filterA;
filterA->SetInputConnection(source->GetOutputPort());
vtkNew<FilterB> filterB;
filterB->SetInputConnection(filterA->GetOutputPort());
vtkNew<FilterC> filterC;
filterC->SetInputConnection(filterB->GetOutputPort());
// 管道：source -> filterA -> filterB -> filterC

// 模式3：不经过Filter，Source直接连Mapper
vtkNew<SourceType> source;
vtkNew<MapperType> mapper;
mapper->SetInputConnection(source->GetOutputPort());
// 管道：source -> mapper
```

这三种模式覆盖了绝大多数VTK使用场景。更深层的分支与合并将在2.4节讨论。

---

## 2.3 按需执行机制（Demand-Driven Execution）

### 2.3.1 "不请求，不计算"——VTK的惰性哲学

VTK管道最反直觉但也是最强大的特性是：**当你创建一个Source或Filter时，它什么数据都没有生成。** 你只是在搭建管道的骨架，数据要等到"有人真正需要"的时候才会被计算出来。

来看一个具体的例子：

```cpp
// 创建Source，设置参数
vtkNew<vtkConeSource> cone;
cone->SetResolution(50);
cone->SetHeight(2.0);
cone->SetRadius(0.5);

// 此时问一个看似合理的问题：cone已经生成数据了吗？
// 答案：没有。cone只是记住了参数（resolution=50, height=2.0, radius=0.5），
// 但还没有执行任何实质的几何计算。

// 创建Filter，连接管道
vtkNew<vtkShrinkPolyData> shrink;
shrink->SetInputConnection(cone->GetOutputPort());
shrink->SetShrinkFactor(0.7);

// 此时问：shrink已经执行Shrink操作了吗？
// 答案：也没有。shrink只是记住了它的上游是谁以及shrink factor是多少。

// 数据直到下面这行才真正被计算：
shrink->Update();  // 此时整条管道执行：Cone生成数据 -> Shrink处理数据
```

这个设计的目的是什么？让我们考虑一个实际的交互场景：

1. 你的程序渲染了一个由锥体Source、ShrinkFilter、WarpFilter组成的复杂场景。
2. 用户拖动一个滑块，调整锥体的高度。
3. 你的回调函数调用`cone->SetHeight(newValue)`。
4. 窗口重新渲染（`RenderWindow->Render()`）。
5. VTK检测到Cone的MTime已变化，重新执行整条受影响的上游管道。
6. 用户看到更新后的结果。

在这个过程中，VTK只重新计算了"必须重新计算的部分"。如果用户只是旋转了相机而没有改变任何数据参数，那么管道的可视化部分完全不需要重新执行——渲染管道只需用新的相机参数重新绘制已有的图形基元。

这就是按需执行的威力：**VTK始终只做最少量的必要计算，没有冗余，没有浪费。**

### 2.3.2 vtkExecutive——管道的大脑

虽然你很少直接在代码中使用`vtkExecutive`，但理解它的存在和工作方式对于掌握管道机制很有帮助。

每个`vtkAlgorithm`内部都持有一个`vtkExecutive`对象。Executive的职责是：

1. **接收下游的更新请求**：当某个算法被要求`Update()`时，实际上是它的Executive收到了请求。
2. **向上游传播请求**：Executive沿着`SetInputConnection()`建立的连接链，向上游追溯，请求上游算法先完成计算。
3. **检查缓存有效性**：Executive比较上游算法的MTime与自己缓存的MTime，判断是否需要重新执行。
4. **调度算法执行**：如果需要重新计算，Executive调用算法的`RequestData()`方法（这是由具体算法子类实现的）。

VTK的标准Executive是`vtkDemandDrivenPipeline`（在VTK 9.5.2中，它继承自`vtkStreamingDemandDrivenPipeline`，但管道相关的核心行为仍然来自`vtkDemandDrivenPipeline`）。大多数情况下，你不需要关心Executive的具体类型——`vtkAlgorithm`在内部自动创建了正确的Executive。

有一个例外值得一提：在并行计算（多线程）场景中，一些算法使用不同的Executive来支持分片（Piece）请求——允许每个线程只处理数据的一部分。但这属于高级话题，将在第十二章（并行处理）中详细讨论。

### 2.3.3 MTime——管道的时间标尺

MTime（Modified Time，修改时间）是VTK按需执行机制的核心引擎。它回答了一个简单但关键的问题：**"你上次给我的数据还新鲜吗？"**

#### MTime的基本概念

VTK中每一个对象（都继承自`vtkObject`）都有一个内部的时间戳——MTime。这个时间戳不是真实世界的时钟时间，而是一个单调递增的全局计数器：

- 当对象被创建时，MTime = 0。
- 当对象的关键状态发生变化时（例如调用了某个`SetXXX()`方法），MTime被更新为当前的全局计数器的值。
- 全局计数器每被查询一次就自增1。

这样，两个MTime的比较就能精确地告诉我们：一个对象在另一个对象的某个参考时间点之后有没有被修改过。

#### MTime在管道中的角色

考虑这样一条管道：

```
ConeSource --> ShrinkFilter --> PolyDataMapper
```

当ShrinkFilter被要求输出结果时，它内部的执行逻辑大致如下（伪代码）：

```
ShrinkFilter::RequestData():
    upstreamData = GetInputConnection()->GetProducer()->GetOutput()
    // 内部逻辑：
    // if (cachedUpstreamMTime < upstreamAlgorithm->GetMTime())
    //     需要重新获取上游数据
    //     对上游数据执行shrink操作
    //     更新 cachedUpstreamMTime = upstreamAlgorithm->GetMTime()
    //     缓存shrink结果, 更新selfMTime
    // else:
    //     上游没变，我也没变，返回缓存的结果
```

关键点在于：**每个算法缓存了它上次执行时上游的MTime**。当新的请求到来时，它只需要比较"缓存的MTime"和"上游当前的MTime"——如果上游的MTime更新，说明上游数据变了，需要重新处理；如果两者相同，说明上游数据没变，可以直接使用缓存结果。

这就像是超市里的食品标签：你买牛奶时会看生产日期（MTime），如果日期和上次买的一样，你就知道这瓶和上次的来自同一批次（数据没变）；如果日期更新，说明是新生产的（数据变了）。

#### 哪些操作会触发MTime更新

```cpp
vtkNew<vtkConeSource> cone;
// 此时 cone->GetMTime() == 某个初始值

cone->SetResolution(30);    // MTime 更新！
cone->SetHeight(2.0);       // MTime 更新！
cone->SetRadius(1.5);       // MTime 更新！
cone->SetCenter(0, 1, 0);   // MTime 更新！

// 但是：
cone->GetResolution();      // MTime 不变——这只是查询，不是修改
```

VTK对象中几乎所有的`SetXXX()`方法都会触发`this->Modified()`，而`Modified()`将MTime更新为当前的全局计数器值。这意味着：**只要你修改了管道的任何参数，MTime就会忠实地记录下这一事实。** 下一次下游请求数据时，管道就会自动传播和执行这些变化。

#### 验证MTime的效果——一个小实验

```cpp
vtkNew<vtkConeSource> cone;
cone->SetResolution(30);

vtkNew<vtkShrinkPolyData> shrink;
shrink->SetInputConnection(cone->GetOutputPort());

// 第一次Update：Cone和Shrink都执行计算
shrink->Update();

// 记录此时的MTime
unsigned long mtimeAfterFirstUpdate = cone->GetMTime();

// 不修改任何参数，再次Update
shrink->Update();
// 第二次Update：MTime没有变化，什么都不会重新计算

// 修改Cone参数
cone->SetResolution(60);
// 现在 cone->GetMTime() != mtimeAfterFirstUpdate

// 再次Update：VTK检测到Cone的MTime变了，重新执行整条管道
shrink->Update();
// Shrink拿着新resolution生成的cone数据，重新执行shrink操作
```

这个实验揭示了MTime机制的优雅：你不需要手动通知下游"数据变了"，你甚至不需要知道下游是谁。你只需要修改参数，VTK在幕后自动追踪一切。

### 2.3.4 Update()——触发的扳机

`Update()`是管道按需执行机制的"扳机"。当你调用`Update()`时，以下过程会依次发生：

1. **向上游追溯**：Executive沿着`SetInputConnection()`的连接链向上一级一级追溯，直到找到Source。
2. **检查MTime链**：对管道中的每一个算法，检查其MTime与缓存的上游MTime是否一致。
3. **重新计算（如果需要）**：从Source开始（因为它没有上游），依次执行每个需要重新计算的算法。
4. **返回结果**：调用`Update()`的算法现在拥有了最新的输出数据。

`Update()`的一个重要变体是`Update(int port)`：当一个算法有多个输出端口时，你可以指定只更新某个端口对应的管道分支。

VTK还提供了更精细的更新方法：

```cpp
// 标准Update：更新到当前算法（包括其所有上游）
algorithm->Update();

// 按分片更新（用于并行处理）：
// piece: 当前分片索引, numPieces: 总分片数, ghostLevels: 重叠层级
algorithm->UpdatePiece(piece, numPieces, ghostLevels);

// 按范围更新（用于大规模数据的子集提取）：
// extent: {xmin, xmax, ymin, ymax, zmin, zmax}
algorithm->UpdateExtent(extent);

// 按时间步更新（用于时变数据）：
algorithm->UpdateTimeStep(timeStep);
```

但在本章的范围内，你只需要记住`Update()`这一种形式。其余变体将在处理大规模数据和时变数据的章节中逐步引入。

---

## 2.4 管道分支与合并

管道之所以被称为"管道网络"（Pipeline Network）而非"管道线"（Pipeline Line），是因为它不仅支持线性的串联，还支持分叉和汇合。

### 2.4.1 管道分支（Forking）

**管道分支指的是：一个算法的一个输出端口，连接到多个下游算法的输入端口。**

这是VTK中最常见的复用模式。假设你有一个数据源，你想同时做三件事：显示原始模型、显示线框叠加、显示提取的边线。在VTK中，你可以这样做：

```cpp
// 公共数据源
vtkNew<vtkConeSource> cone;

// 分支A：显示原始着色模型
vtkNew<vtkPolyDataMapper> mapperA;
mapperA->SetInputConnection(cone->GetOutputPort());
vtkNew<vtkActor> actorA;
actorA->SetMapper(mapperA);
actorA->GetProperty()->SetColor(1.0, 0.8, 0.6);  // 肤色

// 分支B：显示线框叠加
vtkNew<vtkExtractEdges> extractEdges;
extractEdges->SetInputConnection(cone->GetOutputPort());
vtkNew<vtkPolyDataMapper> mapperB;
mapperB->SetInputConnection(extractEdges->GetOutputPort());
vtkNew<vtkActor> actorB;
actorB->SetMapper(mapperB);
actorB->GetProperty()->SetColor(0.0, 0.0, 0.0);       // 黑色边线
actorB->GetProperty()->SetLineWidth(2.0);

// 分支C：显示法线方向（使用Glyph）
vtkNew<vtkSphereSource> sphereGlyph;
sphereGlyph->SetRadius(0.03);
vtkNew<vtkGlyph3DMapper> mapperC;
mapperC->SetInputConnection(cone->GetOutputPort());
mapperC->SetSourceConnection(sphereGlyph->GetOutputPort());
// ... actorC ...
```

数据流图示：

```
                    +---> FilterA ---> MapperA ---> ActorA
                    |
ConeSource ---------+---> FilterB ---> MapperB ---> ActorB
                    |
                    +---> FilterC ---> MapperC ---> ActorC
```

所有三个分支共享同一个ConeSource。当ConeSource的参数发生变化时，所有三个分支都会按需自动更新——每个分支在自己的MTime检查中独立判断是否需要重新计算。

这种分支设计的实际价值在于：

- **数据只生成一次**：无论有多少个下游分支，ConeSource的输出只计算一次。
- **每个分支独立执行**：分支之间的执行互不影响，一个分支的更新不会拖累其他分支。
- **灵活的组合**：你可以随时添加或移除分支，而不需要重构已有的管道。

### 2.4.2 管道合并（Merging）

**管道合并指的是：多个算法的输出，汇聚到一个算法的不同输入端口。**

不是所有Filter都支持多输入，但那些做了设计的Filter可以接收来自多个上游的数据：

```cpp
// 两个独立的数据源
vtkNew<vtkSphereSource> sphere;
vtkNew<vtkConeSource> cone;

// 合并为一个数据集
vtkNew<vtkAppendPolyData> append;
append->AddInputConnection(sphere->GetOutputPort());
append->AddInputConnection(cone->GetOutputPort());

// 下游统一处理合并后的数据
vtkNew<vtkPolyDataMapper> mapper;
mapper->SetInputConnection(append->GetOutputPort());
```

数据流图示：

```
SphereSource -----+
                  +---> AppendFilter ---> Mapper ---> Actor
ConeSource -------+
```

`vtkAppendPolyData`是一个特殊的Filter——它使用`AddInputConnection()`而不是`SetInputConnection()`来添加连接。这是因为它的输入端口数量是动态的（可以合并任意数量的数据集），而不是固定的1个或2个。

除`vtkAppendPolyData`外，以下Filter也支持多输入合并：

| Filter | 用途 | 输入说明 |
|--------|------|---------|
| `vtkAppendPolyData` | 合并多个PolyData | 动态数量的输入端口 |
| `vtkAppendFilter` | 合并多个DataSet | 动态数量，支持多种DataSet类型 |
| `vtkProbeFilter` | 采样 | 端口0：源数据；端口1：采样点 |
| `vtkResampleWithDataSet` | 重采样 | 端口0：源数据；端口1：目标网格 |
| `vtkBooleanOperationPolyDataFilter` | 布尔运算 | 端口0和1：两个操作数 |

### 2.4.3 Filter的类型分类

VTK中所有Filter都继承自`vtkAlgorithm`，但中间还有一些细分层次，用于表达Filter对输入数据类型的约束：

| 基类 | 典型子类 | 说明 |
|------|---------|------|
| `vtkPassInputTypeAlgorithm` | `vtkShrinkPolyData` | 输出类型与输入类型相同（输入PolyData则输出PolyData） |
| `vtkPolyDataAlgorithm` | `vtkSmoothPolyDataFilter`、`vtkDecimatePro` | 专门处理PolyData，输出也是PolyData |
| `vtkImageAlgorithm` | `vtkImageGaussianSmooth` | 专门处理ImageData，输入输出均为ImageData |
| `vtkDataSetAlgorithm` | `vtkClipDataSet`、`vtkAppendFilter` | 处理通用DataSet |
| `vtkGraphAlgorithm` | `vtkBoostBreadthFirstSearch` | 专门处理图数据结构 |

这些中间基类存在的意义是简化Filter的实现。如果你要写一个接收PolyData并输出PolyData的Filter，你只需要继承`vtkPolyDataAlgorithm`并重写`RequestData()`方法——输入类型检查和输出类型设置都已经由基类完成了。自定义Filter的开发将在第十九章详细讲解。

---

## 2.5 Update()与渲染触发

### 2.5.1 谁在调用Update()

在VTK的日常使用中，你显式调用`Update()`的场景其实并不多。大多数时候，`Update()`是由渲染系统自动调用的。理解"谁在什么时候调用了Update()"对于掌握程序的执行时机和排查性能问题至关重要。

VTK中Update()的触发可以分为两类：

**隐式Update（由渲染触发）——最常见的情况：**

```cpp
// 你的代码
renderWindow->Render();  // 这一行会触发整条Update()链
```

当`Render()`被调用时，内部执行路径大致是：

```
RenderWindow::Render()
    --> Renderer::DeviceRender()
        --> Actor::Render()
            --> Mapper::Update()          // Mapper需要图形数据
                --> Filter::Update()      // Filter需要输入数据
                    --> Source::Update()  // Source需要生成数据
                        Source执行计算
                    Filter拿到数据，执行处理
                Mapper拿到处理后的数据，转换为图形基元
            Actor将图形基元提交给OpenGL
        Renderer执行相机投影和光照
    RenderWindow将像素呈现到屏幕
```

这个调用链揭示了一个重要事实：**Update()的传播方向是从下游往上游（从渲染端到数据端）**，但**实际的计算方向是从上游往下游（从数据端到渲染端）**。这就像是——你要吃饭（渲染），于是你喊楼上的厨师（Mapper）做饭，厨师又喊农场（Source）送食材。顺序是："要吃饭" -> "要菜" -> "要食材" -> "食材送达" -> "做菜" -> "吃饭"。

**显式Update（由你手动调用）——特殊场景：**

```cpp
// 场景1：你想在渲染之前检查数据
cone->Update();
int numPoints = cone->GetOutput()->GetNumberOfPoints();
std::cout << "Generated " << numPoints << " points." << std::endl;

// 场景2：你想将数据写入文件，而不需要渲染
cone->Update();
vtkNew<vtkSTLWriter> writer;
writer->SetInputConnection(cone->GetOutputPort());
writer->Write();  // Write内部也会调用Update()
```

在场景1中，你手动调用了`cone->Update()`是因为你想在渲染之前检查数据。这样做是可以的，但它破坏了管道的按需特性——Cone现在被强制提前执行了。如果之后渲染时Cone的参数没有变，这部分计算就是浪费的。

更好的做法是：如果需要检查数据但不打断管道，可以在渲染之后获取数据。

### 2.5.2 完整的Update()级联详解

让我们用一个具体的代码示例来追踪完整的Update()级联过程：

```cpp
vtkNew<vtkConeSource> cone;
cone->SetResolution(20);

vtkNew<vtkShrinkPolyData> shrink;
shrink->SetInputConnection(cone->GetOutputPort());
shrink->SetShrinkFactor(0.8);

vtkNew<vtkPolyDataMapper> mapper;
mapper->SetInputConnection(shrink->GetOutputPort());

vtkNew<vtkActor> actor;
actor->SetMapper(mapper);

renderer->AddActor(actor);
renderWindow->Render();  // <-- 从这一行开始，下面的一切自动发生
```

当`renderWindow->Render()`被执行时，以下是内部发生的完整步骤序列：

**步骤1：RenderWindow::Render()**
RenderWindow知道自己含有一个或多个Renderer。它依次调用每个Renderer的渲染方法，准备将最终的像素结果合成到窗口的各区域。

**步骤2：Renderer遍历Actor列表**
Renderer持有场景中的Actor列表。对于每个可见的Actor，Renderer准备执行渲染——这包括设置该Actor的模型-视图-投影矩阵，以及请求Actor生成用于绘制的图形基元。

**步骤3：Actor请求Mapper执行**
Actor本身不包含任何几何信息——它只知道"我的Mapper是谁"。Actor调用`mapper->Render(renderer, actor)`，这内部会触发`mapper->Update()`，向Mapper索要图形基元。

**步骤4：Mapper::Update()向上游追溯**
`vtkPolyDataMapper::Update()`被调用。Mapper的Executive检查："我上次渲染时缓存的上游MTime是多少？现在的上游MTime是多少？"（首次渲染时，缓存的MTime为0，一定小于当前的MTime，因此一定需要计算。）

**步骤5：ShrinkFilter::Update()向上游追溯**
Mapper的上游是ShrinkFilter。ShrinkFilter的Executive同样执行MTime检查。但ShrinkFilter还需要它自己的上游（ConeSource）的数据才能执行处理——因此ShrinkFilter向上游ConeSource请求数据。

**步骤6：ConeSource执行计算**
ConeSource是Source——它没有上游，没有输入数据可以依赖。ConeSource的Executive直接调用`ConeSource::RequestData()`。这个方法是ConeSource类的核心实现，它根据当前参数（Resolution=20, Height=1.0, Radius=0.5, etc.）生成一个圆锥体的PolyData——包括顶点坐标、面索引、法向量等。

**步骤7：数据向下游返回**
ConeSource的计算完成。它的输出数据（一个`vtkPolyData`对象）现在可以被ShrinkFilter获取。

**步骤8：ShrinkFilter执行处理**
ShrinkFilter拿到了ConeSource的输出数据。它的`RequestData()`方法对PolyData中的每个三角形执行收缩变换（将每个三角形的顶点向其中心点移动一定比例），生成收缩后的PolyData。

**步骤9：Mapper执行图形转换**
ShrinkFilter的输出数据到达Mapper。`vtkPolyDataMapper`的`RequestData()`方法将PolyData的点、线、面转换为OpenGL可直接使用的顶点缓冲对象（VBO）、索引缓冲对象（IBO）和着色器程序。

**步骤10：图形基元沿渲染管道下行**
Mapper的图形输出到达Actor。Actor结合自己的空间变换矩阵（Position、Orientation、Scale），将模型空间的图形基元变换到世界空间。

**步骤11：Renderer执行投影和光照**
Renderer根据Camera的投影矩阵将世界空间的图形基元变换到屏幕空间。Renderer还计算光照效果（如果启用了光照的话），这包括漫反射、镜面反射、环境光分量。

**步骤12：RenderWindow执行最终显示**
所有Renderer的输出被合成到RenderWindow的各个区域。RenderWindow将最终的帧缓冲交换到屏幕（双缓冲机制，避免闪烁）。

**步骤1-4是Update()的向上追溯阶段（"我需要什么"），步骤5-12是计算的向下执行阶段（"给你，拿去吧"）。**

### 2.5.3 首次Update vs 后续Update

让我们继续上面的例子，看看第二次渲染时发生了什么：

```cpp
// 首次渲染
renderWindow->Render();  // 整条管道计算一次

// 不修改任何参数，再次渲染
renderWindow->Render();  // 管道没有计算——MTime都没有变化，全部使用缓存
```

第二个`Render()`的调用链仍然会经历步骤1-4的追溯过程，但在每个算法的MTime检查环节，结论都是"上游MTime == 缓存的MTime，不需要重新计算"。因此，步骤5-12中的实际计算被全部跳过，图形基元直接从缓存中获取。

只有在参数被修改后，相关链路上的MTime才会被打破：

```cpp
// 修改Filter参数
shrink->SetShrinkFactor(0.5);  // ShrinkFilter的MTime更新

renderWindow->Render();  // 这次会重新计算

// 执行链：
// Mapper检查MTime: shrink的MTime变了 -> 需要重新获取shrink的输出
// ShrinkFilter检查MTime: cone的MTime没变 -> 不需要重新获取cone的输出
//   -> ShrinkFilter直接用缓存的cone数据，只重新执行自己的shrink操作
//   -> 新的shrink结果返回到Mapper
// Mapper用新的数据重新生成图形基元
```

注意上面的关键洞察：**修改了ShrinkFilter的参数后，ConeSource不需要重新计算。** ShrinkFilter缓存的ConeSource的MTime显示ConeSource的数据没有变化，因此ShrinkFilter直接使用之前缓存的ConeSource输出数据，只在已有的Cone数据基础上重新做shrink操作。这就是MTime机制的精妙——最小化重计算范围。

### 2.5.4 内存管理——中间结果的生命周期

管道中产生的中间数据何时被释放？这是一个既有理论意义又有实际影响的问题。

**VTK使用引用计数（Reference Counting）来管理所有数据对象的生命周期。** 当管道中一个算法的输出不再被任何对象引用时，它会被自动析构。

在标准的管道使用模式中：

```cpp
vtkNew<vtkConeSource> cone;
vtkNew<vtkShrinkPolyData> shrink;
shrink->SetInputConnection(cone->GetOutputPort());

shrink->Update();  // 此时cone的输出数据被shrink持有引用
// cone的输出数据现在不会被释放——shrink还持有对它的引用

// 如果你断开连接：
shrink->RemoveAllInputConnections(0);
// cone的输出数据现在没有引用者了，会在shrink的引用释放后自动析构
```

在实际的应用程序中，你几乎不需要手动管理这些中间数据的生命周期——VTK的引用计数机制会自动处理。但有两点值得留意：

1. **大型数据集的中间结果可能占用大量内存**。如果你有数十GB的数据在管道中流动，你需要考虑是否某些中间结果可以释放。在某些极端情况下，可以手动调用`RemoveAllInputConnections()`来断开连接，释放上游数据。

2. **SetInputData()传递的数据需要你手动管理生命周期**。因为使用`SetInputData()`时断开的是管道连接，你传入的数据对象的生命周期完全由你（或持有它的智能指针）来管理。

---

## 2.6 完整示例：带过滤器的管道

现在，让我们将本章学到的所有概念融入一个完整的工作示例。这个示例将展示：

- ConeSource -> ShrinkFilter -> PolyDataMapper -> Actor -> Renderer 的完整管道
- 参数修改后管道的自动响应
- 显式Update()与隐式Update()的对比效果

### 2.6.1 示例代码

将以下代码保存为`PipelineExample.cxx`：

```cpp
// ============================================================================
// PipelineExample.cxx
// VTK第二章示例：演示可视化管道的完整构建与按需执行机制
// ============================================================================

#include "vtkActor.h"
#include "vtkConeSource.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkShrinkPolyData.h"

#include <iostream>
#include <string>

int main(int argc, char* argv[])
{
  // ======================================================================
  // 第一步：创建渲染基础设施
  // ======================================================================

  vtkNew<vtkRenderer> renderer;
  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->SetMultiSamples(0);         // 演示用关闭多重采样
  renderWindow->AddRenderer(renderer);
  renderWindow->SetSize(600, 300);          // 宽窗口，便于展示两个视图
  renderWindow->SetWindowName("VTK Pipeline Demo - Cone + ShrinkFilter");

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<vtkInteractorStyleTrackballCamera> style;
  style->SetDefaultRenderer(renderer);
  interactor->SetInteractorStyle(style);

  // ======================================================================
  // 第二步：构建可视化管道
  // ======================================================================
  // 管道结构: ConeSource -> ShrinkFilter -> PolyDataMapper -> Actor

  // --- 数据源：生成圆锥体几何数据 ---
  vtkNew<vtkConeSource> cone;
  cone->SetResolution(30);     // 锥体的面数（越大越光滑）
  cone->SetHeight(2.0);        // 锥体高度
  cone->SetRadius(1.0);        // 锥体底面半径

  // 注意：此时Cone还没有生成任何数据。
  // 它只是"记住"了resolution=30, height=2.0, radius=1.0这些参数。

  // --- 过滤器：收缩多边形 ---
  vtkNew<vtkShrinkPolyData> shrink;
  // 【关键】使用SetInputConnection建立管道连接
  // 这告诉shrink："你的输入数据来自cone的输出端口，需要时去问它要"
  shrink->SetInputConnection(cone->GetOutputPort());
  shrink->SetShrinkFactor(0.8);  // 收缩因子：0=完全收缩到中心，1=保持不变

  // --- 映射器：将几何数据转换为图形基元 ---
  vtkNew<vtkPolyDataMapper> mapper;
  // 【关键】同样使用SetInputConnection保持管道连接
  mapper->SetInputConnection(shrink->GetOutputPort());

  // --- 演员：场景中的可视化实体 ---
  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(0.5, 0.8, 1.0);   // 浅蓝色
  actor->GetProperty()->SetEdgeVisibility(1);        // 显示边界线
  actor->GetProperty()->SetEdgeColor(0.0, 0.0, 0.3); // 深蓝灰色边界线

  // ======================================================================
  // 第三步：添加Actor到场景
  // ======================================================================

  renderer->AddActor(actor);

  // ======================================================================
  // 第四步：在渲染前检查管道状态
  // ======================================================================

  std::cout << "========================================" << std::endl;
  std::cout << "  VTK可视化管道示例" << std::endl;
  std::cout << "  管道: ConeSource -> ShrinkFilter -> Mapper -> Actor" << std::endl;
  std::cout << "========================================" << std::endl;
  std::cout << std::endl;
  std::cout << ">>> 首次渲染前（管道尚未执行）:" << std::endl;

  // 在Update()之前尝试获取输出——这会触发一段零输出或者需要先Update
  // 这里我们演示：在首次渲染之前，可以显式调用Update()来窥探数据

  // 显式Update：触发ConeSource和ShrinkFilter都执行
  shrink->Update();

  // 现在可以安全地查询数据
  vtkPolyData* output = shrink->GetOutput();
  std::cout << "  ShrinkFilter输出数据点数量: "
            << output->GetNumberOfPoints() << std::endl;
  std::cout << "  ShrinkFilter输出数据单元数量: "
            << output->GetNumberOfCells() << std::endl;
  std::cout << "  Cone当前Resolution参数: " << cone->GetResolution() << std::endl;
  std::cout << "  Shrink当前ShrinkFactor参数: " << shrink->GetShrinkFactor() << std::endl;
  std::cout << std::endl;

  // ======================================================================
  // 第五步：首次渲染
  // ======================================================================

  renderer->SetBackground(0.2, 0.3, 0.4);
  renderWindow->Render();
  std::cout << ">>> 首次渲染完成（管道已经执行）" << std::endl;

  // ======================================================================
  // 第六步：修改参数并重新渲染
  // ======================================================================

  std::cout << std::endl;
  std::cout << ">>> 修改Cone参数: SetResolution(30 -> 10)" << std::endl;
  cone->SetResolution(10);

  // 注意：我们不需要手动通知shrink或mapper！
  // MTime机制会自动检测到cone的修改。

  std::cout << ">>> 修改Shrink参数: SetShrinkFactor(0.8 -> 0.5)" << std::endl;
  shrink->SetShrinkFactor(0.5);

  std::cout << ">>> 重新渲染..." << std::endl;
  renderWindow->Render();
  std::cout << ">>> 第二次渲染完成（管道自动重新计算了变化的部分）" << std::endl;

  // 验证：渲染后查询数据
  shrink->Update();  // Update()此时是便宜的——如果数据没变则不会重新计算
  output = shrink->GetOutput();
  std::cout << "  ShrinkFilter输出数据点数量（参数修改后）: "
            << output->GetNumberOfPoints() << std::endl;
  std::cout << "  ShrinkFilter输出数据单元数量（参数修改后）: "
            << output->GetNumberOfCells() << std::endl;
  std::cout << std::endl;

  // ======================================================================
  // 第七步：演示第二次渲染（参数不变）
  // ======================================================================

  std::cout << ">>> 不修改任何参数，再次渲染..." << std::endl;
  std::cout << "    （这次渲染时管道不需要重新计算——所有MTime都没变）" << std::endl;
  renderWindow->Render();
  std::cout << ">>> 第三次渲染完成（无计算发生，直接使用缓存）" << std::endl;

  // ======================================================================
  // 第八步：启动交互循环
  // ======================================================================

  std::cout << std::endl;
  std::cout << "========================================" << std::endl;
  std::cout << "  窗口已打开。你可以:" << std::endl;
  std::cout << "  - 鼠标左键拖拽：旋转场景" << std::endl;
  std::cout << "  - 鼠标右键上下拖拽：缩放场景" << std::endl;
  std::cout << "  - 鼠标中键拖拽：平移场景" << std::endl;
  std::cout << "  关闭窗口退出程序。" << std::endl;
  std::cout << "========================================" << std::endl;

  renderWindowInteractor->Start();

  return 0;
}
```

### 2.6.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(PipelineExample)

# 查找VTK 9.5.2
find_package(VTK 9.5.2 REQUIRED
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
)

# 打印VTK信息（可选，帮助调试）
message(STATUS "VTK version: ${VTK_VERSION}")
message(STATUS "VTK include: ${VTK_INCLUDE_DIRS}")

# 创建可执行文件
add_executable(PipelineExample PipelineExample.cxx)

# 链接VTK库
target_link_libraries(PipelineExample PRIVATE ${VTK_LIBRARIES})

# 自动初始化VTK模块工厂
vtk_module_autoinit(
  TARGETS PipelineExample
  MODULES ${VTK_LIBRARIES}
)
```

注意新增的组件依赖：

- **`CommonExecutionModel`**：提供`vtkExecutive`、`vtkDemandDrivenPipeline`等执行管理类。虽然你在代码中没有直接引用它们，但它们被`vtkAlgorithm`内部使用，链接时需要这个模块。
- **`FiltersCore`**：提供`vtkShrinkPolyData`等核心过滤器。这是我们在本章中使用的Filter所属的模块。

### 2.6.3 程序运行输出解读

运行程序后，终端将输出类似以下内容：

```
========================================
  VTK可视化管道示例
  管道: ConeSource -> ShrinkFilter -> Mapper -> Actor
========================================

>>> 首次渲染前（管道尚未执行）:
  ShrinkFilter输出数据点数量: 62
  ShrinkFilter输出数据单元数量: 90
  Cone当前Resolution参数: 30
  Shrink当前ShrinkFactor参数: 0.8

>>> 首次渲染完成（管道已经执行）

>>> 修改Cone参数: SetResolution(30 -> 10)
>>> 修改Shrink参数: SetShrinkFactor(0.8 -> 0.5)
>>> 重新渲染...
>>> 第二次渲染完成（管道自动重新计算了变化的部分）
  ShrinkFilter输出数据点数量（参数修改后）: 22
  ShrinkFilter输出数据单元数量（参数修改后）: 30

>>> 不修改任何参数，再次渲染...
    （这次渲染时管道不需要重新计算——所有MTime都没变）
>>> 第三次渲染完成（无计算发生，直接使用缓存）

========================================
  窗口已打开。你可以:
  - 鼠标左键拖拽：旋转场景
  - 鼠标右键上下拖拽：缩放场景
  - 鼠标中键拖拽：平移场景
  关闭窗口退出程序。
========================================
```

从输出中可以观察到：

1. **数据点数量**：Resolution=30时产生62个点（30个底面顶点 x 2层 + 1个尖端 + 1个底面中心 = 62），Resolution=10时只有22个点。这说明ConeSource确实在重新计算。

2. **收缩效果**：由于ShrinkFilter的作用，显示的锥体看起来比原始ConeSource生成的小——每个三角形的顶点被向中心收缩了20%（ShrinkFactor=0.8时）或50%（ShrinkFactor=0.5时）。

3. **参数不用手动传播**：我们只调用了`cone->SetResolution(10)`和`shrink->SetShrinkFactor(0.5)`——不需要手动通知下游。MTime机制自动完成了变化检测和计算调度。

### 2.6.4 本示例的管道可视化

```
ConeSource                      可视化管道
    |
    |   SetInputConnection()
    v
ShrinkFilter
    |
    |   SetInputConnection()
    v
PolyDataMapper ============ 可视化/渲染分界 ============ 渲染管道
    |
    |   actor->SetMapper(mapper)
    v
Actor
    |
    |   renderer->AddActor(actor)
    v
Renderer
    |
    |   renderWindow->AddRenderer(renderer)
    v
RenderWindow
```

### 2.6.5 实验建议

运行此示例后，建议你自行尝试以下修改，以加深对管道机制的理解：

1. **将`SetInputConnection`改为`SetInputData`**：把shrink到mapper的连接改为`mapper->SetInputData(shrink->GetOutput())`，观察修改cone参数后渲染是否会更新。你应该会发现不会——因为管道被SetInputData打断了。

2. **在修改参数后手动调用`Update()`**：在`cone->SetResolution(10)`之后立即加上`shrink->Update()`，然后在`renderWindow->Render()`之前再查询一次数据点数量。观察中间结果的变化。

3. **删除`SetInputConnection`改用直接SetInputData构造独立数据**：完全抛弃cone，手动创建一个`vtkPolyData`对象，用`SetInputData`传给mapper。观察在不使用管道连接时，程序的行为有何不同。

这些实验将帮助你从"知道概念"转化为"真正理解"。

---

## 2.7 本章小结

本章深入讲解了VTK可视化管道的完整运作机制。让我们回顾一下核心要点：

### 要点速览

1. **管道由六个核心环节组成**：Source（数据生产）、Filter（数据变换）、Mapper（数据到图形的翻译）、Actor（可视化实体）、Renderer（场景舞台）、RenderWindow（屏幕窗口）。这六个环节分为两层：可视化管道（Source -> Filter -> Mapper）处理数据，渲染管道（Actor -> Renderer -> RenderWindow）处理显示。

2. **`SetInputConnection()`是管道的血液**。它建立算法之间的"活"连接，使得数据可以在需要时按需流动。`SetInputData()`则传递数据快照，打断管道连接——前者是默认选择，后者只在特殊场景使用。

3. **VTK采用按需执行（Demand-Driven）模型**。创建Source和Filter时不会执行任何计算；计算只在`Update()`被调用时（通常由`Render()`自动触发）才发生。这种设计避免了不必要的计算，支持高效的交互式参数探索。

4. **MTime（修改时间戳）是管道的时钟**。每次调用`SetXXX()`修改参数时，对象的MTime都会更新。下游算法通过比较缓存的MTime与上游的当前MTime来判断数据是否需要重新计算——这是VTK自动检测变化、自动调度的核心机制。

5. **管道支持分叉（一个Source驱动多个下游）和合并（多个Source汇聚到一个Filter）**。这使得VTK可以构建复杂的可视化网络，同时保持数据复用和按需计算的效率。

6. **`Render()`触发的Update()级联是从下游往上游追溯的**：RenderWindow -> Renderer -> Actor -> Mapper -> Filter -> Source。追溯完成后，计算从Source开始向下游流动。整个过程对应用代码透明——你只需要修改参数并重新渲染。

### 从本章到全书

如果说VTK是一门外语，那么管道就是它的语法规则。本章的内容将在全书中反复出现：

- 第三章到第十章讲各种数据类型和Filter时，你会不断用到本章的`SetInputConnection()`模式。
- 第十一章处理大规模数据时，管道的按需执行和MTime机制是分块加载（Streaming）的基础。
- 第十二章讲多线程并行时，管道的分片更新（UpdatePiece）是核心概念。
- 第十九章讲自定义Filter开发时，正是基于对`vtkAlgorithm`和管道机制的深入理解。

在进入具体的Filter和数据操作之前，下一章（第三章：几何数据基础与PolyData）将带你了解VTK中最常用、最灵活的数据类型——`vtkPolyData`。你将学习如何构造、遍历和修改多边形数据，这将成为你构建一切可视化应用的数据基础。

---

> **本章关键记忆口诀**："连接用Connection，更新靠MTime，渲染触发Update，数据按需流。"
>
> - 管道连接：`SetInputConnection(algorithm->GetOutputPort())`
> - 按需更新：修改参数 -> MTime变化 -> 下游在`Update()`时检测 -> 自动重算
> - 渲染触发：`Render()` -> 向上游追溯 -> 逐个检查MTime -> 从Source开始执行
> - 数据流动：从Source到Mapper是数据加工，从Actor到窗口是图形渲染
