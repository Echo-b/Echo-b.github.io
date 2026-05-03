# 第十三章 高级渲染技术 (Advanced Rendering)

## 本章导读

在前面几章中，你已经学会了如何构建各种数据类型、配置Mapper、应用颜色映射来为数据赋予有意义的视觉编码。你也掌握了渲染管道的基本概念——Actor、Renderer、RenderWindow、Interactor之间的协作关系，以及如何设置Camera和Light。然而，到目前为止，我们使用的渲染配置是相当基础的：所有Actor都使用默认的白色材质，光照系统只有一个默认的头灯，纹理和透明度等高级特性尚未涉及。

本章将带你深入VTK的渲染系统，系统地讲解四大高级渲染技术：

1. **材质属性（vtkProperty）**——从简单的单一颜色到完整的Phong光照模型参数控制。你将学习漫反射、环境光、镜面反射、自发光等物理启发的外观参数，以及点/线框/表面的表示模式切换。
2. **光照模型（vtkLight）**——VTK的光照系统支持多种光源类型，包括头灯（Headlight）、相机灯（CameraLight）和场景灯（SceneLight）。你将学习如何配置多光源场景，包括经典的三点式布光（三点照明）用于专业级的科学可视化展示。
3. **纹理映射（vtkTexture）**——为几何表面贴上图像纹理，极大地增强视觉丰富度。从加载外部图像文件到程序化生成纹理，再到控制纹理的插值、重复和边缘夹持行为。
4. **透明渲染（Transparency）**——透明度是实现"看穿"效果的关键工具，用于展示物体的内部结构或在不同数据层之间建立深度关系。你将了解深度剥离（Depth Peeling）和深度排序（Depth Sorting）这两种核心的透明度处理方法，以及它们的适用场景和局限性。

此外，本章提供了一个综合性的"材质展示画廊"代码示例，将所有概念整合到一个可运行的演示程序中：在5x4的网格中排列20个球体，每个球体具有不同的材质参数组合，辅助以三点式布光和透明外球壳，让你直观地感受不同参数对渲染效果的影响。

本章需要的前置知识包括第5章（渲染管道基础）和
第12章（颜色映射）。如果你对Actor/Mapper/Renderer的关系还不太清楚，建议先回顾第5章。

---

## 13.1 材质属性（vtkProperty）

### 13.1.1 vtkProperty 概述

在VTK中，`vtkProperty`是Actor的外观描述器。每个`vtkActor`持有一个`vtkProperty`实例，通过`actor->GetProperty()`获取。Property控制着渲染的几乎所有外观参数：颜色、光泽度、透明度、表示模式（点/线框/表面）等。

回顾第5章中Actor的组成：

```
vtkActor
  ├── vtkMapper      -- 定义"画什么"（几何形状）
  ├── vtkProperty    -- 定义"怎么画"（外观材质）
  └── Transform      -- 定义"在哪里"（位置、朝向、缩放）
```

Mapper负责生成几何图元（顶点、索引），Property负责告诉OpenGL如何着色这些图元。当渲染后端（vtkOpenGLPolyDataMapper等）在GPU上绘制时，Property中的参数被翻译为着色器中的统一变量（Uniform），直接影响最终的像素颜色。

### 13.1.2 Phong光照模型参数

VTK使用经典的**Phong光照模型**（Phong Lighting Model）来模拟材质对光的反应。Phong模型将材质对入射光的反应分解为三个分量：

```
最终颜色 = 环境光分量 + 漫反射分量 + 镜面反射分量
         = Ambient     + Diffuse       + Specular
```

**分量说明：**

- **环境光（Ambient）**：模拟场景中无处不在的间接光照。即使物体的某个面不直接面向光源，环境光分量也会确保它不完全黑暗。环境光就像是"包围着整个场景的柔和光线"。
- **漫反射（Diffuse）**：模拟光线打到粗糙表面时向各个方向均匀散射的效果。漫反射的强度取决于表面法线方向与光源方向之间的夹角——当表面对向光源时最亮，背离光源时最暗。这是通常人们所说的"物体颜色"的主要来源。
- **镜面反射（Specular）**：模拟光线在光滑表面上的集中反射，产生高光（Highlight）。镜面反射的强度和集中程度取决于表面的光泽度（Shininess）。镜面反射产生的那一小块明亮区域极大地增强了物体材质的可辨识性——让人一眼看出表面是"金属的"还是"塑料的"。

此外，还有一个第四分量：

- **自发光（Emission）**：模拟物体自身发光的效果。这不是对外部光源的反应，而是物体本身就有颜色输出。自发光颜色会直接加到最终颜色上，不受场景光源的影响。

**VTK中的对应API：**

```cpp
// 获取Property对象
vtkProperty* property = actor->GetProperty();

// 设置漫反射颜色 -- 物体在光照下的"本色"
property->SetDiffuseColor(r, g, b);

// 设置环境光颜色 -- 通常与漫反射颜色相同或稍暗
property->SetAmbientColor(r, g, b);

// 设置镜面反射颜色 -- 高光的颜色，通常是白色或浅色
property->SetSpecularColor(r, g, b);

// 设置镜面反射强度系数（0.0 到 1.0）
property->SetSpecular(0.5);

// 设置镜面反射光泽度/锐度（典型值：10-100）
property->SetSpecularPower(30.0);

// 设置自发光颜色 -- 模拟物体自身发光
property->SetColor(r, g, b);  // 同时设置漫反射、环境光和镜面反射为同一颜色
```

### 13.1.3 SetColor() 与各分量颜色的关系

`SetColor(r, g, b)`是一个便捷方法，它同时设置漫反射颜色、环境光颜色和镜面反射颜色为相同的值。当你只需要快速设置"物体是什么颜色的"时，这一个调用就够了。

但在更高级的材质控制中，你通常希望三个分量有不同的值：

```cpp
// 创建一个"金色金属"的材质效果
property->SetDiffuseColor(0.8, 0.6, 0.2);     // 金色基调
property->SetAmbientColor(0.4, 0.3, 0.1);     // 环境光暗于漫反射（50%）
property->SetSpecularColor(1.0, 0.9, 0.6);    // 高光为浅金色（接近白色）
property->SetSpecular(0.8);                    // 较强的镜面反射
property->SetSpecularPower(60);                 // 集中而尖锐的高光

// 创建一个"哑光塑料"的材质效果
property->SetDiffuseColor(0.2, 0.6, 0.8);     // 蓝绿色基调
property->SetAmbientColor(0.1, 0.3, 0.4);     // 环境光暗于漫反射
property->SetSpecularColor(1.0, 1.0, 1.0);    // 白色高光
property->SetSpecular(0.1);                    // 微弱的镜面反射
property->SetSpecularPower(5.0);               // 分散而柔和的高光（哑光效果）
```

经验法则：
- **金属**：高Specular（0.5-1.0）、高SpecularPower（40-100）、SpecularColor接近光源颜色（白色）。
- **哑光/橡胶/塑料**：低Specular（0.0-0.2）、低SpecularPower（1-15）。
- **陶瓷/光泽塑料**：中Specular（0.2-0.6）、中SpecularPower（15-40）。

### 13.1.4 SetOpacity() -- 透明度

```cpp
// 设置不透明度，范围 0.0（完全透明）到 1.0（完全不透明）
property->SetOpacity(0.5);   // 50%透明
```

`SetOpacity()`控制Actor的整体不透明度。值为1.0时物体完全不透明（默认值），值为0.5时物体半透明，值为0.0时物体完全不可见。

注意：仅仅设置Opacity并不足以获得正确的透明渲染结果。要使透明度正常工作的完整设置，请参考第13.4节（透明渲染）。这里先给出一个快速片段：

```cpp
// 使Actor半透明的快速三步设置
property->SetOpacity(0.5);
renderer->SetUseDepthPeeling(1);              // 开启深度剥离
renderer->SetMaximumNumberOfPeels(10);         // 设置剥离层数
renderer->SetOcclusionRatio(0.0);              // 遮挡比例阈值
```

### 13.1.5 SetRepresentation -- 表示模式

`vtkProperty`支持三种基本的表示模式，通过`SetRepresentation()`切换：

```cpp
// 点模式 -- 每个顶点绘制为一个点
property->SetRepresentationToPoints();

// 线框模式 -- 仅绘制多边形边缘，不填充面
property->SetRepresentationToWireframe();

// 表面模式 -- 标准的多边形填充绘制（默认）
property->SetRepresentationToSurface();
```

**点模式和线框模式下的尺寸控制：**

```cpp
// 点模式下每个点的大小（像素单位）
property->SetPointSize(5.0);

// 线框模式下线的宽度（像素单位）
property->SetLineWidth(2.0);
```

### 13.1.6 SetEdgeVisibility() -- 边缘叠加显示

有时候，同时看到表面和网格线是非常有用的——比如在有限元分析中，你希望看到网格的拓扑结构同时保持对形状的感知。`SetEdgeVisibility()`提供了这种能力：

```cpp
// 在表面绘制的基础上叠加显示多边形的边缘
property->SetEdgeVisibility(true);

// 设置边缘的颜色
property->SetEdgeColor(0.0, 0.0, 0.0);  // 黑色边缘线

// 设置边缘线的宽度
property->SetLineWidth(1.5);
```

当`SetEdgeVisibility(true)`与表面模式组合使用时，Actor首先绘制填充的多边形表面，然后在每个多边形的边缘上叠加线段。这是一种高效的"线框叠加"技术，比纯粹的线框模式更美观，比纯粹的表面模式展示了更多的几何信息。

### 13.1.7 插值模式

对于光照计算在多边形内部的插值方式，VTK支持两种模式：

```cpp
// 平面着色（Flat Shading）-- 每个多边形使用单一法线，产生棱角分明的外观
property->SetInterpolationToFlat();

// 光滑着色（Gouraud Shading）-- 顶点法线在多边形内部线性插值，产生平滑外观
property->SetInterpolationToGouraud();

// Phong着色 -- 逐像素法线插值，比Gouraud产生更真实的高光
property->SetInterpolationToPhong();
```

默认情况下，VTK使用Gouraud着色。对于粗糙的多边形网格（低分辨率球体等），Phong着色可以显著改善视觉质量，但需要额外的GPU计算。

### 13.1.8 小示例：材质球对比

下面是一个快速示例，在并排的视口中展示具有不同材质属性的两个球体：

```cpp
#include <vtkActor.h>
#include <vtkCamera.h>
#include <vtkNamedColors.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkSphereSource.h>

int main()
{
    // --- 数据源：两个球体 ---
    vtkNew<vtkSphereSource> sphere1;
    sphere1->SetThetaResolution(64);
    sphere1->SetPhiResolution(64);

    vtkNew<vtkSphereSource> sphere2;
    sphere2->SetThetaResolution(64);
    sphere2->SetPhiResolution(64);

    // --- 左侧球体：金属金色材质 ---
    vtkNew<vtkPolyDataMapper> mapper1;
    mapper1->SetInputConnection(sphere1->GetOutputPort());

    vtkNew<vtkActor> actor1;
    actor1->SetMapper(mapper1);
    actor1->GetProperty()->SetDiffuseColor(0.8, 0.6, 0.2);
    actor1->GetProperty()->SetAmbientColor(0.4, 0.3, 0.1);
    actor1->GetProperty()->SetSpecularColor(1.0, 0.9, 0.6);
    actor1->GetProperty()->SetSpecular(0.9);
    actor1->GetProperty()->SetSpecularPower(80);

    // --- 右侧球体：蓝色哑光塑料材质 ---
    vtkNew<vtkPolyDataMapper> mapper2;
    mapper2->SetInputConnection(sphere2->GetOutputPort());

    vtkNew<vtkActor> actor2;
    actor2->SetMapper(mapper2);
    actor2->GetProperty()->SetDiffuseColor(0.2, 0.4, 0.8);
    actor2->GetProperty()->SetAmbientColor(0.1, 0.2, 0.4);
    actor2->GetProperty()->SetSpecularColor(1.0, 1.0, 1.0);
    actor2->GetProperty()->SetSpecular(0.15);
    actor2->GetProperty()->SetSpecularPower(8);

    // --- 左侧渲染器 ---
    vtkNew<vtkRenderer> renderer1;
    renderer1->SetViewport(0.0, 0.0, 0.5, 1.0);
    renderer1->AddActor(actor1);
    renderer1->SetBackground(0.15, 0.15, 0.15);
    renderer1->GetActiveCamera()->SetPosition(0, 0, 6);

    // --- 右侧渲染器 ---
    vtkNew<vtkRenderer> renderer2;
    renderer2->SetViewport(0.5, 0.0, 1.0, 1.0);
    renderer2->AddActor(actor2);
    renderer2->SetBackground(0.15, 0.15, 0.15);
    renderer2->SetActiveCamera(renderer1->GetActiveCamera()); // 共享相机

    // --- 渲染窗口 ---
    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(800, 400);
    renderWindow->AddRenderer(renderer1);
    renderWindow->AddRenderer(renderer2);

    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);
    interactor->Initialize();
    interactor->Start();

    return 0;
}
```

从运行结果中，你可以清楚地观察到：左边的金色球体有锐利而明亮的高光（镜面反射分量强），而右边的蓝色球体则呈现柔和均匀的外观（镜面反射分量弱）。

---

## 13.2 光照模型

### 13.2.1 vtkLight 概述

`vtkLight`是VTK中光源的抽象。每个`vtkRenderer`维护一组光源，默认情况下包含一个**头灯**（Headlight）——一个位置始终跟随相机的光源函数。你可以添加、移除和修改光源来创建所需的光照环境。

光源控制是创建专业级可视化作品的关键技能。一个精心布置的光照方案可以使数据的形状和结构更加清晰可辨，而糟糕的光照则会让最精细的模型也显得平板无趣。

### 13.2.2 光源的基本API

```cpp
#include <vtkLight.h>

vtkNew<vtkLight> light;

// 设置光源位置（世界坐标系）
light->SetPosition(x, y, z);

// 设置光源的焦点/目标点 -- 光源指向这个点
light->SetFocalPoint(x, y, z);

// 设置光源颜色
light->SetColor(r, g, b);

// 设置光源强度（默认1.0）
light->SetIntensity(1.0);

// 打开/关闭光源
light->SwitchOn();
light->SwitchOff();

// 将光源添加到渲染器
renderer->AddLight(light);

// 移除默认头灯
renderer->RemoveAllLights();
```

### 13.2.3 光源类型

VTK定义了三种光源类型，通过`SetLightType()`设置：

**1. 头灯（Headlight / VTK_LIGHT_TYPE_HEADLIGHT）**

```cpp
light->SetLightTypeToHeadlight();
```

头灯始终位于相机位置，照射方向与相机视线方向一致。这是默认光源的行为。头灯确保无论你将相机旋转到什么角度，场景始终有来自观察者方向的光照。头灯的缺点是：它会产生"平坦"的光照效果——因为光照方向与视线方向一致，所有面向相机的表面看起来都很亮，缺少阴影和立体感的提示。

**2. 场景灯（SceneLight / VTK_LIGHT_TYPE_SCENE_LIGHT）**

```cpp
light->SetLightTypeToSceneLight();
```

场景灯的位置和焦点在世界坐标系中固定不变。无论相机如何旋转，光源始终照射场景的同一区域。这是创建真实世界光照效果的方式——模拟太阳从特定方向照射场景。

**3. 相机灯（CameraLight / VTK_LIGHT_TYPE_CAMERA_LIGHT）**

```cpp
light->SetLightTypeToCameraLight();
```

相机灯的位置和焦点在相机坐标系中定义。当相机移动时，光源也跟随移动，但光源相对于相机的方向和位置保持不变。这介于Headlight和SceneLight之间——提供了比Headlight更灵活的光照方向控制，同时保持了相机移动时的便利性。

**场景灯 vs. 相机灯的使用场景：**

| 场景 | 推荐类型 | 原因 |
|------|---------|------|
| 单次检查数据 | 头灯 | 简单，不需要额外配置 |
| 从固定角度展示 | 场景灯 | 保持光照一致性 |
| 交互式探索 | 相机灯 | 跟随视角但保持光照方向感 |
| 出版物级别图片 | 场景灯（多点布光） | 精细控制每个光源的位置和方向 |

### 13.2.4 SetColor() 与 SetIntensity()

光源的颜色和强度共同决定照射到物体表面的光的实际效果：

```cpp
// 偏暖的白色光（微微偏黄）
light->SetColor(1.0, 0.95, 0.8);
light->SetIntensity(1.2);  // 稍微增强

// 冷色补光（偏蓝）
light->SetColor(0.7, 0.8, 1.0);
light->SetIntensity(0.5);  // 强度减半
```

光照强度的工作原理：光源颜色和强度相乘得到最终投射到物体表面的光量。因此，`SetColor(1.0, 0.5, 0.5)` + `SetIntensity(2.0)`等价于`SetColor(2.0, 1.0, 1.0)` + `SetIntensity(1.0)`。不过请注意，颜色值超过1.0时可能在渲染后端产生超亮度（HDR-like）效果，通常建议将颜色分量限制在[0, 1]范围内，通过调整Intensity来控制亮度。

### 13.2.5 三点式布光（Three-Point Lighting）

三点式布光是影视和摄影中的经典布光技术，也是科学可视化中获得专业外观的主要方法。三点式布光包含三个光源：

```
        主光源 (Key Light)
        │
        │  45°
        │╲
        │ ╲
        │  ╲
        │  物体
       ╱
      ╱ 45°
     ╱
    补光 (Fill Light)
    强度：主光源的40%-60%

        背光源 (Back Light / Rim Light)
        │ 从物体后方照射
        │ 勾画轮廓边缘
```

**主光源（Key Light）**：最主要的光源，提供场景的基本光照和方向感。通常放置在相机前方偏一侧约30-45度、偏上方约30-45度的位置。主光源的强度最高（1.0），色调偏暖。

**补光（Fill Light）**：放置在另一侧，用于填充主光源产生的阴影，避免暗面完全变黑。强度通常为主光源的40%-60%，色调偏中或偏冷。

**背光源（Back Light / Rim Light）**：从物体的后方或侧面照射，在物体边缘产生亮边（Rim Lighting），将物体与背景分离开来，增强立体感和深度感。强度可以接近主光源，位置在物体的后方。

**三点式布光在VTK中的实现：**

```cpp
vtkNew<vtkRenderer> renderer;
renderer->RemoveAllLights();  // 先移除默认头灯

// --- 主光源（Key Light）：从右上方照射 ---
vtkNew<vtkLight> keyLight;
keyLight->SetLightTypeToSceneLight();
keyLight->SetPosition(5.0, 5.0, 5.0);
keyLight->SetFocalPoint(0.0, 0.0, 0.0);
keyLight->SetColor(1.0, 0.95, 0.9);
keyLight->SetIntensity(1.0);
renderer->AddLight(keyLight);

// --- 补光（Fill Light）：从左下方照射 ---
vtkNew<vtkLight> fillLight;
fillLight->SetLightTypeToSceneLight();
fillLight->SetPosition(-4.0, -2.0, 0.0);
fillLight->SetFocalPoint(0.0, 0.0, 0.0);
fillLight->SetColor(0.7, 0.8, 1.0);
fillLight->SetIntensity(0.5);
renderer->AddLight(fillLight);

// --- 背光源（Back Light）：从后方照射 ---
vtkNew<vtkLight> backLight;
backLight->SetLightTypeToSceneLight();
backLight->SetPosition(0.0, 2.0, -6.0);
backLight->SetFocalPoint(0.0, 0.0, 0.0);
backLight->SetColor(1.0, 1.0, 1.0);
backLight->SetIntensity(0.8);
renderer->AddLight(backLight);
```

### 13.2.6 光照强度衰减

VTK光源支持距离衰减，使得离光源越远的物体越暗（模拟真实世界的平方反比定律）：

```cpp
// 启用光照衰减
light->SetAttenuationValues(a0, a1, a2);

// 最终的衰减因子 = 1.0 / (a0 + a1*d + a2*d*d)
// 其中 d 是光源到物体表面点的距离

// 示例：平方反比衰减（近似真实世界）
light->SetAttenuationValues(0.0, 0.0, 1.0);

// 示例：线性衰减
light->SetAttenuationValues(0.0, 1.0, 0.0);

// 示例：无衰减（默认）
light->SetAttenuationValues(1.0, 0.0, 0.0);
```

默认情况下，VTK光源不使用距离衰减（`a0=1, a1=0, a2=0`）。这在科学可视化中通常是有意的设计选择——你通常希望场景中所有部分的光照均匀一致，以便准确比较不同区域的数据特征。但如果你在创建演示或动画，启用衰减可以增加真实感。

### 13.2.7 锥形光源与位置光源

VTK支持将光源限制在一个锥形区域内（类似聚光灯效果）：

```cpp
// 设置锥形光源
light->SetConeAngle(30.0);         // 锥角（度），范围 0-90

// 设置光源的位置性
light->SetPositional(true);        // 启用位置光源（有锥形和衰减）
light->SetPositional(false);       // 方向光源（平行光，无锥形和衰减，默认）

// 锥形光源的指数衰减
light->SetExponent(5.0);           // 锥形区域内从中心到边缘的衰减指数
```

当`SetPositional(true)`且`SetConeAngle()`小于90度时，光源效果类似于一个聚光灯——只照射锥形区域内的物体。这在需要突出显示场景中某一特定部分时非常有用。

### 13.2.8 环境光

除了点光源和方向光源外，VTK场景还有一个全局环境光设置：

```cpp
// 设置环境光强度（0.0 到 1.0）
renderer->SetEnvironmentalLighting(true);   // 基于上/下方向的环境光（默认false）
```

或者，你可以手动设置一个带有高环境光分量的光源来模拟全局间接光照：

```cpp
// 不推荐，但在某些情况下有用：
// 将renderer的全局环境光（与Property的ambient不同）设置为较高值
renderer->SetEnvironmentLighting(true);
renderer->SetEnvironmentConstantColor(0.2, 0.2, 0.25);   // 偏蓝的暗环境光
```

需要注意的是，`renderer`层面的环境光与`vtkProperty`上的`SetAmbientColor()`是不同的概念——前者是场景中无处不在的统一基底光照，后者是材质对环境光的反射率。两者共同决定了环境光分量对最终像素颜色的贡献。

---

## 13.3 纹理映射

### 13.3.1 纹理映射概述

**纹理映射**（Texture Mapping）是将图像（纹理）贴附到三维几何表面上的技术。纹理映射极大地增强了视觉丰富度——用少量的三角形配合高分辨率的纹理，可以产生比纯几何建模更丰富的视觉细节。

在VTK中，纹理映射由`vtkTexture`类实现。它的核心工作流程是：

```
图像数据（vtkImageData） --> vtkTexture --> Actor（通过 SetTexture()）
```

### 13.3.2 vtkTexture 的基本用法

```cpp
#include <vtkTexture.h>
#include <vtkJPEGReader.h>    // 或 vtkPNGReader, vtkBMPReader 等

// 步骤1：加载图像数据
vtkNew<vtkJPEGReader> reader;
reader->SetFileName("texture.jpg");
reader->Update();

// 步骤2：创建纹理对象
vtkNew<vtkTexture> texture;
texture->SetInputConnection(reader->GetOutputPort());

// 步骤3：将纹理应用到Actor
actor->SetTexture(texture);
```

### 13.3.3 纹理坐标

为了让纹理正确地贴到几何表面上，几何数据必须包含**纹理坐标**（Texture Coordinates，通常称为TCoords或UV坐标）。纹理坐标是定义在每个顶点上的一组二维坐标（u, v），范围通常为[0, 1]，表示在纹理图像上的横向和纵向位置。

```
纹理图像的坐标系统：
(0, 1) +---------------+ (1, 1)
       |               |
       |   纹理图像     |
       |               |
(0, 0) +---------------+ (1, 0)
```

VTK的内置源（`vtkSphereSource`、`vtkPlaneSource`、`vtkCylinderSource`等）默认会生成纹理坐标。如果你使用的是自定义几何数据，需要确保它包含TCoords数组。

```cpp
// 检查数据是否包含纹理坐标
vtkDataArray* tcoords = polydata->GetPointData()->GetTCoords();
if (tcoords)
{
    std::cout << "Texture coordinates found. Dimensions: "
              << tcoords->GetNumberOfComponents() << " components." << std::endl;
}
else
{
    std::cout << "No texture coordinates available." << std::endl;
}

// 一些source提供了控制纹理坐标生成的选项
sphereSource->SetGenerateTCoords(true);   // 某些source支持此选项
```

### 13.3.4 纹理属性控制

`vtkTexture`提供了多种属性来控制纹理采样的行为：

```cpp
// 纹理插值
texture->SetInterpolate(true);    // 开启双线性插值（默认）：纹理放大时平滑
texture->SetInterpolate(false);   // 关闭插值：纹理放大时呈现像素块状

// 纹理重复
texture->SetRepeat(true);         // 允许纹理在[0,1]范围外重复
texture->SetRepeat(false);        // [0,1]范围外的坐标被夹持到边缘（默认）

// 边缘夹持
texture->SetEdgeClamp(true);      // 限制纹理坐标不超出[0,1]范围
texture->SetEdgeClamp(false);     // 允许纹理坐标超出（配合Repeat或镜像）

// 纹理质量
texture->SetQuality(32);          // 纹理采样的最大各向异性级别（默认1）

// 纹理的颜色混合
texture->SetBlendingMode(vtkTexture::VTK_TEXTURE_BLENDING_MODE_MODULATE);
// 支持的模式：
//   VTK_TEXTURE_BLENDING_MODE_NONE          -- 仅使用纹理颜色
//   VTK_TEXTURE_BLENDING_MODE_REPLACE       -- 用纹理替换底层颜色
//   VTK_TEXTURE_BLENDING_MODE_MODULATE      -- 纹理颜色与底层颜色相乘
//   VTK_TEXTURE_BLENDING_MODE_ADD           -- 纹理颜色与底层颜色相加
//   VTK_TEXTURE_BLENDING_MODE_ADD_SIGNED    -- 带符号的加法
//   VTK_TEXTURE_BLENDING_MODE_INTERPOLATE   -- 插值混合
//   VTK_TEXTURE_BLENDING_MODE_SUBTRACT      -- 底层颜色减去纹理颜色
```

### 13.3.5 程序化纹理生成

有时候你不需要从文件加载纹理图像，而是希望程序化地生成纹理。下面是一个创建棋盘格纹理的示例：

```cpp
#include <vtkImageData.h>
#include <vtkUnsignedCharArray.h>

vtkSmartPointer<vtkTexture> CreateCheckerboardTexture(
    int size, int squaresPerRow, double r1, double g1, double b1,
                                   double r2, double g2, double b2)
{
    // 创建图像数据
    vtkNew<vtkImageData> image;
    image->SetDimensions(size, size, 1);
    image->AllocateScalars(VTK_UNSIGNED_CHAR, 3);  // RGB三通道

    unsigned char* pixels = static_cast<unsigned char*>(
        image->GetScalarPointer());

    int squareSize = size / squaresPerRow;

    for (int y = 0; y < size; y++)
    {
        for (int x = 0; x < size; x++)
        {
            int idx = (y * size + x) * 3;   // 3 channels per pixel
            int sqX = x / squareSize;
            int sqY = y / squareSize;

            if ((sqX + sqY) % 2 == 0)
            {
                pixels[idx]     = static_cast<unsigned char>(r1 * 255);
                pixels[idx + 1] = static_cast<unsigned char>(g1 * 255);
                pixels[idx + 2] = static_cast<unsigned char>(b1 * 255);
            }
            else
            {
                pixels[idx]     = static_cast<unsigned char>(r2 * 255);
                pixels[idx + 1] = static_cast<unsigned char>(g2 * 255);
                pixels[idx + 2] = static_cast<unsigned char>(b2 * 255);
            }
        }
    }

    // 创建纹理
    vtkNew<vtkTexture> texture;
    texture->SetInputData(image);
    texture->SetInterpolate(true);   // 插值以获得平滑的纹理采样

    return texture;
}

// 使用示例：创建红色-浅灰色的棋盘格
auto checkerTex = CreateCheckerboardTexture(256, 8,
    0.9, 0.2, 0.2,    // 颜色1：红色
    0.9, 0.9, 0.9);   // 颜色2：浅灰
actor->SetTexture(checkerTex);
```

类似地，你可以创建条纹纹理、渐变纹理、噪点纹理等。程序化纹理在展示几何形状（如通过棋盘格展示曲面的参数化质量）和数据可视化中非常有用。

### 13.3.6 纹理映射到不同几何体

**平面纹理映射：**

对于`vtkPlaneSource`，纹理坐标默认在平面的四个角生成，纹理图像被拉伸以填充整个平面。

```cpp
vtkNew<vtkPlaneSource> plane;
plane->SetOrigin(-2.0, -2.0, 0.0);
plane->SetPoint1(4.0, 0.0, 0.0);
plane->SetPoint2(0.0, 4.0, 0.0);
// plane自动生成纹理坐标：(0,0), (1,0), (0,1), (1,1)
```

**球面纹理映射：**

对于`vtkSphereSource`，纹理坐标通过球面参数化自动生成——纹理在球面上环绕，在两极处汇聚。这在两极附近会产生可见的纹理压缩（纹理畸变），这是球面参数化固有的特性。

```cpp
vtkNew<vtkSphereSource> sphere;
sphere->SetThetaResolution(64);
sphere->SetPhiResolution(64);
// 纹理坐标自动生成：theta -> u, phi -> v
```

**圆柱纹理映射：**

对于`vtkCylinderSource`，纹理沿圆周方向环绕，沿高度方向线性分布。

### 13.3.7 纹理映射中的注意事项

1. **纹理尺寸**：OpenGL对纹理尺寸有限制。大多数现代GPU支持最高16384x16384的纹理，但为了性能和兼容性，建议使用2的幂次方尺寸（256x256、512x512、1024x1024等）。VTK会自动调整非2的幂次方纹理，但这会引入额外的处理开销。

2. **纹理内存**：每个RGB纹理占用的GPU内存为 `width * height * 3` 字节（未压缩）。单个4096x4096的RGB纹理占用约48MB显存。在场景中使用大量大尺寸纹理时要注意显存上限。

3. **纹理坐标缺失**：如果几何数据没有纹理坐标，纹理将无法正确显示。确保数据源包含TCoords，或使用`vtkTextureMapToPlane`/`vtkTextureMapToSphere`/`vtkTextureMapToCylinder`等过滤器来程序化生成纹理坐标。

4. **纹理与光照的交互**：当Actor同时设置了纹理和材质颜色（Property）时，最终颜色是纹理颜色与光照计算结果的组合。具体行为取决于`SetBlendingMode()`的设置。

---

## 13.4 透明渲染

### 13.4.1 透明度在VTK中的挑战

透明度（半透明渲染）是科学可视化中的一个重要但具有挑战性的主题。核心困难在于：**正确的透明度渲染要求从远到近的顺序绘制**。这是因为当光线穿过多个半透明表面时，颜色混合的顺序决定了最终结果。

考虑两个半透明物体：物体A（红色，位于后方）和物体B（蓝色，位于前方）。正确的渲染顺序是：
1. 先绘制A（后方）
2. 再绘制B（前方）——B的半透明蓝色与A的红色混合

如果顺序反过来（先B后A），A的红色会错误地覆盖B的蓝色对应的像素。

### 13.4.2 基本透明度设置

设置透明度的第一步是在Property上设置Opacity：

```cpp
actor->GetProperty()->SetOpacity(0.5);  // 50%透明
```

但仅仅设置Opacity是不够的。为了让VTK正确处理透明度，你需要选择以下几种策略之一。

### 13.4.3 深度剥离（Depth Peeling）

**深度剥离**是VTK为透明渲染提供的最通用的解决方案。它通过多遍渲染（Multi-pass Rendering）逐层"剥离"深度信息，从最前面开始，一层层地渲染，最后按从后到前的顺序合成。

```cpp
// 在渲染器上启用深度剥离
renderer->SetUseDepthPeeling(true);

// 设置最大剥离层数（默认4，建议4-10）
// 层数越多，能正确处理的透明层数越多，但性能越差
renderer->SetMaximumNumberOfPeels(10);

// 设置遮挡比例（0到1，默认0.0）
// 当一段像素的透明片元比例超过此阈值时，停止剥离
renderer->SetOcclusionRatio(0.0);
```

**深度剥离的工作原理：**

深度剥离对每一轮（Peel）执行一次额外的渲染遍历：
1. **第一轮**：渲染所有物体的最前面的可见表面，但不清除这些像素的深度缓冲。然后丢弃这些片元，为下一轮做准备。
2. **第二轮**：渲染所有物体时跳过第一轮已经处理过的像素（通过比较深度缓冲），获得第二层可见表面。
3. **第三轮**：类似地，获得第三层可见表面。
4. ...依此类推。
5. **合成**：将所有层按从后到前的顺序进行alpha混合，生成最终的透明效果。

**优点**：无需对几何体进行排序，可以直接处理交叉和重叠的复杂透明几何体。对`vtkActor`的绝大多数渲染场景都有效。

**缺点**：每增加一层剥离，就需要一次额外的全场景渲染遍历。10层剥离意味着渲染开销大约是正常的1+10=11倍。对于复杂的场景（大量多边形），这可能导致显著的性能下降。对于动画和实时交互场景，建议使用较少的剥离层数（4-6层）。

### 13.4.4 深度排序（Depth Sorting for PolyData）

**深度排序**是针对`vtkPolyData`的另一种透明度处理策略。它通过对多边形进行排序来确保正确的绘制顺序。

```cpp
#include <vtkDepthSortPolyData.h>

// 创建深度排序过滤器
vtkNew<vtkDepthSortPolyData> depthSort;
depthSort->SetInputConnection(source->GetOutputPort());
depthSort->SetDirectionToBackToFront();   // 从后到前排序
// 或 depthSort->SetDirectionToFrontToBack(); // 从前到后
depthSort->SetCamera(renderer->GetActiveCamera());  // 排序依据的相机
depthSort->SetSortScalarMode(VTK_SORT_BY_VALUE);   // 按深度值排序

// 将排序后的数据输入Mapper
vtkNew<vtkPolyDataMapper> mapper;
mapper->SetInputConnection(depthSort->GetOutputPort());
```

**深度排序的工作方式：**

`vtkDepthSortPolyData`计算`vtkPolyData`中每个单元（Cell）的深度（到相机的距离），然后按照指定方向对单元重新排序。重新排序后的数据集被传递给Mapper渲染，由于单元已经按深度排列，alpha混合可以正确进行。

**优点**：单次渲染遍历，比深度剥离更快。对于简单的不交叉的透明PolyData效果很好。

**缺点**：排序是在CPU上按单元进行的，对于非常大的Polydata（数百万多边形），排序本身的开销可能很大。此外，深度排序只对每个模型的单元进行排序，**不能处理不同Actor之间的深度关系**（不同Actor之间的交叉仍然可能出现错误的渲染顺序）。它也不能处理单个多边形内部的深度交叉。

### 13.4.5 两种透明策略的对比

| 特性 | 深度剥离（Depth Peeling） | 深度排序（DepthSortPolyData） |
|------|--------------------------|------------------------------|
| 工作原理 | 多遍渲染，逐层剥离深度 | CPU排序多边形的绘制顺序 |
| 处理交叉几何体 | 正确 | 单个Actor内可能出错 |
| 处理多个Actor | 正确 | 不同Actor间可能出错 |
| 性能 | 慢（每层一遍渲染） | 较快（但CPU排序开销） |
| 适用场景 | 通用透明场景 | 单个Actor的无交叉透明几何体 |
| GPU/CPU负载 | GPU负载高 | CPU排序负载高 |
| 硬件要求 | 需要GPU支持深度纹理 | 无特殊要求 |
| 建议 | 质量优先的选择 | 性能优先的选择 |

### 13.4.6 透明度渲染的实践建议

1. **优先使用深度剥离**：对于大多数科学可视化场景，深度剥离是正确的选择。它虽然慢一些，但渲染结果是正确的。非交互式的出图渲染（如保存高分辨率截图）推荐使用深度剥离。

2. **限制剥离层数**：对于实时交互场景，将`SetMaximumNumberOfPeels()`设为4-6层可以在效果和性能之间取得不错的平衡。

3. **减少透明物体的数量**：如果可能，将不透明的物体和透明物体分开处理。不透明物体不受深度剥离影响，先渲染不透明物体可以减少后续剥离的像素数量。

4. **避免复杂的深度交叉**：尽量避免让多个高多边形数的半透明物体相互穿插。如果需要展示内部结构，考虑使用剪切平面（Clipping Plane）或"爆炸视图"（Exploded View）作为透明度之外的替代方案。

5. **对点精灵使用深度排序**：对于粒子系统（点精灵），使用`vtkDepthSortPolyData`通常是更高效的选择。

6. **测试和验证**：透明度相关的bug往往非常微妙——可能在特定角度看起来正确，旋转后在另一个角度出现明显的渲染错误。在完成透明渲染设置后，多旋转场景从不同角度验证效果。

### 13.4.7 小示例：半透明球体展示内部结构

```cpp
#include <vtkActor.h>
#include <vtkCamera.h>
#include <vtkNamedColors.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkSphereSource.h>

int main()
{
    // --- 外部球体：半透明 ---
    vtkNew<vtkSphereSource> outerSphere;
    outerSphere->SetRadius(2.0);
    outerSphere->SetThetaResolution(100);
    outerSphere->SetPhiResolution(100);

    vtkNew<vtkPolyDataMapper> outerMapper;
    outerMapper->SetInputConnection(outerSphere->GetOutputPort());

    vtkNew<vtkActor> outerActor;
    outerActor->SetMapper(outerMapper);
    outerActor->GetProperty()->SetColor(0.6, 0.8, 1.0);
    outerActor->GetProperty()->SetOpacity(0.3);          // 30%不透明
    outerActor->GetProperty()->SetSpecular(0.4);
    outerActor->GetProperty()->SetSpecularPower(30);
    // 在表面基础上显示线框，增强结构感知
    outerActor->GetProperty()->SetEdgeVisibility(true);
    outerActor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.3);

    // --- 内部球体：不透明 ---
    vtkNew<vtkSphereSource> innerSphere;
    innerSphere->SetRadius(0.8);
    innerSphere->SetThetaResolution(50);
    innerSphere->SetPhiResolution(50);

    vtkNew<vtkPolyDataMapper> innerMapper;
    innerMapper->SetInputConnection(innerSphere->GetOutputPort());

    vtkNew<vtkActor> innerActor;
    innerActor->SetMapper(innerMapper);
    innerActor->GetProperty()->SetColor(1.0, 0.3, 0.3);   // 红色
    innerActor->GetProperty()->SetSpecular(0.5);
    innerActor->GetProperty()->SetSpecularPower(20);

    // --- 渲染器 ---
    vtkNew<vtkRenderer> renderer;
    renderer->AddActor(outerActor);
    renderer->AddActor(innerActor);
    renderer->SetBackground(0.15, 0.15, 0.15);

    // 启用深度剥离以正确处理透明度
    renderer->SetUseDepthPeeling(true);
    renderer->SetMaximumNumberOfPeels(10);
    renderer->SetOcclusionRatio(0.0);

    // --- 渲染窗口 ---
    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(600, 600);
    renderWindow->SetMultiSamples(8);   // 8x多重采样抗锯齿
    renderWindow->AddRenderer(renderer);

    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);
    interactor->Initialize();
    interactor->Start();

    return 0;
}
```

运行这个示例，你会看到一个半透明的浅蓝色外壳包裹着一个不透明的红色内核。旋转场景，你能从不同角度看到内部的红色球体，同时蓝色外壳的线框叠加让你感知到外壳的几何形状。

---

## 13.5 代码示例：材质展示画廊

本节提供一个完整的C++程序，展示本章涵盖的所有核心概念。程序在5x4的网格中创建了20个球体，每个球体展示不同的材质属性组合。此外，还展示了三点式布光、棋盘格纹理映射以及半透明外层球体。

### 13.5.1 程序概述

**布局设计：**

程序在5列 x 4行的网格中排列20个球体，每个列代表一类材质参数变化：

- **第1列（Column 0）：颜色变化**——展示不同漫反射颜色的纯色材质（红、橙、黄、绿）。
- **第2列（Column 1）：镜面反射变化**——相同漫反射色，不同的Specular和SpecularPower（从哑光到高亮金属）。
- **第3列（Column 2）：表示模式变化**——分别使用点模式、线框模式、表面模式、线框叠加模式。
- **第4列（Column 3）：纹理映射**——棋盘格纹理与不同材质参数的组合。
- **第5列（Column 4）：特殊效果**——自发光、极高镜面反射、半透明、线框覆盖。

**光照方案：**使用经典的三点式布光（Key + Fill + Back Light）。

**附加效果：**在场景中心放置一个大的半透明外球壳，展示透明渲染。

### 13.5.2 完整C++代码

```cpp
// ============================================================================
// Chapter 13: Advanced Rendering Techniques
// File: MaterialGallery.cxx
// Description: A comprehensive demonstration of VTK advanced rendering:
//              material properties, three-point lighting, texture mapping,
//              and transparent rendering. 20 spheres are arranged in a 5x4
//              grid, each with different material parameter combinations.
//              A large semi-transparent outer sphere encloses the scene.
// VTK Version: 9.5.2
// ============================================================================

#include <vtkActor.h>
#include <vtkCamera.h>
#include <vtkImageData.h>
#include <vtkLight.h>
#include <vtkNamedColors.h>
#include <vtkNew.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkSmartPointer.h>
#include <vtkSphereSource.h>
#include <vtkTexture.h>
#include <vtkUnsignedCharArray.h>

#include <vector>

// --------------------------------------------------------------------------
// Helper: Create a procedural checkerboard texture
// --------------------------------------------------------------------------
vtkSmartPointer<vtkTexture> CreateCheckerboard(
    int    size,
    int    squaresPerRow,
    double r1, double g1, double b1,
    double r2, double g2, double b2)
{
    vtkNew<vtkImageData> image;
    image->SetDimensions(size, size, 1);
    image->AllocateScalars(VTK_UNSIGNED_CHAR, 3);

    unsigned char* pixels = static_cast<unsigned char*>(
        image->GetScalarPointer());

    int squareSize = size / squaresPerRow;

    for (int y = 0; y < size; y++)
    {
        for (int x = 0; x < size; x++)
        {
            int idx = (y * size + x) * 3;
            int sqX = x / squareSize;
            int sqY = y / squareSize;

            if ((sqX + sqY) % 2 == 0)
            {
                pixels[idx]     = static_cast<unsigned char>(r1 * 255);
                pixels[idx + 1] = static_cast<unsigned char>(g1 * 255);
                pixels[idx + 2] = static_cast<unsigned char>(b1 * 255);
            }
            else
            {
                pixels[idx]     = static_cast<unsigned char>(r2 * 255);
                pixels[idx + 1] = static_cast<unsigned char>(g2 * 255);
                pixels[idx + 2] = static_cast<unsigned char>(b2 * 255);
            }
        }
    }

    vtkNew<vtkTexture> texture;
    texture->SetInputData(image);
    texture->SetInterpolate(true);

    return texture;
}

// --------------------------------------------------------------------------
// Helper: Create a sphere source with specified resolution
// --------------------------------------------------------------------------
vtkSmartPointer<vtkSphereSource> CreateSphere(double radius, int res)
{
    vtkNew<vtkSphereSource> sphere;
    sphere->SetRadius(radius);
    sphere->SetThetaResolution(res);
    sphere->SetPhiResolution(res);
    sphere->Update();
    return sphere;
}

// --------------------------------------------------------------------------
// Helper: Create an actor with a mapper connected to a sphere
// --------------------------------------------------------------------------
vtkSmartPointer<vtkActor> CreateSphereActor(
    vtkSmartPointer<vtkSphereSource> sphere)
{
    vtkNew<vtkPolyDataMapper> mapper;
    mapper->SetInputConnection(sphere->GetOutputPort());

    vtkNew<vtkActor> actor;
    actor->SetMapper(mapper);
    return actor;
}

// --------------------------------------------------------------------------
// Main program
// --------------------------------------------------------------------------
int main(int argc, char* argv[])
{
    // ======================================================================
    // Lighting Setup: Three-point lighting for professional visualization
    // ======================================================================
    vtkNew<vtkLight> keyLight;
    keyLight->SetLightTypeToSceneLight();
    keyLight->SetPosition(10.0, 10.0, 15.0);
    keyLight->SetFocalPoint(0.0, 0.0, 0.0);
    keyLight->SetColor(1.0, 0.96, 0.9);
    keyLight->SetIntensity(1.0);

    vtkNew<vtkLight> fillLight;
    fillLight->SetLightTypeToSceneLight();
    fillLight->SetPosition(-8.0, -3.0, 2.0);
    fillLight->SetFocalPoint(0.0, 0.0, 0.0);
    fillLight->SetColor(0.7, 0.8, 1.0);
    fillLight->SetIntensity(0.5);

    vtkNew<vtkLight> backLight;
    backLight->SetLightTypeToSceneLight();
    backLight->SetPosition(0.0, 5.0, -12.0);
    backLight->SetFocalPoint(0.0, 0.0, 0.0);
    backLight->SetColor(1.0, 1.0, 1.0);
    backLight->SetIntensity(0.7);

    // ======================================================================
    // Texture: Create two checkerboard textures
    // ======================================================================
    auto checkerRed = CreateCheckerboard(256, 8,
        0.95, 0.15, 0.15,     // 颜色1：红色
        0.95, 0.95, 0.95);    // 颜色2：白色

    auto checkerBlue = CreateCheckerboard(256, 8,
        0.15, 0.15, 0.95,     // 颜色1：蓝色
        0.95, 0.95, 0.95);    // 颜色2：白色

    // ======================================================================
    // Scene: Create 20 spheres in a 5x4 grid
    // ======================================================================
    const int COLS = 5;
    const int ROWS = 4;
    const double SPACING = 3.5;
    const double RADIUS = 1.2;

    // Store all actors for later rendering
    std::vector<vtkSmartPointer<vtkActor>> allActors;

    // 5 columns x 4 rows = 20 spheres
    // Each column demonstrates a different material category

    for (int col = 0; col < COLS; col++)
    {
        for (int row = 0; row < ROWS; row++)
        {
            auto sphere = CreateSphere(RADIUS, 64);
            auto actor = CreateSphereActor(sphere);

            // Position the sphere in the grid
            double x = (col - (COLS - 1) / 2.0) * SPACING;
            double y = (row - (ROWS - 1) / 2.0) * SPACING;
            actor->SetPosition(x, y, 0.0);

            // --- Column 0: Color variations ---
            if (col == 0)
            {
                // Same specular parameters, different diffuse colors
                actor->GetProperty()->SetSpecular(0.4);
                actor->GetProperty()->SetSpecularPower(25);
                actor->GetProperty()->SetSpecularColor(1.0, 1.0, 1.0);

                if (row == 0)
                {
                    actor->GetProperty()->SetDiffuseColor(1.0, 0.2, 0.2);   // Red
                    actor->GetProperty()->SetAmbientColor(0.3, 0.05, 0.05);
                }
                else if (row == 1)
                {
                    actor->GetProperty()->SetDiffuseColor(1.0, 0.6, 0.1);   // Orange
                    actor->GetProperty()->SetAmbientColor(0.3, 0.15, 0.02);
                }
                else if (row == 2)
                {
                    actor->GetProperty()->SetDiffuseColor(1.0, 0.9, 0.1);   // Yellow
                    actor->GetProperty()->SetAmbientColor(0.3, 0.22, 0.02);
                }
                else // row == 3
                {
                    actor->GetProperty()->SetDiffuseColor(0.2, 0.8, 0.3);   // Green
                    actor->GetProperty()->SetAmbientColor(0.05, 0.2, 0.07);
                }
            }
            // --- Column 1: Specular variations ---
            else if (col == 1)
            {
                // Same diffuse color, different specular parameters
                actor->GetProperty()->SetDiffuseColor(0.3, 0.5, 0.9);
                actor->GetProperty()->SetAmbientColor(0.07, 0.12, 0.25);

                if (row == 0)
                {
                    // Matte finish -- very low specular
                    actor->GetProperty()->SetSpecular(0.05);
                    actor->GetProperty()->SetSpecularPower(5);
                    actor->GetProperty()->SetSpecularColor(1.0, 1.0, 1.0);
                }
                else if (row == 1)
                {
                    // Satin finish -- medium-low specular
                    actor->GetProperty()->SetSpecular(0.3);
                    actor->GetProperty()->SetSpecularPower(20);
                    actor->GetProperty()->SetSpecularColor(1.0, 1.0, 1.0);
                }
                else if (row == 2)
                {
                    // Glossy finish -- high specular, moderate power
                    actor->GetProperty()->SetSpecular(0.7);
                    actor->GetProperty()->SetSpecularPower(50);
                    actor->GetProperty()->SetSpecularColor(1.0, 0.95, 0.8);
                }
                else // row == 3
                {
                    // Metallic finish -- very high specular and power
                    actor->GetProperty()->SetSpecular(0.95);
                    actor->GetProperty()->SetSpecularPower(90);
                    actor->GetProperty()->SetSpecularColor(1.0, 0.9, 0.6);
                }
            }
            // --- Column 2: Representation modes ---
            else if (col == 2)
            {
                actor->GetProperty()->SetColor(0.8, 0.5, 0.2);
                actor->GetProperty()->SetSpecular(0.3);
                actor->GetProperty()->SetSpecularPower(20);
                actor->GetProperty()->SetAmbientColor(0.2, 0.12, 0.05);

                if (row == 0)
                {
                    // Points mode
                    actor->GetProperty()->SetRepresentationToPoints();
                    actor->GetProperty()->SetPointSize(4.0);
                }
                else if (row == 1)
                {
                    // Wireframe mode
                    actor->GetProperty()->SetRepresentationToWireframe();
                    actor->GetProperty()->SetLineWidth(1.5);
                }
                else if (row == 2)
                {
                    // Surface mode (default)
                    actor->GetProperty()->SetRepresentationToSurface();
                }
                else // row == 3
                {
                    // Surface + wireframe overlay
                    actor->GetProperty()->SetRepresentationToSurface();
                    actor->GetProperty()->SetEdgeVisibility(true);
                    actor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.1);
                    actor->GetProperty()->SetLineWidth(1.0);
                }
            }
            // --- Column 3: Texture mapping ---
            else if (col == 3)
            {
                actor->GetProperty()->SetSpecular(0.3);
                actor->GetProperty()->SetSpecularPower(25);
                actor->GetProperty()->SetSpecularColor(1.0, 1.0, 1.0);

                if (row == 0)
                {
                    // Red checkerboard, no diffuse color -> texture dominates
                    actor->GetProperty()->SetDiffuseColor(1.0, 1.0, 1.0);
                    actor->GetProperty()->SetAmbientColor(0.2, 0.2, 0.2);
                    actor->SetTexture(checkerRed);
                }
                else if (row == 1)
                {
                    // Red checkerboard + yellow tint
                    actor->GetProperty()->SetDiffuseColor(1.0, 0.95, 0.2);
                    actor->GetProperty()->SetAmbientColor(0.25, 0.23, 0.05);
                    actor->SetTexture(checkerRed);
                }
                else if (row == 2)
                {
                    // Blue checkerboard, neutral diffuse
                    actor->GetProperty()->SetDiffuseColor(0.9, 0.9, 0.9);
                    actor->GetProperty()->SetAmbientColor(0.2, 0.2, 0.2);
                    actor->SetTexture(checkerBlue);
                }
                else // row == 3
                {
                    // Blue checkerboard + high specular for gloss
                    actor->GetProperty()->SetDiffuseColor(0.8, 0.8, 0.8);
                    actor->GetProperty()->SetAmbientColor(0.15, 0.15, 0.15);
                    actor->GetProperty()->SetSpecular(0.8);
                    actor->GetProperty()->SetSpecularPower(70);
                    actor->SetTexture(checkerBlue);
                }
            }
            // --- Column 4: Special effects ---
            else if (col == 4)
            {
                if (row == 0)
                {
                    // Self-illuminating (emissive) green
                    actor->GetProperty()->SetColor(0.2, 0.8, 0.3);
                    actor->GetProperty()->SetAmbient(1.0);      // Full ambient
                    actor->GetProperty()->SetDiffuse(0.3);
                    actor->GetProperty()->SetAmbientColor(0.2, 0.8, 0.3);
                    actor->GetProperty()->SetSpecular(0.0);
                }
                else if (row == 1)
                {
                    // Extreme metallic -- pure specular, almost no diffuse
                    actor->GetProperty()->SetDiffuseColor(0.1, 0.1, 0.15);
                    actor->GetProperty()->SetAmbientColor(0.02, 0.02, 0.03);
                    actor->GetProperty()->SetSpecularColor(1.0, 0.95, 0.85);
                    actor->GetProperty()->SetSpecular(1.0);
                    actor->GetProperty()->SetSpecularPower(120);
                }
                else if (row == 2)
                {
                    // Semi-transparent
                    actor->GetProperty()->SetColor(0.4, 0.6, 1.0);
                    actor->GetProperty()->SetSpecular(0.5);
                    actor->GetProperty()->SetSpecularPower(40);
                    actor->GetProperty()->SetOpacity(0.5);
                    actor->GetProperty()->SetEdgeVisibility(true);
                    actor->GetProperty()->SetEdgeColor(0.0, 0.1, 0.3);
                }
                else // row == 3
                {
                    // Wireframe-only with thick lines
                    actor->GetProperty()->SetRepresentationToWireframe();
                    actor->GetProperty()->SetColor(1.0, 0.6, 0.2);
                    actor->GetProperty()->SetLineWidth(3.0);
                    actor->GetProperty()->SetSpecular(0.0);
                }
            }

            allActors.push_back(actor);
        }
    }

    // ======================================================================
    // Add a large semi-transparent outer sphere at the center
    // ======================================================================
    auto outerSphere = CreateSphere(2.8, 120);

    vtkNew<vtkPolyDataMapper> outerMapper;
    outerMapper->SetInputConnection(outerSphere->GetOutputPort());

    vtkNew<vtkActor> outerActor;
    outerActor->SetMapper(outerMapper);
    outerActor->GetProperty()->SetColor(0.7, 0.85, 1.0);
    outerActor->GetProperty()->SetOpacity(0.15);
    outerActor->GetProperty()->SetSpecular(0.2);
    outerActor->GetProperty()->SetSpecularPower(30);
    outerActor->GetProperty()->SetEdgeVisibility(true);
    outerActor->GetProperty()->SetEdgeColor(0.3, 0.4, 0.6);
    outerActor->SetPosition(0.0, 0.0, 0.0);

    // ======================================================================
    // Renderer and RenderWindow setup
    // ======================================================================
    vtkNew<vtkRenderer> renderer;

    // Remove default headlight and add three-point lighting
    renderer->RemoveAllLights();
    renderer->AddLight(keyLight);
    renderer->AddLight(fillLight);
    renderer->AddLight(backLight);

    // Add all actors to the renderer
    for (auto& actor : allActors)
    {
        renderer->AddActor(actor);
    }
    renderer->AddActor(outerActor);

    // Enable depth peeling for transparent rendering
    renderer->SetUseDepthPeeling(true);
    renderer->SetMaximumNumberOfPeels(10);
    renderer->SetOcclusionRatio(0.0);

    // Background gradient: dark gray
    renderer->SetBackground(0.12, 0.12, 0.15);
    renderer->SetBackground2(0.05, 0.05, 0.08);
    renderer->SetGradientBackground(true);
    renderer->SetUseFXAA(true);  // Enable fast approximate anti-aliasing

    // Camera setup
    vtkCamera* camera = renderer->GetActiveCamera();
    camera->SetPosition(0.0, -6.0, 18.0);
    camera->SetFocalPoint(0.0, 0.0, 0.0);
    camera->SetViewUp(0.0, 1.0, 0.0);
    camera->SetClippingRange(0.5, 80.0);

    // ======================================================================
    // Render Window
    // ======================================================================
    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(1400, 1000);
    renderWindow->SetMultiSamples(8);     // 8x MSAA
    renderWindow->SetWindowName("Chapter 13: Advanced Rendering - Material Gallery");
    renderWindow->AddRenderer(renderer);

    // ======================================================================
    // Interactor
    // ======================================================================
    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    // Initialize and start the interaction loop
    renderWindow->Render();
    interactor->Initialize();
    interactor->Start();

    return 0;
}
```

### 13.5.3 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

# ============================================================================
# Chapter 13: Advanced Rendering Techniques
# File: CMakeLists.txt
# ============================================================================

project(MaterialGallery)

# --------------------------------------------------------------------------
# Find VTK 9.5.2
# --------------------------------------------------------------------------
find_package(VTK 9.5 REQUIRED COMPONENTS
    CommonCore
    CommonDataModel
    CommonExecutionModel
    FiltersSources
    ImagingCore
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
)

# If VTK_DIR is not set, user must provide it via -DVTK_DIR=/path/to/vtk/build
if(NOT VTK_FOUND)
    message(FATAL_ERROR "VTK 9.5 not found. Set -DVTK_DIR=/path/to/vtk/build")
endif()

message(STATUS "VTK version: ${VTK_VERSION}")
message(STATUS "VTK libraries: ${VTK_LIBRARIES}")

# --------------------------------------------------------------------------
# Executable
# --------------------------------------------------------------------------
add_executable(MaterialGallery MaterialGallery.cxx)

target_link_libraries(MaterialGallery PRIVATE
    VTK::CommonCore
    VTK::CommonDataModel
    VTK::CommonExecutionModel
    VTK::FiltersSources
    VTK::ImagingCore
    VTK::InteractionStyle
    VTK::RenderingCore
    VTK::RenderingOpenGL2
)

# --------------------------------------------------------------------------
# C++ standard
# --------------------------------------------------------------------------
target_compile_features(MaterialGallery PRIVATE cxx_std_17)

# --------------------------------------------------------------------------
# Platform-specific settings
# --------------------------------------------------------------------------
if(WIN32)
    # Suppress overly verbose MSVC warnings
    target_compile_definitions(MaterialGallery PRIVATE
        _CRT_SECURE_NO_WARNINGS
        NOMINMAX
    )
endif()
```

### 13.5.4 编译与运行说明

**编译步骤：**

```bash
# 1. 创建构建目录
mkdir build && cd build

# 2. 配置CMake（请将路径替换为你本地的VTK安装路径）
cmake .. -DVTK_DIR=/path/to/vtk-9.5.2/build

# 3. 编译
cmake --build . --config Release

# 4. 运行
./MaterialGallery        # Linux/macOS
MaterialGallery.exe      # Windows
```

**程序交互说明：**

- **鼠标左键拖拽**：旋转场景（轨迹球交互）
- **鼠标滚轮 / 鼠标右键拖拽**：缩放场景
- **鼠标中键拖拽**：平移场景
- **R键**：重置相机视角
- **E键**：退出程序

**预期效果：**

运行后你将看到一个5列4行的球体网格，每个列展示不同类别的材质参数：

- 第一列展示从红色到绿色的纯色材质变化，漫反射颜色依次为红、橙、黄、绿。
- 第二列展示从哑光到高亮金属的镜面反射渐变，注意观察高光区域从柔和大面积变为尖锐小面积的过程。
- 第三列展示四种不同的表现模式：点模式、线框模式、表面模式、带线框叠加的表面模式。
- 第四列展示红蓝棋盘格纹理与不同材质参数的组合效果。
- 第五列展示自发光、极限金属、半透明和厚线框等特殊效果。

一个大的半透明浅蓝色球壳包围着所有球体，你可以透过它看到内部的20个球体。旋转场景来观察半透明渲染在不同角度下的效果。

---

## 13.6 本章小结

本章系统地讲解了VTK中的四大高级渲染技术：材质属性、光照模型、纹理映射和透明渲染。这些技术共同构成了你创建专业级和出版物级别可视化作品的核心工具箱。

**一、材质属性（vtkProperty）**

你在13.1节学习了Phong光照模型的各个分量以及如何在VTK中控制它们：
- `SetDiffuseColor()`控制物体的基础颜色（漫反射分量）。
- `SetAmbientColor()`配置环境光反射色。
- `SetSpecularColor()`和`SetSpecularPower()`配置高光的颜色和锐度。
- `SetOpacity()`控制整体透明度。
- `SetRepresentation()`在点模式、线框模式和表面模式之间切换。
- `SetEdgeVisibility()`在表面上叠加网格线。

理解这些参数之间的相互作用是控制材质外观的关键。金属材质需要高镜面反射+高光泽度，哑光材质需要低镜面反射+低光泽度，自发光材质需要将环境光分量设得足够高。

**二、光照模型（vtkLight）**

你在13.2节学习了VTK光源系统的完整能力：
- 三种光源类型：头灯（随相机移动）、场景灯（固定在世界空间）、相机灯（在相机空间中固定偏移）。
- 多点布光：主光源+补光+背光源的三点式布光方案，这是影视摄影中沿用了近百年的经典技术。
- 光源颜色和强度分别控制色调和亮度。
- 距离衰减和锥形光源提供了更高级的光照控制。

三点式布光是需要记住的核心模式——它适用于绝大多数科学可视化场景，能够以最小的光源数量产生立体感和专业感。

**三、纹理映射（vtkTexture）**

你在13.3节学习了纹理映射的完整工作流程，从加载图像文件到程序化生成纹理：
- `vtkTexture`通过`SetInputConnection()`接收图像数据。
- 纹理坐标（TCoords）是几何数据中必需的组成部分。
- `SetInterpolate()`、`SetRepeat()`、`SetEdgeClamp()`控制纹理采样的行为细节。
- 程序化纹理（如棋盘格）可以在不依赖外部文件的情况下产生丰富的视觉效果。

纹理映射是几何复杂度与视觉丰富度之间的桥梁——用几张三角形加上高分辨率纹理，就能产生细节极其丰富的视觉效果。

**四、透明渲染**

你在13.4节学习了透明渲染的两种核心技术及其权衡：
- 深度剥离：通用、正确、但性能开销大。通过多遍渲染逐层剥离获取正确的透明合成。
- 深度排序：对单个Polydata的单元排序，更快但适用场景受限。
- 一般建议：对于质量优先的出图场景使用深度剥离，对于性能敏感的交互场景使用较少的剥离层数或深度排序。

**五、综合示例**

你在13.5节看到了一个完整的材质展示画廊程序，将本章的所有概念整合到一起。这个程序本身也是一个有用的参考——你可以修改其中的参数来观察不同设置对渲染效果的影响，帮助你建立对材质参数的直觉理解。

---

**继续学习建议：**

掌握了本章的高级渲染技术后，你的可视化作品将拥有更丰富的视觉表达。以下是后续章节中将涉及的相关主题：

- **体积渲染（Volume Rendering）**：将材质和光照概念扩展到三维体数据，使用传递函数直接对体积进行光学积分。
- **动画与关键帧**：利用本章的光照和材质控制来创建动态的可视化演示。
- **标注与文字**：为你的可视化场景添加文字标签、标题和图例。
- **剪切平面与等值面**：使用剪切操作来揭示数据内部结构，配合透明度控制可以产生强大的洞察效果。
