# 第十五章 实战篇综合实战：CFD后处理可视化工具

## 本章导读

你在第二卷（实战篇）中走过了很长的一段路。第七章带你深入了PolyData的内部结构——点、线、多边形、三角形条带，以及如何手动构建它们。第八章展开了非结构化网格（UnstructuredGrid）的复杂性——任意单元类型、混合单元、显式拓扑。第九章系统讲解了结构化数据家族——ImageData、RectilinearGrid、StructuredGrid，以及它们的差分和适用场景。第十章赋予你读写世界的能力——从传统.vtk格式到现代.vtu/.vts、从PVD时间序列到FLUENT和OpenFOAM。第十一章是过滤器盛宴——几何变换、数据提取、矢量可视化、网格处理，四大类过滤器的系统讲解。第十二章深入地讲解了颜色映射的机理——LookupTable、标量范围、色彩空间、颜色条（Scalar Bar）、对数映射。第十四章打开了多数据集的大门——合并与复合数据集、多块渲染。

现在你面临的问题不再是"这个过滤器怎么用？"，而是——**如何将所有这些组件组合成一个真实、有用的科学可视化工具？**

这就是本章的使命。我们将构建一个**"CFD后处理可视化工具"（CFD Post-Processing Visualization Tool）**——一个完整的、工程化的桌面应用程序，用于分析和可视化计算流体力学（CFD）的仿真结果。

本章是第二卷的收官之作，同时也是你从"学习者"到"建造者"的转折点。与第一卷的收官项目（3D几何画廊）不同，这个项目不再只是"展示VTK能做什么"——它是一个有实际使用价值的科学可视化工具，其架构和代码组织方式可以直接作为生产级VTK项目的模板。

核心目标：

1. **程序化生成CFD仿真数据**——在没有任何外部数据文件的情况下，用一个数学函数生成符合物理规律的流场数据（速度场 + 压力场），模拟圆柱绕流的CFD结果。
2. **构建分支式管道架构**——同一个数据源驱动四条独立的可视化管道（等值线、流线、变形曲面、箭头Glyph），每一条管道解决一种不同的可视化需求。
3. **实现科学可视化的标准布局**——四视口布局、每个视口独立配色方案、各自带标量条（Scalar Bar）、统一的标注和标题。
4. **自定义交互器**——在标准TrackballCamera交互之外添加键盘快捷键，实现视口切换、截图等高级交互。
5. **完整的CMake构建系统**——展示如何为多文件VTK项目编写规范的CMakeLists.txt。

读完本章后，你将拥有一份可以直接编译运行的完整代码，以及一份理解这份代码所需的全套讲解。更重要的是，你将建立起编写VTK科学可视化应用程序的系统性思维——从数据生成到最终渲染的完整链条。

---

## 15.1 项目概述

### 15.1.1 场景描述

我们的场景是经典的流体力学基准问题：**二维圆柱绕流**（Flow Around a Cylinder）。这是一个广泛用于验证CFD求解器的标准测试案例。流体从左侧入口流入，绕过位于流道中央的圆柱体，从右侧出口流出。

圆柱绕流富含流体力学教科书中的经典现象：

- **驻点（Stagnation Point）**：圆柱正前方和正后方，流速为零，压力达到最大值。
- **边界层分离（Boundary Layer Separation）**：流体在圆柱表面某处脱离，形成尾迹区。
- **尾迹（Wake）**：圆柱后方产生的低速、低压区域。
- **伯努利效应（Bernoulli Effect）**：圆柱表面流速最大的位置压力最低（圆柱顶部和底部）。

一个功能完备的CFD后处理工具应该能够同时回答工程师关心的多种问题：

| 问题 | 可视化手段 |
|------|-----------|
| 压力如何分布？哪里压力最大/最小？ | 填充等值线（Filled Contours） + 颜色条 |
| 流体如何流动？速度场的方向和大小？ | 流线（Streamlines）叠加在速度幅值着色的表面上 |
| 压力场的三维形态？负压区有多深？ | 变形曲面（Warped Surface）以压力为高度位移 |
| 速度方向在空间中的变化？ | 箭头Glyph（Arrow Glyphs）表达速度矢量场 |

我们的工具将在一个窗口中用四个视口同时回答这四个问题。

### 15.1.2 为什么程序化生成数据？

在真实场景中，CFD数据来自求解器（如OpenFOAM、ANSYS Fluent、SU2）的输出文件。但为了保持本章的独立性和可复现性，我们选择用数学函数生成数据。这带来了几个关键优势：

1. **零依赖**：读者不需要安装任何CFD求解器或拥有仿真数据文件即可运行本章代码。
2. **可复现**：数学函数是确定性的，每次运行产生完全相同的数据，便于学习和调试。
3. **教学透明**：我们可以逐步解释数据生成过程，让读者理解"数据长什么样"而不需要先理解CFD理论。
4. **无限扩展**：读者可以修改函数参数（圆柱半径、来流速度、网格密度）并立即观察可视化效果的变化。

当然，我们生成的数据是**近似的、理想化的**——它结合了位势流理论（势流绕圆柱）的解和一些启发式修正来模拟粘性效应（尾迹）。它不是Navier-Stokes方程的数值解，但它在视觉上和工程感上是可信的，并且对于学习VTK可视化来说足够了。

### 15.1.3 项目最终效果

程序运行后你将看到一个约1400x1000像素的窗口，被划分为2行x2列的网格：

```
+---------------------------------------+---------------------------------------+
|  [Top-Left] "压力等值线"               |  [Top-Right] "速度幅值 + 流线"          |
|  Pressure Contour                     |  Velocity Magnitude + Streamlines      |
|                                       |                                       |
|  - 彩色填充的压力等值线                |  - 以速度幅值着色的网格平面              |
|  - 暖色(高压) -> 冷色(低压)            |  - 黑色流线表示流体轨迹                  |
|  - 右侧竖直颜色条                      |  - 右侧竖直颜色条                        |
|                                       |                                       |
+---------------------------------------+---------------------------------------+
|  [Bottom-Left] "压力变形曲面"          |  [Bottom-Right] "速度方向Glyph"         |
|  Pressure Warped Surface              |  Velocity Arrow Glyphs                 |
|                                       |                                       |
|  - 以压力为Z轴位移的曲面               |  - 以速度幅值为大小的箭头                |
|  - 低压区下沉，高压区隆起              |  - 箭头方向=流动方向                     |
|  - 暖色(高压) -> 冷色(低压)            |  - 箭头颜色=速度幅值                     |
|  - 右侧竖直颜色条                      |  - 右侧竖直颜色条                        |
+---------------------------------------+---------------------------------------+
```

- **视口0（左上）**：压力填充等值线——使用`vtkContourFilter`生成15条等值线并用`vtkBandedPolyDataContourFilter`做色带填充（或直接使用`vtkLookupTable`做面着色），展示从入口高压区到圆柱后方低压尾迹的完整压力过渡。
- **视口1（右上）**：速度幅值面着色 + 流线——使用`vtkStreamTracer`从入口平面发射种子点，追踪穿过圆柱周围的流线，叠加在以速度幅值着色的结构化网格平面上。
- **视口2（左下）**：压力变形曲面——使用`vtkWarpScalar`将压力值映射为Z方向的位移，创建一个三维地形效果，让低压尾迹的"山谷"变得直观可见。
- **视口3（右下）**：速度方向Glyph——在网格的均匀采样点上放置箭头，箭头方向与速度矢量对齐，箭头大小和颜色均按速度幅值缩放。

所有四个视口使用TrackballCamera交互风格，并共享一个自定义交互器，支持键盘快捷键在视口之间切换焦点。

---

## 15.2 数据生成：模拟CFD圆柱绕流结果

在编写可视化代码之前，我们需要先生成数据。数据生成是本章最关键的基础工作——它决定了一切后续可视化的信息量。

### 15.2.1 物理域和网格定义

我们的物理域是一个二维流道：

- **域尺寸**：X方向（流向）从-2.0到8.0（总长10.0），Y方向（垂直方向）从-2.0到2.0（总高4.0）。
- **圆柱体**：圆心在原点(0, 0)，半径R=0.5。
- **网格**：使用`vtkStructuredGrid`，分辨率为Nx x Ny = 120 x 60，在X方向上均匀分布，在Y方向上靠近圆柱区域加密。

这里有一个重要细节：在结构化网格中，圆柱内部的点如何处理？对于实际CFD来说，圆柱内部不在计算域内。对于我们的可视化来说，我们在圆柱内设置"掩码"值（mask），让数据查找表将这些单元渲染为特定颜色或设为透明。更简单的做法是直接将圆柱内部流速为零、压力为参考值——这在可视化中仍然有效，因为用户看到的是一个"实心"圆柱区域。

实际上，对于我们的可视化目的，我们可以采用更优雅的做法：使用`vtkClipDataSet`或`vtkThreshold`过滤掉圆柱内部的单元。但在本章中，为了保持管道清晰，我们将直接在数据生成时赋予圆柱内部合理的"空白"值（速度=0，压力=0），然后通过对压力场使用合适的颜色映射范围（让圆柱区域显示为中性色）来处理。

### 15.2.2 压力场生成

压力场的构造基于**伯努利原理**的简化应用：

$$P + \frac{1}{2}\rho v^2 = \text{const}$$

对于不可压缩、无旋流动，绕圆柱体的势流解给出速度场为：

$$v_r = U_\infty \left(1 - \frac{R^2}{r^2}\right)\cos\theta$$
$$v_\theta = -U_\infty \left(1 + \frac{R^2}{r^2}\right)\sin\theta$$

其中$U_\infty$是来流速度，$R$是圆柱半径，$r$是离圆柱中心的距离，$\theta$是方位角。

速度幅值平方为：

$$v^2 = U_\infty^2\left[1 - 2\frac{R^2}{r^2}\cos(2\theta) + \frac{R^4}{r^4}\right]$$

应用伯努利方程，压力系数$C_p = (P - P_\infty)/(\frac{1}{2}\rho U_\infty^2)$为：

$$C_p = 1 - 4\sin^2\theta \quad \text{(在圆柱表面 r=R)}$$

对于整个流场，$C_p = 1 - \frac{v^2}{U_\infty^2}$。

为了模拟粘性效应（尾迹），我们在圆柱后方（x > 0）额外叠加一个低压区域：

$$P_{\text{wake}} = -\Delta P_{\text{wake}} \cdot \exp\left(-\frac{(x - x_c)^2}{2\sigma_x^2} - \frac{y^2}{2\sigma_y^2}\right)$$

其中$(x_c, \sigma_x, \sigma_y)$控制尾迹的中心位置和扩散范围。

此外，我们还需要在入口处叠加一个线性的压力梯度，以驱动流动：

$$P_{\text{gradient}} = P_{\text{inlet}} \cdot \frac{x_{\text{max}} - x}{x_{\text{max}} - x_{\text{min}}}$$

最终压力场 = 伯努利压力场 + 尾迹低压 + 线性梯度。

### 15.2.3 速度场生成

速度场使用势流解作为基础，加上尾迹修正：

- **基础速度**：遵循势流绕圆柱体的解析解。
- **尾迹减速**：在圆柱后方叠加一个速度衰减因子，模拟分离泡中的低速区。
- **圆柱内部**：速度设为(0, 0)。
- **无滑移条件模拟**：在圆柱表面附近（r < R + boundary_layer_thickness），速度线性降低。

### 15.2.4 生成的标量和矢量数据

最终生成的`vtkStructuredGrid`将携带以下数据数组：

| 数组名称 | 类型 | 存储位置 | 含义 |
|---------|------|---------|------|
| `pressure` | vtkFloatArray | Point Data | 压力标量值 |
| `velocity` | vtkFloatArray (3分量) | Point Data | 速度矢量 (vx, vy, 0) |
| `velocity_magnitude` | vtkFloatArray | Point Data | 速度幅值 |

其中`velocity_magnitude`是一个派生量——可以由`velocity`矢量直接计算得到。但是预计算它可以让下游的着色和Glyph缩放更加方便。

---

## 15.3 项目架构

### 15.3.1 管道全景图

CFD后处理工具的数据流遵循一个中心辐射式（Hub-and-Spoke）拓扑：数据生成器是唯一的Hub，四个可视化分支从它向外辐射：

```
                vtkStructuredGrid (CFD Data Source)
                |
                | 包含: pressure (标量), velocity (矢量), velocity_magnitude (标量)
                |
    +-----------+-----------+-----------+-----------+
    |           |           |           |           |
    v           v           v           v           v
[分支1]    [分支2]    [分支3]    [分支4]
Pressure   Velocity   Pressure   Velocity
Contour    Streamln   Warp       Glyphs
    |           |           |           |
    |           |           |           |
[等值线]    [流线]    [变形曲面]  [箭头]
    |           |           |           |
    v           v           v           v
  Mapper      Mapper     Mapper      Mapper
    |           |           |           |
    v           v           v           v
  Actor       Actor       Actor       Actor
    |           |           |           |
    +-----------+-----------+-----------+
                |
    +-----------+-----------+-----------+
    |           |           |           |
    v           v           v           v
  View 0     View 1     View 2     View 3
(Top-Left) (Top-Right)(Bot-Left) (Bot-Right)
    |           |           |           |
    +-----------+-----------+-----------+
                |
                v
        RenderWindow + Interactor
```

注意要点：

1. **管道分支发生在数据源头**——每条分支都使用`SetInputConnection(dataSource->GetOutputPort())`，而不是`SetInputData(dataSource->GetOutput())`。这确保了VTK的惰性求值机制：数据只会被计算一次（当第一条分支触发Update时），后续分支直接复用结果。

2. **每条分支独立于其他分支**——修改`branch1`的过滤器参数不会影响`branch2`，因为它们操作的是同一个输出共享的数据集。这是VTK管道架构的精髓：数据复用 + 处理独立。

3. **颜色映射在Mapper层面**——每条分支有自己独立的`vtkLookupTable`，映射到不同的标量数组（pressure或velocity_magnitude），并且使用不同的颜色方案。

### 15.3.2 四个视口的设计决策

#### 视口0（左上）：压力等值线

- **过滤器**：`vtkContourFilter`（生成等值线）
- **着色**：使用`vtkLookupTable`的Jet颜色方案（蓝=低压，红=高压），将`pressure`标量映射为面颜色
- **颜色条**：竖直方向，标题"Pressure [Pa]"，带数值刻度
- **关键参数**：等值线数量设为15条，等值线值在pressure数据范围内均匀分布
- **技术说明**：这里不使用`vtkBandedPolyDataContourFilter`（那更适合离散的色带外观）。我们直接让`vtkContourFilter`生成等值线对象，每条等值线是一个PolyData，通过`vtkPolyDataMapper`分别着色。更简单的做法是直接在结构化网格的平面上使用`vtkDataSetMapper`以`pressure`为激活标量做平滑着色——这正是我们要做的。

实际上，为了实现"填充等值线"的效果，我们直接对整个网格平面着色（使用压力标量），然后用`vtkContourFilter`在上面叠加黑色的等值线。这样既能看到连续的色彩渐变，又能看到离散的等值线标注。

#### 视口1（右上）：速度幅值 + 流线

- **底层**：结构化网格平面，以`velocity_magnitude`标量着色，使用暖色方案（红=高速，黄=中速，白=低速）
- **上层**：`vtkStreamTracer`从入口平面的多个种子点发射流线
- **种子点**：在x=-1.85处（确保在域内且略远离入口边界），沿Y方向均匀分布11个种子点
- **流线风格**：黑色Tube（使用`vtkTubeFilter`包裹流线使其有厚度），增强与底层彩色面的对比度
- **颜色条**：属于底层平面，标题"Velocity [m/s]"

#### 视口2（左下）：压力变形曲面

- **核心过滤器**：`vtkWarpScalar`，输入结构化网格，位移标量为`pressure`
- **位移缩放因子**：设为0.5左右，使变形效果显著但不过度
- **着色**：使用压力标量着色（与视口0相同的颜色方案），让颜色和高度共同表达压力场
- **相机角度**：设置为倾斜视角（从斜上方观察），使Z方向的位移可见
- **技术说明**：`vtkWarpScalar`修改的是点的坐标（而非标量值），因此变形后的几何体是真正的3D曲面。对于结构化网格，每个网格点被沿其法向量（Z方向）移动`scaleFactor * pressure`的距离。

#### 视口3（右下）：速度方向Glyph

- **数据准备**：首先使用`vtkMaskPoints`降低采样密度（如果不降采样，120x60=7200个箭头会完全遮挡画面）
- **核心过滤器**：`vtkGlyph3D`，以箭头（`vtkArrowSource`）为Glyph形状
- **方向映射**：Glyph方向由`velocity`矢量场决定（OrientationArray = "velocity", ScaleFactor = 0.15）
- **大小映射**：箭头大小也按速度幅值缩放（`SetScaleModeToScaleByVector`）
- **着色**：使用`velocity_magnitude`标量着色，与视口1相同的暖色方案
- **降采样参数**：每隔4个网格点采样一个箭头（在120x60网格上产生约30x15=450个箭头）

### 15.3.3 视口布局计算

在归一化视口坐标（0到1）中，四个视口的布局如下：

```cpp
// 行归一化高度: 上半部分 = [0.55, 1.0], 下半部分 = [0.0, 0.45]
// 列归一化宽度: 左半部分 = [0.00, 0.49], 右半部分 = [0.51, 1.00]

// 视口0（左上）
double topViewport[4]    = {0.00, 0.55, 0.49, 1.00};

// 视口1（右上）
double topRightVp[4]     = {0.51, 0.55, 1.00, 1.00};

// 视口2（左下）
double botLeftVp[4]      = {0.00, 0.00, 0.49, 0.45};

// 视口3（右下）
double botRightVp[4]     = {0.51, 0.00, 1.00, 0.45};
```

在视口之间留有间隙（0.02的归一化宽度/高度），防止渲染内容相互重叠。

---

## 15.4 完整代码

### 15.4.1 主程序

```cpp
// ============================================================================
// CFDPostProcessor.cxx
// ============================================================================
// VTK 9.5.2 CFD后处理可视化工具
// 第二卷（实战篇）收官项目 -- 第十五章
//
// 功能：
//   - 程序化生成圆柱绕流CFD仿真数据（压力+速度场）
//   - 四条并行可视化管道：
//       1. 压力填充等值线（Filled Pressure Contours）
//       2. 速度幅值着色 + 流线（Velocity Magnitude + Streamlines）
//       3. 压力变形曲面（Pressure Warped Surface, 3D）
//       4. 速度矢量箭头Glyph（Velocity Arrow Glyphs）
//   - 每视口独立颜色条（Scalar Bar）
//   - 自定义交互器（键盘快捷键）
//   - 完整的标题标注
// ============================================================================

#include <vtkActor.h>
#include <vtkArrowSource.h>
#include <vtkCamera.h>
#include <vtkColorTransferFunction.h>
#include <vtkContourFilter.h>
#include <vtkDataSetMapper.h>
#include <vtkFloatArray.h>
#include <vtkGlyph3D.h>
#include <vtkInteractorStyleTrackballCamera.h>
#include <vtkLookupTable.h>
#include <vtkMaskPoints.h>
#include <vtkNamedColors.h>
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
#include <vtkStructuredGrid.h>
#include <vtkStreamTracer.h>
#include <vtkStructuredGridGeometryFilter.h>
#include <vtkTextActor.h>
#include <vtkTextProperty.h>
#include <vtkTubeFilter.h>
#include <vtkWarpScalar.h>

#include <cmath>
#include <iostream>
#include <vector>

// ---------------------------------------------------------------------------
// 常量定义
// ---------------------------------------------------------------------------
constexpr int    GRID_NX          = 120;   // X方向网格点数
constexpr int    GRID_NY          = 60;    // Y方向网格点数
constexpr double DOMAIN_XMIN      = -2.0;  // 流道域X最小值
constexpr double DOMAIN_XMAX      = 8.0;   // 流道域X最大值
constexpr double DOMAIN_YMIN      = -2.0;  // 流道域Y最小值
constexpr double DOMAIN_YMAX      = 2.0;   // 流道域Y最大值
constexpr double CYLINDER_RADIUS  = 0.5;   // 圆柱半径
constexpr double CYLINDER_CX      = 0.0;   // 圆柱中心X
constexpr double CYLINDER_CY      = 0.0;   // 圆柱中心Y
constexpr double U_INF            = 1.0;   // 来流速度
constexpr double P_INLET          = 100.0; // 入口参考压力
constexpr double P_REF            = 0.0;   // 参考压力（伯努利常数基值）

// 视口边界（归一化坐标）
constexpr double VP_MARGIN_X      = 0.02;  // 视口间水平间隙
constexpr double VP_MARGIN_Y      = 0.02;  // 视口间垂直间隙
constexpr double VP_TOP_Y         = 0.55;  // 上排视口下边界
constexpr double VP_BTM_Y         = 0.45;  // 下排视口上边界
constexpr double VP_LEFT_X        = 0.49;  // 左排视口右边界
constexpr double VP_RIGHT_X       = 0.51;  // 右排视口左边界

// ---------------------------------------------------------------------------
// 辅助函数：生成CFD数据
// ---------------------------------------------------------------------------
vtkSmartPointer<vtkStructuredGrid> GenerateCFDData()
{
    vtkNew<vtkStructuredGrid> grid;
    vtkNew<vtkPoints>         points;
    vtkNew<vtkFloatArray>     pressureArray;
    vtkNew<vtkFloatArray>     velocityArray;
    vtkNew<vtkFloatArray>     velocityMagArray;

    // 数组配置
    pressureArray->SetName("pressure");
    pressureArray->SetNumberOfComponents(1);

    velocityArray->SetName("velocity");
    velocityArray->SetNumberOfComponents(3);

    velocityMagArray->SetName("velocity_magnitude");
    velocityMagArray->SetNumberOfComponents(1);

    // 网格间距
    double dx = (DOMAIN_XMAX - DOMAIN_XMIN) / (GRID_NX - 1);
    double dy = (DOMAIN_YMAX - DOMAIN_YMIN) / (GRID_NY - 1);

    // 尾迹参数
    const double wake_sigma_x   = 1.5;
    const double wake_sigma_y   = 0.8;
    const double wake_center_x  = 1.2;   // 尾迹低压中心X
    const double wake_strength  = 60.0;  // 尾迹低压强度
    const double bl_thickness   = 0.08;  // 边界层厚度（速度衰减）

    // 逐点计算
    for (int j = 0; j < GRID_NY; ++j)
    {
        double y = DOMAIN_YMIN + j * dy;

        for (int i = 0; i < GRID_NX; ++i)
        {
            double x = DOMAIN_XMIN + i * dx;

            // 插入点
            points->InsertNextPoint(x, y, 0.0);

            // 相对于圆柱中心的极坐标
            double dx_cyl = x - CYLINDER_CX;
            double dy_cyl = y - CYLINDER_CY;
            double r      = std::sqrt(dx_cyl * dx_cyl + dy_cyl * dy_cyl);
            double theta  = std::atan2(dy_cyl, dx_cyl);

            // ---- 速度场 ----
            double vx = 0.0;
            double vy = 0.0;

            if (r <= CYLINDER_RADIUS)
            {
                // 圆柱内部：速度为零
                vx = 0.0;
                vy = 0.0;
            }
            else
            {
                double r2 = r * r;
                double R2 = CYLINDER_RADIUS * CYLINDER_RADIUS;

                // 势流解
                double vr     =  U_INF * (1.0 - R2 / r2) * std::cos(theta);
                double vtheta = -U_INF * (1.0 + R2 / r2) * std::sin(theta);

                vx = vr * std::cos(theta) - vtheta * std::sin(theta);
                vy = vr * std::sin(theta) + vtheta * std::cos(theta);

                // 边界层修正：靠近圆柱表面线性降速（模拟无滑移）
                if (r < CYLINDER_RADIUS + bl_thickness)
                {
                    double factor = (r - CYLINDER_RADIUS) / bl_thickness;
                    vx *= factor;
                    vy *= factor;
                }

                // 尾迹减速
                if (x > wake_center_x - 2.0 * wake_sigma_x)
                {
                    double wx = (x - wake_center_x) / wake_sigma_x;
                    double wy = y / wake_sigma_y;
                    double wake_factor = 1.0 - 0.7 * std::exp(-0.5 * (wx * wx + wy * wy));
                    vx *= wake_factor;
                    vy *= wake_factor;
                }
            }

            velocityArray->InsertNextTuple3(vx, vy, 0.0);

            // ---- 速度幅值 ----
            double vmag = std::sqrt(vx * vx + vy * vy);
            velocityMagArray->InsertNextTuple1(vmag);

            // ---- 压力场（基于伯努利原理）----
            double pressure = 0.0;

            if (r <= CYLINDER_RADIUS)
            {
                // 圆柱内部：设为接近入口压力的值（在可视化中呈现为"实心"）
                pressure = P_INLET * 0.8;
            }
            else
            {
                double r2 = r * r;
                double R2 = CYLINDER_RADIUS * CYLINDER_RADIUS;

                // 伯努利：P + 1/2 * rho * v^2 = const。简化，取 rho = 1
                double v2 = (vx * vx + vy * vy);
                double Cp = 1.0 - v2 / (U_INF * U_INF);

                // Cp * q_inf，其中 q_inf = 0.5 * rho * U_inf^2 = 0.5
                double P_bernoulli = Cp * 0.5 * U_INF * U_INF;

                // 线性压力梯度（入口到出口）
                double P_gradient = P_INLET * (DOMAIN_XMAX - x) / (DOMAIN_XMAX - DOMAIN_XMIN);

                // 尾迹低压叠加
                double wx = (x - wake_center_x) / wake_sigma_x;
                double wy = y / wake_sigma_y;
                double P_wake = -wake_strength * std::exp(-0.5 * (wx * wx + wy * wy));

                pressure = P_bernoulli + P_gradient + P_wake;
            }

            pressureArray->InsertNextTuple1(pressure);
        }
    }

    // 配置网格
    grid->SetDimensions(GRID_NX, GRID_NY, 1);
    grid->SetPoints(points);
    grid->GetPointData()->AddArray(pressureArray);
    grid->GetPointData()->AddArray(velocityArray);
    grid->GetPointData()->AddArray(velocityMagArray);

    // 设置激活标量
    grid->GetPointData()->SetActiveScalars("pressure");
    grid->GetPointData()->SetActiveVectors("velocity");

    std::cout << "[DATA] Generated structured grid: "
              << GRID_NX << " x " << GRID_NY
              << " = " << (GRID_NX * GRID_NY) << " points" << std::endl;

    // 打印数据范围供参考
    double pRange[2], vRange[2];
    pressureArray->GetRange(pRange);
    velocityMagArray->GetRange(vRange);
    std::cout << "[DATA] Pressure range: ["
              << pRange[0] << ", " << pRange[1] << "]" << std::endl;
    std::cout << "[DATA] Velocity magnitude range: ["
              << vRange[0] << ", " << vRange[1] << "]" << std::endl;

    return grid;
}

// ---------------------------------------------------------------------------
// 辅助函数：创建颜色查找表 (LookupTable)
// ---------------------------------------------------------------------------
vtkSmartPointer<vtkLookupTable> CreateLookupTable(
    double minVal, double maxVal, const std::string& scheme)
{
    vtkNew<vtkLookupTable> lut;
    lut->SetTableRange(minVal, maxVal);
    lut->SetNumberOfTableValues(256);

    if (scheme == "jet")
    {
        // Jet色彩方案：蓝 -> 青 -> 绿 -> 黄 -> 红
        // 适合压力场：冷色低压，暖色高压
        const int nColors = 5;
        double r[nColors] = {0.0,  0.0,  0.0,  1.0, 1.0};
        double g[nColors] = {0.0,  0.0,  1.0,  1.0, 0.0};
        double b[nColors] = {0.5625, 1.0, 1.0, 0.0, 0.0};
        double hueRange = (maxVal - minVal);

        for (int i = 0; i < 256; ++i)
        {
            double t = static_cast<double>(i) / 255.0;
            int seg = static_cast<int>(t * (nColors - 1));
            if (seg >= nColors - 1) seg = nColors - 2;
            double local = t * (nColors - 1) - seg;

            double rr = r[seg] + local * (r[seg + 1] - r[seg]);
            double gg = g[seg] + local * (g[seg + 1] - g[seg]);
            double bb = b[seg] + local * (b[seg + 1] - b[seg]);
            lut->SetTableValue(i, rr, gg, bb, 1.0);
        }
    }
    else if (scheme == "warm")
    {
        // 暖色方案：白 -> 黄 -> 橙 -> 红
        // 适合速度幅值：低速白色，中速黄色/橙色，高速红色
        const int nColors = 4;
        double r[nColors] = {1.0, 1.0,  1.0,  0.8};
        double g[nColors] = {1.0, 1.0,  0.5,  0.0};
        double b[nColors] = {1.0, 0.0,  0.0,  0.0};

        for (int i = 0; i < 256; ++i)
        {
            double t = static_cast<double>(i) / 255.0;
            int seg = static_cast<int>(t * (nColors - 1));
            if (seg >= nColors - 1) seg = nColors - 2;
            double local = t * (nColors - 1) - seg;

            double rr = r[seg] + local * (r[seg + 1] - r[seg]);
            double gg = g[seg] + local * (g[seg + 1] - g[seg]);
            double bb = b[seg] + local * (b[seg + 1] - b[seg]);
            lut->SetTableValue(i, rr, gg, bb, 1.0);
        }
    }
    else
    {
        // 默认：灰度
        for (int i = 0; i < 256; ++i)
        {
            double v = static_cast<double>(i) / 255.0;
            lut->SetTableValue(i, v, v, v, 1.0);
        }
    }

    lut->Build();
    return lut;
}

// ---------------------------------------------------------------------------
// 辅助函数：创建标量条（Scalar Bar）
// ---------------------------------------------------------------------------
vtkSmartPointer<vtkScalarBarActor> CreateScalarBar(
    vtkLookupTable* lut, const std::string& title, int numLabels)
{
    vtkNew<vtkScalarBarActor> scalarBar;
    scalarBar->SetLookupTable(lut);
    scalarBar->SetTitle(title.c_str());

    // 竖直标量条
    scalarBar->SetOrientationToVertical();

    // 位置和尺寸（在视口归一化坐标内）
    scalarBar->SetWidth(0.12);
    scalarBar->SetHeight(0.65);
    scalarBar->SetPosition(0.85, 0.15);

    // 标签数量
    scalarBar->SetNumberOfLabels(numLabels);

    // 外观
    scalarBar->GetTitleTextProperty()->SetFontSize(3);
    scalarBar->GetTitleTextProperty()->SetColor(0.0, 0.0, 0.0);  // 黑色标题
    scalarBar->GetLabelTextProperty()->SetFontSize(3);
    scalarBar->GetLabelTextProperty()->SetColor(0.0, 0.0, 0.0);  // 黑色标签
    scalarBar->SetLabelFormat("%.1f");

    // 边框和背景
    scalarBar->SetDrawBackground(1);
    scalarBar->SetBackgroundColor(1.0, 1.0, 1.0);  // 白色背景
    scalarBar->SetBackgroundOpacity(0.7);

    // 边框
    scalarBar->DrawFrameOn();
    scalarBar->GetFrameProperty()->SetColor(0.0, 0.0, 0.0);

    return scalarBar;
}

// ---------------------------------------------------------------------------
// 辅助函数：创建文本标注
// ---------------------------------------------------------------------------
vtkSmartPointer<vtkTextActor> CreateLabel(
    const std::string& text, int fontSize, double position[2])
{
    vtkNew<vtkTextActor> label;
    label->SetInput(text.c_str());
    label->GetTextProperty()->SetFontSize(fontSize);
    label->GetTextProperty()->SetColor(0.0, 0.0, 0.0);
    label->GetTextProperty()->SetBold(1);
    label->GetTextProperty()->SetFontFamilyToArial();
    label->SetDisplayPosition(static_cast<int>(position[0]),
                              static_cast<int>(position[1]));
    return label;
}

// ---------------------------------------------------------------------------
// 自定义交互器风格
// ---------------------------------------------------------------------------
class CFDCustomInteractorStyle : public vtkInteractorStyleTrackballCamera
{
public:
    static CFDCustomInteractorStyle* New();
    vtkTypeMacro(CFDCustomInteractorStyle, vtkInteractorStyleTrackballCamera);

    CFDCustomInteractorStyle()
        : m_activeRenderer(0)
        , m_renderers()
    {
        m_renderers.fill(nullptr);
    }

    void SetRenderer(int idx, vtkRenderer* renderer)
    {
        if (idx >= 0 && idx < 4)
        {
            m_renderers[idx] = renderer;
        }
    }

    virtual void OnChar() override
    {
        vtkRenderWindowInteractor* rwi = this->Interactor;
        std::string key = rwi->GetKeySym();

        if (key == "1")
        {
            // 切换到视口0（左上）
            m_activeRenderer = 0;
            std::cout << "[INTERACTOR] Focus on viewport 0: Pressure Contour" << std::endl;
            if (m_renderers[0])
            {
                this->SetDefaultRenderer(m_renderers[0]);
            }
        }
        else if (key == "2")
        {
            // 切换到视口1（右上）
            m_activeRenderer = 1;
            std::cout << "[INTERACTOR] Focus on viewport 1: Velocity + Streamlines" << std::endl;
            if (m_renderers[1])
            {
                this->SetDefaultRenderer(m_renderers[1]);
            }
        }
        else if (key == "3")
        {
            // 切换到视口2（左下）
            m_activeRenderer = 2;
            std::cout << "[INTERACTOR] Focus on viewport 2: Pressure Warped Surface" << std::endl;
            if (m_renderers[2])
            {
                this->SetDefaultRenderer(m_renderers[2]);
            }
        }
        else if (key == "4")
        {
            // 切换到视口3（右下）
            m_activeRenderer = 3;
            std::cout << "[INTERACTOR] Focus on viewport 3: Velocity Glyphs" << std::endl;
            if (m_renderers[3])
            {
                this->SetDefaultRenderer(m_renderers[3]);
            }
        }
        else if (key == "r" || key == "R")
        {
            // 重置当前视口相机
            std::cout << "[INTERACTOR] Reset camera for viewport "
                      << m_activeRenderer << std::endl;
            if (m_renderers[m_activeRenderer])
            {
                m_renderers[m_activeRenderer]->ResetCamera();
            }
        }
        else if (key == "a" || key == "A")
        {
            // 重置所有视口相机
            std::cout << "[INTERACTOR] Reset all cameras" << std::endl;
            for (int i = 0; i < 4; ++i)
            {
                if (m_renderers[i])
                {
                    m_renderers[i]->ResetCamera();
                }
            }
        }
        else if (key == "f" || key == "F")
        {
            // Fit：让当前视口自动调整视角到所有可见物体
            std::cout << "[INTERACTOR] Fit camera for viewport "
                      << m_activeRenderer << std::endl;
            if (m_renderers[m_activeRenderer])
            {
                m_renderers[m_activeRenderer]->ResetCamera();
                // 稍微调整以更好地包含视口内容
                vtkCamera* cam = m_renderers[m_activeRenderer]->GetActiveCamera();
                if (cam) cam->Zoom(1.1);
            }
        }
        else if (key == "h" || key == "H")
        {
            // 帮助
            std::cout << "\n=== CFD Post-Processor Keyboard Shortcuts ===" << std::endl;
            std::cout << "  1/2/3/4 : Switch active viewport" << std::endl;
            std::cout << "  R       : Reset current viewport camera" << std::endl;
            std::cout << "  A       : Reset all cameras" << std::endl;
            std::cout << "  F       : Fit camera to scene" << std::endl;
            std::cout << "  H       : Show this help" << std::endl;
            std::cout << "  Q/Esc   : Quit" << std::endl;
            std::cout << "===========================================\n" << std::endl;
        }
        else if (key == "q" || key == "Q" || key == "Escape")
        {
            rwi->TerminateApp();
            return;
        }

        // 调用父类处理标准TrackballCamera交互
        vtkInteractorStyleTrackballCamera::OnChar();
    }

private:
    int m_activeRenderer;
    std::array<vtkRenderer*, 4> m_renderers;
};

vtkStandardNewMacro(CFDCustomInteractorStyle);

// ---------------------------------------------------------------------------
// 主函数
// ---------------------------------------------------------------------------
int main(int argc, char* argv[])
{
    (void)argc;
    (void)argv;

    std::cout << "========================================" << std::endl;
    std::cout << "  CFD后处理可视化工具" << std::endl;
    std::cout << "  VTK 9.5.2 第二卷实战篇综合练习" << std::endl;
    std::cout << "========================================" << std::endl;

    // ======================================================================
    // STEP 0: 生成CFD数据
    // ======================================================================
    std::cout << "\n[STEP 0] Generating CFD data..." << std::endl;
    auto cfdGrid = GenerateCFDData();

    // 获取数据范围
    double pressureRange[2];
    double velocityRange[2];
    cfdGrid->GetPointData()->GetArray("pressure")->GetRange(pressureRange);
    cfdGrid->GetPointData()->GetArray("velocity_magnitude")->GetRange(velocityRange);

    std::cout << "[STEP 0] Done. Pressure: [" << pressureRange[0]
              << ", " << pressureRange[1] << "], Velocity: ["
              << velocityRange[0] << ", " << velocityRange[1] << "]" << std::endl;

    // ======================================================================
    // STEP 1: 创建颜色表
    // ======================================================================
    std::cout << "\n[STEP 1] Creating look-up tables..." << std::endl;

    // 压力颜色表 -- Jet方案
    auto pressureLUT = CreateLookupTable(pressureRange[0], pressureRange[1], "jet");
    // 速度颜色表 -- Warm方案
    auto velocityLUT = CreateLookupTable(velocityRange[0], velocityRange[1], "warm");

    // ======================================================================
    // STEP 2: 构建四条可视化管道
    // ======================================================================

    // ------ 分支1: 压力等值线（左上视口） ------
    std::cout << "\n[STEP 2.1] Building Pressure Contour pipeline..." << std::endl;

    // 使用 StructuredGridGeometryFilter 提取2D平面
    vtkNew<vtkStructuredGridGeometryFilter> pressurePlane;
    pressurePlane->SetInputData(cfdGrid);

    // 等值线过滤器（叠加在线条上的黑色等值线）
    vtkNew<vtkContourFilter> pressureContours;
    pressureContours->SetInputConnection(pressurePlane->GetOutputPort());
    pressureContours->SetInputArrayToProcess(
        0, 0, 0, vtkDataObject::FIELD_ASSOCIATION_POINTS, "pressure");
    pressureContours->GenerateValues(15, pressureRange[0], pressureRange[1]);

    // 等值线的Mapper
    vtkNew<vtkPolyDataMapper> contourMapper;
    contourMapper->SetInputConnection(pressureContours->GetOutputPort());
    contourMapper->ScalarVisibilityOff();  // 等值线统一为黑色

    // 等值线的Actor
    vtkNew<vtkActor> contourActor;
    contourActor->SetMapper(contourMapper);
    contourActor->GetProperty()->SetColor(0.0, 0.0, 0.0);   // 黑色等值线
    contourActor->GetProperty()->SetLineWidth(1.0);

    // 压力面着色（使用DataSetMapper直接渲染平面 + 压力标量着色）
    vtkNew<vtkDataSetMapper> pressureMapper;
    pressureMapper->SetInputConnection(pressurePlane->GetOutputPort());
    pressureMapper->SetScalarModeToUsePointFieldData();
    pressureMapper->SelectColorArray("pressure");
    pressureMapper->SetLookupTable(pressureLUT);
    pressureMapper->SetScalarRange(pressureRange[0], pressureRange[1]);

    vtkNew<vtkActor> pressureActor;
    pressureActor->SetMapper(pressureMapper);

    // ------ 分支2: 速度幅值 + 流线（右上视口） ------
    std::cout << "[STEP 2.2] Building Velocity + Streamlines pipeline..." << std::endl;

    // 速度面着色
    vtkNew<vtkStructuredGridGeometryFilter> velocityPlane;
    velocityPlane->SetInputData(cfdGrid);

    vtkNew<vtkDataSetMapper> velocityMapper;
    velocityMapper->SetInputConnection(velocityPlane->GetOutputPort());
    velocityMapper->SetScalarModeToUsePointFieldData();
    velocityMapper->SelectColorArray("velocity_magnitude");
    velocityMapper->SetLookupTable(velocityLUT);
    velocityMapper->SetScalarRange(velocityRange[0], velocityRange[1]);

    vtkNew<vtkActor> velocityActor;
    velocityActor->SetMapper(velocityMapper);

    // 流线种子点：在入口附近沿Y方向均匀分布
    vtkNew<vtkPolyData> seedPoints;
    vtkNew<vtkPoints>   seeds;
    int nSeeds = 15;
    for (int i = 0; i < nSeeds; ++i)
    {
        double y = DOMAIN_YMIN + 0.2 + (DOMAIN_YMAX - DOMAIN_YMIN - 0.4) * i / (nSeeds - 1);
        seeds->InsertNextPoint(DOMAIN_XMIN + 0.15, y, 0.0);
    }
    seedPoints->SetPoints(seeds);

    // 流线追踪器
    vtkNew<vtkStreamTracer> streamTracer;
    streamTracer->SetInputData(cfdGrid);
    streamTracer->SetSourceData(seedPoints);
    streamTracer->SetInputArrayToProcess(
        0, 0, 0, vtkDataObject::FIELD_ASSOCIATION_POINTS, "velocity");
    streamTracer->SetIntegrationDirectionToForward();
    streamTracer->SetIntegratorTypeToRungeKutta4();
    streamTracer->SetMaximumPropagation(200.0);
    streamTracer->SetInitialIntegrationStep(0.05);
    streamTracer->SetMinimumIntegrationStep(0.01);
    streamTracer->SetMaximumIntegrationStep(0.2);
    streamTracer->SetTerminalSpeed(0.001);

    // 管状流线（使用TubeFilter增加流线的视觉厚度）
    vtkNew<vtkTubeFilter> streamTubes;
    streamTubes->SetInputConnection(streamTracer->GetOutputPort());
    streamTubes->SetRadius(0.03);
    streamTubes->SetNumberOfSides(8);

    vtkNew<vtkPolyDataMapper> streamlineMapper;
    streamlineMapper->SetInputConnection(streamTubes->GetOutputPort());
    streamlineMapper->ScalarVisibilityOff();

    vtkNew<vtkActor> streamlineActor;
    streamlineActor->SetMapper(streamlineMapper);
    streamlineActor->GetProperty()->SetColor(0.0, 0.0, 0.0);  // 黑色流线
    streamlineActor->GetProperty()->SetLineWidth(1.5);

    // ------ 分支3: 压力变形曲面（左下视口） ------
    std::cout << "[STEP 2.3] Building Pressure Warped Surface pipeline..." << std::endl;

    vtkNew<vtkWarpScalar> warpFilter;
    warpFilter->SetInputData(cfdGrid);
    warpFilter->SetInputArrayToProcess(
        0, 0, 0, vtkDataObject::FIELD_ASSOCIATION_POINTS, "pressure");
    warpFilter->SetScaleFactor(0.02);  // Z方向位移因子

    // 提取曲面（结构化网格经过Warp后仍是StructuredGrid）
    vtkNew<vtkDataSetMapper> warpMapper;
    warpMapper->SetInputConnection(warpFilter->GetOutputPort());
    warpMapper->SetScalarModeToUsePointFieldData();
    warpMapper->SelectColorArray("pressure");
    warpMapper->SetLookupTable(pressureLUT);
    warpMapper->SetScalarRange(pressureRange[0], pressureRange[1]);

    vtkNew<vtkActor> warpActor;
    warpActor->SetMapper(warpMapper);

    // ------ 分支4: 速度矢量Glyph（右下视口） ------
    std::cout << "[STEP 2.4] Building Velocity Glyphs pipeline..." << std::endl;

    // 降采样以减少箭头密度
    vtkNew<vtkMaskPoints> maskPoints;
    maskPoints->SetInputData(cfdGrid);
    maskPoints->SetOnRatio(4);  // 每隔4个点采样一个
    maskPoints->SetMaximumNumberOfPoints(500);
    maskPoints->RandomModeOff();
    maskPoints->GenerateVerticesOn();

    // 箭头Glyph源
    vtkNew<vtkArrowSource> arrowSource;
    arrowSource->SetTipResolution(8);
    arrowSource->SetShaftResolution(12);

    // Glyph3D生成箭头
    vtkNew<vtkGlyph3D> glyphFilter;
    glyphFilter->SetInputConnection(maskPoints->GetOutputPort());
    glyphFilter->SetSourceConnection(arrowSource->GetOutputPort());
    glyphFilter->SetInputArrayToProcess(
        1, 0, 0, vtkDataObject::FIELD_ASSOCIATION_POINTS, "velocity");
    glyphFilter->SetVectorModeToUseVector();
    glyphFilter->SetScaleModeToScaleByVector();
    glyphFilter->SetScaleFactor(0.15);
    glyphFilter->OrientOn();              // 箭头方向跟随矢量方向
    glyphFilter->ClampingOn();

    vtkNew<vtkPolyDataMapper> glyphMapper;
    glyphMapper->SetInputConnection(glyphFilter->GetOutputPort());
    glyphMapper->SetScalarModeToUsePointFieldData();
    glyphMapper->SelectColorArray("velocity_magnitude");
    glyphMapper->SetLookupTable(velocityLUT);
    glyphMapper->SetScalarRange(velocityRange[0], velocityRange[1]);

    vtkNew<vtkActor> glyphActor;
    glyphActor->SetMapper(glyphMapper);

    // ======================================================================
    // STEP 3: 创建渲染器、视口和颜色条
    // ======================================================================
    std::cout << "\n[STEP 3] Setting up renderers and viewports..." << std::endl;

    // 创建四个渲染器
    vtkNew<vtkRenderer> renderer0;  // 左上：压力等值线
    vtkNew<vtkRenderer> renderer1;  // 右上：速度+流线
    vtkNew<vtkRenderer> renderer2;  // 左下：压力变形曲面
    vtkNew<vtkRenderer> renderer3;  // 右下：速度箭头Glyph

    // 设置视口边界
    renderer0->SetViewport(0.00, VP_TOP_Y, VP_LEFT_X, 1.00);
    renderer1->SetViewport(VP_RIGHT_X, VP_TOP_Y, 1.00, 1.00);
    renderer2->SetViewport(0.00, 0.00, VP_LEFT_X, VP_BTM_Y);
    renderer3->SetViewport(VP_RIGHT_X, 0.00, 1.00, VP_BTM_Y);

    // 设置背景色
    double bgColor[3] = {0.95, 0.95, 0.98};  // 浅蓝灰色
    renderer0->SetBackground(bgColor);
    renderer1->SetBackground(bgColor);
    renderer2->SetBackground(bgColor);
    renderer3->SetBackground(bgColor);

    // 向渲染器添加Actor
    renderer0->AddActor(pressureActor);
    renderer0->AddActor(contourActor);

    renderer1->AddActor(velocityActor);
    renderer1->AddActor(streamlineActor);

    renderer2->AddActor(warpActor);

    renderer3->AddActor(glyphActor);

    // 创建标量条
    auto pressureBar = CreateScalarBar(pressureLUT, "Pressure\n[Pa]", 6);
    auto velocityBar = CreateScalarBar(velocityLUT, "Velocity\n[m/s]", 6);

    // 向对应的渲染器添加标量条（通过AddActor2D添加为2D覆盖层）
    renderer0->AddActor2D(pressureBar);
    renderer1->AddActor2D(velocityBar);
    renderer2->AddActor2D(pressureBar);
    renderer3->AddActor2D(velocityBar);

    // ======================================================================
    // STEP 4: 设置相机
    // ======================================================================
    std::cout << "[STEP 4] Configuring cameras..." << std::endl;

    // 视口0和视口1：俯视（Top-Down View, XY平面）
    // 先重置再调整
    renderer0->ResetCamera();
    renderer1->ResetCamera();

    vtkCamera* cam0 = renderer0->GetActiveCamera();
    cam0->SetPosition(3.0, 0.0, 15.0);    // 从远处高处俯视
    cam0->SetFocalPoint(3.0, 0.0, 0.0);    // 焦点在网格中心
    cam0->SetViewUp(0.0, 1.0, 0.0);        // Y轴向上
    cam0->ParallelProjectionOn();            // 平行投影（更适合2D视图）
    cam0->SetParallelScale(2.5);

    vtkCamera* cam1 = renderer1->GetActiveCamera();
    cam1->SetPosition(3.0, 0.0, 15.0);
    cam1->SetFocalPoint(3.0, 0.0, 0.0);
    cam1->SetViewUp(0.0, 1.0, 0.0);
    cam1->ParallelProjectionOn();
    cam1->SetParallelScale(2.5);

    // 视口2和视口3：倾斜视角（Perspective, 3D效果可见）
    renderer2->ResetCamera();
    renderer3->ResetCamera();

    vtkCamera* cam2 = renderer2->GetActiveCamera();
    cam2->SetPosition(3.0, -6.0, 8.0);     // 从斜前方/下方观察
    cam2->SetFocalPoint(3.0, 0.0, 0.0);
    cam2->SetViewUp(0.0, 0.0, 1.0);
    cam2->SetElevation(30);
    cam2->SetAzimuth(30);

    vtkCamera* cam3 = renderer3->GetActiveCamera();
    cam3->SetPosition(3.0, -6.0, 8.0);
    cam3->SetFocalPoint(3.0, 0.0, 0.0);
    cam3->SetViewUp(0.0, 0.0, 1.0);
    cam3->SetElevation(30);
    cam3->SetAzimuth(30);

    // ======================================================================
    // STEP 5: 添加文本标注
    // ======================================================================
    std::cout << "[STEP 5] Adding text labels..." << std::endl;

    // 使用 vtkTextActor 为每个视口添加标题
    // 位置基于归一化视口坐标（Display坐标）
    // 视口0 标签（左上）
    vtkNew<vtkTextActor> label0;
    label0->SetInput("(a) Pressure Contour - Filled Contours");
    label0->GetTextProperty()->SetFontSize(14);
    label0->GetTextProperty()->SetColor(0.1, 0.1, 0.3);
    label0->GetTextProperty()->SetBold(1);
    label0->GetTextProperty()->SetFontFamilyToArial();
    label0->SetPosition(10, 10);

    // 视口1 标签（右上）
    vtkNew<vtkTextActor> label1;
    label1->SetInput("(b) Velocity Magnitude + Streamlines");
    label1->GetTextProperty()->SetFontSize(14);
    label1->GetTextProperty()->SetColor(0.1, 0.1, 0.3);
    label1->GetTextProperty()->SetBold(1);
    label1->GetTextProperty()->SetFontFamilyToArial();
    label1->SetPosition(10, 10);

    // 视口2 标签（左下）
    vtkNew<vtkTextActor> label2;
    label2->SetInput("(c) Pressure Warped Surface (3D)");
    label2->GetTextProperty()->SetFontSize(14);
    label2->GetTextProperty()->SetColor(0.1, 0.1, 0.3);
    label2->GetTextProperty()->SetBold(1);
    label2->GetTextProperty()->SetFontFamilyToArial();
    label2->SetPosition(10, 10);

    // 视口3 标签（右下）
    vtkNew<vtkTextActor> label3;
    label3->SetInput("(d) Velocity Arrow Glyphs");
    label3->GetTextProperty()->SetFontSize(14);
    label3->GetTextProperty()->SetColor(0.1, 0.1, 0.3);
    label3->GetTextProperty()->SetBold(1);
    label3->GetTextProperty()->SetFontFamilyToArial();
    label3->SetPosition(10, 10);

    renderer0->AddActor2D(label0);
    renderer1->AddActor2D(label1);
    renderer2->AddActor2D(label2);
    renderer3->AddActor2D(label3);

    // 全局标题
    vtkNew<vtkTextActor> titleActor;
    titleActor->SetInput("CFD Post-Processor: Flow Around a Cylinder");
    titleActor->GetTextProperty()->SetFontSize(18);
    titleActor->GetTextProperty()->SetColor(0.05, 0.05, 0.2);
    titleActor->GetTextProperty()->SetBold(1);
    titleActor->GetTextProperty()->SetFontFamilyToArial();
    titleActor->SetPosition(10, 10);

    // ======================================================================
    // STEP 6: 创建渲染窗口
    // ======================================================================
    std::cout << "[STEP 6] Creating render window..." << std::endl;

    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(1400, 1000);
    renderWindow->SetWindowName("CFD 后处理可视化工具 -- Flow Around a Cylinder");

    // 添加所有渲染器
    renderWindow->AddRenderer(renderer0);
    renderWindow->AddRenderer(renderer1);
    renderWindow->AddRenderer(renderer2);
    renderWindow->AddRenderer(renderer3);

    // ======================================================================
    // STEP 7: 配置交互器
    // ======================================================================
    std::cout << "[STEP 7] Configuring custom interactor..." << std::endl;

    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    vtkNew<CFDCustomInteractorStyle> customStyle;
    customStyle->SetRenderer(0, renderer0);
    customStyle->SetRenderer(1, renderer1);
    customStyle->SetRenderer(2, renderer2);
    customStyle->SetRenderer(3, renderer3);
    customStyle->SetDefaultRenderer(renderer0);

    interactor->SetInteractorStyle(customStyle);

    // ======================================================================
    // STEP 8: 进入渲染循环
    // ======================================================================
    std::cout << "\n========================================" << std::endl;
    std::cout << "  Starting interactive visualization..." << std::endl;
    std::cout << "  Keyboard Shortcuts:" << std::endl;
    std::cout << "    1/2/3/4 : Switch viewport" << std::endl;
    std::cout << "    Mouse   : Rotate / Pan / Zoom (TrackballCamera)" << std::endl;
    std::cout << "    R       : Reset current camera" << std::endl;
    std::cout << "    H       : Help" << std::endl;
    std::cout << "    Q/Esc   : Quit" << std::endl;
    std::cout << "========================================\n" << std::endl;

    renderWindow->Render();
    interactor->Start();

    std::cout << "\n[DONE] CFD Post-Processor terminated." << std::endl;
    return 0;
}
```

### 15.4.2 CMakeLists.txt

```cmake
# ============================================================================
# CMakeLists.txt -- CFD后处理可视化工具
# 第二卷（实战篇）收官项目 -- 第十五章
# VTK 9.5.2
# ============================================================================

cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

# ---------------------------------------------------------------------------
# 项目信息
# ---------------------------------------------------------------------------
project(CFDPostProcessor VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ---------------------------------------------------------------------------
# 查找VTK依赖
# ---------------------------------------------------------------------------
# 对于VTK 9.5.2，推荐使用 find_package(VTK)
# 下面的配置列出了本项目需要的全部VTK模块
find_package(VTK REQUIRED COMPONENTS
  # 核心模块
  CommonCore
  CommonDataModel
  CommonExecutionModel
  CommonMath
  CommonSystem
  CommonTransforms
  CommonMisc

  # 数据模型
  FiltersSources          # vtkArrowSource 在此
  FiltersCore             # vtkContourFilter, vtkStreamTracer, vtkTubeFilter
  FiltersGeneral          # vtkWarpScalar
  FiltersFlowPaths        # vtkStreamTracer

  # 渲染模块
  RenderingCore
  RenderingOpenGL2
  RenderingAnnotation     # vtkScalarBarActor
  RenderingFreeType       # 文本渲染
  InteractionStyle
  ViewsCore
  ViewsContext2D
)

# 如果VTK未找到，报错
if(NOT VTK_FOUND)
  message(FATAL_ERROR "VTK 9.5.2 not found. Please set VTK_DIR to the VTK build directory.")
endif()

message(STATUS "VTK_VERSION: ${VTK_VERSION}")
message(STATUS "VTK_DIR: ${VTK_DIR}")

# ---------------------------------------------------------------------------
# 包含VTK头文件和使用要求
# ---------------------------------------------------------------------------
include(${VTK_USE_FILE})

# ---------------------------------------------------------------------------
# 可执行文件
# ---------------------------------------------------------------------------
add_executable(CFDPostProcessor
  CFDPostProcessor.cxx
)

# 设置目标属性
set_target_properties(CFDPostProcessor PROPERTIES
  CXX_STANDARD 17
  CXX_STANDARD_REQUIRED ON
  CXX_EXTENSIONS OFF
)

# 链接VTK库
target_link_libraries(CFDPostProcessor PRIVATE
  ${VTK_LIBRARIES}
)

# 包含目录
target_include_directories(CFDPostProcessor PRIVATE
  ${VTK_INCLUDE_DIRS}
)

# 编译定义（VTK 9.5.2需要的特定宏）
target_compile_definitions(CFDPostProcessor PRIVATE
  ${VTK_DEFINITIONS}
)

# ---------------------------------------------------------------------------
# 安装规则
# ---------------------------------------------------------------------------
install(TARGETS CFDPostProcessor
  RUNTIME DESTINATION bin
)

# ---------------------------------------------------------------------------
# 输出配置摘要
# ---------------------------------------------------------------------------
message(STATUS "=============================================")
message(STATUS "  CFDPostProcessor Configuration Summary")
message(STATUS "  C++ Standard: ${CMAKE_CXX_STANDARD}")
message(STATUS "  VTK Version:  ${VTK_VERSION}")
message(STATUS "  Build Type:   ${CMAKE_BUILD_TYPE}")
message(STATUS "=============================================")
```

---

## 15.5 代码详解

本节不逐行复述代码（完整的带注释代码已在15.4节中给出），而是聚焦于设计决策、技术要点和实现细节，帮助读者理解"为什么要这样写"而不仅仅是"写了什么"。

### 15.5.1 数据生成策略

数据生成是本章的第一个核心技术挑战。我们把`GenerateCFDData()`作为辅助函数独立出来，这是工程中的最佳实践——将数据源与可视化逻辑解耦。

#### 网格选择：为什么是 vtkStructuredGrid？

对于二维CFD可视化，结构化网格（`vtkStructuredGrid`）是自然的选择：

1. **拓扑隐含**：结构化网格的点排列是一个规则的IxJ数组，单元连接关系不需要显式存储。对于120x60的网格（7200个点），数据结构非常紧凑。
2. **可视化友好**：`vtkContourFilter`、`vtkStreamTracer`、`vtkWarpScalar`、`vtkGlyph3D`均可直接操作结构化网格，无需预先三角化。
3. **易于参数化**：修改分辨率只需改变`GRID_NX`和`GRID_NY`两个常量，代码的其他部分自动适应。

如果换成`vtkUnstructuredGrid`，同样的点数需要显式定义大约14000个四边形单元的连接关系，代码将变得更加冗长。

#### 物理建模的层次

我们的速度场生成采用了**分层建模**的思路，每一层增加一种物理效应：

**第1层：势流解。** 不可压缩无旋流动绕圆柱的解析解——这是位势流理论的基本结果。它在远离圆柱的地方非常准确。

**第2层：边界层修正。** 在圆柱表面附近（`r < R + bl_thickness`），我们线性地降低速度以模拟无滑移边界条件的影响。这虽然是一种启发式修正，但它正确地表达了"壁面附近流速降低"的物理现实。

**第3层：尾迹衰减。** 在圆柱后方（`x > 0`附近），叠加高斯形状的速度衰减因子。这个因子在中心最强（减速70%），向两侧逐渐减弱。它模拟了真实粘性流动中分离泡和尾迹区的低速特征。

这种分层方法使得代码具有很好的可调性和可解释性。每个物理效应的强度都可以独立调节——例如将`wake_strength`从60改为30就能使尾迹低压区减半，方便读者做参数实验。

#### 圆柱内部的"遮罩"处理

在真实CFD中，圆柱内部不在求解域内。本章采用的方法是将圆柱内部的速度设为零、压力设为固定值。在可视化中：
- 压力等值线图中，圆柱区域显示为均匀的浅色（因为它在`pressureLUT`的较高端附近），这实际上清晰地在视觉上区分了固体障碍物和流体域。
- 流线会自动绕开圆柱（因为内部速度为零，流线追踪器不会进入）。
- 在变形曲面中，圆柱区域保持平坦（零位移），也自然地指示固体区域。

这种处理方式虽然简单，但在教学演示中效果很好。

### 15.5.2 管道分支设计

四条可视化管道从同一个数据源分支出来。这个设计充分利用了VTK的两个核心特性：

**特性1：管道复用。** 由于所有四个分支都使用`SetInputConnection(cfdGrid->GetOutputPort())`（通过中间过滤器间接连接），当第一个分支触发`Update()`时，数据只需生成一次。后续三条分支复用同一个计算结果。这在实际大型数据集中能节省显著的CPU和内存。

**特性2：管道独立性。** 每条分支在收到共享数据后，各自执行独立的数据处理。修改分支1的等值线数量不会影响分支2的流线种子点配置。这种独立性让代码组织清晰——你可以分别开发和调试每条可视化管道。

值得注意的是，虽然我们在代码中使用`SetInputData(cfdGrid)`（`cfdGrid`是一个`vtkSmartPointer<vtkStructuredGrid>`），但这仍然触发了管道执行：当Mapper需要数据时，如果Grid的MTime表示数据需要更新，`GenerateCFDData()`的所有计算会被触发。在实际工程中，如果你想确保数据只生成一次，可以显式调用`cfdGrid->Update()`或者使用更明确的管道连接方式。

### 15.5.3 相机设置策略

四个视口使用两种不同的相机配置：

**视口0和视口1（顶视图）：平行投影**

```cpp
cam0->ParallelProjectionOn();
cam0->SetParallelScale(2.5);
```

平行投影消除了透视变形，使得二维流道平面上的等值线和流线保持真实的几何形状。这对于定量分析非常重要——如果使用透视投影，等值线在近处和远处的间距会不一致，给视觉判断压力梯度带来困难。平行投影的`SetParallelScale(2.5)`值决定了视口可见的垂直半高（在Z=0平面上，Y方向的可见范围约为[-2.5, 2.5]），略大于流道高度(4.0)确保边界可见。

**视口2和视口3（倾斜视角）：透视投影**

```cpp
cam2->SetPosition(3.0, -6.0, 8.0);
cam2->SetElevation(30);
cam2->SetAzimuth(30);
```

透视投影用于3D效果的视口——压力变形曲面和Glyph箭头在透视下更具深度感。相机位置(3.0, -6.0, 8.0)意味着相机从流道的右下方观察，Z方向的高度使变形曲面的"山峰"和"山谷"清晰可辨。`SetElevation(30)`和`SetAzimuth(30)`提供了经典的"工程等距视角"——在视觉吸引力和数据可读性之间取得平衡。

### 15.5.4 颜色映射选择

两个不同的颜色方案服务于不同的可视化目的：

**压力场：Jet方案（蓝->青->绿->黄->红）**

Jet色彩方案尽管在可视化社区中存在争议（因为其不均匀的亮度过渡可能导致视觉偏差），但在压力场的渲染中它提供了出色的感知对比度。高压区（入口附近）的红色与低压区（尾迹中心）的深蓝色形成强烈的对比，使得压力的空间变化模式一目了然。对于教学演示和工程预览，这种高对比度是有价值的。

**速度场：Warm方案（白->黄->橙->深红）**

暖色方案避免了Jet方案中青色和黄绿色可能引入的视觉混乱。白色对应低速（入口均匀来流和尾迹低速区），深红色对应高速（圆柱两侧加速区）。这种从白到红的渐变在感知上具有自然的序数意义（越红越快），并且不受常见色盲类型的影响（红-黄色盲的读者仍然能感知亮度差异）。

### 15.5.5 vtkScalarBarActor 的精细调校

每个视口的颜色条通过`CreateScalarBar()`辅助函数创建。几个关键的设计决策：

- **SetWidth(0.12) 和 SetHeight(0.65)**：颜色条占视口宽度的12%和高度的65%。这个比例确保颜色条足够大，标签文字可读，同时不侵占主要的3D渲染空间。
- **SetPosition(0.85, 0.15)**：将颜色条放置在视口右侧中心。x=0.85意味着颜色条左边缘在视口80%宽度处，右侧留15%余量。y=0.15意味着底部在视口15%高度处。
- **SetNumberOfLabels(6)**：6个标签在颜色条长度上均匀分布，提供足够的数据参考而不显得拥挤。
- **白色半透明背景**：`SetBackgroundOpacity(0.7)`和白色背景确保标签在可能变化的渲染内容上始终可读。
- **黑色标签**：与Jet和Warm色彩方案中的任何颜色都形成高对比度。

### 15.5.6 流线追踪的精度控制

`vtkStreamTracer`的精度参数对一个好的流线可视化至关重要：

```cpp
streamTracer->SetIntegratorTypeToRungeKutta4();
streamTracer->SetMaximumPropagation(200.0);
streamTracer->SetInitialIntegrationStep(0.05);
streamTracer->SetTerminalSpeed(0.001);
```

- **RungeKutta4积分器**：四阶龙格-库塔方法提供了精度和速度的良好平衡。对于圆柱绕流这种曲率较大的流场，低阶积分器（如RK2）会产生可见的锯齿。
- **MaximumPropagation(200.0)**：流线最大长度为单位来流速度可传播200个单位距离。考虑到流道总长约10个单位，这个值对于确保流线"跑完"整个流道来说绰绰有余。
- **InitialIntegrationStep(0.05)**：初始步长约为网格间距的1/2（dx=10/120≈0.083），确保流线在穿越单个网格单元时至少经过两个采样点。
- **TerminalSpeed(0.001)**：当流速降至来流速度的0.1%以下时终止追踪。这自然防止了流线在死水区（如圆柱正后方）中无限螺旋。

### 15.5.7 Glyph的降采样策略

`vtkMaskPoints`的降采样是Glyph可视化成败的关键。在120x60=7200个点上各放置一个箭头会完全遮挡画面。我们采用两种互补的降采样策略：

```cpp
maskPoints->SetOnRatio(4);   // 均匀降采样：每隔4个点取1个
maskPoints->SetMaximumNumberOfPoints(500);  // 上限：最多500个点
```

- **SetOnRatio(4)**：均匀采样产生排列整齐的箭头网格，视觉上呈现良好的规律性。在120x60的网格上，这会选择约30x15≈450个点。
- **SetMaximumNumberOfPoints(500)**：安全上限，防止高密度网格情况下箭头过多。
- **RandomModeOff()**：保持确定性采样。对于演示和教学的场景，排列整齐的箭头比随机排列的箭头更易理解。如果你在真实工程中需要更自然的视觉效果，可以开启RandomMode。

### 15.5.8 自定义交互器

`CFDCustomInteractorStyle`继承自`vtkInteractorStyleTrackballCamera`，在保留标准鼠标交互（左键旋转、中键平移、右键缩放）的基础上增加了键盘快捷键层：

```cpp
class CFDCustomInteractorStyle : public vtkInteractorStyleTrackballCamera
```

**视口切换（1/2/3/4键）**：通过调用`SetDefaultRenderer()`，将渲染窗口的焦点切换到对应视口。当用户按下数字键后，后续的鼠标操作只影响该视口的相机。这对于需要分别精调每个视口角度的场景非常有用。

**相机重置（R键）**：重置当前激活视口的相机到初始位置。这对于在旋转/缩放中"迷失方向"时快速回到标准视图很有帮助。

**全部重置（A键）**：一次性重置所有四个视口，恢复到程序启动时的标准视图配置。

**帮助（H键）**：在控制台输出完整的快捷键列表。这对于用户来说比记忆快捷键更友好。

**设计注意事项**：我们调用了`vtkInteractorStyleTrackballCamera::OnChar()`在自定义处理的末尾。这确保了未被我们捕获的按键（例如'w'用于线框模式等标准VTK按键）仍然由父类处理，保持了标准交互行为。

---

## 15.6 第二卷总结与第三卷预告

### 15.6.1 第二卷回顾：你学到了什么

第二卷（实战篇）涵盖第7章到第15章，其核心主题是**从数据到可视化的完整链条**。让我们回顾一下你走过的路程：

| 章节 | 主题 | 核心技能 |
|------|------|---------|
| 第7章 | PolyData深入 | 手动构建Points/Cells，理解Topology与Geometry的关系，Verts/Lines/Polys/Strips |
| 第8章 | UnstructuredGrid | 任意单元类型，显式拓扑定义，混合单元网格，CellTypes枚举 |
| 第9章 | 结构化数据 | ImageData/RectilinearGrid/StructuredGrid三大家族，坐标系统的差异 |
| 第10章 | 数据I/O | Legacy格式(.vtk)，XML格式(.vtu/.vts/.vtp)，PVD时间序列，FLUENT/OpenFOAM读取 |
| 第11章 | 常用过滤器 | 几何变换、数据提取、矢量可视化、网格处理四类滤波器 |
| 第12章 | 颜色映射 | LookupTable机制，标量范围，色彩空间，ScalarBar，对数映射 |
| 第13章 | 高级渲染 | 透明度、光照模型、纹理映射、Offscreen渲染、立体渲染 |
| 第14章 | 多数据集 | vtkAppendPolyData/vtkAppendFilter合并，vtkMultiBlockDataSet层次化组织 |
| 第15章 | 综合实战 | CFD后处理工具：数据生成、分支管道、多视口、颜色条、自定义交互 |

通过这一卷的学习和第十五章的综合实践，你现在应该能够：

1. **为任意数据类型选择合适的网格容器**——结构化、非结构化还是PolyData，每种选择背后的性能和应用权衡。
2. **将外部数据导入VTK**——无论是标准VTK格式、CFD求解器格式，还是通过手动编程构建。
3. **在数据上应用过滤器链**——将多个过滤器串联起来，实现复杂的数据处理逻辑。
4. **设计有效的科学可视化方案**——选择合适的颜色映射、表达方式（面着色/等值线/流线/Glyph/变形曲面），以及多视口布局。
5. **构建工程化的VTK应用程序**——清晰的代码组织、辅助函数、CMake构建系统、自定义交互。

这些技能组合起来，使你具备了使用VTK解决**真实科学可视化问题**的能力。你不再只是"能跑通示例代码"的初学者，而是可以独立设计和实现可视化工具。

### 15.6.2 第三卷预告：VTK高级专题

第二卷为你打下了坚实的基础，但VTK的广度和深度远不止于此。第三卷（高级篇）将带你进入以下领域：

| 章节 | 主题 | 内容预告 |
|------|------|---------|
| 第16章 | 体渲染 | vtkFixedPointVolumeRayCastMapper，传递函数设计，GPU体渲染，多分量数据体渲染 |
| 第17章 | 2D上下文与图表 | vtkChartXY/vtkPlot，折线图、散点图、柱状图的VTK原生绘制，vtkContextView |
| 第18章 | 标注系统 | vtkCaptionActor2D，vtkLegendBoxActor，自定义标注，与交互联动 |
| 第19章 | 交互Widgets | vtkBoxWidget/vtkSphereWidget/vtkSliderWidget/vtkDistanceWidget，3D测量工具 |
| 第20章 | Qt与VTK集成 | QVTKOpenGLNativeWidget，模型-视图架构，信号槽与VTK事件桥接 |
| 第21章 | SMP并行处理 | vtkSMPTools，多线程过滤器和映射器，并行加速策略，性能测量 |
| 第22章 | 导出与发布 | 屏幕截图、高分辨率离线渲染、视频导出、ParaView集成、Web可视化 |
| 第23章 | 第三卷综合实战 | 构建一个完整的科学可视化桌面应用（Qt界面 + 多视口 + 体渲染 + Widgets + 数据I/O + 并行加速） |

第三卷的每一章都是一个独立的知识领域，但它们也相互关联：你可以在Qt界面中嵌入VTK渲染窗口（第20章），使用交互Widgets操作3D场景（第19章），将处理结果通过体渲染呈现（第16章），并利用SMP并行加速整个管线（第21章）。

当第三卷结束时，你将具备使用VTK进行**生产级**科学可视化开发的完整技能集——从数据处理到高级渲染，从用户界面到性能优化。

### 15.6.3 结语

完成第二卷的学习是一个值得庆祝的里程碑。你已经从第一卷的"管道和数据源"基础知识跨越到了"真实数据处理和完整可视化应用构建"的能力层次。第十五章的CFD后处理工具不是终点——它是你开始构建自己可视化工具群的起点。

VTK是一个超过25年历史的成熟库，它的功能远超出任何单本书能覆盖的范围。但你现在拥有的知识体系（管道架构、数据模型、过滤器、渲染器、交互器、颜色映射、数据I/O）构成了一个坚实的"功能集"——无论你将来面对什么样的可视化问题，你都可以从这个功能集中挑选合适的工具来构建解决方案。

建议的后续学习路径：

1. **反复实践**：修改本章的CFD工具代码——改变网格密度、调整物理参数、添加新的过滤器分支、尝试不同的颜色方案。修改代码是比阅读代码更好的学习方式。
2. **连接真实数据**：找到手头的仿真数据（OpenFOAM案例、ANSYS导出、甚至是CSV表格数据），尝试用VTK程序加载和可视化它们。
3. **研究VTK示例**：VTK源码自带的`Examples/`目录包含了数百个经过验证的示例程序，覆盖了本书未涉及的许多高级功能。
4. **深入第三卷**：当你准备好时，第三卷的高级主题将进一步提升你的技能层次。

祝你学习进步，构建出令人赞叹的可视化作品。

---

*第二卷（实战篇）完。感谢阅读。*
