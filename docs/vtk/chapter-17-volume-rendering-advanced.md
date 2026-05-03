# 第十七章 体渲染高级技术 (Advanced Volume Rendering)

## 本章导读

在第十六章中，你第一次接触了体渲染（Volume Rendering）的基本概念——使用`vtkFixedPointVolumeRayCastMapper`进行光线投射、通过传递函数（Transfer Function）将标量值映射为颜色和不透明度、以及标量/梯度不透明度的基础配置。那些基础知识足以让你完成一次简单的体渲染：加载数据、设置颜色传递函数、渲染出一个半透明的立方体。

但是，如果你曾经尝试过将体渲染用于真实数据，你很快会遇到一系列实际问题：渲染速度慢得无法交互怎么办？体数据中的表面结构模糊不清怎么办？面对包含骨骼、皮肤、血管等多重组织的医学影像，如何设计传递函数来清晰地区分它们？GPU能带来多少性能提升？

本章就是为回答这些问题而写的。我们将深入到体渲染的核心机制中，系统地讲解四大高级主题：

1. **渲染质量与性能平衡**（17.1节）——采样距离、自适应采样、图像采样距离。你将理解体渲染"质量-速度"曲线的每一个控制点，并学会根据应用场景选择最优的参数组合。

2. **光照与着色**（17.2节）——在体渲染中开启光照计算、使用梯度不透明度来突出表面结构、配置环境光/漫反射/镜面反射参数。你将理解为什么开启Shade之后体数据中的骨骼和血管突然变得"立体"了。

3. **传递函数设计进阶**（17.3节）——多峰传递函数用于多材料数据集、利用直方图信息定位控制点、常见医学预设（骨骼/皮肤/血管造影/工业CT）、交互式传递函数Widget的简介。你将获得实用的传递函数设计方法论。

4. **GPU体渲染详解**（17.4节）——`vtkGPUVolumeRayCastMapper`的工作原理、硬件要求、与CPU的对比、多体渲染、性能优化策略。

此外，17.5节提供了一个完整的医学影像体渲染示例程序：生成合成脑部数据、实现MIP/复合/类等值面三种渲染模式、通过键盘在模式间切换、配置医学调优的传递函数、以及完整的CMakeLists.txt。

本章需要的前置知识包括第5章（渲染管道）、第9章（结构化数据——ImageData是体渲染的数据基础）、第12章（颜色映射——传递函数与标量范围的关系）以及第十六章（体渲染基础）。如果你对体渲染的基本概念（如传递函数、光线投射、alpha合成）还不太熟悉，建议先阅读第十六章。

---

## 17.1 渲染质量与性能平衡

体渲染本质上是一个计算密集型任务。对于每个像素，渲染器需要沿着穿过体数据的视线方向采样数十到数百个点，对每个采样点进行三线性插值以获取标量值，然后通过传递函数将标量值映射为颜色和不透明度，最后按从前到后的顺序合成这些颜色贡献。对于一个800x600的视口，使用200个采样步长，这意味着每次渲染需要进行约9600万次标量插值和传递函数查询。

控制计算量的最直接参数是**采样距离**（Sample Distance）。本节将系统讲解所有与采样质量相关的参数，以及它们如何影响渲染质量和交互帧率。

### 17.1.1 SetSampleDistance() -- 采样距离

采样距离是体渲染质量与性能之间最重要的调节旋钮。它决定了光线投射时沿视线方向的采样点间距。

```cpp
#include <vtkGPUVolumeRayCastMapper.h>

vtkNew<vtkGPUVolumeRayCastMapper> mapper;
mapper->SetSampleDistance(0.5);  // 采样间距为0.5个体素单位
```

**采样距离的含义：**

在光线投射过程中，从视点出发穿过体数据的每一条光线都会被等间距采样。采样距离（Sample Distance）指定了两个连续采样点之间的空间距离，单位是体素空间中的长度单位（通常与数据的spacing一致）。

- **较小的采样距离**（如0.3）：每个体素内可能有3-4个采样点。采样越密集，渲染结果越平滑细腻，丢失的细节越少。但采样点数量与采样距离成反比——采样距离减半意味着采样点数量翻倍，渲染时间也随之翻倍。
- **较大的采样距离**（如1.5）：每个采样点跨越多个体素。渲染速度快，但可能出现"条纹"或"阶梯"伪影，特别是在标量值急剧变化的区域（如组织边界）。

**采样距离的默认值：**

在VTK 9.5.2中，如果不设置采样距离，`vtkGPUVolumeRayCastMapper`会使用默认值1.0（即每个体素单位一个采样点）。这是一个折中的默认值——在典型数据上提供合理的质量同时保持可交互的帧率。

### 17.1.2 采样距离与体数据维度的关系

采样距离的"合理"值取决于体数据的空间分辨率和物理尺寸。一个经验法则是：

**采样距离应该与体数据中最小有意义特征的特征尺度相匹配。**

考虑以下场景：

| 数据类型 | 体素间距 | 推荐采样距离 | 说明 |
|---------|---------|-------------|------|
| 高分辨率CT (512x512x512) | 0.5 x 0.5 x 0.5 mm | 0.5 - 1.0 | 需要细采样以分辨亚毫米结构 |
| 常规CT (256x256x128) | 1.0 x 1.0 x 2.0 mm | 1.0 - 2.0 | 中等采样密度即可 |
| 低分辨率MRI (128x128x64) | 2.0 x 2.0 x 4.0 mm | 1.5 - 3.0 | 密集采样不会增加有效信息 |
| 海洋涡旋模拟 | 各向异性间距 | 与最小间距匹配 | 确保所有方向都充分采样 |

**数学关系：**

如果体数据的维度为$(N_x, N_y, N_z)$，spacing为$(s_x, s_y, s_z)$，则体数据的物理尺寸为$(N_x s_x, N_y s_y, N_z s_z)$。当采样距离为$\Delta s$时，一条跨越整个体对角线的光线的采样点数量约为：

$$N_{\text{samples}} \approx \frac{\sqrt{(N_x s_x)^2 + (N_y s_y)^2 + (N_z s_z)^2}}{\Delta s}$$

对于高分辨率数据，这个数字增长得非常快。一个512x512x512的体在采样距离为0.5时，最长光线的采样点数约为$\sqrt{(512 \cdot 0.5)^2 \times 3} / 0.5 \approx 886$个采样点。对于约800x600=48万像素的视口，这意味着约4.25亿次标量查询——这解释了为什么细采样距离下的交互式体渲染需要GPU。

### 17.1.3 AutoAdjustSampleDistances -- 自适应采样距离

VTK提供了一个"自动"模式，允许Mapper根据与相机之间的距离动态调整采样距离：

```cpp
mapper->AutoAdjustSampleDistancesOn();   // 开启自适应（默认行为）
mapper->AutoAdjustSampleDistancesOff();  // 关闭自适应
```

当自适应模式开启时，VTK会根据渲染器中的其他几何体的深度范围来自动设置采样距离。其背后的逻辑是：

- 如果体数据是场景中唯一的物体，且占据了视口的大部分像素，VTK使用`SetSampleDistance()`指定的值（或默认值1.0）。
- 如果场景中还有其他几何体（如等值面、裁剪平面指示器等），VTK会尝试调整采样距离，使得体数据的采样频率与场景中的深度范围相匹配。

**实际建议：**

在大多数场景中，保持`AutoAdjustSampleDistancesOn()`（默认）是合理的选择。但在以下情况中应该关闭它：

1. **当你需要精确控制采样质量时**——例如在出版物级别的离线渲染中，你需要保证每次渲染使用完全相同的采样参数。
2. **当体数据与几何面混合渲染且出现明显伪影时**——自适应机制有时会选择过大的采样距离，导致体数据的某些部分出现可见的采样不足。
3. **在动画中**——自适应采样距离可能在帧之间产生不一致，导致"闪烁"效果。

```cpp
// 精确控制的场景
mapper->AutoAdjustSampleDistancesOff();
mapper->SetSampleDistance(0.3);  // 固定使用细粒度采样
```

### 17.1.4 ImageSampleDistance -- 图像空间采样距离

除了沿光线的采样距离外，VTK还提供了一个**图像空间**的采样控制参数：

```cpp
mapper->SetImageSampleDistance(1.0);  // 默认值：每个像素一条光线
mapper->SetImageSampleDistance(2.0);  // 每隔一个像素投射一条光线
```

`ImageSampleDistance`控制屏幕空间中的光线密度。值为1.0时，每个像素投射一条光线；值为2.0时，每隔一个像素投射一条光线（总光线数减少到1/4）；值为3.0时，每三个像素投射一条光线（总光线数减少到1/9）。

当`ImageSampleDistance > 1.0`时，缺失的像素通过插值填充。这在交互操作期间（旋转/缩放）非常有用——你可以临时将`ImageSampleDistance`设为2.0或3.0来维持交互帧率，然后在交互停止时恢复到1.0进行最终渲染。

**实际使用场景：**

```cpp
// 交互期间：快速渲染
mapper->SetImageSampleDistance(2.5);
renderWindow->Render();

// 最终出图：全分辨率渲染
mapper->SetImageSampleDistance(1.0);
renderWindow->Render();
```

### 17.1.5 质量与性能权衡表

下面的表格总结了不同场景下的推荐参数配置，以及大致可以达到的交互帧率（以中端GPU为参考，如NVIDIA RTX 3060，512x512x512体数据）：

| 场景 | Sample Distance | Image Sample Distance | Shade | 光线数(1024x768) | 单帧时间 | 帧率 | 适用情况 |
|------|----------------|----------------------|-------|------------------|---------|------|---------|
| 实时交互（最低质量） | 2.0 | 2.0 | Off | ~196k | ~15ms | ~60fps | 快速导航和数据定位 |
| 实时交互（平衡质量） | 1.0 | 1.5 | Off | ~349k | ~30ms | ~33fps | 常规交互式探索 |
| 实时交互（高质量） | 0.7 | 1.0 | On | ~786k | ~50ms | ~20fps | 精细的交互式观察 |
| 半交互（高精度） | 0.5 | 1.0 | On | ~786k | ~70ms | ~14fps | 需要光照的细致检查 |
| 离线渲染（出版级） | 0.25 | 1.0 | On | ~786k | ~200ms+ | ~5fps | 出版物截图/海报 |
| 预览渲染 | 1.5 | 2.0 | Off | ~196k | ~10ms | ~100fps | 传递函数调整预览 |

**性能优化决策树：**

```
需要更快的渲染速度？
├── 步骤1: 增大 SampleDistance（1.0 -> 1.5 -> 2.0）
│   └── 注意：超过2.0时采样不足伪影会变得明显
├── 步骤2: 增大 ImageSampleDistance（1.0 -> 1.5 -> 2.0）
│   └── 注意：超过2.5时图像会明显模糊
├── 步骤3: 关闭光照（ShadeOff()）
│   └── 光照计算在每个采样点增加约20-30%计算量
├── 步骤4: 降低视口分辨率（更小的RenderWindow）
│   └── 800x600 vs 1920x1080：光线数减少约75%
└── 步骤5: 减少传递函数的不透明度峰值
    └── 更少的累积采样意味着提前光线终止概率更高
```

### 17.1.6 提前光线终止（Early Ray Termination）

提前光线终止是体渲染的一个重要优化机制，它在VTK中默认启用。其原理是：在光线从前到后穿过体数据的过程中，当累积的不透明度接近1.0时（即之后的所有采样点对最终颜色的贡献可以忽略不计），光线投射提前终止。

```cpp
// 提前终止由采样过程自动管理
// 你可以通过增加不透明度传递函数的不透明值来增加提前终止的频率
// 这无形中提高了渲染性能——不透明区域的"后面"不需要被采样
```

提前光线终止对性能的影响很大，特别是对于包含大面积不透明区域（如骨骼CT数据）的体数据。在极端情况下（如骨骼数据），光线的平均传播距离可能只有体对角线长度的20-30%，这意味着60-70%的采样计算被跳过。

---

## 17.2 光照与着色

在基础的体渲染中（`ShadeOff()`），每个采样点的颜色仅由传递函数决定——标量值映射到RGBA颜色，然后直接用于alpha合成。这种"无光照"的渲染模式计算速度快，但产生的图像缺乏深度感和表面信息。

开启光照（`ShadeOn()`）后，VTK在每个采样点额外计算局部光照，使用体数据的**梯度**（Gradient）作为"表面法线"的代理。这极大地增强了体数据中隐含表面的可视化效果。

### 17.2.1 ShadeOn() 与 ShadeOff()

光照开关位于`vtkVolumeProperty`上：

```cpp
#include <vtkVolumeProperty.h>

vtkNew<vtkVolumeProperty> volumeProperty;

// 开启光照——计算漫反射和镜面反射分量
volumeProperty->ShadeOn();

// 关闭光照——仅使用传递函数颜色（默认）
volumeProperty->ShadeOff();
```

**ShadeOff（无光照模式）：**

在无光照模式下，每个采样点的最终颜色直接取自颜色传递函数在该标量值处的RGBA值。这相当于假设体数据"自发光"——每个体素独立地发出其传递函数指定的颜色。无光照渲染的优势在于速度快，且传递函数的颜色设计可以非常直观（你设置的颜色就是你看到的颜色）。

**ShadeOn（光照模式）：**

在光照模式下，每个采样点的颜色计算变为：

$$C_{\text{final}} = C_{\text{ambient}} \cdot k_a + C_{\text{diffuse}} \cdot k_d \cdot (\mathbf{N} \cdot \mathbf{L}) + C_{\text{specular}} \cdot k_s \cdot (\mathbf{R} \cdot \mathbf{V})^n$$

其中$\mathbf{N}$是梯度方向（表面法线的代理），$\mathbf{L}$是光源方向，$\mathbf{R}$是反射方向，$\mathbf{V}$是视线方向，$n$是镜面反射指数。

光照计算的结果乘以传递函数颜色，使面向光源的区域更亮，背离光源的区域更暗。这种明暗变化为体数据中的"隐含表面"提供了立体感和深度线索。

**关键直觉：为什么光照有助于体渲染？**

考虑一个包含骨骼的CT数据。在无光照模式下，所有骨骼体素都呈现相同的白色（传递函数映射）。骨骼的内部和表面看起来完全一样——你只能看到一个白色的块状区域。开启光照后，骨骼的"表面"区域（梯度较大的地方，即从软组织到骨骼的过渡区）会根据梯度方向与光源的关系产生明暗变化。骨骼的内部（梯度较小，周围都是相似的骨骼体素）则保持均匀的亮度。结果就是：骨骼的边界突然变得清晰可见，三维形状一目了然。

### 17.2.2 梯度不透明度 -- 表面强调

传递函数中的**梯度不透明度**（Gradient Opacity）是体渲染中最强大但常被忽视的特性之一。标量不透明度（Scalar Opacity）根据标量值控制不透明度，而梯度不透明度根据**梯度幅值**（Gradient Magnitude）来**调节**不透明度。

```cpp
// 添加梯度不透明度控制点
// AddGradientOpacityPoint(gradientMagnitude, opacity)
volumeProperty->AddGradientOpacityPoint(0.0, 0.0);    // 零梯度：不透明
volumeProperty->AddGradientOpacityPoint(50.0, 0.3);   // 中等梯度：30%不透明度
volumeProperty->AddGradientOpacityPoint(200.0, 1.0);  // 大梯度：完全不透明
```

**梯度不透明度的工作原理：**

体数据的梯度幅值在标量值快速变化的区域（组织边界、材料界面）达到最大值，在均匀区域（组织内部）接近零。通过将梯度不透明度设置为在低梯度区域低、在高梯度区域高，你可以选择性地"衰减"均匀区域的不透明度贡献，同时保持边界区域的不透明度。

最终的采样点不透明度为：

$$\alpha_{\text{final}} = \alpha_{\text{scalar}}(s) \times \alpha_{\text{gradient}}(\|\nabla s\|)$$

其中$\alpha_{\text{scalar}}(s)$是标量传递函数给出的不透明度，$\alpha_{\text{gradient}}(\|\nabla s\|)$是梯度传递函数给出的调制因子，$s$是标量值，$\|\nabla s\|$是梯度幅值。

**典型应用场景：**

| 场景 | 梯度不透明度曲线 | 效果 |
|------|----------------|------|
| 骨骼成像 | 低梯度->0, 高梯度->1 | 骨骼表面清晰，内部结构可见（因为内部梯度低，透明度高） |
| 血管造影 | 低梯度->0.1, 高梯度->1 | 血管壁突出显示，血液内部半透明 |
| 软组织检查 | 低梯度->1, 高梯度->1 | 无梯度衰减——所有组织均匀显示 |
| 工业CT（缺陷检测） | 低梯度->0.4, 高梯度->1 | 均匀材料半透明，裂纹/孔洞边界突出 |
| "类等值面"效果 | 低梯度->0, 高梯度->1 | 仅显示表面附近的体素，模拟等值面渲染 |

**梯度幅值的合理范围：**

梯度幅值的数值范围取决于数据的标量值范围和spacing。对于一个标量值在[0, 1000]范围内的CT数据（HU单位），梯度幅值可能在[0, 500]或更高的范围内。你需要根据实际数据的梯度分布来设置梯度不透明度的控制点。一个实用的方法是：

1. 先用`ShadeOn()`渲染一次。
2. 观察体数据中哪些区域（表面边界）被过度强调或不充分强调。
3. 调整梯度不透明度曲线——增大高梯度端的不透明度来加强表面效果，增大低梯度端的不透明度来恢复均匀区域的可见性。

### 17.2.3 局部光照参数

`vtkVolumeProperty`提供了完整的Phong光照模型参数控制，与你在第十三章学习的`vtkProperty`的材质参数在概念上是相同的，但应用于体渲染的每个采样点而非多边形表面：

```cpp
vtkNew<vtkVolumeProperty> volumeProperty;

// 环境光系数 -- 控制环境光分量的强度（0.0到1.0，默认0.0）
volumeProperty->SetAmbient(0.2);

// 漫反射系数 -- 控制漫反射分量的强度（0.0到1.0，默认1.0）
volumeProperty->SetDiffuse(0.9);

// 镜面反射系数 -- 控制镜面反射分量的强度（0.0到1.0，默认0.0）
volumeProperty->SetSpecular(0.4);

// 镜面反射指数 -- 控制高光的集中度（默认10.0，较大值=更锐利的高光）
volumeProperty->SetSpecularPower(30.0);
```

**环境光（Ambient）：**

环境光模拟间接光照——从场景各处均匀地照亮所有采样点，无论其梯度方向如何。增大环境光系数可以提高体数据的整体亮度，但过高的环境光会"洗掉"光照产生的表面细节。对于体渲染，推荐使用较低的环境光值（0.1-0.3），因为过高的环境光分量会导致体数据内部和表面看起来一样亮，失去了ShadeOn()的意义。

**漫反射（Diffuse）：**

漫反射是体渲染光照的主要分量。漫反射强度取决于梯度方向与光源方向的夹角——梯度指向光源时（"面向"光源的表面）得到最亮的值，梯度背离光源时得到最暗的值。漫反射系数通常设置为较高的值（0.7-1.0）以确保良好的方向感。

**镜面反射（Specular）：**

镜面反射产生高光效果——在梯度方向使得反射光恰好进入观察者眼睛的位置产生亮斑。在体渲染中，镜面反射高光有助于：
- 增强曲面的感知（如股骨头的球形曲面）。
- 区分不同材料（骨骼比软组织有更强的镜面反射）。
- 增强整体画面的立体感和"3D渲染感"。

在医学体渲染中，适度的镜面反射（0.3-0.6）配合较高的镜面反射指数（20-50）可以产生类似"湿润骨骼标本"的专业外观。

### 17.2.4 梯度计算与法线方向

在体渲染的光照计算中，"法线方向"由体数据的**梯度**（Gradient）来近似。梯度是一个向量，指向标量值增加最快的方向：

$$\mathbf{g} = \nabla s = \left(\frac{\partial s}{\partial x}, \frac{\partial s}{\partial y}, \frac{\partial s}{\partial z}\right)$$

在VTK中，梯度通过中心差分法（Central Difference）在每个采样点近似计算：使用采样点前后各0.5个采样距离处的标量值来计算偏导数。

**梯度方向作为法线的物理意义：**

在体数据中，标量值从一个材料过渡到另一个材料时急剧变化——这正好发生在材料界面上。在骨骼-软组织界面，CT值从约1000 HU（骨骼）急剧下降到约50 HU（软组织），梯度向量指向从软组织到骨骼的方向（即CT值增加最快的方向，垂直于界面向外）。

这个梯度方向垂直于材料界面，因此它可以作为"表面法线"用于光照计算。这解释了为什么开启`ShadeOn()`后体渲染中的表面结构变得明显——只有在标量值急剧变化的地方（材料界面），梯度方向才明确，光照计算才产生明显的明暗变化。

**梯度和光照的协同工作：**

```
无Shade的体渲染：
  每个采样点 = 传递函数颜色 + alpha合成
  -> 表面边界模糊，缺乏三维感

有Shade的体渲染：
  每个采样点 = (传递函数颜色) * (环境光 + 漫反射(n·l) + 镜面反射(r·v)^n) + alpha合成
  其中 n = gradient / |gradient|
  -> 表面边界清晰，三维立体感强
```

---

## 17.3 传递函数设计进阶

传递函数是体渲染的"灵魂"。一个精心设计的传递函数可以将同一组体数据呈现为完全不同的可视化结果——从透明软组织包裹的半透明骨骼（骨骼模式），到仅显示血管的血管造影模式，再到高对比度的工业缺陷检测模式。

第十六章介绍了传递函数的基本API（`AddRGBPoint()`和`AddScalarOpacityPoint()`）。本节将深入探讨传递函数设计的方法论和实用技巧。

### 17.3.1 多峰传递函数 -- 多材料数据集的分离

现实世界的体数据通常包含多种材料（组织类型、物质相态），每种材料占据标量值轴上的一个特定区间。多峰传递函数（Multi-Peak Transfer Function）的目标是为每种材料分配独立的不透明度峰值，从而在渲染中清晰地区分它们。

**医学CT数据的材料区间（典型HU值范围）：**

| 材料 | HU值范围 | 视觉编码建议 |
|------|---------|-------------|
| 空气 | -1000 至 -800 | 完全透明 |
| 肺组织 | -800 至 -400 | 低不透明度，浅蓝灰 |
| 脂肪 | -100 至 -50 | 中低不透明度，浅黄 |
| 水/软组织 | -10 至 40 | 中不透明度，肤色 |
| 肌肉/肝脏 | 40 至 80 | 中高不透明度，红色系 |
| 造影剂增强血管 | 100 至 300 | 高不透明度，亮红/白色 |
| 松质骨 | 300 至 600 | 高不透明度，浅棕 |
| 密质骨 | 600 至 1500 | 最高不透明度，白色/象牙 |
| 金属植入物 | 2000+ | 最高不透明度，亮白 |

**多峰传递函数示例代码：**

```cpp
#include <vtkPiecewiseFunction.h>
#include <vtkColorTransferFunction.h>

// ---- 标量不透明度传递函数（多峰设计） ----
vtkNew<vtkPiecewiseFunction> scalarOpacity;

// 空气：完全透明
scalarOpacity->AddPoint(-1000.0, 0.0);
scalarOpacity->AddPoint(-800.0, 0.0);

// 软组织：低不透明度（包围但不遮挡骨骼）
scalarOpacity->AddPoint(-50.0, 0.0);
scalarOpacity->AddPoint(40.0, 0.08);     // 软组织峰值
scalarOpacity->AddPoint(80.0, 0.02);

// 造影剂增强血管：高不透明度（突出显示）
scalarOpacity->AddPoint(90.0, 0.0);
scalarOpacity->AddPoint(200.0, 0.7);     // 血管峰值
scalarOpacity->AddPoint(300.0, 0.1);

// 骨骼：最高不透明度
scalarOpacity->AddPoint(350.0, 0.0);
scalarOpacity->AddPoint(700.0, 1.0);     // 骨骼峰值
scalarOpacity->AddPoint(1500.0, 1.0);

// ---- 颜色传递函数（与材料区间配套） ----
vtkNew<vtkColorTransferFunction> colorTF;

// 空气区间：不可见（颜色无关紧要）
colorTF->AddRGBPoint(-1000.0, 0.0, 0.0, 0.0);

// 软组织区间：温暖的肤色
colorTF->AddRGBPoint(-50.0, 0.9, 0.7, 0.5);
colorTF->AddRGBPoint(40.0, 1.0, 0.8, 0.6);

// 血管区间：亮红色
colorTF->AddRGBPoint(100.0, 0.8, 0.1, 0.1);
colorTF->AddRGBPoint(250.0, 1.0, 0.2, 0.1);

// 骨骼区间：象牙白
colorTF->AddRGBPoint(350.0, 0.9, 0.8, 0.7);
colorTF->AddRGBPoint(700.0, 1.0, 0.95, 0.85);
colorTF->AddRGBPoint(1500.0, 1.0, 1.0, 1.0);
```

**多峰传递函数的设计原则：**

1. **峰谷分离**：在不同材料的标量值区间之间设置"谷"（低不透明度区域），形成一个透明度"窗口"，让每个材料区间在视觉上分离。
2. **重要材料高峰值**：对需要突出显示的材料（如骨折线、肿瘤、血管）使用较高的不透明度峰值。
3. **上下文材料低峰值**：对提供解剖上下文的材料（如软组织、骨骼轮廓）使用较低的不透明度，形成半透明的背景层。
4. **颜色关联**：为每种材料区间使用不同的颜色（或同一色调的不同饱和度/亮度），使观察者可以一目了然地区分材料类型。

### 17.3.2 利用直方图信息定位控制点

盲目地在标量值轴上放置控制点是低效的。**标量直方图**（Scalar Histogram）告诉你数据中标量值的分布——哪些值区间包含大量体素，哪些值区间几乎是空的。利用直方图信息，你可以将传递函数的控制点精确地放置在数据中实际存在材料的标量值区间上。

**获取体数据的直方图：**

```cpp
#include <vtkImageHistogram.h>

// 计算标量直方图
vtkNew<vtkImageHistogram> histogram;
histogram->SetInputConnection(reader->GetOutputPort());
histogram->SetNumberOfBins(256);                 // 256个分箱
histogram->SetAutoRange(true);                   // 自动确定范围
histogram->Update();

// 获取直方图数据
vtkImageData* histOutput = histogram->GetOutput();
int* bins = static_cast<int*>(histOutput->GetScalarPointer());

// 打印直方图峰值位置
double range[2];
histOutput->GetScalarRange(range);
double binWidth = (range[1] - range[0]) / 256.0;

std::cout << "Histogram peaks (bins with > 1% of total voxels):" << std::endl;
for (int i = 0; i < 256; ++i)
{
    double ratio = static_cast<double>(bins[i]) / (dataDims[0] * dataDims[1] * dataDims[2]);
    if (ratio > 0.01)
    {
        std::cout << "  Value " << (range[0] + i * binWidth)
                  << " : " << (ratio * 100) << "%" << std::endl;
    }
}
```

**基于直方图的控制点放置策略：**

1. **找到直方图的峰值**——每个峰值对应一种主要的材料类型。
2. **在峰值位置放置不透明度控制点**——不透明度值与峰的面积（材料占比）相关。主要材料使用高不透明度，次要材料使用较低的不透明度。
3. **在直方图的谷值位置放置不透明度为零的控制点**——这些标量值区间几乎没有体素，是不同材料之间的自然分隔。
4. **颜色控制点与不透明度控制点对齐**——在相同的关键标量值处设置颜色，确保颜色变化与不透明度变化一致。

### 17.3.3 常见场景的传递函数预设

以下是经过实践验证的传递函数预设，适用于常见的体渲染场景。这些预设可以直接作为起点，然后根据具体数据进行微调。

**预设1：骨骼模式（Bone Preset）——用于CT骨骼检查**

```cpp
void ApplyBonePreset(vtkPiecewiseFunction* opacity,
                     vtkColorTransferFunction* color,
                     double scalarRange[2])
{
    double minVal = scalarRange[0];
    double maxVal = scalarRange[1];

    // 不透明度：骨骼高不透明，软组织几乎透明
    opacity->RemoveAllPoints();
    opacity->AddPoint(minVal, 0.0);
    opacity->AddPoint(150.0, 0.0);      // 软组织以下透明
    opacity->AddPoint(300.0, 0.15);     // 过渡区
    opacity->AddPoint(600.0, 0.85);     // 松质骨
    opacity->AddPoint(1000.0, 1.0);     // 密质骨
    opacity->AddPoint(maxVal, 1.0);

    // 颜色：象牙白
    color->RemoveAllPoints();
    color->AddRGBPoint(minVal, 0.0, 0.0, 0.0);
    color->AddRGBPoint(300.0, 0.85, 0.75, 0.60);
    color->AddRGBPoint(600.0, 0.95, 0.90, 0.80);
    color->AddRGBPoint(maxVal, 1.0, 1.0, 1.0);
}
```

**预设2：皮肤/软组织模式（Skin Preset）——用于CT表面重建**

```cpp
void ApplySkinPreset(vtkPiecewiseFunction* opacity,
                     vtkColorTransferFunction* color,
                     double scalarRange[2])
{
    double minVal = scalarRange[0];
    double maxVal = scalarRange[1];

    // 不透明度：软组织区间有较高不透明度
    opacity->RemoveAllPoints();
    opacity->AddPoint(minVal, 0.0);
    opacity->AddPoint(-500.0, 0.0);     // 肺部透明
    opacity->AddPoint(-200.0, 0.05);
    opacity->AddPoint(0.0, 0.3);        // 皮肤/软组织峰值
    opacity->AddPoint(80.0, 0.15);
    opacity->AddPoint(200.0, 0.05);     // 骨骼开始变透明
    opacity->AddPoint(maxVal, 0.0);     // 骨骼区域隐藏

    // 颜色：温暖的皮肤色调
    color->RemoveAllPoints();
    color->AddRGBPoint(minVal, 0.0, 0.0, 0.0);
    color->AddRGBPoint(-200.0, 0.7, 0.5, 0.35);
    color->AddRGBPoint(0.0, 1.0, 0.75, 0.55);
    color->AddRGBPoint(80.0, 1.0, 0.85, 0.7);
    color->AddRGBPoint(maxVal, 1.0, 1.0, 1.0);
}
```

**预设3：血管造影模式（Angiography Preset）**

```cpp
void ApplyAngiographyPreset(vtkPiecewiseFunction* opacity,
                            vtkColorTransferFunction* color,
                            double scalarRange[2])
{
    double minVal = scalarRange[0];
    double maxVal = scalarRange[1];

    // 不透明度：造影剂增强区间（100-300 HU）高不透明
    opacity->RemoveAllPoints();
    opacity->AddPoint(minVal, 0.0);
    opacity->AddPoint(90.0, 0.0);       // 造影前透明
    opacity->AddPoint(130.0, 0.8);      // 造影剂峰值
    opacity->AddPoint(300.0, 0.3);
    opacity->AddPoint(400.0, 0.0);      // 骨骼区域透明
    opacity->AddPoint(maxVal, 0.0);

    // 颜色：亮红色血管
    color->RemoveAllPoints();
    color->AddRGBPoint(minVal, 0.0, 0.0, 0.0);
    color->AddRGBPoint(100.0, 0.8, 0.1, 0.1);
    color->AddRGBPoint(200.0, 1.0, 0.15, 0.1);
    color->AddRGBPoint(300.0, 1.0, 0.3, 0.2);
    color->AddRGBPoint(maxVal, 1.0, 1.0, 1.0);
}
```

**预设4：工业CT缺陷检测模式**

```cpp
void ApplyIndustrialCTPreset(vtkPiecewiseFunction* opacity,
                             vtkColorTransferFunction* color,
                             double scalarRange[2])
{
    double minVal = scalarRange[0];
    double maxVal = scalarRange[1];

    // 不透明度：材料主体半透明，缺陷（低密度区）突出
    opacity->RemoveAllPoints();
    opacity->AddPoint(minVal, 0.0);
    // 空腔/缺陷区间：高不透明度，便于发现
    opacity->AddPoint(minVal + 0.0  * (maxVal - minVal), 0.0);
    opacity->AddPoint(minVal + 0.05 * (maxVal - minVal), 0.9);  // 缺陷区
    opacity->AddPoint(minVal + 0.2  * (maxVal - minVal), 0.05); // 过渡
    // 材料主体：低不透明度
    opacity->AddPoint(minVal + 0.4  * (maxVal - minVal), 0.15);
    opacity->AddPoint(minVal + 0.7  * (maxVal - minVal), 0.3);
    opacity->AddPoint(minVal + 0.85 * (maxVal - minVal), 0.5);
    opacity->AddPoint(maxVal, 0.7);

    // 颜色：缺陷红色，材料灰色
    color->RemoveAllPoints();
    color->AddRGBPoint(minVal, 0.0, 0.0, 0.0);
    color->AddRGBPoint(minVal + 0.05 * (maxVal - minVal), 1.0, 0.1, 0.1);  // 缺陷红色
    color->AddRGBPoint(minVal + 0.2  * (maxVal - minVal), 0.75, 0.75, 0.75); // 过渡
    color->AddRGBPoint(minVal + 0.5  * (maxVal - minVal), 0.6, 0.6, 0.65);   // 材料灰蓝
    color->AddRGBPoint(maxVal, 0.9, 0.9, 0.95);  // 高密度区亮色
}
```

### 17.3.4 交互式传递函数Widget

VTK提供了`vtkVolumePropertyWidget`（在VTK 9.x中通过`vtkAbstractWidget`体系支持），允许用户在渲染窗口中交互式地调整传递函数的控制点。这极大地加速了传递函数的设计过程——你不再需要反复修改代码、编译、运行来观察效果。

**基本用法：**

```cpp
#include <vtkVolumePropertyWidget.h>
#include <vtkBiDimensionalWidget.h>  // 类似的Widget模式

// vtkVolumePropertyWidget的使用模式（示意）
// 注：VTK 9.5.2中具体的传递函数Widget API可能因版本而异
// 以下展示的是概念层面的用法

// 创建传递函数Widget
vtkNew<vtkVolumePropertyWidget> tfWidget;
tfWidget->SetInteractor(interactor);
tfWidget->SetVolumeProperty(volumeProperty);

// 启用Widget
tfWidget->On();
```

需要注意的是，`vtkVolumePropertyWidget`是一个相对较新的组件，在VTK 9.5.2中可能需要通过以下方式使用：

```cpp
// 通过vtkBoxWidget2等通用Widget间接控制传递函数
// 或者使用ParaView等更高级的工具进行交互式传递函数设计

// 作为替代方案，你可以自己实现简单的键盘驱动传递函数调整：
// - '+'/'-' 调整当前选中控制点的不透明度
// - 上下箭头 移动当前选中控制点的标量位置
// - 'c' 切换控制点
```

对于需要交互式传递函数设计的复杂场景，推荐使用ParaView（基于VTK的可视化应用程序），它提供了完整的传递函数编辑器（Transfer Function Editor），支持拖拽控制点、实时预览、预设导入导出等功能。ParaView的传递函数编辑器是理解传递函数设计的最佳工具。

### 17.3.5 传递函数设计工作流程

一个系统的传递函数设计工作流程如下：

```
步骤1：分析数据
├── 获取标量值范围（GetScalarRange）
├── 计算直方图（vtkImageHistogram）
├── 识别材料类型及其标量值区间
└── 确定目标：哪些材料需要突出显示？

步骤2：设置不透明度
├── 在每种目标材料的标量区间上设置不透明度峰值
├── 在不感兴趣的区域设置低不透明度或零不透明度
├── 在材料区间之间设置"谷值"实现视觉分离
└── 考虑梯度不透明度来强调表面

步骤3：设置颜色
├── 为每种材料分配区分度高的颜色
├── 在颜色过渡区放置中间控制点确保平滑过渡
├── 高不透明度区域使用更饱和的颜色
└── 考虑色盲友好的配色方案

步骤4：迭代调整
├── 渲染并评估：目标材料是否清晰可见？
├── 上下文材料是否适当？（过多会遮挡目标）
├── 颜色是否有足够的区分度？
└── 调整并重复，直到满意为止
```

---

## 17.4 GPU体渲染详解

在VTK 9.5.2中，`vtkGPUVolumeRayCastMapper`是交互式体渲染的首选Mapper。它利用GPU的大规模并行计算能力，将光线投射算法映射到GPU着色器上执行，实现了比CPU端Mapper快1-2个数量级的渲染速度。

### 17.4.1 vtkGPUVolumeRayCastMapper -- 交互式体渲染的主力

```cpp
#include <vtkGPUVolumeRayCastMapper.h>
#include <vtkVolume.h>
#include <vtkVolumeProperty.h>

// 创建GPU体渲染Mapper
vtkNew<vtkGPUVolumeRayCastMapper> mapper;
mapper->SetInputConnection(reader->GetOutputPort());

// 创建Volume（体渲染的"Actor"等价物）
vtkNew<vtkVolume> volume;
volume->SetMapper(mapper);
volume->SetProperty(volumeProperty);

// 添加到渲染器
renderer->AddVolume(volume);
```

**GPU Mapper的核心优势：**

1. **并行光线投射**：每条像素光线在GPU上独立并行处理。现代GPU拥有数千个CUDA核心/流处理器，可以同时处理数千条光线。相比之下，CPU端的Mapper只能在有限的核心上并行（或串行处理）。

2. **硬件三线性插值**：GPU纹理单元提供了硬件加速的三线性插值——在体渲染中每个采样点都需要进行的三线性插值被GPU免费执行。在CPU上，三线性插值需要约20-30次浮点运算和大量内存访问。

3. **片元着色器中的传递函数查询**：传递函数被存储为1D纹理（依赖纹理/Dependent Texture），在GPU片元着色器中查询传递函数的延迟极低。

4. **提前终止**：GPU支持片元级的提前终止（通过discard或深度测试），不需要渲染完全被遮挡的片元。

### 17.4.2 硬件要求

| 要求 | 最低配置 | 推荐配置 |
|------|---------|---------|
| OpenGL版本 | 3.2 或更高 | 4.5+ |
| GPU显存 | 1 GB | 4 GB+ |
| OpenGL扩展 | GL_ARB_texture_float, GL_EXT_gpu_shader4 | GL_ARB_shader_image_load_store |
| 纹理最大尺寸 | 至少2048x2048x2048（3D纹理） | 支持大于数据尺寸的3D纹理 |
| 驱动版本 | 较新的稳定驱动 | 最新稳定驱动 |

**显存估算：**

一个512x512x512的16位标量体数据占用的GPU显存为：
$$512 \times 512 \times 512 \times 2 \text{ bytes} = 256 \text{ MB}$$

加上渲染目标、传递函数纹理、深度缓冲等开销，总显存占用约为数据大小的1.5-2倍。因此，1GB显存可以容纳约300-500MB的体数据（相当于约400x400x400的16位数据或512x512x256的数据）。

### 17.4.3 支持的特性与CPU回退

| 特性 | GPU Mapper | CPU Mapper (FixedPoint) |
|------|-----------|------------------------|
| 标量不透明度传递函数 | 支持 | 支持 |
| 颜色传递函数 | 支持 | 支持 |
| 梯度不透明度 | 支持 | 支持 |
| 光照（Shade） | 支持 | 支持 |
| 多分量数据 | 支持（2-4分量） | 支持 |
| 裁剪平面 | 支持（最多6个） | 支持 |
| 混合渲染模式(MIP等) | 支持 | 支持 |
| 最大体数据尺寸 | GPU纹理限制 | 无硬性限制（但极慢） |
| 渲染速度 | 快（10-60fps典型值） | 慢（1-10fps典型值） |
| 内存使用 | GPU显存 | 系统RAM |
| 跨平台 | 需要OpenGL 3.2+ | 任何平台 |
| 精度 | float（32-bit） | fixed-point（12-bit） |

**CPU回退：**

当GPU Mapper不可用时（如系统不支持必需的OpenGL扩展），VTK会尝试回退到CPU端的Mapper。你可以通过以下代码检查是否正在使用GPU渲染：

```cpp
#include <vtkGPUVolumeRayCastMapper.h>

vtkGPUVolumeRayCastMapper* gpuMapper =
    vtkGPUVolumeRayCastMapper::SafeDownCast(mapper.Get());
if (gpuMapper)
{
    std::cout << "Using GPU volume rendering." << std::endl;
}
else
{
    std::cout << "GPU volume rendering not available. "
              << "Falling back to CPU mapper." << std::endl;
}
```

### 17.4.4 多体渲染 -- 重叠体数据

VTK支持在同一个场景中渲染多个重叠的体数据。这对于以下场景非常有用：

- **术前/术后对比**：将术前CT和术后CT数据叠加在同一视图中。
- **多模态融合**：将PET（功能信息）叠加在CT（解剖信息）之上。
- **部分体数据组合**：将来自不同扫描的体数据片段组合成完整的视图。

```cpp
// 创建两个独立的体数据
vtkNew<vtkGPUVolumeRayCastMapper> mapper1;
mapper1->SetInputConnection(reader1->GetOutputPort());

vtkNew<vtkVolume> volume1;
volume1->SetMapper(mapper1);
volume1->SetProperty(property1);

vtkNew<vtkGPUVolumeRayCastMapper> mapper2;
mapper2->SetInputConnection(reader2->GetOutputPort());

vtkNew<vtkVolume> volume2;
volume2->SetMapper(mapper2);
volume2->SetProperty(property2);

// 将两个体都添加到同一个渲染器
renderer->AddVolume(volume1);
renderer->AddVolume(volume2);
```

**多体渲染的注意事项：**

1. **渲染顺序**：体数据按照添加到渲染器的顺序进行渲染。后添加的体渲染在前一个体之上。
2. **深度剥离**：为了正确处理多个体之间的透明度关系，建议在渲染器上启用深度剥离。
3. **性能**：每个额外的体数据都会显著增加渲染开销。两个512x512x512的体数据在GPU上的渲染时间大约是单个体的1.8-2.0倍。
4. **显存**：每个体数据在GPU中都需要分配独立的3D纹理。两个512x512x512的体数据占用约512MB显存（仅纹理）。

### 17.4.5 性能优化策略

**策略1：减少有效体范围（SubVolume Extraction）**

只渲染感兴趣的区域（ROI），忽略其余部分：

```cpp
#include <vtkExtractVOI.h>

// 提取感兴趣的子体积
vtkNew<vtkExtractVOI> extractVOI;
extractVOI->SetInputConnection(reader->GetOutputPort());
extractVOI->SetVOI(xmin, xmax, ymin, ymax, zmin, zmax);
extractVOI->Update();

mapper->SetInputConnection(extractVOI->GetOutputPort());
```

子体积提取的效果是线性的：提取整体积的50%意味着约50%的渲染时间减少（实际略少，因为光线穿越距离也缩短了）。

**策略2：使用LOD（Level of Detail）**

在交互期间使用较低分辨率的体数据，静止时切换到全分辨率：

```cpp
// 交互状态检测
class LODInteractorStyle : public vtkInteractorStyleTrackballCamera
{
public:
    static LODInteractorStyle* New();
    vtkTypeMacro(LODInteractorStyle, vtkInteractorStyleTrackballCamera);

    void SetMapper(vtkGPUVolumeRayCastMapper* m) { mapper = m; }

    void OnLeftButtonDown() override
    {
        // 交互开始：降低质量
        if (mapper)
        {
            mapper->SetSampleDistance(2.0);
            mapper->SetImageSampleDistance(2.0);
        }
        vtkInteractorStyleTrackballCamera::OnLeftButtonDown();
    }

    void OnLeftButtonUp() override
    {
        // 交互结束：恢复高质量
        if (mapper)
        {
            mapper->SetSampleDistance(0.5);
            mapper->SetImageSampleDistance(1.0);
        }
        vtkInteractorStyleTrackballCamera::OnLeftButtonUp();
        this->Interactor->GetRenderWindow()->Render();
    }

private:
    vtkGPUVolumeRayCastMapper* mapper = nullptr;
};
vtkStandardNewMacro(LODInteractorStyle);
```

**策略3：优化传递函数以加速提前终止**

设计传递函数时，在数据的外围区域（包围盒附近）使用高不透明度，可以触发提前终止，减少在"空"区域中的采样次数。

**策略4：使用合适的渲染模式**

对于仅需要最大/最小强度投影的场景，使用`SetBlendModeToMaximumIntensity()`或`SetBlendModeToMinimumIntensity()`比默认的复合模式（Composite）更快，因为它们不需要完整的从前到后alpha合成。

```cpp
mapper->SetBlendModeToMaximumIntensity();  // MIP：更快
mapper->SetBlendModeToComposite();         // 默认复合模式：标准体渲染
mapper->SetBlendModeToMinimumIntensity();  // MinIP
mapper->SetBlendModeToAdditive();          // 加性混合
```

**策略5：减少视口尺寸**

在交互式探索期间使用较小的渲染窗口（如800x600而非1920x1200），在最终渲染或截图时再扩展到全分辨率。

**策略6：使用`vtkSmartVolumeMapper`自动选择最优Mapper**

```cpp
#include <vtkSmartVolumeMapper.h>

vtkNew<vtkSmartVolumeMapper> smartMapper;
smartMapper->SetInputConnection(reader->GetOutputPort());
// vtkSmartVolumeMapper会自动选择GPU Mapper（如果可用），
// 否则回退到CPU Mapper
```

`vtkSmartVolumeMapper`是一个智能代理：它检测系统是否支持GPU体渲染，如果支持则使用`vtkGPUVolumeRayCastMapper`，否则回退到`vtkFixedPointVolumeRayCastMapper`。对于需要跨平台兼容性的应用程序，这是推荐的选择。

---

## 17.5 代码示例：医学影像体渲染

本节提供一个完整的C++程序，展示如何构建一个功能完备的医学影像体渲染应用程序。程序使用合成脑部数据（由多个椭球体和球体组合创建，模拟简化的脑部结构），支持四种渲染模式之间的实时切换。

### 17.5.1 程序概述

**功能特性：**

1. 程序化生成合成脑部数据——包含"颅骨"、"脑组织"、"脑室"、"肿瘤"四层结构
2. 四种体渲染模式：
   - **模式1 (MIP)**: 最大强度投影，适合快速骨骼概览
   - **模式2 (Composite)**: 标准复合体渲染，展示所有组织的空间关系
   - **模式3 (IsoSurface-like)**: 类等值面效果，仅将狭窄标量区间设为不透明，模拟等值面外观
   - **模式4 (Bone-only)**: 仅显示高密度结构，软组织隐藏
3. 每种模式关联一个医学调优的传递函数
4. 通过键盘数字键1-4在模式间切换
5. 窗口标题实时显示当前模式名称
6. 完整的CMakeLists.txt

**合成数据的结构：**

```
合成脑部数据（150x150x150，spacing 1.0x1.0x1.0 mm，模拟CT HU值）：
├── 背景：HU = -1000（空气等效）
├── 颅骨壳：HU = 800-1000（椭球壳，中心区域薄，边缘厚）
├── 脑组织：HU = 20-40（填充颅骨内部）
├── 脑室（左右）：HU = 5-15（脑组织内的低密度区）
└── 肿瘤（右侧）：HU = 60-90（略高于正常脑组织）
```

### 17.5.2 完整C++代码

```cpp
// ============================================================================
// Chapter 17: Advanced Volume Rendering
// File: MedicalVolumeRenderer.cxx
// Description: Complete medical volume rendering application with:
//   - Synthetic brain-like volume data generation
//   - 4 rendering modes: MIP, Composite, IsoSurface-like, Bone-only
//   - Keyboard-driven mode switching (keys 1-4)
//   - Transfer functions tuned for medical data visualization
//   - GPU-accelerated ray casting
// VTK Version: 9.5.2
// ============================================================================

#include <vtkActor.h>
#include <vtkCamera.h>
#include <vtkColorTransferFunction.h>
#include <vtkGPUVolumeRayCastMapper.h>
#include <vtkImageData.h>
#include <vtkInteractorStyleTrackballCamera.h>
#include <vtkNew.h>
#include <vtkPiecewiseFunction.h>
#include <vtkPointData.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkShortArray.h>
#include <vtkSmartPointer.h>
#include <vtkSmartVolumeMapper.h>
#include <vtkTextActor.h>
#include <vtkTextProperty.h>
#include <vtkVolume.h>
#include <vtkVolumeProperty.h>

#include <cmath>
#include <iostream>
#include <string>

// ---------------------------------------------------------------------------
// Constants for synthetic data generation
// ---------------------------------------------------------------------------
constexpr int    VOLUME_DIM        = 150;     // Volume dimensions (cubic)
constexpr double VOLUME_SPACING    = 1.0;     // mm per voxel
constexpr double VOLUME_CENTER_X   = 75.0;    // Center of volume in voxel units
constexpr double VOLUME_CENTER_Y   = 75.0;
constexpr double VOLUME_CENTER_Z   = 75.0;

// "Anatomical" parameters
constexpr double SKULL_OUTER_RX    = 55.0;    // Skull outer ellipsoid X radius
constexpr double SKULL_OUTER_RY    = 65.0;    // Skull outer ellipsoid Y radius
constexpr double SKULL_OUTER_RZ    = 60.0;    // Skull outer ellipsoid Z radius
constexpr double SKULL_INNER_RX    = 48.0;    // Skull inner ellipsoid X radius
constexpr double SKULL_INNER_RY    = 58.0;    // Skull inner ellipsoid Y radius
constexpr double SKULL_INNER_RZ    = 53.0;    // Skull inner ellipsoid Z radius
constexpr double BRAIN_RX          = 45.0;    // Brain tissue ellipsoid X
constexpr double BRAIN_RY          = 55.0;    // Brain tissue ellipsoid Y
constexpr double BRAIN_RZ          = 50.0;    // Brain tissue ellipsoid Z
constexpr double VENTRICLE_RX      = 10.0;    // Ventricle X radius
constexpr double VENTRICLE_RY      = 18.0;    // Ventricle Y radius
constexpr double VENTRICLE_RZ      = 15.0;    // Ventricle Z radius
constexpr double VENTRICLE_OFFSET  = 18.0;    // Ventricle offset from midline
constexpr double TUMOR_RX          = 12.0;    // Tumor X radius
constexpr double TUMOR_RY          = 14.0;    // Tumor Y radius
constexpr double TUMOR_RZ          = 10.0;    // Tumor Z radius
constexpr double TUMOR_OFFSET_X    = 22.0;    // Tumor center offset X
constexpr double TUMOR_OFFSET_Y    = 5.0;     // Tumor center offset Y
constexpr double TUMOR_OFFSET_Z    = 0.0;     // Tumor center offset Z

// ---------------------------------------------------------------------------
// Helper: Check if a point is inside an ellipsoid
// ---------------------------------------------------------------------------
bool InsideEllipsoid(double x, double y, double z,
                     double cx, double cy, double cz,
                     double rx, double ry, double rz)
{
    double dx = (x - cx) / rx;
    double dy = (y - cy) / ry;
    double dz = (z - cz) / rz;
    return (dx * dx + dy * dy + dz * dz) <= 1.0;
}

// ---------------------------------------------------------------------------
// Helper: Generate synthetic brain-like CT volume data
//
// Simulated CT Hounsfield Units (HU):
//   Air / background:   -1000 HU
//   Skull (cortical):   +800 to +1000 HU
//   Brain tissue:        +20 to +40 HU
//   Ventricles (CSF):    +5 to +15 HU
//   Tumor:               +60 to +90 HU
// ---------------------------------------------------------------------------
vtkSmartPointer<vtkImageData> GenerateSyntheticBrainVolume()
{
    std::cout << "[GENERATE] Creating synthetic brain volume ("
              << VOLUME_DIM << " x " << VOLUME_DIM << " x " << VOLUME_DIM
              << ")..." << std::endl;

    vtkNew<vtkImageData> imageData;
    imageData->SetDimensions(VOLUME_DIM, VOLUME_DIM, VOLUME_DIM);
    imageData->SetSpacing(VOLUME_SPACING, VOLUME_SPACING, VOLUME_SPACING);
    imageData->SetOrigin(0.0, 0.0, 0.0);
    imageData->AllocateScalars(VTK_SHORT, 1);

    short* ptr = static_cast<short*>(imageData->GetScalarPointer());

    for (int z = 0; z < VOLUME_DIM; ++z)
    {
        double zpos = static_cast<double>(z);

        for (int y = 0; y < VOLUME_DIM; ++y)
        {
            double ypos = static_cast<double>(y);

            for (int x = 0; x < VOLUME_DIM; ++x)
            {
                double xpos = static_cast<double>(x);
                int idx = z * VOLUME_DIM * VOLUME_DIM + y * VOLUME_DIM + x;
                short huValue = -1000;  // Default: air

                // --- Layer 1: Skull (outer shell) ---
                bool inOuterSkull = InsideEllipsoid(
                    xpos, ypos, zpos,
                    VOLUME_CENTER_X, VOLUME_CENTER_Y, VOLUME_CENTER_Z,
                    SKULL_OUTER_RX, SKULL_OUTER_RY, SKULL_OUTER_RZ);
                bool inInnerSkull = InsideEllipsoid(
                    xpos, ypos, zpos,
                    VOLUME_CENTER_X, VOLUME_CENTER_Y, VOLUME_CENTER_Z,
                    SKULL_INNER_RX, SKULL_INNER_RY, SKULL_INNER_RZ);

                if (inOuterSkull && !inInnerSkull)
                {
                    // Skull bone: vary HU to simulate thickness variation
                    double distOuter = std::sqrt(
                        ((xpos - VOLUME_CENTER_X) / SKULL_OUTER_RX) *
                        ((xpos - VOLUME_CENTER_X) / SKULL_OUTER_RX) +
                        ((ypos - VOLUME_CENTER_Y) / SKULL_OUTER_RY) *
                        ((ypos - VOLUME_CENTER_Y) / SKULL_OUTER_RY) +
                        ((zpos - VOLUME_CENTER_Z) / SKULL_OUTER_RZ) *
                        ((zpos - VOLUME_CENTER_Z) / SKULL_OUTER_RZ));

                    double distInner = std::sqrt(
                        ((xpos - VOLUME_CENTER_X) / SKULL_INNER_RX) *
                        ((xpos - VOLUME_CENTER_X) / SKULL_INNER_RX) +
                        ((ypos - VOLUME_CENTER_Y) / SKULL_INNER_RY) *
                        ((ypos - VOLUME_CENTER_Y) / SKULL_INNER_RY) +
                        ((zpos - VOLUME_CENTER_Z) / SKULL_INNER_RZ) *
                        ((zpos - VOLUME_CENTER_Z) / SKULL_INNER_RZ));

                    // Bone density decreases toward inner surface
                    double boneFraction = (distOuter - distInner) /
                        (1.0 - distInner + 0.01);
                    boneFraction = std::max(0.0, std::min(1.0, boneFraction));
                    huValue = static_cast<short>(800.0 + 200.0 * boneFraction);
                }

                // --- Layer 2: Brain tissue (inside skull) ---
                if (inInnerSkull)
                {
                    bool inBrain = InsideEllipsoid(
                        xpos, ypos, zpos,
                        VOLUME_CENTER_X, VOLUME_CENTER_Y, VOLUME_CENTER_Z,
                        BRAIN_RX, BRAIN_RY, BRAIN_RZ);

                    if (inBrain)
                    {
                        // Base brain tissue value with slight noise
                        double noise = std::sin(xpos * 0.5) *
                                       std::cos(ypos * 0.5) *
                                       std::sin(zpos * 0.3);
                        huValue = static_cast<short>(30.0 + noise * 5.0);

                        // --- Layer 3: Ventricles ---
                        bool inLeftVentricle = InsideEllipsoid(
                            xpos, ypos, zpos,
                            VOLUME_CENTER_X - VENTRICLE_OFFSET,
                            VOLUME_CENTER_Y + 2.0, VOLUME_CENTER_Z,
                            VENTRICLE_RX, VENTRICLE_RY, VENTRICLE_RZ);

                        bool inRightVentricle = InsideEllipsoid(
                            xpos, ypos, zpos,
                            VOLUME_CENTER_X + VENTRICLE_OFFSET,
                            VOLUME_CENTER_Y + 2.0, VOLUME_CENTER_Z,
                            VENTRICLE_RX, VENTRICLE_RY, VENTRICLE_RZ);

                        if (inLeftVentricle || inRightVentricle)
                        {
                            huValue = static_cast<short>(10.0 + noise * 2.0);
                        }

                        // --- Layer 4: Tumor ---
                        bool inTumor = InsideEllipsoid(
                            xpos, ypos, zpos,
                            VOLUME_CENTER_X + TUMOR_OFFSET_X,
                            VOLUME_CENTER_Y + TUMOR_OFFSET_Y,
                            VOLUME_CENTER_Z + TUMOR_OFFSET_Z,
                            TUMOR_RX, TUMOR_RY, TUMOR_RZ);

                        if (inTumor)
                        {
                            // Tumor: slightly higher density than normal brain
                            double tumorNoise = std::sin(xpos * 0.8) *
                                                std::cos(ypos * 0.7);
                            huValue = static_cast<short>(75.0 + tumorNoise * 5.0);
                        }
                    }
                    else
                    {
                        // Between brain and skull: CSF-like fluid
                        huValue = static_cast<short>(8.0);
                    }
                }

                ptr[idx] = huValue;
            }
        }
    }

    // Report data range
    short minVal, maxVal;
    short* data = ptr;
    minVal = maxVal = data[0];
    for (int i = 1; i < VOLUME_DIM * VOLUME_DIM * VOLUME_DIM; ++i)
    {
        if (data[i] < minVal) minVal = data[i];
        if (data[i] > maxVal) maxVal = data[i];
    }
    std::cout << "[GENERATE] Volume scalar range: ["
              << minVal << ", " << maxVal << "] HU" << std::endl;

    return imageData;
}

// ---------------------------------------------------------------------------
// Transfer Function Preset: MIP Mode (Maximum Intensity Projection)
// MIP ignores opacity -- color mapping alone defines the result
// We still need an opacity function but VTK's MIP mode handles it internally
// ---------------------------------------------------------------------------
void SetupMIPTransferFunction(vtkVolumeProperty* property)
{
    // MIP mode: color is mapped linearly to scalar value
    vtkNew<vtkPiecewiseFunction> scalarOpacity;
    scalarOpacity->AddPoint(-1000.0, 1.0);
    scalarOpacity->AddPoint(1000.0, 1.0);

    vtkNew<vtkColorTransferFunction> colorTF;
    colorTF->AddRGBPoint(-1000.0, 0.0, 0.0, 0.0);
    colorTF->AddRGBPoint(-500.0, 0.1, 0.1, 0.2);
    colorTF->AddRGBPoint(0.0, 0.3, 0.3, 0.4);
    colorTF->AddRGBPoint(500.0, 0.7, 0.7, 0.8);
    colorTF->AddRGBPoint(1000.0, 1.0, 1.0, 1.0);

    property->SetScalarOpacity(scalarOpacity);
    property->SetColor(colorTF);
    property->ShadeOff();
}

// ---------------------------------------------------------------------------
// Transfer Function Preset: Composite Mode (Standard volume rendering)
// All tissues visible with appropriate opacity levels
// ---------------------------------------------------------------------------
void SetupCompositeTransferFunction(vtkVolumeProperty* property)
{
    // ---- Scalar Opacity: multi-peak for different tissues ----
    vtkNew<vtkPiecewiseFunction> scalarOpacity;

    // Air: transparent
    scalarOpacity->AddPoint(-1000.0, 0.0);
    scalarOpacity->AddPoint(-800.0, 0.0);

    // Soft tissue / brain: low-medium opacity for context
    scalarOpacity->AddPoint(-100.0, 0.0);
    scalarOpacity->AddPoint(0.0, 0.05);
    scalarOpacity->AddPoint(30.0, 0.12);      // Brain tissue peak
    scalarOpacity->AddPoint(50.0, 0.03);

    // Tumor: slightly higher opacity to highlight
    scalarOpacity->AddPoint(60.0, 0.0);
    scalarOpacity->AddPoint(80.0, 0.25);      // Tumor peak
    scalarOpacity->AddPoint(95.0, 0.05);

    // Bone transition
    scalarOpacity->AddPoint(100.0, 0.0);
    scalarOpacity->AddPoint(400.0, 0.15);
    scalarOpacity->AddPoint(700.0, 0.7);       // Bone peak
    scalarOpacity->AddPoint(1000.0, 1.0);

    // ---- Color Transfer Function ----
    vtkNew<vtkColorTransferFunction> colorTF;

    // Air: dark
    colorTF->AddRGBPoint(-1000.0, 0.0, 0.0, 0.0);

    // Brain: pinkish gray
    colorTF->AddRGBPoint(0.0, 0.75, 0.55, 0.50);
    colorTF->AddRGBPoint(40.0, 0.85, 0.65, 0.58);

    // Tumor: reddish
    colorTF->AddRGBPoint(65.0, 0.9, 0.3, 0.25);
    colorTF->AddRGBPoint(85.0, 0.95, 0.25, 0.20);

    // Bone: ivory
    colorTF->AddRGBPoint(400.0, 0.88, 0.82, 0.70);
    colorTF->AddRGBPoint(700.0, 0.95, 0.92, 0.85);
    colorTF->AddRGBPoint(1000.0, 1.0, 1.0, 1.0);

    // ---- Gradient Opacity: emphasize surfaces ----
    vtkNew<vtkPiecewiseFunction> gradientOpacity;
    gradientOpacity->AddPoint(0.0, 0.0);       // Uniform regions: transparent
    gradientOpacity->AddPoint(200.0, 0.2);     // Medium gradient: semi-transparent
    gradientOpacity->AddPoint(1000.0, 1.0);    // Strong gradient: opaque (surfaces)

    property->SetScalarOpacity(scalarOpacity);
    property->SetGradientOpacity(gradientOpacity);
    property->SetColor(colorTF);

    // Lighting parameters for 3D effect
    property->ShadeOn();
    property->SetAmbient(0.15);
    property->SetDiffuse(0.9);
    property->SetSpecular(0.3);
    property->SetSpecularPower(20.0);
}

// ---------------------------------------------------------------------------
// Transfer Function Preset: IsoSurface-like Mode
// Only a narrow scalar range is opaque, simulating an isosurface
// ---------------------------------------------------------------------------
void SetupIsoSurfaceTransferFunction(vtkVolumeProperty* property, double isoValue)
{
    double halfWidth = 30.0;  // Narrow window around the isovalue

    vtkNew<vtkPiecewiseFunction> scalarOpacity;
    scalarOpacity->AddPoint(isoValue - halfWidth - 10.0, 0.0);
    scalarOpacity->AddPoint(isoValue - halfWidth, 0.0);
    scalarOpacity->AddPoint(isoValue, 1.0);            // Peak at isovalue
    scalarOpacity->AddPoint(isoValue + halfWidth, 0.0);
    scalarOpacity->AddPoint(isoValue + halfWidth + 10.0, 0.0);

    vtkNew<vtkColorTransferFunction> colorTF;
    colorTF->AddRGBPoint(isoValue - halfWidth, 0.2, 0.2, 0.2);
    colorTF->AddRGBPoint(isoValue, 1.0, 0.9, 0.8);
    colorTF->AddRGBPoint(isoValue + halfWidth, 1.0, 1.0, 1.0);

    // Gradient opacity: high at surfaces
    vtkNew<vtkPiecewiseFunction> gradientOpacity;
    gradientOpacity->AddPoint(0.0, 0.0);
    gradientOpacity->AddPoint(500.0, 1.0);

    property->SetScalarOpacity(scalarOpacity);
    property->SetGradientOpacity(gradientOpacity);
    property->SetColor(colorTF);

    property->ShadeOn();
    property->SetAmbient(0.2);
    property->SetDiffuse(0.8);
    property->SetSpecular(0.5);
    property->SetSpecularPower(40.0);
}

// ---------------------------------------------------------------------------
// Transfer Function Preset: Bone-Only Mode
// Only high-density (bone) region is visible
// ---------------------------------------------------------------------------
void SetupBoneOnlyTransferFunction(vtkVolumeProperty* property)
{
    vtkNew<vtkPiecewiseFunction> scalarOpacity;
    scalarOpacity->AddPoint(-1000.0, 0.0);
    scalarOpacity->AddPoint(200.0, 0.0);         // All soft tissue transparent
    scalarOpacity->AddPoint(350.0, 0.05);
    scalarOpacity->AddPoint(600.0, 0.6);          // Bone becomes visible
    scalarOpacity->AddPoint(800.0, 0.95);
    scalarOpacity->AddPoint(1000.0, 1.0);

    vtkNew<vtkColorTransferFunction> colorTF;
    colorTF->AddRGBPoint(200.0, 0.0, 0.0, 0.0);
    colorTF->AddRGBPoint(400.0, 0.7, 0.65, 0.50);
    colorTF->AddRGBPoint(700.0, 0.95, 0.90, 0.80);
    colorTF->AddRGBPoint(1000.0, 1.0, 1.0, 1.0);

    // Strong gradient opacity to emphasize bone surface
    vtkNew<vtkPiecewiseFunction> gradientOpacity;
    gradientOpacity->AddPoint(0.0, 0.0);
    gradientOpacity->AddPoint(300.0, 0.1);
    gradientOpacity->AddPoint(800.0, 1.0);

    property->SetScalarOpacity(scalarOpacity);
    property->SetGradientOpacity(gradientOpacity);
    property->SetColor(colorTF);

    property->ShadeOn();
    property->SetAmbient(0.1);
    property->SetDiffuse(0.9);
    property->SetSpecular(0.6);
    property->SetSpecularPower(50.0);
}

// ---------------------------------------------------------------------------
// Custom Interactor Style with keyboard-driven mode switching
// ---------------------------------------------------------------------------
class VolumeModeInteractorStyle : public vtkInteractorStyleTrackballCamera
{
public:
    static VolumeModeInteractorStyle* New();
    vtkTypeMacro(VolumeModeInteractorStyle, vtkInteractorStyleTrackballCamera);

    void SetVolumeProperty(vtkVolumeProperty* prop) { m_volumeProperty = prop; }
    void SetMapper(vtkSmartVolumeMapper* mapper)     { m_mapper = mapper; }
    void SetRenderWindow(vtkRenderWindow* rw)       { m_renderWindow = rw; }

    void ApplyMode(int mode)
    {
        if (!m_volumeProperty || !m_mapper) return;

        switch (mode)
        {
        case 1:
            std::cout << "[MODE] Switching to MIP (Maximum Intensity Projection)"
                      << std::endl;
            m_mapper->SetBlendModeToMaximumIntensity();
            SetupMIPTransferFunction(m_volumeProperty);
            m_currentModeName = "MIP (Maximum Intensity Projection)";
            break;

        case 2:
            std::cout << "[MODE] Switching to Composite (Standard Volume Rendering)"
                      << std::endl;
            m_mapper->SetBlendModeToComposite();
            SetupCompositeTransferFunction(m_volumeProperty);
            m_currentModeName = "Composite (Standard Volume Rendering)";
            break;

        case 3:
            std::cout << "[MODE] Switching to IsoSurface-like (Narrow Window)"
                      << std::endl;
            m_mapper->SetBlendModeToComposite();
            SetupIsoSurfaceTransferFunction(m_volumeProperty, 700.0);  // Bone isovalue
            m_currentModeName = "IsoSurface-like (Bone Surface, ~700 HU)";
            break;

        case 4:
            std::cout << "[MODE] Switching to Bone-Only"
                      << std::endl;
            m_mapper->SetBlendModeToComposite();
            SetupBoneOnlyTransferFunction(m_volumeProperty);
            m_currentModeName = "Bone-Only (High-Density Structures)";
            break;

        default:
            return;
        }

        // Update window title
        if (m_renderWindow)
        {
            std::string title = "Medical Volume Renderer -- Mode: "
                              + m_currentModeName;
            m_renderWindow->SetWindowName(title.c_str());
        }

        m_renderWindow->Render();
    }

    void OnChar() override
    {
        vtkRenderWindowInteractor* rwi = this->Interactor;
        std::string key = rwi->GetKeySym();

        if (key == "1") ApplyMode(1);
        else if (key == "2") ApplyMode(2);
        else if (key == "3") ApplyMode(3);
        else if (key == "4") ApplyMode(4);
        else if (key == "h" || key == "H")
        {
            std::cout << "\n=== Medical Volume Renderer -- Help ===" << std::endl;
            std::cout << "  1 : MIP mode" << std::endl;
            std::cout << "  2 : Composite mode" << std::endl;
            std::cout << "  3 : IsoSurface-like mode (bone surface)" << std::endl;
            std::cout << "  4 : Bone-only mode" << std::endl;
            std::cout << "  R : Reset camera" << std::endl;
            std::cout << "  H : Show this help" << std::endl;
            std::cout << "  Q / Escape : Quit" << std::endl;
            std::cout << "======================================\n" << std::endl;
        }
        else if (key == "r" || key == "R")
        {
            std::cout << "[CAMERA] Reset camera." << std::endl;
            rwi->GetRenderWindow()->GetRenderers()->GetFirstRenderer()
                ->ResetCamera();
            rwi->GetRenderWindow()->Render();
        }
        else if (key == "q" || key == "Q" || key == "Escape")
        {
            rwi->TerminateApp();
            return;
        }

        vtkInteractorStyleTrackballCamera::OnChar();
    }

private:
    vtkVolumeProperty*   m_volumeProperty = nullptr;
    vtkSmartVolumeMapper* m_mapper         = nullptr;
    vtkRenderWindow*     m_renderWindow   = nullptr;
    std::string          m_currentModeName;
};
vtkStandardNewMacro(VolumeModeInteractorStyle);

// ---------------------------------------------------------------------------
// Main function
// ---------------------------------------------------------------------------
int main(int argc, char* argv[])
{
    (void)argc;
    (void)argv;

    std::cout << "========================================" << std::endl;
    std::cout << "  Medical Volume Renderer" << std::endl;
    std::cout << "  Chapter 17: Advanced Volume Rendering" << std::endl;
    std::cout << "  VTK 9.5.2" << std::endl;
    std::cout << "========================================" << std::endl;

    // ========================================================================
    // STEP 1: Generate or load volume data
    // ========================================================================
    std::cout << "\n[STEP 1] Generating synthetic brain volume..." << std::endl;
    auto imageData = GenerateSyntheticBrainVolume();

    // ========================================================================
    // STEP 2: Create volume mapper and property
    // ========================================================================
    std::cout << "[STEP 2] Setting up GPU volume mapper..." << std::endl;

    vtkNew<vtkSmartVolumeMapper> volumeMapper;
    volumeMapper->SetInputData(imageData);
    // vtkSmartVolumeMapper auto-selects GPU if available

    vtkNew<vtkVolumeProperty> volumeProperty;
    // Default to composite mode transfer function
    SetupCompositeTransferFunction(volumeProperty);

    // Apply initial quality settings
    volumeMapper->SetSampleDistance(0.8);
    volumeMapper->SetImageSampleDistance(1.0);
    volumeMapper->AutoAdjustSampleDistancesOn();

    // ========================================================================
    // STEP 3: Create volume actor
    // ========================================================================
    std::cout << "[STEP 3] Creating volume actor..." << std::endl;

    vtkNew<vtkVolume> volume;
    volume->SetMapper(volumeMapper);
    volume->SetProperty(volumeProperty);

    // ========================================================================
    // STEP 4: Set up renderer
    // ========================================================================
    std::cout << "[STEP 4] Configuring renderer..." << std::endl;

    vtkNew<vtkRenderer> renderer;
    renderer->AddVolume(volume);
    renderer->SetBackground(0.08, 0.08, 0.12);  // Dark blue-gray background

    // Set up depth peeling for correct transparency
    renderer->SetUseDepthPeeling(true);
    renderer->SetMaximumNumberOfPeels(8);
    renderer->SetOcclusionRatio(0.0);

    // Camera positioning
    renderer->ResetCamera();
    vtkCamera* camera = renderer->GetActiveCamera();
    camera->SetPosition(VOLUME_CENTER_X,
                        VOLUME_CENTER_Y - 200.0,
                        VOLUME_CENTER_Z + 100.0);
    camera->SetFocalPoint(VOLUME_CENTER_X,
                          VOLUME_CENTER_Y,
                          VOLUME_CENTER_Z);
    camera->SetViewUp(0.0, 0.0, 1.0);
    camera->SetClippingRange(10.0, 600.0);

    // ========================================================================
    // STEP 5: Set up render window
    // ========================================================================
    std::cout << "[STEP 5] Creating render window..." << std::endl;

    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(900, 800);
    renderWindow->SetMultiSamples(8);  // 8x MSAA
    renderWindow->SetWindowName(
        "Medical Volume Renderer -- Mode: Composite (Standard Volume Rendering)");
    renderWindow->AddRenderer(renderer);

    // ========================================================================
    // STEP 6: Set up interactor with custom style
    // ========================================================================
    std::cout << "[STEP 6] Configuring interactor..." << std::endl;

    vtkNew<vtkRenderWindowInteractor> interactor;
    interactor->SetRenderWindow(renderWindow);

    vtkNew<VolumeModeInteractorStyle> style;
    style->SetVolumeProperty(volumeProperty);
    style->SetMapper(volumeMapper);
    style->SetRenderWindow(renderWindow);

    interactor->SetInteractorStyle(style);

    // ========================================================================
    // STEP 7: Start rendering loop
    // ========================================================================
    std::cout << "\n========================================" << std::endl;
    std::cout << "  Rendering initialized." << std::endl;
    std::cout << "  Modes:" << std::endl;
    std::cout << "    Press 1: MIP" << std::endl;
    std::cout << "    Press 2: Composite (default)" << std::endl;
    std::cout << "    Press 3: IsoSurface-like" << std::endl;
    std::cout << "    Press 4: Bone-only" << std::endl;
    std::cout << "    Press H: Help" << std::endl;
    std::cout << "    Press Q: Quit" << std::endl;
    std::cout << "  Mouse: Rotate/Pan/Zoom (TrackballCamera)" << std::endl;
    std::cout << "========================================\n" << std::endl;

    renderWindow->Render();
    interactor->Initialize();
    interactor->Start();

    std::cout << "\n[DONE] Medical Volume Renderer terminated." << std::endl;
    return 0;
}
```

### 17.5.3 CMakeLists.txt

```cmake
# ============================================================================
# Chapter 17: Advanced Volume Rendering
# File: CMakeLists.txt
# Medical Volume Renderer -- synthetic brain data, multi-mode visualization
# VTK 9.5.2
# ============================================================================

cmake_minimum_required(VERSION 3.16 FATAL_ERROR)

project(MedicalVolumeRenderer VERSION 1.0.0 LANGUAGES CXX)

# --------------------------------------------------------------------------
# C++ Standard
# --------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# --------------------------------------------------------------------------
# Find VTK 9.5.2
# --------------------------------------------------------------------------
find_package(VTK 9.5 REQUIRED COMPONENTS
    CommonCore
    CommonDataModel
    CommonExecutionModel
    CommonMath
    CommonSystem

    ImagingCore
    ImagingSources          # For potential future image data operations

    RenderingCore
    RenderingOpenGL2
    RenderingVolume
    RenderingVolumeOpenGL2   # GPU volume rendering
    RenderingFreeType         # Text rendering

    InteractionStyle
    FiltersSources
)
if(NOT VTK_FOUND)
    message(FATAL_ERROR "VTK 9.5 not found. "
        "Please set VTK_DIR to the VTK build directory.")
endif()

message(STATUS "VTK version: ${VTK_VERSION}")
message(STATUS "VTK_DIR: ${VTK_DIR}")

# --------------------------------------------------------------------------
# Executable
# --------------------------------------------------------------------------
add_executable(MedicalVolumeRenderer MedicalVolumeRenderer.cxx)

target_link_libraries(MedicalVolumeRenderer PRIVATE
    VTK::CommonCore
    VTK::CommonDataModel
    VTK::CommonExecutionModel
    VTK::CommonMath
    VTK::ImagingCore
    VTK::RenderingCore
    VTK::RenderingOpenGL2
    VTK::RenderingVolume
    VTK::RenderingVolumeOpenGL2
    VTK::RenderingFreeType
    VTK::InteractionStyle
)

target_include_directories(MedicalVolumeRenderer PRIVATE
    ${VTK_INCLUDE_DIRS}
)

target_compile_definitions(MedicalVolumeRenderer PRIVATE
    ${VTK_DEFINITIONS}
)

# --------------------------------------------------------------------------
# Platform-specific settings
# --------------------------------------------------------------------------
if(WIN32)
    target_compile_definitions(MedicalVolumeRenderer PRIVATE
        _CRT_SECURE_NO_WARNINGS
        NOMINMAX
    )
endif()

if(MSVC)
    target_compile_options(MedicalVolumeRenderer PRIVATE
        /W3 /MP
    )
endif()

# --------------------------------------------------------------------------
# Installation
# --------------------------------------------------------------------------
install(TARGETS MedicalVolumeRenderer
    RUNTIME DESTINATION bin
)

# --------------------------------------------------------------------------
# Configuration summary
# --------------------------------------------------------------------------
message(STATUS "=============================================")
message(STATUS "  MedicalVolumeRenderer Configuration")
message(STATUS "  C++ Standard: ${CMAKE_CXX_STANDARD}")
message(STATUS "  VTK Version:   ${VTK_VERSION}")
message(STATUS "  Build Type:    ${CMAKE_BUILD_TYPE}")
message(STATUS "=============================================")
```

### 17.5.4 编译与运行

**编译步骤：**

```bash
# 1. 创建构建目录
mkdir build && cd build

# 2. CMake配置（替换为你的VTK构建路径）
cmake .. -DVTK_DIR=/path/to/vtk-9.5.2/build -DCMAKE_BUILD_TYPE=Release

# 3. 编译
cmake --build . --config Release

# 4. 运行
./MedicalVolumeRenderer        # Linux/macOS
MedicalVolumeRenderer.exe      # Windows
```

**运行后的操作说明：**

| 按键 | 功能 |
|------|------|
| 1 | 切换到MIP模式——最大强度投影，适合快速观察整体结构 |
| 2 | 切换到Composite模式——标准体渲染，所有组织可见（默认） |
| 3 | 切换到IsoSurface-like模式——类等值面效果，模拟骨表面渲染 |
| 4 | 切换到Bone-only模式——仅显示高密度骨骼结构 |
| H | 显示帮助信息（控制台输出） |
| R | 重置相机到初始位置 |
| Q / Escape | 退出程序 |
| 鼠标左键拖拽 | 旋转场景 |
| 鼠标滚轮 | 缩放 |
| 鼠标中键拖拽 | 平移 |

**预期渲染效果：**

- **模式1 (MIP)**：显示一个明亮的"头骨"投影，所有高密度体素沿视线方向的最大值被投影到屏幕上。图像为灰度，骨骼和部分脑组织都可见但缺乏深度感。
- **模式2 (Composite)**：显示完整的半透明头部模型。颅骨呈现象牙白色，脑组织呈现粉灰色半透明，两个脑室在中心位置可见，肿瘤在右侧呈现红色调。
- **模式3 (IsoSurface-like)**：仅显示颅骨的表面——因为传递函数仅在约700HU的狭窄窗口内有不透明度。颅骨的3D形态清晰可见，带有光照产生的高光和阴影。
- **模式4 (Bone-only)**：仅颅骨可见，软组织完全透明。骨骼表面有光泽的高光效果（高SpecularPower）。

---

## 17.6 本章小结

本章深入讲解了体渲染的四大高级主题：质量与性能平衡、光照与着色、传递函数设计进阶、GPU体渲染。这些知识将你的体渲染能力从"能跑通示例"提升到"能针对真实数据进行有效调优"的水平。

**一、渲染质量与性能平衡（17.1节）**

- **采样距离（SampleDistance）**是体渲染质量-性能曲线的主要控制参数。较小的值提供更平滑的渲染但需要更多采样点数。
- **图像采样距离（ImageSampleDistance）**控制屏幕空间的光线密度。在交互期间增大此值以获得更好帧率，在最终渲染时恢复到1.0。
- **自适应采样（AutoAdjustSampleDistances）**在大多数场景中保持了合理的默认行为，但精确控制场景应该关闭它。
- 性能优化遵循一个清晰的决策树：增大采样距离 -> 增大图像采样距离 -> 关闭光照 -> 降低视口分辨率 -> 减少不透明度峰值。

核心法则：采样距离决定了沿每条光线的采样密度，图像采样距离决定了投射光线的数量。两者协同控制总计算量。

**二、光照与着色（17.2节）**

- **ShadeOn()**在体渲染中开启Phong光照计算，使用体数据的梯度作为表面法线的代理。这是让体数据"看起来像3D物体"的关键设置。
- **梯度不透明度**根据梯度幅值调制不透明度——在均匀区域降低不透明度，在边界区域保持不透明度，从而自动强调材料界面。它是"类等值面"体渲染效果的核心机制。
- **环境光（Ambient）**、**漫反射（Diffuse）**和**镜面反射（Specular）**在体渲染中具有与多边形渲染相同的语义，但作用于每个采样点而非面片顶点。
- 光照计算增加了约20-30%的渲染时间，但对于需要展示表面结构的数据（骨骼、牙齿、工业部件），视觉效果的提升远超性能成本。

**三、传递函数设计进阶（17.3节）**

- **多峰传递函数**通过在标量值轴上为每种材料设置独立的不透明度峰值，实现了多材料数据集的清晰视觉分离。
- **直方图分析**是传递函数设计的系统化起点——通过标量直方图识别数据中存在的主要材料类型和它们的标量值区间。
- 四种经过验证的医学预设（骨骼、皮肤、血管造影、工业CT）可以作为真实场景下的传递函数起点。
- 传递函数设计是一个迭代过程：分析数据 -> 设置不透明度 -> 设置颜色 -> 渲染评估 -> 调整并重复。
- 对于需要频繁调整传递函数的场景，ParaView的传递函数编辑器是一个高效的工具。

**四、GPU体渲染详解（17.4节）**

- `vtkGPUVolumeRayCastMapper`利用GPU的大规模并行计算能力，将光线投射映射到片元着色器。在典型数据上，GPU Mapper比CPU Mapper快1-2个数量级。
- GPU体渲染需要OpenGL 3.2+和足够的GPU显存（推荐4GB+）。
- `vtkSmartVolumeMapper`自动检测GPU可用性并选择最优Mapper——是编写跨平台体渲染应用程序的推荐选择。
- 多体渲染支持在同一场景中渲染多个重叠的体数据，适用于术前/术后对比和多模态融合。
- 性能优化策略包括：子体积提取（Reduce Volume Extent）、交互LOD（动态调整采样参数）、优化传递函数以促进提前终止、选择合适的渲染模式（MIP比Composite更快）、在交互期间减小视口尺寸。

**五、综合示例（17.5节）**

17.5节提供了一个完整可运行的医学体渲染程序，涵盖了本章的所有核心概念。该程序生成合成的脑部CT数据，支持四种渲染模式（MIP、Composite、IsoSurface-like、Bone-only）之间的键盘切换，并为每种模式配置了医学调优的传递函数。这个程序可以作为一个实用的参考实现——你可以替换数据源、修改传递函数预设，或添加新的渲染模式来满足自己的需求。

---

**继续学习建议：**

掌握了本章的高级体渲染技术后，你可以将体渲染与其他VTK高级功能结合使用：

- **裁剪平面（Clipping Planes）**：在体渲染中使用剪切平面来揭示数据内部结构，与透明度结合使用可以产生更强大的洞察效果。
- **多视口布局**：在一个窗口中并排显示不同渲染模式（如MIP和Composite），提供互补的数据视图。
- **动画**：创建传递函数随时间变化的动画（如"溶解"效果，从皮肤到骨骼的渐进过渡），用于演示和教学。
- **Qt + VTK集成**：构建带有滑块和按钮的图形界面，让用户交互式地调整传递函数和渲染参数。
- **大规模数据**：对于超过GPU显存的大体数据，使用分块（Bricking）和LOD策略进行渲染。

体渲染是科学可视化领域中视觉效果最具冲击力的技术之一，而你现在已经掌握了将其应用于真实数据的核心技能。

---

*第十七章完。*
