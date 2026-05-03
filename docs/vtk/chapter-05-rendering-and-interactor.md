# 第五章 渲染与交互机（Rendering and Interactor）

## 本章导读

第二章我们深入学习了可视化管道（Visualization Pipeline）——从Source到Mapper这一整条数据加工链路。你理解了数据如何沿管道流动、MTime如何驱动缓存失效、`SetInputConnection()`与`SetInputData()`的根本区别。但管道最右侧的部分——Mapper之后发生了什么——我们只是简略提及。

本章将聚焦于管道的"右半侧"：**渲染管道（Rendering Pipeline）**。这是VTK将你的数据真正呈现为屏幕上可交互像素的最后一段旅程。具体包括：

1. **渲染管线的四个核心层级**——Actor、Renderer、RenderWindow、RenderWindowInteractor各自承担什么职责，以及它们之间的协作关系。
2. **vtkRenderer**——场景管理者，它不只是一个"画布"，更是Actor、Camera、Light的管理中心。
3. **vtkRenderWindow**——渲染窗口，操作系统窗口的VTK抽象，以及多渲染器布局的物理载体。
4. **多视口（Multi-Viewport）布局**——如何使用`SetViewport()`在同一个窗口中创建多个视图，如"前视图+顶视图+侧视图+透视图"的四视图科学可视化布局。
5. **vtkCamera**——相机控制，透视投影与平行投影、相机参数的含义、各种旋转/平移/缩放语义。
6. **vtkRenderWindowInteractor**——交互机，事件循环的启动、默认键盘/鼠标绑定、交互样式的概念与切换。
7. **一个完整的代码示例**——在一个窗口中展示多视口布局，不同视口显示不同数据源和相机角度，并通过键盘回调切换预设视角。

如果你在之前的章节中一直在问"为什么我的数据画出来后总是白色？""如何在同一个窗口里显示四个视图？""旋转场景的轨迹球行为能不能改？"——那么这一章就是为你准备的答案。

---

## 5.1 渲染管线回顾

### 5.1.1 渲染管线 vs 可视化管线

在第二章2.1.3节，我们画出了VTK两条管道的分界线。这里我们再做一个精炼的回顾，因为深入理解第5章必须锚定这两者的区别。

```
+------------------- 可视化管道（第二章重点）-------------------+
|                                                              |
| Source --> Filter --> Filter --> ... --> Mapper              |
|                                         |                    |
+-----------------------------------------+--------------------+
                                          |
                  actor->SetMapper(mapper) |
                                          v
+------------------- 渲染管道（本章重点）-----------------------+
|                                                              |
| Actor --> Renderer --> RenderWindow --> Interactor           |
|                                                              |
+--------------------------------------------------------------+
```

**可视化管道**处理的问题是："我有数据，我想把它变成图形"。它的输入是数据集（vtkDataSet），输出是图形基元（顶点缓冲、索引缓冲、着色器等）。核心抽象是`vtkAlgorithm`及其子类。

**渲染管道**处理的问题是："我有图形，我想把它展示给用户并允许交互"。它的输入是Actor/Mapper产出的图形基元，输出是屏幕上的像素和用户输入事件的分发。核心抽象是`vtkProp`（及其子类`vtkActor`、`vtkVolume`等）、`vtkRenderer`、`vtkRenderWindow`、`vtkRenderWindowInteractor`。

一条经验法则：**如果你在修改数据参数（`SetResolution()`、`SetRadius()`、`SetShrinkFactor()`），你操作的是可视化管道；如果你在修改颜色、背景、相机角度、窗口大小，你操作的是渲染管道。**

### 5.1.2 Actor -> Renderer -> RenderWindow -> Interactor 角色回顾

这是你从第一章第一个Cone示例中就已经见过的四层结构。既然我们要在本章中对它们逐一深入，先做一个快速的角色定位：

```
vtkActor          -- "演员"：场景中的可视化实体。
                    持有一个Mapper（决定几何形状）和一组Property（决定外观）。
                    还负责自身的空间变换（位置、朝向、缩放）。

vtkRenderer       -- "舞台"：管理一组Actor、一组Light、一台Camera。
                    负责Actor的添加/移除、背景设置、以及将这组Actor渲染到目标区域。

vtkRenderWindow    -- "剧院"：屏幕上的一个窗口。
                    可以容纳多个Renderer（每个占据窗口的一块矩形区域）。
                    封装操作系统窗口的创建、管理、像素显示。

vtkRenderWindowInteractor -- "观众与演出之间的交互"：
                    捕获键盘鼠标事件，分发给当前的InteractorStyle处理。
                    启动事件循环，驱动持续的交互刷新。
```

### 5.1.3 VTK如何抽象图形后端

VTK的渲染系统构建在**后端抽象层**之上。在VTK 9.5.2中，默认且推荐的渲染后端是**OpenGL2**（对应的CMake模块为`RenderingOpenGL2`）。此外，VTK还支持以下后端：

| 后端 | CMake模块 | 说明 |
|------|----------|------|
| OpenGL2 | `RenderingOpenGL2` | 桌面OpenGL 3.2+，默认且最成熟的后端 |
| OpenGL ES | `RenderingOpenGLES2` | 移动/嵌入式OpenGL ES 3.0 |
| WebGPU | `RenderingWebGPU` | 通过WebGPU标准的现代渲染（实验性） |
| ANARI | `RenderingANARI` | 通过ANARI接口支持ray tracing渲染器 |

对于C++开发者和本教程而言，你只需要使用OpenGL2后端。VTK通过**对象工厂（Object Factory）** 机制自动选择正确的具体实现类。例如：

- 你代码中写的是`vtkPolyDataMapper`，运行时使用的实际上是`vtkOpenGLPolyDataMapper`。
- 你代码中写的是`vtkRenderer`，运行时使用的实际上是`vtkOpenGLRenderer`。

这种抽象使得你的应用程序代码与底层渲染API完全解耦——同一个程序可以不加修改地在不同后端上运行（只要在CMake配置时选择了对应的后端模块即可）。你在CMakeLists.txt中声明`RenderingOpenGL2`，加上`vtk_module_autoinit`宏，VTK的工厂系统就会在程序启动时自动注册OpenGL2后端的所有具体实现类。

**不需要你关心的事情（VTK已经帮你做了）：**

- 顶点缓冲对象（VBO）的创建与填充
- 着色器程序（Shader Program）的编译与链接
- 帧缓冲对象（FBO）的管理
- 纹理单元（Texture Unit）的分配
- OpenGL状态机的管理（绑定、解绑、状态推入/弹出）
- 窗口系统集成（WGL/GLX/CGL/EGL）

**需要你关心的事情（本章的重点）：**

- 将Actor添加到正确的Renderer
- 设置背景色、渐变效果
- 使用`SetViewport()`划分多视图布局
- 调整相机位置、焦点、投影类型
- 选择交互样式、启动事件循环

---

## 5.2 vtkRenderer——场景管理者

### 5.2.1 vtkRenderer的职责

`vtkRenderer`是VTK渲染系统中的中央指挥。你可以把它想象成"灯光师+摄像师+舞台总监"的综合体。它的核心职责包括：

1. **Actor管理**：维护场景中的可视实体列表，管理它们的添加、移除和可见性。
2. **光照管理**：持有光源（`vtkLight`）列表，控制场景照明环境。
3. **相机管理**：持有一台`vtkCamera`对象，定义观察者的位置、朝向和投影方式。
4. **背景设置**：设置背景颜色、渐变颜色、或背景纹理。
5. **渲染执行**：当收到渲染请求时，遍历Actor列表，调用每个Actor的渲染方法，最终将结果绘制到RenderWindow。

每个`vtkRenderer`内部维护着一个Prop列表（`vtkPropCollection`）。`vtkProp`是所有可渲染对象的基类，`vtkActor`、`vtkVolume`、`vtkActor2D`都是它的派生类。本章我们主要讨论`vtkActor`；体渲染（`vtkVolume`）将在第十三章专门讲解。

### 5.2.2 AddActor() / RemoveActor()——管理可视元素

`AddActor()`和`RemoveActor()`是渲染器中最常用的两个方法：

```cpp
// 将一个Actor添加到渲染器的场景中
void AddActor(vtkProp* prop);

// 将一个Actor从渲染器中移除
void RemoveActor(vtkProp* prop);

// 获取渲染器中当前的Actor数量
int VisibleActorCount();

// 获取渲染器中第i个Actor
vtkProp* GetActors() 返回的集合中的元素；

// 移除所有Actor
void RemoveAllViewProps();
```

添加的顺序很重要：**渲染器按照Actor的添加顺序进行渲染**（除非你启用了透明度排序，此时半透明Actor会根据深度重新排序）。通常你应该先添加不透明对象，再添加半透明对象。

一个常见的模式是：你需要临时隐藏某个Actor时，除了`RemoveActor()`之外，你也可以直接操作Actor的可见性开关：

```cpp
actor->SetVisibility(0);   // 隐藏（但Actor仍留在渲染器中）
actor->SetVisibility(1);   // 显示
```

使用`SetVisibility()`比反复`RemoveActor()`/`AddActor()`更高效——它避免了从Prop列表中删除和重新插入对象的开销。

### 5.2.3 SetBackground() / SetBackground2() / GradientBackgroundOn()——背景样式

VTK支持三种背景模式：纯色背景、色块渐变背景、和纹理背景。

**纯色背景（默认）：**

```cpp
renderer->SetBackground(0.2, 0.3, 0.4);   // RGB（分量范围0.0~1.0）
renderer->SetBackground(0, 0, 0);          // 纯黑背景
renderer->SetBackground(1.0, 1.0, 1.0);    // 纯白背景
```

**渐变背景：**

通过设置两种颜色并启用渐变模式，可以实现从上到下的平滑过渡：

```cpp
renderer->SetBackground(0.1, 0.2, 0.3);    // 顶部颜色：深蓝灰色
renderer->SetBackground2(0.7, 0.8, 0.9);   // 底部颜色：浅灰蓝色
renderer->GradientBackgroundOn();           // 启用渐变
```

渐变方向为从上到下（沿Y轴）。如果需要在代码中切换回纯色背景：

```cpp
renderer->GradientBackgroundOff();
```

**纹理背景：**

通过设置背景纹理来实现图片背景或天空盒效果：

```cpp
// 读取图片作为纹理
vtkNew<vtkJPEGReader> reader;
reader->SetFileName("background.jpg");

// 创建纹理对象
vtkNew<vtkTexture> texture;
texture->SetInputConnection(reader->GetOutputPort());

// 设置为渲染器的背景纹理
renderer->SetBackgroundTexture(texture);
renderer->TexturedBackgroundOn();
```

这是显示地球影像、医学DICOM背景或公司Logo背景的常用方式。

### 5.2.4 SetViewport()——定义子区域（多视图布局的基础）

`SetViewport()`定义了渲染器在RenderWindow中的矩形子区域。这是实现多视图布局的核心方法。

```cpp
void SetViewport(double xmin, double ymin, double xmax, double ymax);
```

四个参数都是归一化坐标（normalized viewport coordinates），取值范围为 **[0.0, 1.0]**：

- `(0, 0)` 表示窗口的左下角。
- `(1, 1)` 表示窗口的右上角。
- `(0.5, 0.5)` 表示窗口的正中心。

**重要约定：** VTK的viewport坐标原点在**左下角**，这与许多GUI框架（原点在左上角）不同。在设计多视口布局时务必注意这一点。

几个常用布局示例：

```cpp
// 单个全窗口（默认行为）
renderer->SetViewport(0.0, 0.0, 1.0, 1.0);

// 左右并排两个视图
rendererLeft->SetViewport(0.0, 0.0, 0.5, 1.0);   // 左半部
rendererRight->SetViewport(0.5, 0.0, 1.0, 1.0);   // 右半部

// 上下并排两个视图
rendererTop->SetViewport(0.0, 0.5, 1.0, 1.0);     // 上半部
rendererBottom->SetViewport(0.0, 0.0, 1.0, 0.5);   // 下半部

// 四象限视图（2x2网格）
rendererTL->SetViewport(0.0, 0.5, 0.5, 1.0);   // 左上
rendererTR->SetViewport(0.5, 0.5, 1.0, 1.0);   // 右上
rendererBL->SetViewport(0.0, 0.0, 0.5, 0.5);   // 左下
rendererBR->SetViewport(0.5, 0.0, 1.0, 0.5);   // 右下

// 主视图+三辅助视图（类似3D建模软件的布局）
rendererMain->SetViewport(0.0, 0.3, 0.7, 1.0);    // 主视图（大面积透视）
rendererTop->SetViewport(0.0, 0.0, 0.3, 0.3);     // 顶视图
rendererFront->SetViewport(0.3, 0.0, 0.6, 0.3);   // 前视图
rendererSide->SetViewport(0.6, 0.0, 0.9, 0.3);    // 侧视图
```

**SetViewport()的视觉效果：**

```
Window (e.g., 800x400 pixels)
+----------------------------------------+
| (0,0)                        (1,1)     |
|                                        |
|  [Renderer A]   |   [Renderer B]      |
|  viewport(0,0,  |   viewport(0.5,0,   |
|           0.5,1)|            1.0,1)   |
|                                        |
+----------------------------------------+
```

需要注意的是，viewport坐标与窗口的物理像素大小无关——无论窗口是300x200还是1920x1080，viewport坐标始终在[0,1]范围内。VTK内部根据当前的窗口物理尺寸将viewport坐标映射到实际的像素区域。这意味着视图布局会随着窗口缩放自动适配。

### 5.2.5 SetLayer()——渲染器分层

当一个RenderWindow中多个渲染器的viewport重合时，渲染顺序决定了哪个渲染器的内容显示在最上层。`SetLayer()`可以精确控制这个顺序：

```cpp
renderer->SetLayer(int layer);
```

- 默认layer为0。
- Layer值越大的渲染器越靠上层（后渲染，覆盖在下层之上）。
- 常用于覆盖层效果：例如在主三维视图上叠加一个二维信息面板或比例尺。

```cpp
// 背景三维视图（下层，先渲染）
renderer3D->SetViewport(0.0, 0.0, 1.0, 1.0);
renderer3D->SetLayer(0);

// 前景信息面板（上层，后渲染，需要透明背景）
rendererOverlay->SetViewport(0.7, 0.7, 1.0, 1.0);
rendererOverlay->SetLayer(1);
```

当使用覆盖层时，上层渲染器的背景应该设置为透明（或让上层渲染器不设置背景色，使其内容"浮"在下层之上）。

### 5.2.6 SetDraw()——启用/禁用渲染器

```cpp
renderer->SetDraw(0);   // 禁用此渲染器（不参与渲染）
renderer->SetDraw(1);   // 启用此渲染器（默认行为）
```

当`SetDraw(0)`被调用时，该渲染器在`RenderWindow::Render()`时完全被跳过——它的所有Actor都不会被渲染。这对于以下场景有用：

- 临时隐藏某个视图而不需要移除它的Actor。
- 在程序运行时动态切换显示哪些视图。
- 性能优化：不需要的视图可以暂时关闭其渲染。

---

## 5.3 vtkRenderWindow——渲染窗口

### 5.3.1 vtkRenderWindow的职责

`vtkRenderWindow`是VTK对操作系统窗口的抽象。它负责：

1. **窗口创建与管理**：创建原生平台窗口（Windows上使用Win32 API、Linux上使用X11或Wayland、macOS上使用Cocoa）。
2. **渲染器容纳**：一个RenderWindow可以包含多个Renderer。
3. **像素呈现**：将各Renderer的渲染结果合成并交换到屏幕（双缓冲机制）。
4. **事件路由**：将窗口系统的鼠标/键盘事件转发给Interactor。
5. **窗口属性**：管理窗口大小、位置、标题、全屏等属性。

### 5.3.2 AddRenderer()——渲染器注册

```cpp
void AddRenderer(vtkRenderer* renderer);
```

将一个渲染器注册到窗口中。一个RenderWindow可以容纳任意数量的Renderer，每个Renderer通过`SetViewport()`定义其在窗口中的矩形区域。

```cpp
vtkNew<vtkRenderWindow> renderWindow;

// 创建三个渲染器
vtkNew<vtkRenderer> rendererA, rendererB, rendererC;

// 分别设置各自的viewport
rendererA->SetViewport(0.0, 0.0, 0.33, 1.0);
rendererB->SetViewport(0.33, 0.0, 0.67, 1.0);
rendererC->SetViewport(0.67, 0.0, 1.0, 1.0);

// 将三个渲染器都注册到同一个窗口中
renderWindow->AddRenderer(rendererA);
renderWindow->AddRenderer(rendererB);
renderWindow->AddRenderer(rendererC);
```

每次`Render()`调用时，RenderWindow会按添加顺序依次调用每个渲染器的渲染方法。

### 5.3.3 SetSize() / SetPosition()——窗口几何

```cpp
// 设置窗口的客户区大小（像素）
renderWindow->SetSize(800, 600);

// 设置窗口在屏幕上的位置（像素，相对于屏幕左上角）
renderWindow->SetPosition(100, 100);

// 获取当前窗口大小
int* size = renderWindow->GetSize();       // size[0]=宽, size[1]=高
int w = renderWindow->GetSize()[0];
int h = renderWindow->GetSize()[1];

// 获取当前屏幕尺寸
int* screenSize = renderWindow->GetScreenSize();

// 设置全屏模式
renderWindow->SetFullScreen(1);
renderWindow->SetFullScreen(0);            // 退出全屏

// 设置窗口在屏幕上的显示模式
renderWindow->FullScreenOn();
renderWindow->FullScreenOff();
```

**重要细节：** `SetSize()`设置的是**客户区**大小（可渲染区域的大小），不包括窗口标题栏、边框和菜单栏。实际窗口占用的屏幕空间会大于此值。

### 5.3.4 SetWindowName()——窗口标题

```cpp
renderWindow->SetWindowName("My VTK Application - Cone with Shrink Filter");
```

设置窗口的标题栏文字。通常建议包含应用程序名称和当前数据集的描述信息，方便用户识别窗口内容。

### 5.3.5 SetMultiSamples()——多重采样抗锯齿

```cpp
// 启用4x多重采样抗锯齿（MSAA）
renderWindow->SetMultiSamples(4);

// 关闭多重采样（降低GPU消耗）
renderWindow->SetMultiSamples(0);
```

多重采样（Multi-Sample Anti-Aliasing）是一种硬件支持的抗锯齿技术，通过在像素内采样多个点来平滑边缘锯齿。VTK中MSAA的启用与否对一个程序的视觉效果有显著影响：

- **SetMultiSamples(0)**：无抗锯齿，边缘可能有明显锯齿，GPU消耗最低。适合开发调试阶段或对性能要求极高的场景。
- **SetMultiSamples(4)**：4x MSAA，边缘明显光滑，是性能与质量兼顾的推荐设置。
- **SetMultiSamples(8)**：8x MSAA，最高质量，但GPU消耗更高。

**注意事项：**

1. MSAA必须在**创建vtkRenderWindow之后、首次调用Render()之前**设置。如果在首次Render()之后动态修改MSAA设置，行为可能因平台和GPU驱动而异。
2. MSAA在离屏渲染（Off-Screen Rendering）模式下可能不可用，取决于图形驱动支持。
3. 某些渲染技术（如将渲染结果读取回CPU的`vtkWindowToImageFilter`）在与MSAA配合时可能需要特殊处理。

### 5.3.6 Render()——触发管道级联

```cpp
renderWindow->Render();
```

`Render()`是渲染管道的触发器。它的执行流程在第二章2.5节中已详细解释，这里做简要回顾：

```
renderWindow->Render()
  --> 对每个renderer: renderer->Render()
    --> 遍历renderer中的每个actor: actor->Render()
      --> mapper->Update()               // 触发可视化管道按需执行
      --> mapper->Render(renderer, actor) // 执行图形绘制
    --> 执行相机投影变换
    --> 执行光照计算
  --> 合成所有renderer的输出
  --> 交换帧缓冲到屏幕（双缓冲）
```

### 5.3.7 立体渲染（Stereo Rendering）

VTK支持生成立体图像对（stereo pair），用于VR头显、3D电视或红蓝立体眼镜。

```cpp
// 启用立体渲染
renderWindow->SetStereoRender(1);

// 设置立体渲染类型
renderWindow->SetStereoTypeToRedBlue();     // 红蓝立体（最常用）
renderWindow->SetStereoTypeToAnaglyph();    // 也是红蓝立体的一种
renderWindow->SetStereoTypeToInterlaced();   // 隔行扫描
renderWindow->SetStereoTypeToLeft();         // 仅左眼
renderWindow->SetStereoTypeToRight();        // 仅右眼
renderWindow->SetStereoTypeToCrystalEyes();  // Crystal Eyes主动快门式
renderWindow->SetStereoTypeToDresden();      // Dresden立体显示
```

当立体渲染启用时，VTK在每个渲染周期内分别生成左眼和右眼的视图（通过水平偏移相机位置模拟双眼视差），然后将两幅图像合成到输出帧中。

```cpp
// 调整双眼间距（默认值约0.05）
renderWindow->SetStereoTypeToRedBlue();
renderWindow->StereoRenderOn();
// renderWindow->GetStereoCapableWindow() 查询是否可用
```

立体渲染在本教程中不做重点展开，因为它需要特定显示硬件的支持。如果你从事VR/AR或科学数据沉浸式可视化开发，可以查阅VTK官方文档中关于立体渲染的详细说明。

### 5.3.8 离屏渲染（Off-Screen Rendering）

离屏渲染允许VTK在不显示任何窗口的情况下完成渲染——渲染结果被写入到内存中的帧缓冲（FBO），可以随后读取回CPU或保存为图像文件。

```cpp
// 在创建renderWindow后、首次Render()前设置
renderWindow->SetOffScreenRendering(1);

// 之后正常使用：添加Renderer、Actor，调用Render()
renderWindow->Render();  // 渲染到内存中的帧缓冲，不会弹出窗口
```

离屏渲染的典型应用场景：

- **服务器端/无头模式**：在没有图形界面的服务器（headless server）上生成可视化图像。
- **批量图像生成**：自动生成大量可视化截图，如参数扫描结果的可视化。
- **图像导出**：将渲染结果通过`vtkWindowToImageFilter`导出为图像文件（PNG、JPEG等）。
- **单元测试中的可视化验证**（Regression Testing）：自动对比渲染图像与参考图像。

```cpp
// 离屏渲染 + 导出PNG的示例骨架
vtkNew<vtkRenderWindow> renderWindow;
renderWindow->SetOffScreenRendering(1);
// ... 添加renderer, actor, 配置相机 ...

renderWindow->Render();

// 将渲染结果读取为vtkImageData
vtkNew<vtkWindowToImageFilter> windowToImage;
windowToImage->SetInput(renderWindow);

// 写入PNG文件
vtkNew<vtkPNGWriter> writer;
writer->SetInputConnection(windowToImage->GetOutputPort());
writer->SetFileName("output.png");
writer->Write();
```

---

## 5.4 多视口布局

### 5.4.1 用SetViewport()构建栅格布局

科学可视化中最常见的需求之一是在同一个窗口中同时显示数据的多个视图——例如透视图、顶视图、前视图和侧视图。这完全通过`SetViewport()`实现。

核心思路是：**创建多个vtkRenderer，每个设置不同的viewport矩形区域，全部添加到同一个vtkRenderWindow中。**

```
        (0,1)                (1,1)
          +-------------------+
          |    |    |    |    |
          | R0 | R1 | R2 | R3 |   四个等宽竖直条
          |    |    |    |    |
          +-------------------+
        (0,0)                (1,0)

        (0,1)                (1,1)
          +---------+---------+
          |         |         |
          |   R0    |   R1    |   2x2 网格
          |         |         |
          +---------+---------+
          |         |         |
          |   R2    |   R3    |
          |         |         |
          +---------+---------+
        (0,0)                (1,0)
```

viewport设置模板：

```cpp
// 2x2 等分网格
// 左上 (0.0, 0.5, 0.5, 1.0)
// 右上 (0.5, 0.5, 1.0, 1.0)
// 左下 (0.0, 0.0, 0.5, 0.5)
// 右下 (0.5, 0.0, 1.0, 0.5)
```

### 5.4.2 常见布局模式

**模式一：左右并排（Side-by-Side Comparison）**

用于对比两种不同参数的可视化结果，或同时展示数据模型与其线框表示。

```cpp
rendererLeft->SetViewport(0.0, 0.0, 0.5, 1.0);
rendererRight->SetViewport(0.5, 0.0, 1.0, 1.0);
```

**模式二：2x2四视图（Four-View Layout）**

科学仿真中最经典的布局：透视图 + 三正交视图。

```cpp
// 透视图（左上）
rendererPersp->SetViewport(0.0, 0.5, 0.5, 1.0);
// 前视图（右上）
rendererFront->SetViewport(0.5, 0.5, 1.0, 1.0);
// 侧视图（左下）
rendererSide->SetViewport(0.0, 0.0, 0.5, 0.5);
// 顶视图（右下）
rendererTop->SetViewport(0.5, 0.0, 1.0, 0.5);
```

**模式三：主视图 + 缩略图叠加（Picture-in-Picture）**

```cpp
// 主视图（全窗口）
rendererMain->SetViewport(0.0, 0.0, 1.0, 1.0);
rendererMain->SetLayer(0);

// 小缩略图（右下角）
rendererThumb->SetViewport(0.7, 0.7, 1.0, 1.0);
rendererThumb->SetLayer(1);
```

**模式四：2x3或更多**

viewport坐标可任意划分，不仅限于等分。例如下面的布局产生一个大的主视图配合右侧两个小图：

```cpp
rendererMain->SetViewport(0.0, 0.0, 0.65, 1.0);    // 左侧大图
rendererTopRight->SetViewport(0.65, 0.5, 1.0, 1.0); // 右上小图
rendererBotRight->SetViewport(0.65, 0.0, 1.0, 0.5); // 右下小图
```

### 5.4.3 处理窗口缩放事件

当窗口被用户拖拽改变大小时，VTK会自动将各viewport重新映射到新的像素区域——viewport是归一化坐标，因此**无需手动处理缩放逻辑**。每个Renderer会自适应新的区域大小。

但是，有一种情况需要注意：**你需要在多视口创建后调用一次`Render()`使所有viewport生效**。在交互运行期间，每次窗口大小改变后，VTK的Interactor会自动触发重新渲染，viewport映射随之更新。

```cpp
// 窗口创建时设置初始大小
renderWindow->SetSize(800, 600);

// 添加所有renderer并设置viewport后
// 首次Render()会根据当前窗口尺寸计算各viewport的实际像素范围
renderWindow->Render();

// 之后窗口大小变化时，viewport自动适配——无需额外代码
```

---

## 5.5 vtkCamera——相机控制

### 5.5.1 平行投影 vs 透视投影

VTK支持两种投影类型：

**透视投影（Perspective Projection）——默认：**

模拟人眼或真实相机的视觉效果：离相机近的物体更大，远的物体更小，平行线在远处汇聚到灭点。这是观察三维模型时最自然的选择。

```cpp
vtkCamera* camera = renderer->GetActiveCamera();
camera->ParallelProjectionOff();  // 使用透视投影（默认）

// 调整透视视角宽度（单位为度）
camera->SetViewAngle(30.0);       // 默认值，值越大视锥越广
```

**平行投影（Parallel Projection / Orthographic）：**

所有投影线相互平行，物体无论远近看起来大小相同。在科学可视化中，正交投影用于需要保持几何尺寸比例不变的三视图（前视图、侧视图、顶视图）。

```cpp
camera->ParallelProjectionOn();

// 设置平行投影的缩放比例（控制可见区域大小）
camera->SetParallelScale(5.0); // 值越大"看到"的范围越广
```

`SetParallelScale()`的含义是：在平行投影模式下，渲染器垂直方向的一半能容纳多少世界坐标单位。例如，`SetParallelScale(5.0)`意味着垂直方向上可以看到10个单位范围（-5到+5），水平方向的范围根据窗口宽高比自动缩放。

### 5.5.2 相机位置参数

VTK相机使用三个核心参数来定义观察姿态：

```cpp
vtkCamera* camera = renderer->GetActiveCamera();

// 设置相机在世界坐标系中的位置
camera->SetPosition(x, y, z);       // 相机所在位置（"眼睛"的位置）

// 设置焦点（相机正在看的目标点）
camera->SetFocalPoint(x, y, z);     // 视线汇聚到的点

// 设置"上"方向（定义画面中"向上"的方向）
camera->SetViewUp(vx, vy, vz);      // 通常是 (0, 1, 0)，即Y轴向上
```

这三个参数共同定义了观察者的坐标系：

```
                Camera Position (眼睛所在位置)
                     \
                      \
                       \  视线方向(View Direction)
                        \
                         \
                     Focal Point (焦点，视线汇聚处)
                         
  View Up (通常指向屏幕上方 -- 决定画面中的"上"是哪个方向)
```

**默认值（以vtkConeSource为例的典型场景）：**

- Position: `(0, 0, 1)` —— 相机在Z轴正方向
- Focal Point: `(0, 0, 0)` —— 看原点
- View Up: `(0, 1, 0)` —— Y轴为"上"

**一些预置视角示例：**

```cpp
// 前视图（从Z轴正方向看，Y向上）
camera->SetPosition(0, 0, 10);
camera->SetFocalPoint(0, 0, 0);
camera->SetViewUp(0, 1, 0);

// 顶视图（从Y轴正方向俯视，Z向上）
camera->SetPosition(0, 10, 0);
camera->SetFocalPoint(0, 0, 0);
camera->SetViewUp(0, 0, 1);     // 注意：ViewUp变为Z轴

// 侧视图（从X轴正方向看，Y向上）
camera->SetPosition(10, 0, 0);
camera->SetFocalPoint(0, 0, 0);
camera->SetViewUp(0, 1, 0);

// 等轴测视角（三个轴等间距）
camera->SetPosition(5, 5, 5);
camera->SetFocalPoint(0, 0, 0);
camera->SetViewUp(0, 1, 0);
```

### 5.5.3 SetClippingRange()——近远裁剪平面

```cpp
camera->SetClippingRange(double dNear, double dFar);
```

近裁剪面（Near Clipping Plane）和远裁剪面（Far Clipping Plane）定义了**与相机的距离区间**，只有在这个区间内的物体才会被渲染。离相机距离小于`dNear`或大于`dFar`的几何体将被裁剪（不渲染）。

- **dNear**：近裁剪面与相机的距离。任何比这更靠近相机的几何体都不会被渲染。dNear必须大于0（相机位置本身不可见）。
- **dFar**：远裁剪面与相机的距离。任何比这更远的几何体都不会被渲染。

```cpp
// 合理的默认裁剪范围
camera->SetClippingRange(0.01, 1000.0);  // 很近到非常远

// 对于大型场景（如地形、建筑、CFD域）
camera->SetClippingRange(1.0, 10000.0);

// 对于微尺度场景（如显微数据）
camera->SetClippingRange(0.0001, 10.0);
```

如果`dNear`和`dFar`之间的比值过大（例如`dFar / dNear > 10^5`），在透视投影中可能出现**深度缓冲精度问题（Z-Fighting）**——远处的两个面无法正确区分深度，导致闪烁。解决方案是尽可能收窄裁剪范围，使其刚好覆盖场景的深度范围。

### 5.5.4 相机运动方法

VTK的`vtkCamera`提供了一套面向交互的相机运动方法。这些方法的名字来自摄影和航空术语，理解它们的语义对于操控相机至关重要。

```cpp
// 缩放：沿视线方向向前/向后移动相机（改变Position）
camera->Zoom(double factor);       // factor > 1 放大, factor < 1 缩小

// 绕视线轴旋转（Roll -- 滚转）
camera->Roll(double angle);        // 角度，单位：度。正值顺时针

// 方位角（Azimuth -- 绕ViewUp轴水平旋转）
camera->Azimuth(double angle);     // 正值：从上方看，逆时针旋转

// 俯仰角（Elevation -- 绕与视线和ViewUp垂直的轴旋转）
camera->Elevation(double angle);   // 正值：相机仰头（向上看）

// 偏航（Yaw -- 绕ViewUp轴旋转视点，类似Azimuth）
camera->Yaw(double angle);         // 与Azimuth相同语义

// 俯仰（Pitch -- 相机抬头/低头，类似Elevation）
camera->Pitch(double angle);       // 与Elevation相同语义

// 推拉（Dolly -- 沿视线方向移动相机）
camera->Dolly(double distance);    // 正值：向焦点靠近；负值：远离焦点
```

这里需要辨析几个容易混淆的方法：

- **Zoom vs Dolly**：`Zoom()`通过修改视角角度（View Angle）来改变放大率——就像相机的变焦镜头；`Dolly()`通过实际移动相机位置来改变放大率——就像你拿着相机往前走。
- **Azimuth vs Yaw**：两者在VTK中语义相同，都是绕ViewUp轴旋转。
- **Elevation vs Pitch**：两者在VTK中语义相同，都是抬头/低头。
- **Roll**：绕视线轴旋转——画面顺时针/逆时针旋转。此操作不改变Position和FocalPoint，只改变ViewUp的方向。

**Azimuth/Elevation的几何直观：**

想象你跟焦点之间有一根看不见的线。Azimuth是绕着ViewUp轴旋转这根线（从上方看逆时针转）；Elevation是保持视线长度不变，相机在包含视线和ViewUp的平面内上下摆动。

### 5.5.5 ResetCamera()——自动适配可见物体

```cpp
// 自动调整相机使场景中所有可见Actor都在视野内且居中
renderer->ResetCamera();

// 针对某个特定数据集的边界重置相机
renderer->ResetCamera(dataSet->GetBounds());

// 重置裁剪范围为场景边界的合适倍数
renderer->ResetCameraClippingRange();
```

`ResetCamera()`是一个极其便利的方法：它根据当前渲染器中所有可见Actor的组合包围盒（Bounds），自动计算并设置：
- 最佳相机位置和焦点
- 合适的平行缩放比例或透视视角角度
- 合适的近/远裁剪平面

这是大多数VTK程序的"起点"——在添加完所有Actor之后，调用一次`ResetCamera()`让相机自动定位到一个合适的观察位置，然后再根据需要进行微调。

**一个常见的使用模式：**

```cpp
// 添加所有Actor...
renderer->AddActor(actor1);
renderer->AddActor(actor2);
renderer->AddActor(actor3);

// 让相机自动适配到所有物体的包围盒
renderer->ResetCamera();

// 小小的后缩，留出一点呼吸空间
vtkCamera* camera = renderer->GetActiveCamera();
camera->Dolly(1.2);  // 向后推拉一点

renderWindow->Render();
```

### 5.5.6 vtkCamera vs 交互机的相机

一个重要的注意事项：**每个`vtkRenderer`拥有一台属于它自己的`vtkCamera`实例。** 交互机的交互样式（InteractorStyle）通过`style->SetDefaultRenderer(renderer)`知道的渲染器，然后通过该渲染器的`GetActiveCamera()`获取相机对象进行操作。

```cpp
// 获取渲染器的相机
vtkCamera* camera = renderer->GetActiveCamera();

// 交互样式操作相机
vtkNew<vtkInteractorStyleTrackballCamera> style;
style->SetDefaultRenderer(renderer);
// 内部：当用户拖拽鼠标时，style通过
//   renderer->GetActiveCamera() 获取相机
//   调用 Azimuth(), Elevation(), Dolly() 等方法进行调整
```

这意味着：**如果你更换了渲染器的相机（通过`renderer->SetActiveCamera(anotherCamera)`），交互样式也将自动使用新相机。** 这对于多视口场景很有用——每个视口的渲染器拥有一台独立的相机，从而可以从不同角度观察同一个或不同的数据。

---

## 5.6 vtkRenderWindowInteractor——交互机

### 5.6.1 交互机的角色

`vtkRenderWindowInteractor`是VTK的事件处理中枢。如果`vtkRenderer`是"舞台"、`vtkRenderWindow`是"剧院"，那么Interactor就是"观众——它既是观众与演出之间的交互通道，也是为演出提供持续动力的幕后引擎。

Interactor的核心职责：

1. **事件循环管理**：`Start()`方法进入阻塞的事件循环，持续等待并处理用户输入。
2. **事件路由**：将来自窗口系统的鼠标/键盘/窗口事件转发给当前的InteractorStyle。
3. **交互样式管理**：持有一个`vtkInteractorStyle`对象，定义"鼠标左键拖拽是什么意思？"、"按R键应该做什么？"等交互语义。
4. **渲染触发**：在交互操作完成后自动调用`RenderWindow->Render()`刷新画面。
5. **定时器管理**：支持定时器回调，用于实现动画或周期性更新。

### 5.6.2 SetRenderWindow()——绑定窗口

```cpp
interactor->SetRenderWindow(renderWindow);
```

这行代码建立了交互器与渲染窗口的双向关联：
- Interactor从RenderWindow获取窗口事件（键盘、鼠标、窗口大小变化等）。
- Interactor在交互操作完成后调用`renderWindow->Render()`重新绘制画面。

### 5.6.3 Start()——进入事件循环

```cpp
renderWindowInteractor->Start();
```

`Start()`进入**阻塞事件循环**。程序会在这行代码处"停下来"，直到用户关闭窗口或程序调用`TerminateApp()`。事件循环的内部工作流程如下：

```
while (渲染窗口仍然打开)
{
    等待窗口系统事件（鼠标移动、按键、窗口尺寸变化等）

    将事件转发给当前的 InteractorStyle

    InteractorStyle 根据事件类型执行相应操作：
        - 鼠标拖拽 → 调整相机参数（Azimuth, Elevation, Dolly 等）
        - 键盘按键 → 执行按键对应的操作（r=Reset, w=Wireframe 等）
        - 窗口大小变化 → 重新映射viewport → 触发渲染

    调用 RenderWindow->Render() 刷新画面
}
```

这个事件循环是交互式VTK程序的"心脏"。`Start()`之前的所有代码都是搭建舞台和布景，`Start()`之后一切都进入自动运行。

**为什么不在`Start()`之前调用`Render()`？** 因为如果没有首帧渲染（在`Start()`之前调用`Render()`），事件循环开始后用户将看到一片空白（或垃圾像素），直到第一次交互事件触发渲染。因此，**标准做法是：`renderWindow->Render()`然后在`renderWindowInteractor->Start()`之前调用一次`renderWindow->Render()`。** 这两次调用看似多余，实则保证了首帧的及时显示。

### 5.6.4 默认鼠标事件

`vtkInteractorStyleTrackballCamera`（大多数VTK示例的默认交互样式）定义了以下鼠标绑定：

| 鼠标操作 | 相机效果 | 对应相机方法 |
|----------|---------|-------------|
| **左键拖拽** | 绕焦点旋转 | `Azimuth()` + `Elevation()` |
| **中键拖拽** | 平移（Pan） | 修改FocalPoint与Position（保持视线方向不变） |
| **右键拖拽** | 缩放（Zoom/Dolly） | `Dolly()` |
| **滚轮上滚** | 放大（Zoom In） | 与右键向上拖拽相同 |
| **滚轮下滚** | 缩小（Zoom Out） | 与右键向下拖拽相同 |
| **Shift + 左键拖拽** | 平移 | 与中键拖拽相同 |

### 5.6.5 默认键盘事件

相同的默认交互样式还提供了以下键盘绑定（大小写不敏感）：

| 按键 | 功能 | 说明 |
|------|------|------|
| **r** | Reset | 重置相机到自动计算的默认视角（`ResetCamera()`） |
| **w** | Wireframe | 将所有Actor的表示模式设为线框（Wireframe） |
| **s** | Surface | 将所有Actor的表示模式设为表面（Surface，默认） |
| **p** | Pick | 触发拾取操作（Pick），在终端输出被点击的Actor信息 |
| **f** | Fly-to | 飞行动画：相机平滑过渡到新的观察位置 |
| **j** | Joystick | 切换到操纵杆式（Joystick）交互样式 |
| **t** | Trackball | 切换到轨迹球式（Trackball）交互样式 |
| **e** | Exit | 退出程序（关闭窗口） |
| **q** | 退出 | 同 e |
| **3** | Stereo | 切换立体渲染模式（红蓝立体） |
| **u** | User | 调用用户自定义回调（UserCallback） |

这些默认按键绑定让你在开发阶段可以不写任何键盘处理代码就进行基本的交互操作。特别是在调试模型时，按`w`切换到线框模式查看网格质量、按`s`切回表面模式查看最终效果、按`r`重置视角——这些便利性是VTK默认提供的，不需要写一行额外的代码。

### 5.6.6 vtkInteractorStyle——交互样式的概念

`vtkInteractorStyle`是交互行为的定义类。它决定了"用户拖拽鼠标时发生什么"——这个"什么"可以完全不同，取决于你使用哪种Style。

**VTK 9.5.2内置的主要交互样式：**

| 样式类 | 适用场景 | 核心行为 |
|--------|---------|---------|
| `vtkInteractorStyleTrackballCamera` | 三维场景观察（默认） | 鼠标操控相机绕固定焦点旋转。适合观察静态模型 |
| `vtkInteractorStyleJoystickCamera` | 三维漫游/游戏式操控 | 鼠标操控相机沿视线方向平滑移动。适合第一人称漫游 |
| `vtkInteractorStyleTrackballActor` | 单个物体操控 | 鼠标操控的是选中的Actor自身旋转，而非相机 |
| `vtkInteractorStyleImage` | 二维图像浏览 | 适合医学图像（CT/MRI切片）的平移、缩放、窗宽窗位调整 |
| `vtkInteractorStyleFlight` | 飞行动画 | 提供平滑的相机飞行动画过渡 |
| `vtkInteractorStyleMultiTouchCamera` | 多点触控设备 | 支持触摸屏上的捏合缩放、双指旋转等手势 |
| `vtkInteractorStyleUnicam` | 单手操控 | 简化版的TrackballCamera，优化单手操作体验 |

**设置交互样式：**

```cpp
// 创建交互样式对象
vtkNew<vtkInteractorStyleTrackballCamera> style;

// 绑定到交互机
renderWindowInteractor->SetInteractorStyle(style);

// 指定默认渲染器（交互操作作用于这个渲染器的相机）
style->SetDefaultRenderer(renderer);
```

**TrackballCamera vs JoystickCamera——如何选择？**

- **TrackballCamera**：相机绕固定的焦点旋转。适合"观察者围绕物体转"的场景——如查看一个机械零件、一个医学模型、一个分子结构。
- **JoystickCamera**：相机以自身位置为基准平移和旋转（类似第一人称游戏）。适合"观察者在场景中走动"的场景——如建筑漫游、地质勘探场景中的穿行浏览。

**TrackballActor——操控物体而非相机：**

```cpp
vtkNew<vtkInteractorStyleTrackballActor> actorStyle;
renderWindowInteractor->SetInteractorStyle(actorStyle);
```

在这种模式下，鼠标拖拽操作的是**被选中的Actor**——你实际上在旋转、平移、缩放场景中的物体本身，而不是改变观察者的位置。这对于需要精确操作单个模型的场景（如CAD装配验证）非常有用。

### 5.6.7 自定义交互样式

对于需要自定义交互行为的应用，你只需继承`vtkInteractorStyle`或其子类，重写感兴趣的事件处理方法：

```cpp
class MyCustomStyle : public vtkInteractorStyleTrackballCamera
{
public:
  static MyCustomStyle* New();
  vtkTypeMacro(MyCustomStyle, vtkInteractorStyleTrackballCamera);

  // 重写左键按下事件
  void OnLeftButtonDown() override
  {
    // 自定义处理...
    std::cout << "Left button pressed at "
              << this->Interactor->GetEventPosition()[0] << ", "
              << this->Interactor->GetEventPosition()[1] << std::endl;

    // 也可以调用父类实现保留默认行为
    vtkInteractorStyleTrackballCamera::OnLeftButtonDown();
  }

  // 重写键盘事件
  void OnKeyPress() override
  {
    // 获取按键符号
    std::string key = this->Interactor->GetKeySym();

    if (key == "space")
    {
      std::cout << "Space key pressed -- custom action!" << std::endl;
      // 你的自定义操作...
      return;
    }

    // 其他按键交给父类处理（保留默认行为）
    vtkInteractorStyleTrackballCamera::OnKeyPress();
  }
};
```

自定义Style中可重写的事件处理方法包括：

| 方法 | 触发时机 |
|------|---------|
| `OnLeftButtonDown()` / `OnLeftButtonUp()` | 鼠标左键按下/释放 |
| `OnMiddleButtonDown()` / `OnMiddleButtonUp()` | 鼠标中键按下/释放 |
| `OnRightButtonDown()` / `OnRightButtonUp()` | 鼠标右键按下/释放 |
| `OnMouseMove()` | 鼠标移动 |
| `OnMouseWheelForward()` / `OnMouseWheelBackward()` | 鼠标滚轮前滚/后滚 |
| `OnKeyPress()` / `OnKeyRelease()` | 键盘按键按下/释放 |
| `OnChar()` | 字符事件（可打印字符） |
| `OnConfigure()` | 窗口大小变化 |
| `OnEnter()` / `OnLeave()` | 鼠标进入/离开窗口 |
| `OnTimer()` | 定时器事件 |

---

## 5.7 代码示例：交互式多视口场景展示

本节提供一个完整的C++示例程序，综合演示本章涵盖的所有核心概念：

- 多渲染器、多视口布局（2x2网格）
- 每个视口显示不同的数据源和相机角度
- 键盘回调：按1/2/3切换不同预设视角
- 完整的CMakeLists.txt

### 5.7.1 完整C++代码

将以下代码保存为`MultiViewportDemo.cxx`：

```cpp
// ============================================================================
// MultiViewportDemo.cxx
// VTK第五章示例：多视口布局、相机控制与交互样式综合演示
// ============================================================================

#include "vtkActor.h"
#include "vtkCamera.h"
#include "vtkConeSource.h"
#include "vtkCubeSource.h"
#include "vtkCylinderSource.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkSphereSource.h"

#include <iostream>
#include <string>

// ============================================================================
// 自定义交互样式：在TrackballCamera基础上增加键盘回调
// ============================================================================
class MultiViewportStyle : public vtkInteractorStyleTrackballCamera
{
public:
  static MultiViewportStyle* New();
  vtkTypeMacro(MultiViewportStyle, vtkInteractorStyleTrackballCamera);

  // 设置四个渲染器，供键盘回调使用
  void SetRenderers(vtkRenderer* tl, vtkRenderer* tr,
                    vtkRenderer* bl, vtkRenderer* br)
  {
    this->RendererTL = tl;
    this->RendererTR = tr;
    this->RendererBL = bl;
    this->RendererBR = br;
  }

protected:
  MultiViewportStyle()
    : RendererTL(nullptr), RendererTR(nullptr),
      RendererBL(nullptr), RendererBR(nullptr)
  {
  }

  // 将相机重置到预设位置
  void ResetToPresetView(int presetIndex)
  {
    if (!this->RendererTL || !this->RendererTR ||
        !this->RendererBL || !this->RendererBR)
    {
      return;
    }

    vtkCamera* camTL = this->RendererTL->GetActiveCamera();
    vtkCamera* camTR = this->RendererTR->GetActiveCamera();
    vtkCamera* camBL = this->RendererBL->GetActiveCamera();
    vtkCamera* camBR = this->RendererBR->GetActiveCamera();

    // =====================================================
    // 预设视角定义
    // =====================================================

    if (presetIndex == 1)  // 等轴测视角（经典科学可视化视角）
    {
      std::cout << ">>> 切换到等轴测视角" << std::endl;

      camTL->SetPosition(5, 5, 5);
      camTL->SetFocalPoint(0, 0, 0);
      camTL->SetViewUp(0, 1, 0);

      camTR->SetPosition(5, 5, 5);
      camTR->SetFocalPoint(0, 0, 0);
      camTR->SetViewUp(0, 1, 0);

      camBL->SetPosition(5, 5, 5);
      camBL->SetFocalPoint(0, 0, 0);
      camBL->SetViewUp(0, 1, 0);

      camBR->SetPosition(5, 5, 5);
      camBR->SetFocalPoint(0, 0, 0);
      camBR->SetViewUp(0, 1, 0);

      // 所有渲染器设为透视投影
      for (auto* cam : {camTL, camTR, camBL, camBR})
      {
        cam->ParallelProjectionOff();
        cam->SetViewAngle(30.0);
      }
    }
    else if (presetIndex == 2)  // 正交三视图 + 透视图
    {
      std::cout << ">>> 切换到三视图+透视图" << std::endl;

      // 左上：透视（保持之前的透视设置）
      camTL->SetPosition(5, 5, 5);
      camTL->SetFocalPoint(0, 0, 0);
      camTL->SetViewUp(0, 1, 0);
      camTL->ParallelProjectionOff();

      // 右上：前视图（从+Z方向看，正交）
      camTR->SetPosition(0, 0, 5);
      camTR->SetFocalPoint(0, 0, 0);
      camTR->SetViewUp(0, 1, 0);
      camTR->ParallelProjectionOn();
      camTR->SetParallelScale(2.0);

      // 左下：侧视图（从+X方向看，正交）
      camBL->SetPosition(5, 0, 0);
      camBL->SetFocalPoint(0, 0, 0);
      camBL->SetViewUp(0, 1, 0);
      camBL->ParallelProjectionOn();
      camBL->SetParallelScale(2.0);

      // 右下：顶视图（从+Y方向看，正交）
      camBR->SetPosition(0, 5, 0);
      camBR->SetFocalPoint(0, 0, 0);
      camBR->SetViewUp(0, 0, 1);
      camBR->ParallelProjectionOn();
      camBR->SetParallelScale(2.0);
    }
    else if (presetIndex == 3)  // 不同距离的透视视角
    {
      std::cout << ">>> 切换到不同距离的透视视角" << std::endl;

      // 左上：近距离（近景）
      camTL->SetPosition(2, 2, 2);
      camTL->SetFocalPoint(0, 0, 0);
      camTL->SetViewUp(0, 1, 0);
      camTL->ParallelProjectionOff();

      // 右上：中等距离
      camTR->SetPosition(5, 3, 4);
      camTR->SetFocalPoint(0, 0, 0);
      camTR->SetViewUp(0, 1, 0);
      camTR->ParallelProjectionOff();

      // 左下：远距离（远景）
      camBL->SetPosition(10, 8, 10);
      camBL->SetFocalPoint(0, 0, 0);
      camBL->SetViewUp(0, 1, 0);
      camBL->ParallelProjectionOff();

      // 右下：俯视
      camBR->SetPosition(0, 8, 0.01);
      camBR->SetFocalPoint(0, 0, 0);
      camBR->SetViewUp(0, 0, 1);
      camBR->ParallelProjectionOff();
    }

    // 每个渲染器重置裁剪范围并刷新
    this->RendererTL->ResetCameraClippingRange();
    this->RendererTR->ResetCameraClippingRange();
    this->RendererBL->ResetCameraClippingRange();
    this->RendererBR->ResetCameraClippingRange();

    this->Interactor->GetRenderWindow()->Render();
  }

  void OnKeyPress() override
  {
    std::string key = this->Interactor->GetKeySym();

    if (key == "1")
    {
      ResetToPresetView(1);
    }
    else if (key == "2")
    {
      ResetToPresetView(2);
    }
    else if (key == "3")
    {
      ResetToPresetView(3);
    }
    else if (key == "h" || key == "H")
    {
      // 打印帮助信息
      std::cout << std::endl;
      std::cout << "========== 快捷键 ==========" << std::endl;
      std::cout << " 1 : 等轴测视角（所有视口相同）" << std::endl;
      std::cout << " 2 : 三视图+透视图" << std::endl;
      std::cout << " 3 : 不同距离的透视视角" << std::endl;
      std::cout << " r : 重置相机到自动计算视角" << std::endl;
      std::cout << " w : 线框模式 / s : 表面模式" << std::endl;
      std::cout << " q / e : 退出程序" << std::endl;
      std::cout << "============================" << std::endl;
      std::cout << std::endl;
    }
    else
    {
      // 其他按键交给父类处理（保留默认行为如 r/w/s/q 等）
      vtkInteractorStyleTrackballCamera::OnKeyPress();
    }
  }

private:
  vtkRenderer* RendererTL;
  vtkRenderer* RendererTR;
  vtkRenderer* RendererBL;
  vtkRenderer* RendererBR;
};

vtkStandardNewMacro(MultiViewportStyle);

// ============================================================================
// 辅助函数：创建一个带有颜色和边界线的Actor
// ============================================================================
vtkSmartPointer<vtkActor> CreateColoredActor(
    vtkAlgorithmOutput* sourcePort,
    double r, double g, double b,
    bool showEdges = true)
{
  vtkNew<vtkPolyDataMapper> mapper;
  mapper->SetInputConnection(sourcePort);

  vtkNew<vtkActor> actor;
  actor->SetMapper(mapper);
  actor->GetProperty()->SetColor(r, g, b);
  if (showEdges)
  {
    actor->GetProperty()->SetEdgeVisibility(1);
    actor->GetProperty()->SetEdgeColor(0.1, 0.1, 0.1);
  }
  actor->GetProperty()->SetSpecular(0.4);
  actor->GetProperty()->SetSpecularPower(20.0);

  return actor;
}

// ============================================================================
// main
// ============================================================================
int main(int argc, char* argv[])
{
  // ====================================================================
  // 第一步：创建四个Source -- 每个视口显示不同的几何体
  // ====================================================================

  // 左上：球体
  vtkNew<vtkSphereSource> sphere;
  sphere->SetThetaResolution(48);
  sphere->SetPhiResolution(48);
  sphere->SetRadius(1.5);

  // 右上：立方体
  vtkNew<vtkCubeSource> cube;
  cube->SetXLength(2.0);
  cube->SetYLength(2.0);
  cube->SetZLength(2.0);

  // 左下：圆锥体
  vtkNew<vtkConeSource> cone;
  cone->SetResolution(48);
  cone->SetHeight(3.0);
  cone->SetRadius(1.5);

  // 右下：圆柱体
  vtkNew<vtkCylinderSource> cylinder;
  cylinder->SetResolution(48);
  cylinder->SetHeight(3.0);
  cylinder->SetRadius(1.5);

  // ====================================================================
  // 第二步：创建四个Actor -- 不同颜色，不同数据源
  // ====================================================================

  vtkSmartPointer<vtkActor> actorTL =
    CreateColoredActor(sphere->GetOutputPort(),    0.8, 0.4, 0.4);  // 红色系
  vtkSmartPointer<vtkActor> actorTR =
    CreateColoredActor(cube->GetOutputPort(),      0.4, 0.8, 0.4);  // 绿色系
  vtkSmartPointer<vtkActor> actorBL =
    CreateColoredActor(cone->GetOutputPort(),      0.4, 0.4, 0.8);  // 蓝色系
  vtkSmartPointer<vtkActor> actorBR =
    CreateColoredActor(cylinder->GetOutputPort(),  0.8, 0.7, 0.3);  // 黄色系

  // 为每个Actor设置不同的朝向（增加视觉多样性）
  actorTR->SetOrientation(30, 0, 0);
  actorBL->SetOrientation(0, 45, 0);
  actorBR->SetOrientation(-20, 30, 0);

  // ====================================================================
  // 第三步：创建四个Renderer，配置2x2视口布局
  // ====================================================================

  // 左上 -- 透视图 (viewport: 左半部上)
  vtkNew<vtkRenderer> rendererTL;
  rendererTL->SetViewport(0.0, 0.5, 0.5, 1.0);
  rendererTL->SetBackground(0.15, 0.15, 0.18);
  rendererTL->AddActor(actorTL);

  // 右上 -- 前视图 (viewport: 右半部上)
  vtkNew<vtkRenderer> rendererTR;
  rendererTR->SetViewport(0.5, 0.5, 1.0, 1.0);
  rendererTR->SetBackground(0.18, 0.15, 0.15);
  rendererTR->AddActor(actorTR);

  // 左下 -- 侧视图 (viewport: 左半部下)
  vtkNew<vtkRenderer> rendererBL;
  rendererBL->SetViewport(0.0, 0.0, 0.5, 0.5);
  rendererBL->SetBackground(0.15, 0.18, 0.15);
  rendererBL->AddActor(actorBL);

  // 右下 -- 顶视图 (viewport: 右半部下)
  vtkNew<vtkRenderer> rendererBR;
  rendererBR->SetViewport(0.5, 0.0, 1.0, 0.5);
  rendererBR->SetBackground(0.18, 0.15, 0.15);
  rendererBR->AddActor(actorBR);

  // ====================================================================
  // 第四步：设置初始相机角度
  // ====================================================================

  // 左上：默认透视（等轴测视角）
  vtkCamera* camTL = rendererTL->GetActiveCamera();
  camTL->SetPosition(5, 5, 5);
  camTL->SetFocalPoint(0, 0, 0);
  camTL->SetViewUp(0, 1, 0);

  // 右上：前视图方向
  vtkCamera* camTR = rendererTR->GetActiveCamera();
  camTR->SetPosition(0, 0, 5);
  camTR->SetFocalPoint(0, 0, 0);
  camTR->SetViewUp(0, 1, 0);

  // 左下：侧视图方向
  vtkCamera* camBL = rendererBL->GetActiveCamera();
  camBL->SetPosition(5, 0, 0);
  camBL->SetFocalPoint(0, 0, 0);
  camBL->SetViewUp(0, 1, 0);

  // 右下：顶视图方向（正交投影）
  vtkCamera* camBR = rendererBR->GetActiveCamera();
  camBR->SetPosition(0, 5, 0);
  camBR->SetFocalPoint(0, 0, 0);
  camBR->SetViewUp(0, 0, 1);
  camBR->ParallelProjectionOn();
  camBR->SetParallelScale(2.0);

  // 为所有渲染器重置裁剪范围
  rendererTL->ResetCameraClippingRange();
  rendererTR->ResetCameraClippingRange();
  rendererBL->ResetCameraClippingRange();
  rendererBR->ResetCameraClippingRange();

  // ====================================================================
  // 第五步：创建RenderWindow，添加所有Renderer
  // ====================================================================

  vtkNew<vtkRenderWindow> renderWindow;
  renderWindow->SetSize(900, 700);
  renderWindow->SetWindowName("VTK Chapter 5 -- Multi Viewport Demo");
  renderWindow->SetMultiSamples(4);  // 启用4x MSAA

  renderWindow->AddRenderer(rendererTL);
  renderWindow->AddRenderer(rendererTR);
  renderWindow->AddRenderer(rendererBL);
  renderWindow->AddRenderer(rendererBR);

  // ====================================================================
  // 第六步：创建Interactor和自定义交互样式
  // ====================================================================

  vtkNew<vtkRenderWindowInteractor> interactor;
  interactor->SetRenderWindow(renderWindow);

  vtkNew<MultiViewportStyle> style;
  style->SetDefaultRenderer(rendererTL);  // 默认交互作用于左上渲染器
  style->SetRenderers(rendererTL, rendererTR, rendererBL, rendererBR);
  interactor->SetInteractorStyle(style);

  // ====================================================================
  // 第七步：首次渲染并启动事件循环
  // ====================================================================

  renderWindow->Render();

  std::cout << "========================================" << std::endl;
  std::cout << "  VTK多视口布局演示" << std::endl;
  std::cout << "  左上: 球体（透视）   右上: 立方体（前视图）" << std::endl;
  std::cout << "  左下: 锥体（侧视图） 右下: 圆柱体（顶视图）" << std::endl;
  std::cout << "========================================" << std::endl;
  std::cout << std::endl;
  std::cout << "快捷键:" << std::endl;
  std::cout << "  1 : 等轴测视角" << std::endl;
  std::cout << "  2 : 三视图+透视图" << std::endl;
  std::cout << "  3 : 不同距离透视" << std::endl;
  std::cout << "  h : 打印帮助信息" << std::endl;
  std::cout << "  r : 重置相机    w : 线框模式    s : 表面模式" << std::endl;
  std::cout << "  q / e : 退出" << std::endl;
  std::cout << std::endl;
  std::cout << "交互操作:" << std::endl;
  std::cout << "  鼠标左键拖拽: 旋转     鼠标右键拖拽: 缩放" << std::endl;
  std::cout << "  鼠标中键拖拽: 平移     鼠标滚轮: 缩放" << std::endl;
  std::cout << "  注意: 交互操作作用于当前活跃的渲染器" << std::endl;
  std::cout << "========================================" << std::endl;

  interactor->Start();

  return 0;
}
```

### 5.7.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.8 FATAL_ERROR)
project(MultiViewportDemo VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(VTK 9.5.2
  COMPONENTS
    CommonCore
    CommonColor
    CommonDataModel
    FiltersSources
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
  REQUIRED
)

message(STATUS "VTK version: ${VTK_VERSION}")

add_executable(MultiViewportDemo MultiViewportDemo.cxx)

target_link_libraries(MultiViewportDemo
  PRIVATE
    ${VTK_LIBRARIES}
)

vtk_module_autoinit(
  TARGETS MultiViewportDemo
  MODULES ${VTK_LIBRARIES}
)
```

### 5.7.3 代码逐段解析

#### 头文件和依赖

```cpp
#include "vtkActor.h"
#include "vtkCamera.h"
#include "vtkConeSource.h"
#include "vtkCubeSource.h"
#include "vtkCylinderSource.h"
#include "vtkInteractorStyleTrackballCamera.h"
#include "vtkNew.h"
#include "vtkPolyData.h"
#include "vtkPolyDataMapper.h"
#include "vtkProperty.h"
#include "vtkRenderWindow.h"
#include "vtkRenderWindowInteractor.h"
#include "vtkRenderer.h"
#include "vtkSphereSource.h"
```

本示例使用了四种不同的Source（Sphere、Cube、Cone、Cylinder）来区分四个视口。每种Source都来自`FiltersSources`模块。`vtkCamera`头文件被显式包含，因为我们在代码中直接操作相机参数。

#### 自定义交互样式

`MultiViewportStyle`继承自`vtkInteractorStyleTrackballCamera`，重写了`OnKeyPress()`方法：

- 按键`1`、`2`、`3`分别切换到三种预设视角配置。
- 按键`h`在终端打印帮助信息。
- 其他按键（如`r`、`w`、`s`、`q`等）通过调用父类的`OnKeyPress()`保留默认行为。

自定义样式保存了四个渲染器的指针，以便在键盘回调中访问所有四个渲染器的相机。`SetRenderers()`是公共的初始化方法，在main函数中调用。

#### 辅助函数CreateColoredActor()

这个函数封装了"创建Mapper + Actor + 设置颜色"的样板代码。它接收一个输出端口（`vtkAlgorithmOutput*`）——注意这适用于任何Source或Filter的输出，保持管道连接——然后创建对应的Actor并配置颜色和材质属性。

#### 2x2视口布局

```cpp
rendererTL->SetViewport(0.0, 0.5, 0.5, 1.0);  // 左上
rendererTR->SetViewport(0.5, 0.5, 1.0, 1.0);  // 右上
rendererBL->SetViewport(0.0, 0.0, 0.5, 0.5);  // 左下
rendererBR->SetViewport(0.5, 0.0, 1.0, 0.5);  // 右下
```

这是标准的2x2等分网格。注意viewport坐标的Y轴方向——`0.5`到`1.0`是上半部（高Y值），`0.0`到`0.5`是下半部（低Y值）。

#### 预设视角的设计

三种预设视角分别演示了本章的不同知识点：

- **预设1（等轴测）**：所有四个视口使用相同的相机设置——这是一个简单的统一视角。
- **预设2（三视图+透视图）**：左上为透视投影，其余三个为正交投影，模拟工程中常用的三视图布局。注意顶视图中ViewUp设为`(0, 0, 1)`（Z轴向上），这是因为从正Y方向俯视时，屏幕上的"上"对应世界空间的Z轴。
- **预设3（不同距离）**：四个视口使用不同相机距离的透视投影——从特写到远眺，展示`SetPosition()`对画面尺度的直接影响。

#### 每个视口使用独立相机

这是本示例的一个关键设计：**每个Renderer拥有独立的Camera实例**。当你在任何一个视口中拖拽鼠标旋转场景时，只有该视口的相机发生变化——其他三个视口的相机保持不变。这使得你可以在不同视口中独立调整各自的角度。

交互机通过自定义样式绑定到四个渲染器，键盘回调一次性设置所有四个相机。

### 5.7.4 编译和运行

将上述两个文件（`MultiViewportDemo.cxx`和`CMakeLists.txt`）放在同一目录中，然后：

```bash
mkdir build
cd build
cmake ..
cmake --build . --config Release
./Release/MultiViewportDemo.exe   # Windows
./MultiViewportDemo               # Linux/macOS
```

运行后，你将看到一个900x700像素的窗口，其中包含2x2四视口布局：

- 左上：红色球体，等轴测透视视角
- 右上：绿色立方体，前视图方向
- 左下：蓝色锥体，侧视图方向
- 右下：黄色圆柱体，俯视正交投影

按`2`切换到三视图+透视图模式，按`3`观察不同距离的效果，按`1`回到统一的等轴测视角。

### 5.7.5 实验建议

1. **修改viewport尺寸**：尝试将2x2网格改为3x2或1x3布局，观察viewport坐标与视觉效果的对应关系。
2. **增加第五个Source**：添加一个`vtkArrowSource`或`vtkDiskSource`，为其创建一个新的Renderer和viewport区域。
3. **增加预设4**：在自定义Style中增加一个"鸟瞰"预设——所有相机从非常高的Y位置俯视。
4. **尝试离屏渲染**：在`renderWindow->SetOffScreenRendering(1)`模式下运行（注释掉`interactor->Start()`），通过`vtkWindowToImageFilter`保存四个预设视角的截图。
5. **动态切换单个渲染器**：修改Style使得按方向键时只旋转当前活跃的Renderer（左上），其他三个不变——验证每个Renderer的相机独立性。

---

## 5.8 本章小结

本章系统讲解了VTK渲染管线的四个核心层级以及它们之间的协作关系。让我们回顾一下关键要点：

### 要点速览

1. **渲染管线由四个核心对象组成**：`vtkActor`（可视化实体）、`vtkRenderer`（场景管理者/舞台）、`vtkRenderWindow`（渲染窗口/剧院）、`vtkRenderWindowInteractor`（交互机/观众通道）。可视化管道（Source->Filter->Mapper）负责数据到图形的转换，渲染管道（Actor->Renderer->RenderWindow）负责图形到屏幕的呈现。

2. **vtkRenderer是场景的中央管理者**。它管理Actor列表（`AddActor()`/`RemoveActor()`）、背景样式（纯色/渐变/纹理）、以及通过`SetViewport()`定义在窗口中的矩形显示区域。`SetDraw()`和`SetLayer()`提供了启用/禁用和深度排序控制。

3. **vtkRenderWindow封装了操作系统窗口**。通过`SetSize()`、`SetPosition()`、`SetWindowName()`管理窗口属性；`SetMultiSamples()`控制抗锯齿质量；`Render()`触发整条管道的级联执行。离屏渲染（`SetOffScreenRendering()`）允许在没有图形界面的环境中生成渲染结果。

4. **多视口布局通过SetViewport()实现**。viewport坐标在[0,1]范围内，原点在左下角。通过创建多个Renderer、设置各自的viewport、全部添加到同一个RenderWindow中，可以构建任意复杂的多视图布局——从简单的左右并排到2x2四视图再到自定义的不等分栅格。

5. **vtkCamera控制观察视角**。三个核心参数——`SetPosition()`（相机位置）、`SetFocalPoint()`（视线焦点）、`SetViewUp()`（上方方向）——定义了世界空间中的观察坐标系。支持透视投影（`ParallelProjectionOff()`）和平行投影（`ParallelProjectionOn()`）。`Azimuth()`、`Elevation()`、`Dolly()`、`Roll()`等方法提供了面向交互的相机操控语义。

6. **vtkRenderWindowInteractor驱动交互循环**。`Start()`进入阻塞事件循环；默认交互样式`vtkInteractorStyleTrackballCamera`提供了丰富的鼠标键盘绑定（左键旋转、右键缩放、中键平移、r=重置、w=线框、s=表面等）。通过继承`vtkInteractorStyle`可以自定义交互行为，实现完全个性化的操作逻辑。

### 从本章到全书

渲染管线是VTK程序与用户交互的最终接口。你在本章学到的知识将在全书后续章节中持续发挥作用：

- **第六章（颜色、材质与光照）**将深入`vtkProperty`——如何通过材质属性（漫反射、镜面反射、环境光、透明度）和光源配置来精细化外观。
- **第八章（标量可视化与颜色映射）**将展示如何将数据标量场映射为Actor表面的颜色——这依赖Mapper的查找表（LookupTable）功能。
- **第十三章（体渲染）**将介绍`vtkVolume`——它的渲染方式与`vtkActor`完全不同，但同样依赖于Renderer、RenderWindow和Camera体系。
- **第十六章（交互与控件）**将深入Widget体系——在场景中添加可拖拽的三维控件（如裁切平面、测量标尺、种子点），它们与Interactor深度集成。

### 进入下一章之前

建议你完成以下练习：

1. 运行5.7节的完整示例程序，通过鼠标操作在每个视口中独立旋转场景，验证"每个Renderer有独立Camera"的行为。
2. 修改5.7节的代码，增加一个覆盖层Renderer（`SetLayer(1)`），在上面用`vtkTextActor`显示当前活跃视口的名称。
3. 尝试创建你自己的自定义InteractorStyle——例如重写`OnMouseWheelForward()`使得滚轮改变Sphere的Resolution而不是缩放场景。

下一章（第六章：颜色、材质与光照）将教你如何让场景看起来真正"漂亮"——从单调的纯色模型转变为具有金属光泽、透明效果和丰富光照的高质量渲染。

---

> **本章关键记忆口诀**："Renderer管舞台，Window管窗口，Camera管视角，Interactor管交互。SetViewport划地盘，四骑士各司其职。"
>
> - Actor：`SetMapper()` + `GetProperty()` —— 几何+外观
> - Renderer：`AddActor()` + `SetViewport()` + `SetBackground()` —— 场景管理+视口+背景
> - RenderWindow：`AddRenderer()` + `SetSize()` + `Render()` —— 窗口+尺寸+触发
> - Camera：`SetPosition()` + `SetFocalPoint()` + `SetViewUp()` —— 相机三要素
> - Interactor：`SetRenderWindow()` + `SetInteractorStyle()` + `Start()` —— 绑定+样式+事件循环
