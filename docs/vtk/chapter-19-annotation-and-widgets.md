# 第十九章 三维标注与交互组件 (3D Annotation and Display Components)

## 本章导读

在前面的章节中，你已经掌握了如何使用VTK构建数据可视化管线的核心能力：从数据读取、过滤变换、颜色映射、材质渲染，到体绘制和复合可视化。这些技能让你能够将原始数据转化为富有视觉表现力的三维场景。然而，一个有效的可视化作品不仅要"好看"，还要**易于理解**——观察者需要知道数据的大小比例、空间方向、数值含义，以及场景中各对象的身份。

这就是**标注系统**（Annotation System）的用武之地。标注是叠加在三维可视化之上的"信息层"，它为观察者提供理解场景所需的上下文信息，包括但不限于：

- **数值信息**：标量条（颜色图例）告诉观察者"某种颜色对应什么数值"。
- **空间方向**：方向标（坐标轴指示器）告诉观察者"数据在三维空间中的哪个朝向"。
- **文字说明**：标题、角标、数据描述文字提供了场景的元信息。
- **比例尺度**：比例尺告诉观察者"屏幕上的距离对应数据中的多少物理单位"。
- **图例信息**：多数据集的图例帮助观察者区分不同的数据对象。

本章将系统地讲解VTK中与标注相关的核心类和组件，涵盖以下主题：

1. **标注系统概览**（19.1节）——理解二维标注与三维标注的区别、标注Actor与场景Actor的关系。
2. **标量条深度使用**（19.2节）——在第12章标量条基础之上，深入学习精细外观控制、多标量条布局管理、文本属性定制。
3. **方向标与坐标轴**（19.3节）——使用`vtkOrientationMarkerWidget`和`vtkAxesActor`创建方向指示器，以及`vtkAnnotatedCubeActor`创建带文字标签的方向立方体。
4. **文字标注**（19.4节）——四种主要的文字标注方式：固定屏幕文字（`vtkTextActor`）、跟随3D点的引线文字（`vtkCaptionActor2D`）、角标注（`vtkCornerAnnotation`）和自定义的3D文字。
5. **比例尺与图例**（19.5节）——创建空间比例尺和数据集图例的多种方法。
6. **完整代码示例**（19.6节）——一个约300行的综合演示程序，整合了所有标注类型。
7. **本章小结**（19.7节）。

本章需要的前置知识包括：第5章（渲染管道基础）、第12章（颜色映射与标量条基础）、第13章（高级渲染技术）。如果你对这些内容不太熟悉，建议先回顾相关章节。

---

## 19.1 标注系统概览

### 19.1.1 标注的本质：信息层

在VTK的架构中，标注可以被理解为一个独立的**信息层**（Information Layer），叠加在三维数据可视化之上。这个层次不修改底层的可视化数据，而是通过额外的视觉元素来增强观察者的理解。

关键区别在于：

- **数据可视化**：将数据集通过Mapper映射为可渲染的图元（点、线、三角形），形成对数据的视觉呈现。
- **标注**：在数据可视化之上，添加文字、线条、图例、图标等辅助元素，它们不表示数据本身，而是**解释数据**。

这种分离设计是科学可视化的一项基本原则。一个典型的例子是：当你展示一个有限元分析的应力分布时，彩色曲面是"数据可视化"，而旁边的标量条（标出应力值范围）和左上角的方向标（标出坐标轴方向）是"标注"。两者协同工作，但你可以独立地打开或关闭标注层而不影响数据层。

### 19.1.2 二维标注 vs 三维标注

VTK中的标注根据其坐标空间分为两大类：

**二维标注（2D Annotation）**——也称为"叠加标注"（Overlay Annotation）。这些标注位于**屏幕空间**中，使用归一化视口坐标（Normalized Viewport Coordinates），位置以渲染窗口的左下角为原点(0, 0)，右上角为(1, 1)。二维标注不随三维场景的旋转、缩放或平移而改变位置。它们通过`renderer->AddActor2D()`添加到渲染器中。

二维标注的典型代表：
- `vtkScalarBarActor`：标量条/颜色图例
- `vtkTextActor`：屏幕空间固定文字
- `vtkCornerAnnotation`：角标注
- `vtkLegendBoxActor`、`vtkLegendScaleActor`：图例和比例尺

**三维标注（3D Annotation）**——这些标注位于**世界空间**中，使用场景的三维坐标系统。它们像普通的三维Actor一样，可以随场景旋转、缩放和平移。三维标注通过`renderer->AddActor()`（或`AddViewProp()`）添加到渲染器中。

三维标注的典型代表：
- `vtkCaptionActor2D`：带有引线的文字标注（虽然Actor本身是2D的，但它跟随一个3D锚点）
- `vtkAxesActor`：三维坐标轴
- `vtkCubeAxesActor`：包围盒坐标轴
- `vtkAnnotatedCubeActor`：带文字标签的方向立方体

此外，还有一种特殊的**交互标注**——这些标注通过**Widget**（交互组件）来管理，支持用户交互：
- `vtkOrientationMarkerWidget`：管理方向指示标记
- `vtkScalarBarWidget`：可交互的标量条
- `vtkCaptionWidget`：可交互的引线文字

表19.1总结了两种标注类型的对比：

| 特性 | 二维标注 (2D) | 三维标注 (3D) |
|------|---------------|---------------|
| 坐标空间 | 屏幕空间（归一化视口坐标） | 世界空间（三维坐标） |
| 添加方式 | `renderer->AddActor2D()` | `renderer->AddActor()` |
| 随场景旋转变化 | 否 | 是 |
| 随场景缩放变化 | 否（可用像素限制尺寸） | 是 |
| 适用场景 | 图例、标题、固定信息、标量条 | 方向指示器、数据标签、空间参考 |
| 交互支持 | 部分通过Widget支持 | Widget支持拖拽等交互 |

### 19.1.3 标注不修改数据的核心原则

一个重要的设计原则需要在此强调：**标注组件不应修改底层数据集或可视化管线的状态**。标注Actor接收的是对已有查找表、相机或数据集的**引用**（而非拷贝），它们读取这些引用中的信息并将其可视化，但不应更改这些引用的内容。

例如，`vtkScalarBarActor`持有对`vtkLookupTable`的引用（通过`SetLookupTable()`），它读取查找表中的颜色和范围信息来绘制颜色条，但绝不会修改查找表中的任何条目。同样，`vtkOrientationMarkerWidget`读取渲染器的当前相机状态来同步方向指示，但不会修改相机参数。

这种"读取但不修改"的设计确保了标注系统的安全性和可组合性——你可以随时添加或移除标注组件，而不必担心它们对数据可视化产生副作用。

---

## 19.2 标量条（vtkScalarBarActor深度使用）

在第12章中，你已经学习了`vtkScalarBarActor`的基本用法：创建标量条、关联查找表、设置标题和标签数量。本节将在这些基础知识之上，深入探讨标量条的高级外观控制和多标量条布局管理。

### 19.2.1 精细位置与尺寸控制

在第12章中，我们使用了`SetPosition()`、`SetWidth()`和`SetHeight()`来控制标量条的位置和尺寸。这些方法使用的是**归一化视口坐标**。在实际的复杂可视化布局中，你还需要了解以下几个关键API来精确控制标量条的几何外观：

```cpp
#include <vtkScalarBarActor.h>

vtkNew<vtkScalarBarActor> scalarBar;
scalarBar->SetLookupTable(lut);

// --- 位置设置（归一化视口坐标） ---
// SetPosition(x, y) 设置标量条左下角的位置
scalarBar->SetPosition(0.85, 0.1);

// --- 尺寸设置（归一化视口坐标） ---
scalarBar->SetWidth(0.1);         // 标量条的整体宽度
scalarBar->SetHeight(0.6);        // 标量条的整体高度

// --- 像素单位的尺寸限制 ---
// 这些限制优先级高于归一化坐标的尺寸设置
scalarBar->SetMaximumWidthInPixels(120);   // 最大宽度（像素），超过此值不再增大
scalarBar->SetMaximumHeightInPixels(600);  // 最大高度（像素），超过此值不再增大

// --- 颜色条本身的几何比例 ---
// SetBarRatio() 控制在标量条的总宽度中，颜色条部分占据的比例
// 默认值约为0.375（颜色条占37.5%，其余为文字区域）
scalarBar->SetBarRatio(0.4);     // 颜色条占40%，文字区域的标注占60%
```

`SetMaximumWidthInPixels()`和`SetMaximumHeightInPixels()`在实际应用中非常重要。由于归一化坐标依赖于窗口大小，当一个窗口被拉伸到非常大时，用归一化坐标定义的标量条可能会变得不协调地大。像素限制提供了一种"上限"约束。

### 19.2.2 标签与标题的精细文本控制

标量条的外观很大程度上取决于其文本元素（标题和数值标签）的质量。VTK提供了丰富的API来分别控制标题和标签的文本属性：

```cpp
// --- 标题文本属性 ---
vtkTextProperty* titleProp = scalarBar->GetTitleTextProperty();
titleProp->SetFontSize(14);                            // 字体大小（点数）
titleProp->SetFontFamilyToArial();                     // 字体族
// 可选字体族:
//   VTK_ARIAL, VTK_COURIER, VTK_TIMES
titleProp->SetColor(0.0, 0.0, 0.0);                   // 标题颜色（黑色）
titleProp->SetBold(true);                              // 粗体
titleProp->SetItalic(false);                           // 非斜体
titleProp->SetShadow(false);                           // 无阴影
titleProp->SetOpacity(1.0);                            // 完全不透明

// --- 标签文本属性 ---
vtkTextProperty* labelProp = scalarBar->GetLabelTextProperty();
labelProp->SetFontSize(10);
labelProp->SetFontFamilyToArial();
labelProp->SetColor(0.2, 0.2, 0.2);                   // 深灰色字体
labelProp->SetBold(false);
labelProp->SetItalic(false);

// --- 标签格式化 ---
scalarBar->SetLabelFormat("%.2f");                     // 保留两位小数的浮点数
scalarBar->SetLabelFormat("%4.1f");                    // 至少4位宽度，一位小数
scalarBar->SetLabelFormat("%.1e");                     // 科学计数法

// --- 标签数量 ---
scalarBar->SetNumberOfLabels(7);                       // 显示7个数值标签（含两端）

// --- 标签相对于颜色条的位置 ---
scalarBar->SetTextPositionToPrecedeScalarBar();        // 文字在颜色条之前（左侧，垂直条）
scalarBar->SetTextPositionToSucceedScalarBar();        // 文字在颜色条之后（右侧，垂直条）
```

`SetTextPositionToPrecedeScalarBar()`和`SetTextPositionToSucceedScalarBar()`这两个方法控制数值标签显示在颜色条的哪一侧。对于垂直方向的标量条，"Precede"意味着标签出现在颜色条的左边；"Succeed"意味着标签出现在右边。在水平条中，"Precede"是上方，"Succeed"是下方。

### 19.2.3 边框、背景与其他外观设置

标量条的外观还可以通过以下选项进一步定制：

```cpp
// --- 边框设置 ---
scalarBar->SetDrawBorder(true);                        // 是否绘制标量条的外边框
scalarBar->SetBorderColor(0.0, 0.0, 0.0);              // 边框颜色
scalarBar->SetBorderThickness(2.0);                    // 边框厚度

// --- 背景设置 ---
scalarBar->SetDrawBackground(true);                    // 是否在标量条背后绘制背景
scalarBar->SetBackgroundColor(1.0, 1.0, 1.0);          // 背景颜色（白色）
scalarBar->SetBackgroundOpacity(0.7);                  // 背景不透明度（0.7 = 70%不透明）

// --- 颜色条的方向 ---
scalarBar->SetOrientationToVertical();                 // 垂直方向（默认）
scalarBar->SetOrientationToHorizontal();               // 水平方向

// --- 颜色条的框线 ---
scalarBar->SetDrawColorBar(true);                      // 是否绘制颜色条本身
scalarBar->SetDrawTickMarks(true);                     // 是否绘制刻度标记
scalarBar->SetTickLabelPrecision(2);                   // 刻度标签精度

// --- 设置Unconstrained字体大小 ---
// 默认情况下，字体大小可能受到标量条尺寸的限制
scalarBar->SetUnconstrainedFontSize(true);             // 字体大小不受限制
```

### 19.2.4 多标量条布局管理

当你在一个渲染窗口中展示多个数据集，或者使用多个渲染器时，你可能需要为每个数据集配置独立的标量条。这就涉及到多标量条的布局管理问题。

**场景一：单个渲染器中的多个标量条**

在一个渲染器中，你可以通过仔细设置每个标量条的位置和尺寸，来避免它们互相重叠：

```cpp
// 标量条1：右侧，上半部分
scalarBar1->SetOrientationToVertical();
scalarBar1->SetPosition(0.85, 0.55);
scalarBar1->SetWidth(0.12);
scalarBar1->SetHeight(0.35);

// 标量条2：右侧，下半部分
scalarBar2->SetOrientationToVertical();
scalarBar2->SetPosition(0.85, 0.12);
scalarBar2->SetWidth(0.12);
scalarBar2->SetHeight(0.35);

// 添加到渲染器
renderer->AddActor2D(scalarBar1);
renderer->AddActor2D(scalarBar2);
```

**场景二：多视口布局中每个渲染器有自己的标量条**

这种架构在12.6节的多方案对比示例中已经使用过。每个Renderer有自己独立的视口（Viewport），其内部的标量条位置相对于该Renderer的视口而言。这是最清晰的管理方式：

```cpp
// 渲染器1占左半部分
renderer1->SetViewport(0.0, 0.0, 0.5, 1.0);

// 标量条1在渲染器1中定位
scalarBar1->SetPosition(0.85, 0.1);  // 相对于renderer1的视口

// 渲染器2占右半部分
renderer2->SetViewport(0.5, 0.0, 1.0, 1.0);

// 标量条2在渲染器2中定位
scalarBar2->SetPosition(0.85, 0.1);  // 相对于renderer2的视口
```

### 19.2.5 多标量条布局的计算辅助函数

当需要在一个渲染器中布置多个标量条时，手动计算每个标量条的位置可能会很繁琐。下面是一个辅助函数，它将渲染器的右侧区域均匀分配给指定数量的垂直标量条：

```cpp
// 在渲染器右侧均匀布置多个垂直标量条
// renderer: 目标渲染器
// scalarBars: 标量条数组
// startX: 第一个标量条左边缘的归一化x坐标 (建议0.82)
// barWidth: 每个标量条的宽度 (建议0.12)
// minY: 最下方标量条的底部y坐标 (建议0.05)
// maxY: 最上方标量条的顶部y坐标 (建议0.95)
// gap: 标量条之间的间距 (建议0.03)

void LayoutMultipleScalarBars(
    std::vector<vtkSmartPointer<vtkScalarBarActor>>& scalarBars,
    double startX, double barWidth,
    double minY, double maxY, double gap)
{
    int n = static_cast<int>(scalarBars.size());
    if (n == 0) return;

    // 计算每个标量条的可用高度
    double totalGap = (n - 1) * gap;
    double totalHeight = maxY - minY - totalGap;
    double eachHeight = totalHeight / n;

    for (int i = 0; i < n; ++i)
    {
        // 自下而上排列
        double y = minY + i * (eachHeight + gap);
        scalarBars[i]->SetPosition(startX, y);
        scalarBars[i]->SetWidth(barWidth);
        scalarBars[i]->SetHeight(eachHeight);
    }
}
```

这个函数的使用示例：

```cpp
// 创建三个标量条，分别对应温度、压力和速度
std::vector<vtkSmartPointer<vtkScalarBarActor>> bars;
bars.push_back(tempScalarBar);
bars.push_back(pressureScalarBar);
bars.push_back(velocityScalarBar);

// 在渲染器右侧均匀布置
LayoutMultipleScalarBars(bars, 0.84, 0.12, 0.08, 0.92, 0.02);

// 全部添加到渲染器
for (auto& bar : bars)
{
    renderer->AddActor2D(bar);
}
```

### 19.2.6 自定义颜色条中的颜色段

如果你需要创建带有多段离散颜色的标量条（例如，针对分类数据的标量条），你可以通过`SetAnnotationTextEntry()`或自定义标量条的外观来实现。但对于大多数连续数据可视化场景，标准的`vtkScalarBarActor`就已经足够了。

---

## 19.3 方向标与坐标轴

方向标（Orientation Marker）是三维可视化最基本的标注元素之一。当用户旋转场景时，很容易失去对空间方向的感知——XYZ三个轴到底指向哪里？方向标通过始终显示坐标轴的方向来解决这个问题。

### 19.3.1 vtkOrientationMarkerWidget概述

`vtkOrientationMarkerWidget`是VTK中管理方向指示器的主要组件。它本质上是一个Widget（交互组件），负责在渲染窗口的一个角落显示一个小型的坐标轴指示器。当用户旋转场景时，这个小坐标轴也跟随旋转，始终指明世界坐标系的方向。

`vtkOrientationMarkerWidget`的工作机制：
1. 它持有一个Actor（通常是`vtkAxesActor`或`vtkAnnotatedCubeActor`）作为方向指示标记。
2. 它创建一个独立的小型Renderer，专门用于渲染这个标记。
3. 它将这个小型Renderer叠加在主渲染窗口的指定位置（通过`SetViewport()`控制）。
4. 它监听主Renderer的相机变化，并自动更新标记的朝向。

```cpp
#include <vtkOrientationMarkerWidget.h>
#include <vtkAxesActor.h>

// 步骤1：创建坐标轴Actor作为方向标记
vtkNew<vtkAxesActor> axesActor;
axesActor->SetTotalLength(2.0, 2.0, 2.0);   // 每个轴的长度
axesActor->SetShaftTypeToCylinder();         // 轴干用圆柱体表示
axesActor->SetCylinderRadius(0.05);          // 圆柱体半径
axesActor->SetConeRadius(0.15);              // 箭头圆锥半径
axesActor->SetConeResolution(16);            // 圆锥的边数
axesActor->SetAxisLabels(true);              // 显示X、Y、Z标签
axesActor->SetXAxisLabelText("X");
axesActor->SetYAxisLabelText("Y");
axesActor->SetZAxisLabelText("Z");

// 步骤2：创建OrientationMarkerWidget
vtkNew<vtkOrientationMarkerWidget> orientationWidget;
orientationWidget->SetOrientationMarker(axesActor);
orientationWidget->SetInteractor(renderWindowInteractor);

// 步骤3：设置位置（视口坐标）
orientationWidget->SetViewport(0.0, 0.0, 0.2, 0.2);
// 含义：方向标占据渲染窗口左下角，从左下角(0,0)到(0.2, 0.2)
// 即窗口宽度的20%，窗口高度的20%

// 步骤4：设置行为
orientationWidget->SetEnabled(true);          // 启用方向标
orientationWidget->SetInteractive(true);      // 允许用户与方向标交互
// 当SetInteractive(true)时，用户可以点击方向标来重置相机朝向
```

`SetViewport()`的四个参数定义了方向标在渲染窗口中的矩形区域（归一化坐标）。推荐的典型位置有：
- 左下角：`(0.0, 0.0, 0.2, 0.2)`
- 右下角：`(0.8, 0.0, 1.0, 0.2)`
- 左上角：`(0.0, 0.8, 0.2, 1.0)`

### 19.3.2 vtkAxesActor深度配置

`vtkAxesActor`是VTK中标准的坐标轴Actor，它由三个带箭头的轴组成（X=红色、Y=绿色、Z=蓝色，遵循传统的RGB-XYZ映射）。除了基本的轴长和标签设置外，它还支持以下高级配置：

```cpp
vtkNew<vtkAxesActor> axesActor;

// --- 轴长 ---
axesActor->SetTotalLength(1.5, 1.5, 1.5);     // X, Y, Z轴各自的总长度

// --- 轴干类型 ---
axesActor->SetShaftTypeToCylinder();           // 圆柱体轴干（默认）
axesActor->SetShaftTypeToLine();               // 线段轴干（更简洁）
axesActor->SetShaftTypeToUserDefined();        // 用户自定义

// --- 轴干尺寸 ---
axesActor->SetCylinderRadius(0.04);            // 圆柱体半径
axesActor->SetLineWidth(1.5);                  // 线型轴干的线宽

// --- 箭头设置 ---
axesActor->SetConeRadius(0.12);                // 箭头圆锥的底面半径
axesActor->SetConeResolution(20);              // 圆锥的边数（越大越光滑）
axesActor->SetTipTypeToCone();                 // 圆锥形箭头
// axesActor->SetTipTypeToSphere();            // 球形箭头

// --- 标签设置 ---
axesActor->SetAxisLabels(true);                // 是否显示轴标签
axesActor->SetXAxisLabelText("水平X");          // 自定义X轴标签文字
axesActor->SetYAxisLabelText("垂直Y");          // 自定义Y轴标签文字
axesActor->SetZAxisLabelText("深度Z");          // 自定义Z轴标签文字

// --- 轴的颜色 ---
// 默认: X=红色, Y=绿色, Z=蓝色（RGB-XYZ对应）
// 可以手动修改单个轴的各个部分的颜色：
// X轴: axesActor->GetXAxisShaftProperty()->SetColor(r, g, b);
//      axesActor->GetXAxisTipProperty()->SetColor(r, g, b);
// Y轴: axesActor->GetYAxisShaftProperty()->SetColor(r, g, b);
//      axesActor->GetYAxisTipProperty()->SetColor(r, g, b);
// Z轴: axesActor->GetZAxisShaftProperty()->SetColor(r, g, b);
//      axesActor->GetZAxisTipProperty()->SetColor(r, g, b);

// 自定义轴颜色示例：将Z轴改为黄色
axesActor->GetZAxisShaftProperty()->SetColor(1.0, 1.0, 0.0);
axesActor->GetZAxisTipProperty()->SetColor(1.0, 1.0, 0.0);

// --- 标签文本属性 ---
// X轴标签属性
axesActor->GetXAxisCaptionActor2D()->GetTextActor()->GetTextProperty()->SetFontSize(14);
axesActor->GetXAxisCaptionActor2D()->GetTextActor()->GetTextProperty()->SetBold(true);

// --- 整体姿态 ---
// 设置坐标轴的朝向（通过四元数或旋转角度）
// 默认情况下，X轴向右、Y轴向上、Z轴指向观察者
axesActor->SetOrientation(30, 20, 0);         // 绕X旋转30°，绕Y旋转20°
```

### 19.3.3 vtkAnnotatedCubeActor：带文字标签的方向立方体

`vtkAnnotatedCubeActor`是一种替代性的方向指示器。与`vtkAxesActor`不同，它显示的是一个小型的标注立方体——每个面标有文字标签（通常是"R/L"、"A/P"、"S/I"等解剖学术语，或者"+X/-X"等坐标标签）。

```cpp
#include <vtkAnnotatedCubeActor.h>
#include <vtkNamedColors.h>

vtkNew<vtkAnnotatedCubeActor> cubeActor;

// --- 设置六个面的文字标签 ---
// SetFaceText(faces) 设置所有六个面的文字
// 面的顺序: 1=X+, 2=X-, 3=Y+, 4=Y-, 5=Z+, 6=Z-
// 或使用vtkAnnotatedCubeActor中定义的枚举常量
cubeActor->SetXPlusFaceText("+X");            // X+面：右
cubeActor->SetXMinusFaceText("-X");           // X-面：左
cubeActor->SetYPlusFaceText("+Y");            // Y+面：上
cubeActor->SetYMinusFaceText("-Y");           // Y-面：下
cubeActor->SetZPlusFaceText("+Z");            // Z+面：前
cubeActor->SetZMinusFaceText("-Z");           // Z-面：后

// --- 外观设置 ---
// 获取立方体的外观属性
cubeActor->GetCubeProperty()->SetColor(0.8, 0.85, 0.9);  // 浅灰色立方体
cubeActor->GetCubeProperty()->SetOpacity(0.8);            // 轻微半透明

// 获取面文字的文本属性
cubeActor->GetTextEdgesProperty()->SetColor(0.0, 0.0, 0.0);  // 文字颜色：黑色
cubeActor->GetTextEdgesProperty()->SetLineWidth(2.0);

// 获取X+面文字属性
cubeActor->GetXPlusFaceProperty()->SetColor(1.0, 0.0, 0.0);   // 红色文字
cubeActor->GetXPlusFaceProperty()->SetFontSize(16);

// --- 使用OrientationMarkerWidget来管理 ---
vtkNew<vtkOrientationMarkerWidget> cubeWidget;
cubeWidget->SetOrientationMarker(cubeActor);
cubeWidget->SetInteractor(renderWindowInteractor);
cubeWidget->SetViewport(0.0, 0.0, 0.2, 0.2);
cubeWidget->SetEnabled(true);
cubeWidget->SetInteractive(true);
```

`vtkAnnotatedCubeActor`在医学可视化中特别受欢迎，因为它可以用解剖学术语来标注方向：上方(Superior)、下方(Inferior)、左方(Left)、右方(Right)、前方(Anterior)、后方(Posterior)。这种表达方式比单纯的XYZ坐标更符合领域专家的使用习惯。

### 19.3.4 同时使用坐标轴和立方体

你可以在同一个渲染窗口中同时使用`vtkAxesActor`（通过一个OrientationMarkerWidget）和`vtkAnnotatedCubeActor`（通过另一个OrientationMarkerWidget），将它们放置在不同角落。但通常来说，选择其中一种就足够了。推荐的做法是：

- 对于工程和物理仿真场景，使用`vtkAxesActor`（更精确地表示坐标轴）。
- 对于医学和生物可视化场景，使用`vtkAnnotatedCubeActor`（用术语替代XYZ）。
- 对于一般科学可视化，任意一种都可以。

---

## 19.4 文字标注

文字标注是可视化场景中传递信息的根本方式。VTK提供了多种文字标注组件，适用于不同的使用场景。本节将系统地介绍四种主要的文字标注工具。

### 19.4.1 vtkTextActor：屏幕空间固定文字

`vtkTextActor`是最基本的文字标注组件。它在屏幕空间中显示文字，位置以归一化视口坐标表示，不受相机运动的影响。它适合用作场景的标题、固定的数据说明或任何不需要随3D场景移动的文字信息。

```cpp
#include <vtkTextActor.h>
#include <vtkTextProperty.h>

// 创建屏幕空间文字
vtkNew<vtkTextActor> textActor;

// 设置文字内容
textActor->SetInput("有限元应力分析结果\n最大主应力分布图");

// 设置显示位置（归一化视口坐标）
// 支持SetDisplayPosition(x, y)和SetPosition(x, y)两种方式
textActor->SetDisplayPosition(10, 50);          // 像素坐标：距离左边缘10px，下边缘50px

// 或者使用归一化坐标
textActor->SetPosition(0.02, 0.95);             // 视口坐标：靠近左上角

// --- 文本属性设置 ---
vtkTextProperty* textProp = textActor->GetTextProperty();
textProp->SetFontSize(18);                       // 字体大小
textProp->SetFontFamilyToArial();                // 字体族
textProp->SetColor(0.0, 0.0, 0.0);              // 文字颜色（黑色）
textProp->SetBold(true);                         // 粗体
textProp->SetItalic(false);                      // 非斜体
textProp->SetShadow(true);                       // 开启阴影
textProp->SetShadowOffset(2, -2);                // 阴影偏移量
textProp->SetOpacity(1.0);                       // 完全不透明
textProp->SetJustificationToLeft();              // 左对齐
// 可选对齐方式:
//   SetJustificationToLeft()   -- 左对齐
//   SetJustificationToCentered() -- 居中对齐
//   SetJustificationToRight()  -- 右对齐

textProp->SetVerticalJustificationToTop();       // 顶部对齐
// 可选垂直对齐方式:
//   SetVerticalJustificationToTop()
//   SetVerticalJustificationToCentered()
//   SetVerticalJustificationToBottom()

// --- 文字背景设置 ---
textProp->SetBackgroundColor(1.0, 1.0, 0.9);     // 浅黄色背景
textProp->SetBackgroundOpacity(0.6);              // 背景半透明
textProp->SetFrame(true);                         // 在文字周围绘制边框
textProp->SetFrameColor(0.0, 0.0, 0.0);          // 边框颜色
textProp->SetFrameWidth(2);                       // 边框宽度

// --- 添加到渲染器 ---
renderer->AddActor2D(textActor);                  // 作为2D叠加层添加
```

`vtkTextActor`支持多行文字——在输入的字符串中使用`\n`来换行。你可以用它来创建结构化的信息显示。

### 19.4.2 vtkCaptionActor2D：带引线的跟随文字

`vtkCaptionActor2D`是一种特殊的文字标注，它由两部分组成：
1. **文字部分**：在屏幕上显示的文字。
2. **引线**（Leader Line）：一条从文字指向场景中某个3D点的线段。

当场景旋转或缩放时，引线的锚点始终指向3D世界中的固定位置，而文字部分保持在屏幕空间中。这非常适合标注数据中的特定位置（例如"最高温度点"、"入口边界"等）。

```cpp
#include <vtkCaptionActor2D.h>

// 创建引线文字标注
vtkNew<vtkCaptionActor2D> captionActor;

// 设置标注文字
captionActor->SetCaption("峰值应力\n450 MPa");

// 设置引线锚点（3D世界坐标）
captionActor->SetAttachmentPoint(peakStressX, peakStressY, peakStressZ);

// --- 文字部分的外观设置 ---
vtkTextProperty* capTextProp = captionActor->GetCaptionTextProperty();
capTextProp->SetFontSize(12);
capTextProp->SetFontFamilyToArial();
capTextProp->SetColor(1.0, 0.0, 0.0);              // 红色文字
capTextProp->SetBold(true);
capTextProp->SetShadow(true);
capTextProp->SetFrame(true);
capTextProp->SetFrameColor(0.0, 0.0, 0.0);

// --- 引线设置 ---
captionActor->SetLeader(true);                       // 显示引线
captionActor->SetLeaderGlyphSize(5.0);               // 引线端点的箭头大小
// SetLeaderGlyphSize() 控制引线在锚点端的箭头大小，
// 设置为0则没有箭头

// --- 边框与填充 ---
captionActor->SetPadding(4);                         // 文字矩形框的内边距
captionActor->SetBorder(true);                       // 显示文字边框
captionActor->SetThreeDimensionalLeader(false);      // 引线是2D还是3D
// true: 引线作为3D线绘制（可能被其他几何体遮挡）
// false: 引线在屏幕空间中绘制（始终可见，默认）

// --- 添加到渲染器 ---
// vtkCaptionActor2D 通过 AddActor() 添加（因为它有3D锚点）
renderer->AddActor(captionActor);
```

### 19.4.3 vtkCornerAnnotation：角标注

`vtkCornerAnnotation`专门用于在渲染窗口的四个角落显示文本信息。它的典型用途是显示数据的元信息——文件来源、时间步、数据范围、当前相机参数等。

```cpp
#include <vtkCornerAnnotation.h>

vtkNew<vtkCornerAnnotation> cornerAnnotation;

// 设置四个角落的文本
// SetText(int corner, const char* text)
// corner参数:
//   0 = 左下角 (Lower Left)
//   1 = 右下角 (Lower Right)
//   2 = 左上角 (Upper Left)
//   3 = 右上角 (Upper Right)

// 左下角：显示数据来源信息
cornerAnnotation->SetText(0, "数据来源: NASA SRTM v4\n分辨率: 90m");

// 右下角：显示时间步和范围
cornerAnnotation->SetText(1, "时间: t=3.5s\n高程范围: [-109, 8852] m");

// 左上角：显示场景名称
cornerAnnotation->SetText(2, "VTK地形可视化演示");

// 右上角：显示渲染信息
cornerAnnotation->SetText(3, "帧率: 未知\n相机制: 透视");

// --- 清除某个角落的文本 ---
cornerAnnotation->SetText(1, "");                    // 清空右下角

// --- 文本属性 ---
vtkTextProperty* cornerProp = cornerAnnotation->GetTextProperty();
cornerProp->SetFontSize(11);
cornerProp->SetFontFamilyToCourier();                // 等宽字体适合对齐
cornerProp->SetColor(0.9, 0.9, 0.9);                // 浅灰色文字
cornerProp->SetShadow(true);
cornerProp->SetShadowOffset(1, -1);

// --- 设置最大字体大小（防止在大窗口中文字过大） ---
cornerAnnotation->SetMaximumFontSize(14);

// --- 设置最小字体大小（防止在小窗口中文字过小） ---
cornerAnnotation->SetMinimumFontSize(8);

// --- 线性渐变与非线性字体缩放 ---
cornerAnnotation->SetNonlinearFontScaleFactor(0.35);
cornerAnnotation->SetLinearFontScaleFactor(0.015);

// --- 添加到渲染器 ---
renderer->AddActor2D(cornerAnnotation);
```

角标注的一个关键特性是，它的字体大小可以随窗口大小的变化而自动调整。通过`SetMaximumFontSize()`和`SetMinimumFontSize()`可以限制这种自适应行为的上界和下界，确保文字在任何窗口尺寸下都保持可读。

### 19.4.4 综合文字标注示例

下面是一个综合示例，展示如何在单个渲染器中组合使用多种文字标注类型：

```cpp
// --- 场景标题（vtkTextActor，固定在顶部居中） ---
vtkNew<vtkTextActor> titleActor;
titleActor->SetInput("三维标注系统演示");
vtkTextProperty* titleProp = titleActor->GetTextProperty();
titleProp->SetFontSize(22);
titleProp->SetFontFamilyToArial();
titleProp->SetColor(0.0, 0.0, 0.0);
titleProp->SetBold(true);
titleProp->SetJustificationToCentered();
titleProp->SetVerticalJustificationToTop();
titleActor->SetPosition(0.5, 0.98);                   // 顶部居中

// --- 角标注（vtkCornerAnnotation，左上角和右上角） ---
vtkNew<vtkCornerAnnotation> cornerAnnot;
cornerAnnot->SetText(2, "数据集: RandomHills\n标量: 高程值");   // 左上角
cornerAnnot->SetText(3, "相机状态: 用户交互模式");              // 右上角
cornerAnnot->GetTextProperty()->SetFontSize(10);
cornerAnnot->GetTextProperty()->SetColor(0.3, 0.3, 0.3);

// --- 数据统计信息（vtkTextActor，左下角固定位置） ---
vtkNew<vtkTextActor> statsActor;
statsActor->SetInput("高程统计:\n  最小值: 0.00 m\n"
                     "  最大值: 6.00 m\n  平均值: 3.00 m\n  标准差: 1.41 m");
statsActor->GetTextProperty()->SetFontSize(10);
statsActor->GetTextProperty()->SetFontFamilyToCourier();
statsActor->GetTextProperty()->SetColor(0.1, 0.1, 0.1);
statsActor->GetTextProperty()->SetBackgroundColor(0.95, 0.95, 0.95);
statsActor->GetTextProperty()->SetBackgroundOpacity(0.7);
statsActor->GetTextProperty()->SetFrame(true);
statsActor->GetTextProperty()->SetFrameColor(0.5, 0.5, 0.5);
statsActor->SetDisplayPosition(10, 80);               // 左下角，距左10px，距下80px

// --- 将所有标注添加到渲染器 ---
renderer->AddActor2D(titleActor);
renderer->AddActor2D(cornerAnnot);
renderer->AddActor2D(statsActor);
```

### 19.4.5 字体选择与中文支持

在VTK中使用中文字符串标注时，需要确保渲染后端配置了支持中文的字体。VTK依赖FreeType进行字体渲染，中文字体需要在系统字体路径中可用，或者通过`vtkFreeTypeTools`显式指定字体文件。

```cpp
#include <vtkFreeTypeTools.h>

// 显式设置中文字体文件（Windows示例）
textProp->SetFontFamily(VTK_FONT_FILE);
textProp->SetFontFile("C:/Windows/Fonts/msyh.ttc");   // 微软雅黑
// 或
textProp->SetFontFile("C:/Windows/Fonts/simsun.ttc");  // 宋体
// 或
textProp->SetFontFile("C:/Windows/Fonts/simhei.ttf");  // 黑体

// 在Linux/macOS上：
// textProp->SetFontFile("/usr/share/fonts/truetype/droid/DroidSansFallbackFull.ttf");
// 或使用Noto Sans CJK等开放字体
```

没有正确配置中文字体时，中文字符可能显示为方框或空白。如果你主要在英文环境中工作，建议在代码中使用ASCII字符并配合数字来表示需要的信息；如果需要中文标注，务必在部署环境中检查字体可用性。

---

## 19.5 比例尺与图例

比例尺和图例是两类重要的辅助标注组件，它们帮助观察者理解场景中的空间尺度和数据对象身份。

### 19.5.1 vtkLegendScaleActor：自动比例尺

`vtkLegendScaleActor`是一个二合一的标注组件：它同时显示**水平比例尺**和**垂直比例尺**（在渲染窗口的底部和右侧），为观察者提供场景的物理尺度参考。

```cpp
#include <vtkLegendScaleActor.h>

vtkNew<vtkLegendScaleActor> legendScale;

// --- 基本设置 ---
legendScale->SetLabelModeToDistance();               // 按距离显示刻度
// 可选模式:
//   SetLabelModeToDistance()  -- 显示物理距离（默认）
//   SetLabelModeToCoordinates() -- 显示坐标值

legendScale->SetNumberOfTicks(4);                    // 刻度标记数量

// --- 刻度方向 ---
legendScale->RightAxisVisibilityOn();                // 显示右侧（垂直）比例尺
legendScale->TopAxisVisibilityOn();                  // 显示顶部（水平）比例尺
legendScale->LeftAxisVisibilityOff();                // 隐藏左侧比例尺
legendScale->BottomAxisVisibilityOff();              // 隐藏底部比例尺

// --- 添加到渲染器 ---
// vtkLegendScaleActor 通过 AddActor2D() 添加
// 它自动将自己定位在渲染窗口的边缘
renderer->AddActor2D(legendScale);
```

`vtkLegendScaleActor`的一个重要特性是它的**自动缩放**：当场景缩放时，比例尺上的刻度标记会自动调整，避免过密或过疏。这意味着你不需要为不同的缩放级别手动调整刻度——它会自动选择合适的间距。

然而，它有两个局限性需要了解：
1. 它假定世界坐标系中的距离单位与物理单位一致（它无法知道你的数据使用的是什么单位，1个世界单位始终显示为1）。
2. 它的外观定制选项相对有限，不支持自定义字体大小和颜色（在VTK 9.5.2中）。

### 19.5.2 vtkLegendBoxActor：图例框

当场景中包含多个数据Actor时，图例（Legend Box）帮助观察者区分不同的数据对象。`vtkLegendBoxActor`允许你创建一个包含颜色块和对应文字的图例框。

```cpp
#include <vtkLegendBoxActor.h>

vtkNew<vtkLegendBoxActor> legendBox;

// --- 每个条目由一个颜色块和一个文本标签组成 ---
// AddEntry(i, icon, color, text, colorStr)
// icon: 图例条目的图标类型
//    1 = 实心颜色块
//    2 = 线型图标
// 注意：在VTK 9.5.2中，推荐使用SetEntry()方法

// --- 设置图例条目的方法 ---
// 方法1：逐个添加条目（索引从0开始）
int entryIdx = 0;

// 添加"温度"条目（红色块）
vtkNew<vtkSphereSource> dummySphere;   // 需要一个虚拟的Symbol用于颜色
// 实际而言，推荐直接设置颜色矩形：
legendBox->SetEntry(entryIdx++,
    nullptr,               // icon (nullptr使用默认颜色矩形)
    "温度场 (Temperature)", // 文字标签
    VTK_COLOR_RED);         // 颜色字符串

// 添加"压力"条目（蓝色块）
legendBox->SetEntry(entryIdx++,
    nullptr,
    "压力场 (Pressure)",
    VTK_COLOR_BLUE);

// --- 条目数量 ---
legendBox->SetNumberOfEntries(entryIdx);

// --- 外观设置 ---
legendBox->SetBorder(true);                          // 显示边框
legendBox->SetBox(true);                             // 每个条目周围显示框线
legendBox->SetPadding(6);                            // 内边距
legendBox->SetBackgroundColor(1.0, 1.0, 1.0);       // 白色背景
legendBox->SetEntryTextProperty(textProp);           // 文字属性

// --- 添加到渲染器 ---
renderer->AddActor2D(legendBox);                     // 作为2D叠加层
```

在实际使用中，`vtkLegendBoxActor`的使用比标量条要少。这是因为在大多数科学可视化场景中，每个数据集都有自己独立的颜色映射和标量条，图例的需求可以通过标量条来满足。`vtkLegendBoxActor`更适合于展示来自多个不同数据源的、使用单一颜色（而非颜色映射）表示的Actor。

### 19.5.3 创建自定义比例尺

如果自动比例尺的外观音准不能满足你的需求，你可以通过手动创建线段和文字标注来实现完全自定义的比例尺。下面是一个简化的自定义比例尺示例：

```cpp
// 创建自定义比例尺的辅助代码
// 使用 vtkAxisActor2D 或手动绘制线条

// 方案：使用 vtkPolyData + vtkPolyDataMapper2D 绘制自定义比例尺条
// 这在需要极高度自定义外观的场景下有用。
//
// 步骤概要：
// 1. 创建表示比例尺线条的 vtkPolyData（使用 vtkPoints + vtkCellArray）
// 2. 使用 vtkPolyDataMapper2D 将其映射为2D可渲染图元
// 3. 使用 vtkActor2D 添加到渲染器
// 4. 手动创建 vtkTextActor 用于刻度标签
//
// 由于代码较长（~60-80行），此处仅给出策略概要。
// 对于大多数场景，vtkLegendScaleActor已经足够。
```

---

## 19.6 代码示例：完整标注系统

本节提供一个完整的C++程序（约300行），展示本章涵盖的所有核心标注组件。程序创建一个具有高度变化的三维地形表面，并为其配备全套标注系统。

### 19.6.1 程序概述

本演示程序创建以下场景和标注：

**场景内容：**
1. 使用`vtkParametricRandomHills`生成地形曲面，以高程作为标量值。
2. 使用彩虹色阶（`vtkColorTransferFunction`）进行颜色映射。
3. 程序化生成一个平面用作"海平面"参考面。

**标注系统（完整的六层信息叠加）：**
1. **标量条**（`vtkScalarBarActor`）——自定义样式的垂直颜色图例，展示高程色阶。
2. **方向标**（`vtkOrientationMarkerWidget` + `vtkAxesActor`）——左下角的方向指示器。
3. **角标注**（`vtkCornerAnnotation`）——右上角显示场景元信息。
4. **场景标题**（`vtkTextActor`）——顶部居中的主标题。
5. **数据统计信息**（`vtkTextActor`）——左下角的min/max/mean统计。
6. **引线文字标注**（`vtkCaptionActor2D`）——指向地形最高点的引线标注。

### 19.6.2 完整C++代码

```cpp
// ============================================================================
// Chapter 19: 3D Annotation and Display Components
// File: AnnotationDemo.cxx
// Description: A comprehensive demonstration of VTK annotation system:
//              scalar bar, orientation marker, corner annotation, text actors,
//              caption with leader line, and custom styling.
//              Creates a terrain surface visualization with a complete
//              annotation layer.
// VTK Version: 9.5.2
// ============================================================================

#include <vtkActor.h>
#include <vtkAxesActor.h>
#include <vtkCamera.h>
#include <vtkCaptionActor2D.h>
#include <vtkCleanPolyData.h>
#include <vtkColorTransferFunction.h>
#include <vtkCornerAnnotation.h>
#include <vtkElevationFilter.h>
#include <vtkFloatArray.h>
#include <vtkLookupTable.h>
#include <vtkNamedColors.h>
#include <vtkNew.h>
#include <vtkOrientationMarkerWidget.h>
#include <vtkParametricFunctionSource.h>
#include <vtkParametricRandomHills.h>
#include <vtkPlaneSource.h>
#include <vtkPointData.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkScalarBarActor.h>
#include <vtkSmartPointer.h>
#include <vtkSphereSource.h>
#include <vtkTextActor.h>
#include <vtkTextProperty.h>

#include <cmath>
#include <sstream>
#include <string>

// ============================================================================
// Helper: Create the terrain surface
// ============================================================================
vtkSmartPointer<vtkPolyData> CreateTerrainSurface()
{
    // Create a random hills parametric surface
    vtkNew<vtkParametricRandomHills> hillsFunc;
    hillsFunc->SetHillXVariance(2.5);
    hillsFunc->SetHillYVariance(2.5);
    hillsFunc->SetAmplitude(3.0);
    hillsFunc->SetAllowRandomGeneration(true);
    hillsFunc->SetRandomSeed(42);
    hillsFunc->SetNumberOfHills(8);

    // Generate surface as polydata
    vtkNew<vtkParametricFunctionSource> source;
    source->SetParametricFunction(hillsFunc);
    source->SetUResolution(100);
    source->SetVResolution(100);
    source->SetWResolution(100);
    source->SetScalarModeToZ();              // Use Z (height) as scalar value
    source->Update();

    // Clean the polydata to merge duplicate points
    vtkNew<vtkCleanPolyData> cleaner;
    cleaner->SetInputConnection(source->GetOutputPort());
    cleaner->Update();

    return cleaner->GetOutput();
}

// ============================================================================
// Helper: Compute scalar statistics (min, max, mean) from polydata
// ============================================================================
void ComputeScalarStats(vtkPolyData* data,
    double& minVal, double& maxVal, double& meanVal)
{
    vtkDataArray* scalars = data->GetPointData()->GetScalars();
    if (!scalars)
    {
        minVal = maxVal = meanVal = 0.0;
        return;
    }

    double range[2];
    scalars->GetRange(range);
    minVal = range[0];
    maxVal = range[1];

    // Compute mean
    double sum = 0.0;
    vtkIdType n = scalars->GetNumberOfTuples();
    for (vtkIdType i = 0; i < n; ++i)
    {
        sum += scalars->GetComponent(i, 0);
    }
    meanVal = sum / static_cast<double>(n);
}

// ============================================================================
// Helper: Find the 3D point with the maximum scalar value
// ============================================================================
void FindMaxScalarPoint(vtkPolyData* data,
    double& maxX, double& maxY, double& maxZ, double& maxVal)
{
    vtkDataArray* scalars = data->GetPointData()->GetScalars();
    vtkPoints* points = data->GetPoints();
    if (!scalars || !points)
    {
        maxX = maxY = maxZ = maxVal = 0.0;
        return;
    }

    maxVal = -std::numeric_limits<double>::max();
    vtkIdType maxIdx = 0;
    vtkIdType n = scalars->GetNumberOfTuples();
    for (vtkIdType i = 0; i < n; ++i)
    {
        double val = scalars->GetComponent(i, 0);
        if (val > maxVal)
        {
            maxVal = val;
            maxIdx = i;
        }
    }

    double pt[3];
    points->GetPoint(maxIdx, pt);
    maxX = pt[0];
    maxY = pt[1];
    maxZ = pt[2];
}

// ============================================================================
// Main
// ============================================================================
int main(int argc, char* argv[])
{
    // ========================================================================
    // 1. Create the terrain surface data
    // ========================================================================
    auto surface = CreateTerrainSurface();

    // Get data range
    double dataRange[2];
    surface->GetScalarRange(dataRange);
    std::cout << "Terrain scalar range: [" << dataRange[0]
              << ", " << dataRange[1] << "]" << std::endl;

    // Compute statistics
    double minVal, maxVal, meanVal;
    ComputeScalarStats(surface, minVal, maxVal, meanVal);
    std::cout << "Stats -- Min: " << minVal << ", Max: " << maxVal
              << ", Mean: " << meanVal << std::endl;

    // ========================================================================
    // 2. Create the color transfer function (terrain elevation colormap)
    // ========================================================================
    vtkNew<vtkColorTransferFunction> ctf;
    ctf->AddRGBPoint(0.0,   0.2, 0.3, 0.7);    // Deep water: dark blue
    ctf->AddRGBPoint(0.15,  0.3, 0.5, 0.9);    // Shallow water: medium blue
    ctf->AddRGBPoint(0.3,   0.2, 0.7, 0.3);    // Lowland: green
    ctf->AddRGBPoint(0.5,   0.5, 0.8, 0.2);    // Midland: yellow-green
    ctf->AddRGBPoint(0.7,   0.7, 0.7, 0.3);    // Hills: tan
    ctf->AddRGBPoint(0.85,  0.6, 0.4, 0.2);    // Mountains: brown
    ctf->AddRGBPoint(1.0,   1.0, 1.0, 1.0);    // Snow peak: white
    ctf->SetColorSpaceToRGB();

    // ========================================================================
    // 3. Setup the terrain mapper and actor
    // ========================================================================
    vtkNew<vtkPolyDataMapper> terrainMapper;
    terrainMapper->SetInputData(surface);
    terrainMapper->SetLookupTable(ctf);
    terrainMapper->SetScalarRange(dataRange);
    terrainMapper->SetScalarVisibility(true);

    vtkNew<vtkActor> terrainActor;
    terrainActor->SetMapper(terrainMapper);
    terrainActor->GetProperty()->SetSpecular(0.3);
    terrainActor->GetProperty()->SetSpecularPower(20);
    terrainActor->GetProperty()->SetAmbient(0.15);
    terrainActor->GetProperty()->SetDiffuse(0.8);

    // ========================================================================
    // 4. Create a reference plane (sea level)
    // ========================================================================
    vtkNew<vtkPlaneSource> planeSource;
    planeSource->SetOrigin(-5.0, -5.0, 0.0);
    planeSource->SetPoint1(5.0, -5.0, 0.0);
    planeSource->SetPoint2(-5.0, 5.0, 0.0);
    planeSource->SetXResolution(1);
    planeSource->SetYResolution(1);
    planeSource->Update();

    vtkNew<vtkPolyDataMapper> planeMapper;
    planeMapper->SetInputConnection(planeSource->GetOutputPort());

    vtkNew<vtkActor> planeActor;
    planeActor->SetMapper(planeMapper);
    planeActor->GetProperty()->SetColor(0.3, 0.5, 0.8);
    planeActor->GetProperty()->SetOpacity(0.3);
    planeActor->GetProperty()->SetAmbient(0.5);
    planeActor->GetProperty()->SetSpecular(0.0);

    // ========================================================================
    // 5. Create the renderer
    // ========================================================================
    vtkNew<vtkRenderer> renderer;
    renderer->AddActor(terrainActor);
    renderer->AddActor(planeActor);
    renderer->SetBackground(0.08, 0.08, 0.15);         // Dark navy background
    renderer->SetBackground2(0.15, 0.15, 0.25);
    renderer->SetGradientBackground(true);

    // Enable depth peeling for transparent plane
    renderer->SetUseDepthPeeling(true);
    renderer->SetMaximumNumberOfPeels(6);
    renderer->SetOcclusionRatio(0.0);

    // Camera: isometric-like view
    vtkCamera* camera = renderer->GetActiveCamera();
    camera->SetPosition(-3.0, -8.0, 5.0);
    camera->SetFocalPoint(0.0, 0.0, 1.8);
    camera->SetViewUp(0.0, 0.0, 1.0);
    camera->SetClippingRange(0.5, 50.0);
    renderer->ResetCamera();

    // ========================================================================
    // 6. ANNOTATION LAYER
    // ========================================================================

    // --- 6a. Scalar Bar (标量条) ---
    vtkNew<vtkScalarBarActor> scalarBar;
    scalarBar->SetLookupTable(ctf);
    scalarBar->SetTitle("高程 (m)");
    scalarBar->SetNumberOfLabels(8);
    scalarBar->SetLabelFormat("%.1f");
    scalarBar->SetOrientationToVertical();
    scalarBar->SetTextPositionToPrecedeScalarBar();
    scalarBar->SetPosition(0.85, 0.12);
    scalarBar->SetWidth(0.10);
    scalarBar->SetHeight(0.65);
    scalarBar->SetMaximumWidthInPixels(100);
    scalarBar->SetMaximumHeightInPixels(600);
    scalarBar->SetBarRatio(0.4);
    scalarBar->SetDrawBorder(true);
    scalarBar->SetBorderColor(0.8, 0.8, 0.8);
    scalarBar->SetBackgroundColor(0.1, 0.1, 0.15);
    scalarBar->SetBackgroundOpacity(0.5);
    scalarBar->SetDrawBackground(true);

    // Title text properties
    scalarBar->GetTitleTextProperty()->SetFontSize(14);
    scalarBar->GetTitleTextProperty()->SetFontFamilyToArial();
    scalarBar->GetTitleTextProperty()->SetColor(1.0, 1.0, 1.0);
    scalarBar->GetTitleTextProperty()->SetBold(true);
    scalarBar->GetTitleTextProperty()->SetShadow(false);

    // Label text properties
    scalarBar->GetLabelTextProperty()->SetFontSize(11);
    scalarBar->GetLabelTextProperty()->SetFontFamilyToArial();
    scalarBar->GetLabelTextProperty()->SetColor(0.9, 0.9, 0.9);
    scalarBar->GetLabelTextProperty()->SetBold(false);

    renderer->AddActor2D(scalarBar);

    // --- 6b. Orientation Marker (方向标) ---
    vtkNew<vtkAxesActor> axesActor;
    axesActor->SetTotalLength(1.5, 1.5, 1.5);
    axesActor->SetShaftTypeToCylinder();
    axesActor->SetCylinderRadius(0.04);
    axesActor->SetConeRadius(0.12);
    axesActor->SetConeResolution(16);
    axesActor->SetAxisLabels(true);
    axesActor->SetXAxisLabelText("X");
    axesActor->SetYAxisLabelText("Y");
    axesActor->SetZAxisLabelText("Z");

    // Customize Z-axis color for better visibility against dark background
    axesActor->GetZAxisShaftProperty()->SetColor(1.0, 1.0, 0.3);
    axesActor->GetZAxisTipProperty()->SetColor(1.0, 1.0, 0.3);

    // --- 6c. Scene Title (场景标题) ---
    vtkNew<vtkTextActor> titleActor;
    titleActor->SetInput("地形可视化 - 完整标注系统演示");
    vtkTextProperty* titleProp = titleActor->GetTextProperty();
    titleProp->SetFontSize(20);
    titleProp->SetFontFamilyToArial();
    titleProp->SetColor(1.0, 1.0, 1.0);
    titleProp->SetBold(true);
    titleProp->SetShadow(true);
    titleProp->SetShadowOffset(2, -2);
    titleProp->SetJustificationToCentered();
    titleProp->SetVerticalJustificationToTop();
    titleActor->SetPosition(0.5, 0.97);

    renderer->AddActor2D(titleActor);

    // --- 6d. Corner Annotation (角标注 - 数据信息) ---
    vtkNew<vtkCornerAnnotation> cornerAnnot;
    cornerAnnot->SetText(2, "VTK 9.5.2 教程\n第十九章: 标注系统");  // 左上角
    cornerAnnot->SetText(3, "渲染模式: 表面+半透明参考面\n"
                            "颜色映射: 地形高程色阶");               // 右上角
    cornerAnnot->GetTextProperty()->SetFontSize(10);
    cornerAnnot->GetTextProperty()->SetFontFamilyToArial();
    cornerAnnot->GetTextProperty()->SetColor(0.8, 0.8, 0.8);
    cornerAnnot->SetMaximumFontSize(12);
    cornerAnnot->SetMinimumFontSize(8);

    renderer->AddActor2D(cornerAnnot);

    // --- 6e. Statistics Text (数据统计文字) ---
    std::ostringstream statsStream;
    statsStream << "数据统计:\n";
    statsStream << "  最小值: " << std::fixed << std::setprecision(2)
                << minVal << " m\n";
    statsStream << "  最大值: " << maxVal << " m\n";
    statsStream << "  平均值: " << meanVal << " m\n";
    statsStream << "  数据点: " << surface->GetNumberOfPoints();

    vtkNew<vtkTextActor> statsActor;
    statsActor->SetInput(statsStream.str().c_str());
    vtkTextProperty* statsProp = statsActor->GetTextProperty();
    statsProp->SetFontSize(11);
    statsProp->SetFontFamilyToCourier();
    statsProp->SetColor(1.0, 1.0, 1.0);
    statsProp->SetBackgroundColor(0.05, 0.05, 0.15);
    statsProp->SetBackgroundOpacity(0.7);
    statsProp->SetFrame(true);
    statsProp->SetFrameColor(0.5, 0.5, 0.5);
    statsProp->SetFrameWidth(1);
    statsProp->SetJustificationToLeft();
    statsProp->SetVerticalJustificationToBottom();
    statsActor->SetDisplayPosition(12, 110);

    renderer->AddActor2D(statsActor);

    // --- 6f. Caption with Leader Line (引线文字标注，指向最高点) ---
    double maxPtX, maxPtY, maxPtZ, maxScalar;
    FindMaxScalarPoint(surface, maxPtX, maxPtY, maxPtZ, maxScalar);

    std::ostringstream captionStream;
    captionStream << "最高点\n" << std::fixed << std::setprecision(2)
                  << maxScalar << " m";

    vtkNew<vtkCaptionActor2D> captionActor;
    captionActor->SetCaption(captionStream.str().c_str());
    captionActor->SetAttachmentPoint(maxPtX, maxPtY, maxPtZ);
    captionActor->SetLeader(true);
    captionActor->SetLeaderGlyphSize(8.0);
    captionActor->SetThreeDimensionalLeader(false);
    captionActor->SetPadding(4);
    captionActor->SetBorder(true);

    vtkTextProperty* captionTextProp = captionActor->GetCaptionTextProperty();
    captionTextProp->SetFontSize(12);
    captionTextProp->SetFontFamilyToArial();
    captionTextProp->SetColor(1.0, 1.0, 0.3);          // 亮黄色
    captionTextProp->SetBold(true);
    captionTextProp->SetShadow(true);
    captionTextProp->SetShadowOffset(1, -1);
    captionTextProp->SetBackgroundColor(0.1, 0.1, 0.1);
    captionTextProp->SetBackgroundOpacity(0.8);
    captionTextProp->SetFrame(true);
    captionTextProp->SetFrameColor(1.0, 1.0, 0.3);
    captionTextProp->SetFrameWidth(1);
    captionTextProp->SetJustificationToCentered();

    renderer->AddActor(captionActor);

    // ========================================================================
    // 7. Render Window
    // ========================================================================
    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(1200, 800);
    renderWindow->SetMultiSamples(8);
    renderWindow->SetWindowName(
        "Chapter 19: 3D Annotation and Display Components");
    renderWindow->AddRenderer(renderer);

    // ========================================================================
    // 8. Interactor + Orientation Marker Widget
    // ========================================================================
    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    // Orientation marker widget must be created AFTER the interactor
    vtkNew<vtkOrientationMarkerWidget> orientationWidget;
    orientationWidget->SetOrientationMarker(axesActor);
    orientationWidget->SetInteractor(interactor);
    orientationWidget->SetViewport(0.0, 0.0, 0.18, 0.18);   // 左下角
    orientationWidget->SetEnabled(true);
    orientationWidget->SetInteractive(true);

    // ========================================================================
    // 9. Render and Start
    // ========================================================================
    renderWindow->Render();
    interactor->Initialize();
    interactor->Start();

    return 0;
}
```

### 19.6.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

# ============================================================================
# Chapter 19: 3D Annotation and Display Components
# File: CMakeLists.txt
# ============================================================================

project(AnnotationDemo)

# --------------------------------------------------------------------------
# Find VTK 9.5.2
# --------------------------------------------------------------------------
find_package(VTK 9.5 REQUIRED COMPONENTS
    CommonCore
    CommonColor
    CommonComputationalGeometry
    CommonDataModel
    CommonExecutionModel
    CommonMath
    CommonSystem
    CommonTransforms
    FiltersCore
    FiltersGeneral
    FiltersSources
    InteractionStyle
    InteractionWidgets
    RenderingAnnotation
    RenderingCore
    RenderingFreeType
    RenderingOpenGL2
)

if(NOT VTK_FOUND)
    message(FATAL_ERROR
        "VTK 9.5 not found. Please set -DVTK_DIR=/path/to/vtk-9.5.2/build")
endif()

message(STATUS "VTK version: ${VTK_VERSION}")
message(STATUS "VTK_DIR: ${VTK_DIR}")

# --------------------------------------------------------------------------
# Executable
# --------------------------------------------------------------------------
add_executable(AnnotationDemo AnnotationDemo.cxx)

target_link_libraries(AnnotationDemo PRIVATE
    VTK::CommonCore
    VTK::CommonColor
    VTK::CommonComputationalGeometry
    VTK::CommonDataModel
    VTK::CommonExecutionModel
    VTK::CommonMath
    VTK::CommonSystem
    VTK::CommonTransforms
    VTK::FiltersCore
    VTK::FiltersGeneral
    VTK::FiltersSources
    VTK::InteractionStyle
    VTK::InteractionWidgets
    VTK::RenderingAnnotation
    VTK::RenderingCore
    VTK::RenderingFreeType
    VTK::RenderingOpenGL2
)

# --------------------------------------------------------------------------
# C++ Standard
# --------------------------------------------------------------------------
target_compile_features(AnnotationDemo PRIVATE cxx_std_17)

# --------------------------------------------------------------------------
# Platform-specific settings
# --------------------------------------------------------------------------
if(WIN32)
    target_compile_definitions(AnnotationDemo PRIVATE
        _CRT_SECURE_NO_WARNINGS
        NOMINMAX
    )
endif()

if(MSVC)
    target_compile_options(AnnotationDemo PRIVATE
        /bigobj
    )
endif()
```

### 19.6.4 编译与运行

**编译步骤：**

```bash
# 1. 创建构建目录
mkdir build && cd build

# 2. 配置CMake（请将路径替换为你本地的VTK安装路径）
cmake .. -DVTK_DIR=/path/to/vtk-9.5.2/build

# 3. 编译
cmake --build . --config Release

# 4. 运行
./AnnotationDemo        # Linux/macOS
AnnotationDemo.exe      # Windows
```

**交互说明：**

- 鼠标左键拖拽：旋转场景
- 鼠标滚轮：缩放
- 鼠标中键拖拽：平移
- R键：重置相机
- 点击左下角的方向标：将相机重置到坐标轴的默认视角

### 19.6.5 预期效果

运行程序后，你将看到一个包含地形曲面可视化的窗口，叠加了完整的标注系统：

- **场景中心**：随机丘陵地形曲面，使用从上到下的蓝色(低)到白色(高)的地形高程色阶。半透明的蓝色参考平面位于z=0处作为"海平面"。
- **右侧**：垂直方向的标量条，白色标题"高程 (m)"位于顶部，数值标签清晰可读。
- **左下角**：白色的方向坐标轴标记（X=红色、Y=绿色、Z=黄色），随场景旋转同步更新方向。点击它可以重置视角。
- **左上角**：灰色小字显示"VTK 9.5.2 教程 / 第十九章: 标注系统"。
- **右上角**：灰色小字显示渲染模式和颜色映射信息。
- **顶部中央**：白色粗体大标题"地形可视化 - 完整标注系统演示"，带阴影效果。
- **左下角统计面板**：深色背景半透明的统计信息框，显示最小值、最大值、平均值和数据点总数。
- **最高点标注**：亮黄色引线从文字指向地形曲面上的最高点位置，文字显示该点的精确高程值。

通过旋转和缩放场景，你可以观察到：标量条、标题文字、角标注和统计面板保持固定不变（它们是屏幕空间的2D标注），而引线标注的锚点始终跟随地形上的最高点，方向标随场景旋转同步更新。这正是二维标注和三维标注之间区别的直观体现。

---

## 19.7 本章小结

本章系统地介绍了VTK中的三维标注与交互组件系统。以下是本章的核心要点：

### 核心概念

1. **标注作为信息层**：标注是叠加在三维数据可视化之上的辅助信息层，用于增强观察者对场景的理解，而不修改底层数据或可视化管线。

2. **二维标注 vs 三维标注**：二维标注（`AddActor2D()`）位于屏幕空间，位置固定，不随场景变换而改变；三维标注（`AddActor()`）位于世界空间，随场景一起旋转、缩放和平移。

3. **Widget管理**：部分标注组件（如方向标、可交互标量条）通过Widget机制管理，提供了交互性（鼠标点击、拖拽等）。

### 关键类与API速查

| 类 | 主要用途 | 添加方式 | 坐标空间 |
|----|---------|---------|---------|
| `vtkScalarBarActor` | 颜色图例/标量条 | `AddActor2D()` | 屏幕空间 |
| `vtkAxesActor` | 三维坐标轴 | OrientationMarkerWidget | 世界空间 |
| `vtkAnnotatedCubeActor` | 标注方向立方体 | OrientationMarkerWidget | 世界空间 |
| `vtkOrientationMarkerWidget` | 管理方向指示标记 | Widget(SetEnabled) | 混合 |
| `vtkTextActor` | 屏幕固定文字 | `AddActor2D()` | 屏幕空间 |
| `vtkCaptionActor2D` | 带引线的跟随文字标注 | `AddActor()` | 混合(锚点3D+文字2D) |
| `vtkCornerAnnotation` | 四角文字标注 | `AddActor2D()` | 屏幕空间 |
| `vtkLegendScaleActor` | 自动比例尺 | `AddActor2D()` | 屏幕空间 |
| `vtkLegendBoxActor` | 图例框 | `AddActor2D()` | 屏幕空间 |

### 标量条高级配置要点

- `SetTextPositionToPrecedeScalarBar()`和`SetTextPositionToSucceedScalarBar()`控制标签在颜色条的左侧还是右侧。
- `SetMaximumWidthInPixels()`和`SetMaximumHeightInPixels()`防止标量条在窗口放大时过度膨胀。
- `SetBarRatio()`控制颜色条部分在总宽度中的比例。
- 使用`GetTitleTextProperty()`和`GetLabelTextProperty()`分别控制标题和标签的字体样式。
- 多标量条布局需要仔细计算每个标量条的位置和尺寸，可使用辅助函数进行管理。

### 方向标配置要点

- `vtkOrientationMarkerWidget`负责管理方向标，通过`SetViewport()`定义其在窗口中的矩形区域。
- `vtkAxesActor`提供标准的RGB-XYZ坐标轴，支持自定义轴长、轴干类型（圆柱/线）、箭头形状和轴标签。
- `vtkAnnotatedCubeActor`提供带文字标签的方向立方体，特别适合医学可视化中的解剖学方向标注。
- `SetInteractive(true)`允许用户点击方向标来重置相机视角。

### 文字标注选择指南

- **场景标题/固定说明**：使用`vtkTextActor`（屏幕空间固定文字）。
- **数据中特定位置的标注**：使用`vtkCaptionActor2D`（带引线的跟随文字）。
- **展示元信息和状态**：使用`vtkCornerAnnotation`（四角标注）。
- **需要交互的标注**：使用`vtkCaptionWidget`（可拖拽的引线标注）。

### 综合演示

19.6节提供的一个完整程序演示了六种标注组件的协同使用。通过运行这个程序，你可以直观地理解各标注组件的工作方式和视觉效果。建议在运行程序时：
- 旋转场景观察方向标的同步更新；
- 注意哪些标注保持固定（2D）而哪些跟随场景变化（3D/混合）；
- 尝试点击左下角的方向标来重置相机；
- 放大和缩小窗口观察标量条的像素限制和角标注的字体缩放行为。

### 继续学习建议

标注系统是创建专业级、出版物级别可视化作品的关键组成部分。掌握了本章的内容后，你可以将标注技术与以下主题结合使用：

- **第十二章（颜色映射）**和**第十三章（高级渲染）**的知识，在数据可视化之上叠加完整的标注信息。
- **Widget交互系统**——进一步探索VTK中丰富的交互组件（如`vtkDistanceWidget`、`vtkAngleWidget`、`vtkImageTracerWidget`等），实现交互式的测量和标注。
- **动画与时序数据**——在时变数据可视化中动态更新角标注的时间信息。
- **导出与出版**——在生成高分辨率截图或出版用图片时，确保标注的字体大小和布局是经过优化的。

一个设计良好的标注系统可以让观察者在不阅读任何外部文档的情况下，就能理解可视化的内容。这是将数据可视化从"探索工具"提升为"交流媒介"的关键一步。
