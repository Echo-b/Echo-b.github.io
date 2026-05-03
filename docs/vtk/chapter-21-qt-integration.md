# 第21章 Qt集成 (Qt Integration)

VTK 提供了强大的可视化渲染能力，但在构建完整的科学计算或医学影像应用程序时，仅有三维渲染是远远不够的。开发者还需要按钮、滑块、文件对话框、树形视图、表格等丰富的用户界面控件。Qt 作为跨平台 C++ GUI 框架的标杆，与 VTK 的集成可以实现功能完备的专业级科学可视化应用。本章将深入讲解 VTK 9.5.2 中 Qt 集成的两种方式，并提供完整的可运行示例代码。

## 21.1 Qt与VTK结合的意义

### 21.1.1 为什么选择 Qt

Qt 是一个跨平台的 C++ 应用程序开发框架，具备以下显著优势：

- **跨平台支持**：一套代码可以同时运行于 Windows、Linux 和 macOS 操作系统，无需修改核心逻辑。这对于需要部署在不同实验环境中的科学软件至关重要。
- **信号与槽机制（Signals & Slots）**：Qt 的核心通信机制允许对象之间进行松耦合的事件传递。一个滑块的值变化（`valueChanged` 信号）可以直接触发 VTK 管线中某个滤波器的参数更新，无需手动编写复杂的回调逻辑。
- **丰富的 Widget 库**：Qt 提供了 `QTreeView`、`QTableView`、`QComboBox`、`QSlider`、`QDockWidget` 等大量成熟的 UI 组件，可以快速搭建专业的控制面板。
- **完善的布局管理**：`QVBoxLayout`、`QHBoxLayout`、`QSplitter` 等布局管理器使得 VTK 渲染窗口与控制面板的排布变得简单而灵活。
- **集成开发工具**：Qt Designer 可以可视化设计界面，Qt Creator 提供一站式 IDE 体验。

### 21.1.2 两种集成方式

VTK 9.5.2 提供了两种将渲染窗口嵌入 Qt 窗口的方案：

| 特性 | QVTKOpenGLNativeWidget | QVTKRenderWindowInteractor |
|------|----------------------|--------------------------|
| 基类 | QOpenGLWidget | QWidget |
| 推荐程度 | **推荐（VTK 9.x 新方案）** | 传统兼容方案 |
| OpenGL 上下文管理 | Qt 托管 | 手动管理 |
| 窗口叠加层支持 | 原生支持 | 有限支持 |
| Qt 信号/槽集成 | 完全兼容 | 兼容 |
| 多窗口共享上下文 | 原生支持 | 需要额外配置 |
| 未来维护方向 | 主要开发方向 | 逐步淘汰 |

**QVTKOpenGLNativeWidget** 是 VTK 9.x 推荐使用的现代方案。它继承自 `QOpenGLWidget`，让 Qt 来管理 OpenGL 上下文，从而更好地与 Qt 的渲染管线集成，避免了上下文冲突问题。对于新项目，应优先选择此方案。

**QVTKRenderWindowInteractor** 是传统方案，通过创建独立的窗口 ID 来管理 OpenGL 上下文。在一些旧代码或特殊的平台需求下仍可使用，但官方已不再作为首选推荐。

### 21.1.3 所需的 CMake 模块

在使用 Qt 集成之前，需要确保 CMake 找到了 VTK 的 `GUISupportQt` 模块。典型的 CMake 配置如下：

```cmake
find_package(VTK COMPONENTS
  CommonCore
  CommonDataModel
  FiltersSources
  InteractionStyle
  RenderingOpenGL2
  GUISupportQt
  ViewsQt
)
```

`GUISupportQt` 是核心模块，提供了 QVTKOpenGLNativeWidget 和相关辅助类。`ViewsQt` 提供了与 Qt 视图框架（QTreeView、QTableView 等）集成的相关类。

## 21.2 QVTKOpenGLNativeWidget

### 21.2.1 基本概念

`QVTKOpenGLNativeWidget` 继承自 Qt 的 `QOpenGLWidget`，因此它天然具备以下能力：

- 由 Qt 框架管理 OpenGL 上下文的生命周期和当前状态切换
- 支持 Qt 的透明窗口叠加（Overlays）
- 支持多个 QOpenGLWidget 实例共享同一个 OpenGL 上下文
- 与 Qt 的绘制循环（`paintGL()`）无缝集成

其内部持有 `vtkGenericOpenGLRenderWindow` 实例，将 VTK 的渲染管线桥接到 Qt 的 OpenGL 环境中。

### 21.2.2 在 Qt 布局中嵌入 VTK 窗口

将 QVTKOpenGLNativeWidget 嵌入 Qt 界面的核心步骤：

1. 创建 `QVTKOpenGLNativeWidget` 实例（需要传入 parent widget）
2. 获取其内部的 `vtkGenericOpenGLRenderWindow`
3. 创建一个 `vtkRenderer` 并添加到渲染窗口中
4. 创建 `vtkRenderWindowInteractor` 并绑定到渲染窗口
5. 将 widget 添加到 Qt 布局管理器中
6. 在 widget 上调用 `show()`

基本代码结构：

```cpp
#include <QVTKOpenGLNativeWidget.h>
#include <vtkGenericOpenGLRenderWindow.h>
#include <vtkRenderer.h>
#include <vtkRenderWindowInteractor.h>

// 创建 VTK Qt widget
QVTKOpenGLNativeWidget* vtkWidget = new QVTKOpenGLNativeWidget(this);

// 获取内部的 VTK 渲染窗口
vtkGenericOpenGLRenderWindow* renderWindow = vtkWidget->renderWindow();
// 或者使用 GetRenderWindow() 成员函数

// 创建渲染器和交互器
vtkNew<vtkRenderer> renderer;
renderWindow->AddRenderer(renderer);

vtkNew<vtkRenderWindowInteractor> interactor;
interactor->SetRenderWindow(renderWindow);
interactor->Initialize();

// 添加到布局
QVBoxLayout* layout = new QVBoxLayout;
layout->addWidget(vtkWidget);
setLayout(layout);
```

### 21.2.3 连接 Qt 信号到 VTK 更新

信号与槽机制是实现 Qt 与 VTK 交互的桥梁。典型的使用方式为：用户在 Qt 控件上的操作触发信号，槽函数中修改 VTK 管线的参数并调用 `renderWindow->Render()` 更新显示。

```cpp
// 滑块控制等值面值
connect(isoSlider, &QSlider::valueChanged, this, [this](int value) {
    double isoValue = value / 100.0;
    marchingCubes->SetValue(0, isoValue);
    renderWindow->Render();
});

// 下拉框切换颜色映射
connect(colorCombo, QOverload<int>::of(&QComboBox::currentIndexChanged),
        this, [this](int index) {
    lookupTable->SetNumberOfTableValues(4);
    switch (index) {
        case 0: // 彩虹色
            lookupTable->SetTableValue(0, 0.0, 0.0, 1.0, 1.0);
            lookupTable->SetTableValue(1, 0.0, 1.0, 0.0, 1.0);
            lookupTable->SetTableValue(2, 1.0, 1.0, 0.0, 1.0);
            lookupTable->SetTableValue(3, 1.0, 0.0, 0.0, 1.0);
            break;
        case 1: // 灰度
            lookupTable->SetTableValue(0, 0.0, 0.0, 0.0, 1.0);
            lookupTable->SetTableValue(1, 0.33, 0.33, 0.33, 1.0);
            lookupTable->SetTableValue(2, 0.66, 0.66, 0.66, 1.0);
            lookupTable->SetTableValue(3, 1.0, 1.0, 1.0, 1.0);
            break;
    }
    lookupTable->Modified();
    renderWindow->Render();
});
```

### 21.2.4 示例：QMainWindow 嵌入 VTK 窗口

下面展示一个最基本但完整的 QMainWindow 嵌入 VTK 渲染窗口的示例：

```cpp
#include <QApplication>
#include <QMainWindow>
#include <QVBoxLayout>
#include <QVTKOpenGLNativeWidget.h>

#include <vtkActor.h>
#include <vtkConeSource.h>
#include <vtkGenericOpenGLRenderWindow.h>
#include <vtkNamedColors.h>
#include <vtkNew.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderer.h>

class MainWindow : public QMainWindow {
public:
    MainWindow(QWidget* parent = nullptr) : QMainWindow(parent) {
        // 创建中央 widget 和布局
        QWidget* central = new QWidget(this);
        QVBoxLayout* layout = new QVBoxLayout(central);

        // 创建 VTK 渲染 Widget
        vtkWidget = new QVTKOpenGLNativeWidget(central);
        layout->addWidget(vtkWidget);
        setCentralWidget(central);

        // 设置渲染管线
        SetupVTKPipeline();

        resize(800, 600);
        setWindowTitle("VTK + Qt 基础示例");
    }

private:
    void SetupVTKPipeline() {
        vtkNew<vtkNamedColors> colors;

        // 创建一个圆锥体数据源
        vtkNew<vtkConeSource> cone;
        cone->SetHeight(3.0);
        cone->SetRadius(1.0);
        cone->SetResolution(50);

        // 映射器
        vtkNew<vtkPolyDataMapper> mapper;
        mapper->SetInputConnection(cone->GetOutputPort());

        // Actor
        vtkNew<vtkActor> actor;
        actor->SetMapper(mapper);
        actor->GetProperty()->SetColor(
            colors->GetColor3d("Tomato").GetData());

        // 渲染器
        vtkNew<vtkRenderer> renderer;
        renderer->AddActor(actor);
        renderer->SetBackground(
            colors->GetColor3d("SteelBlue").GetData());

        // 获取渲染窗口并添加渲染器
        vtkGenericOpenGLRenderWindow* renderWindow =
            vtkWidget->renderWindow();
        renderWindow->AddRenderer(renderer);
    }

    QVTKOpenGLNativeWidget* vtkWidget;
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    MainWindow window;
    window.show();
    return app.exec();
}
```

## 21.3 Qt与VTK的数据交互

### 21.3.1 Qt Widget 到 VTK 管线

这是最常用的交互方向：用户在 Qt 控件上的操作改变 VTK 管线的参数。整个流程如下：

```
用户操作 Qt Widget → Qt 信号发射 → 槽函数执行 →
修改 VTK Filter 参数 → 调用 Render() → 视图更新
```

关键设计原则：

1. **参数映射**：Qt 控件的值域需要映射到 VTK 参数的合理范围。例如，`QSlider` 的整数值 0-100 可以映射到等值面的 0.0-2.0 范围。
2. **即時反馈**：对于滑块等连续变化控件，应在 `valueChanged` 信号中直接触发渲染更新，给用户即时的视觉反馈。
3. **批量更新**：对于多个相关参数的修改，可以先暂停渲染、修改完所有参数后再一次性渲染，避免中间状态闪烁。

```cpp
// 参数映射示例
connect(densitySlider, &QSlider::valueChanged, this, [this](int val) {
    // QSlider 范围 0-200 → 实际参数范围 0.0-1.0
    double density = val / 200.0;
    thresholdFilter->SetUpperThreshold(density);
    m_renderWindow->Render();
});
```

### 21.3.2 VTK 到 Qt Widget

数据也可以从 VTK 流回 Qt 界面，例如将数据集统计信息显示在状态栏、属性面板或表格中：

```cpp
// 从 VTK 中提取数据并更新 Qt Label
void UpdateStatsDisplay() {
    vtkDataSet* data = reader->GetOutput();
    vtkIdType numPoints = data->GetNumberOfPoints();
    vtkIdType numCells  = data->GetNumberOfCells();

    double bounds[6];
    data->GetBounds(bounds);

    QString info = QString("点数: %1, 单元数: %2, "
                           "范围: [%3, %4] x [%5, %6] x [%7, %8]")
        .arg(numPoints).arg(numCells)
        .arg(bounds[0], 0, 'f', 2).arg(bounds[1], 0, 'f', 2)
        .arg(bounds[2], 0, 'f', 2).arg(bounds[3], 0, 'f', 2)
        .arg(bounds[4], 0, 'f', 2).arg(bounds[5], 0, 'f', 2);

    statusLabel->setText(info);
    statusBar()->showMessage(info);
}
```

### 21.3.3 线程安全

VTK 的渲染管线**不是线程安全的**。默认情况下，所有 VTK 操作应在主线程（Qt 的 GUI 线程）中执行。原因如下：

- VTK 的渲染类直接操作 OpenGL 状态，而 OpenGL 上下文是线程绑定的。
- VTK 内部大量使用引用计数和观察者模式，跨线程访问可能导致竞态条件。
- Qt 的 Widget 操作也必须在 GUI 线程中进行。

**标准做法**：

- 所有 VTK 管线的创建、修改和渲染操作都在主线程执行。
- 如果需要后台加载大型数据集，使用 `QThread` 或 `QtConcurrent` 在后台线程中执行文件 I/O 和数据读取，然后通过信号/槽将读取的数据传回主线程，在主线程中完成管线构建。

```cpp
// 后台加载示例
class DataLoader : public QObject {
    Q_OBJECT
public:
    void load(const QString& path) {
        // 在后台线程中读取文件
        vtkNew<vtkXMLImageDataReader> reader;
        reader->SetFileName(path.toStdString().c_str());
        reader->Update(); // 耗时操作，在后台线程
        // 将结果通过信号传回主线程
        emit dataLoaded(reader->GetOutput());
    }
signals:
    void dataLoaded(vtkImageData* data);
};
```

### 21.3.4 使用 VTK Observer 更新 Qt UI

VTK 的观察者/命令模式（Observer/Command Pattern）可以用于在 VTK 事件发生时更新 Qt 界面。不过需要注意，Observer 的回调可能在任意线程中执行，更新 Qt UI 时需要确保在主线程中进行：

```cpp
#include <vtkCallbackCommand.h>

// 创建观察者回调
vtkNew<vtkCallbackCommand> progressCallback;
progressCallback->SetCallback([](vtkObject* caller, unsigned long eventId,
                                 void* clientData, void* callData) {
    // VTK 进度事件
    double progress =
        *(reinterpret_cast<double*>(callData));
    // 注意：确保更新 UI 的操作在主线程执行
    // 可以使用 QMetaObject::invokeMethod 跨线程调度
    QMetaObject::invokeMethod(
        QApplication::instance(),
        [progress]() {
            // 更新进度条
            if (auto* bar = QApplication::activeWindow()
                    ->findChild<QProgressBar*>("progressBar")) {
                bar->setValue(static_cast<int>(progress * 100));
            }
        },
        Qt::QueuedConnection);
});

filter->AddObserver(vtkCommand::ProgressEvent, progressCallback);
```

## 21.4 完整的Qt+VTK应用框架

### 21.4.1 总体架构设计

一个专业的 Qt+VTK 应用程序通常采用以下布局：

```
+--------------------------------------------------------+
|  Menu Bar  (文件 视图 工具 帮助)                         |
+--------------------------------------------------------+
|  Tool Bar  (打开 保存截图 重置视图)                       |
+--------------------------------------------------------+
+---------------------------+----------------------------+
|                           |                            |
|     VTK 渲染视口          |   [控制面板 Dock]           |
|   (QVTKOpenGLNativeWidget)|   等值面值: [====滑块====] |
|                           |   颜色映射: [下拉框]       |
|                           |   显示轮廓: [复选框]       |
|                           |   不透明度: [====滑块====] |
|                           |                           |
|                           |   [数据信息]               |
|                           |   点数: 1,048,576          |
|                           |   单元数: 1,000,000        |
|                           |   ...                      |
+---------------------------+----------------------------+
|  Status Bar  (显示当前操作状态和数据信息)                |
+--------------------------------------------------------+
```

### 21.4.2 各组件职责

- **QMainWindow**：主窗口容器，管理菜单、工具栏、状态栏和中央区域
- **QVTKOpenGLNativeWidget（中央区域）**：VTK 的三维渲染视口，占据主要界面空间
- **QDockWidget（右侧面板）**：控制面板容器，可浮动、可关闭、可调整大小
- **QSlider / QDoubleSpinBox**：调整连续参数，如等值面值、轮廓阈值
- **QComboBox**：选择离散选项，如颜色映射方案、管线切换
- **QCheckBox**：开关布尔选项，如是否显示轮廓线、是否启用光照
- **QTableWidget / QTreeWidget**：显示数据统计和结构化信息
- **QStatusBar**：显示当前操作状态、坐标信息、性能数据
- **QFileDialog**：文件打开对话框，选择并加载数据文件

### 21.4.3 控制面板设计

```cpp
// 典型的控制面板创建代码
QDockWidget* CreateControlPanel() {
    QDockWidget* dock = new QDockWidget("控制面板", this);
    QWidget* panel = new QWidget(dock);
    QVBoxLayout* layout = new QVBoxLayout(panel);

    // --- 等值面控制组 ---
    QGroupBox* isoGroup = new QGroupBox("等值面设置");
    QVBoxLayout* isoLayout = new QVBoxLayout(isoGroup);

    QLabel* isoLabel = new QLabel("等值面值:");
    QSlider* isoSlider = new QSlider(Qt::Horizontal);
    isoSlider->setRange(0, 200);
    isoSlider->setValue(100);
    QDoubleSpinBox* isoSpin = new QDoubleSpinBox;
    isoSpin->setRange(0.0, 2.0);
    isoSpin->setSingleStep(0.01);
    isoSpin->setValue(1.0);

    QHBoxLayout* isoHBox = new QHBoxLayout;
    isoHBox->addWidget(isoLabel);
    isoHBox->addWidget(isoSpin);
    isoLayout->addLayout(isoHBox);
    isoLayout->addWidget(isoSlider);

    // 滑块和数值框双向同步
    connect(isoSlider, &QSlider::valueChanged, isoSpin,
            [isoSpin](int v) { isoSpin->setValue(v / 100.0); });
    connect(isoSpin, QOverload<double>::of(&QDoubleSpinBox::valueChanged),
            isoSlider, [isoSlider](double v) {
                isoSlider->setValue(static_cast<int>(v * 100)); });

    layout->addWidget(isoGroup);

    // --- 颜色映射控制组 ---
    QGroupBox* colorGroup = new QGroupBox("颜色映射");
    QVBoxLayout* colorLayout = new QVBoxLayout(colorGroup);
    QComboBox* cmapCombo = new QComboBox;
    cmapCombo->addItems({"彩虹色", "灰度", "冷暖色", "蓝-红发散"});
    colorLayout->addWidget(cmapCombo);
    layout->addWidget(colorGroup);

    // --- 显示选项 ---
    QGroupBox* displayGroup = new QGroupBox("显示选项");
    QVBoxLayout* displayLayout = new QVBoxLayout(displayGroup);
    QCheckBox* outlineCheck = new QCheckBox("显示轮廓线");
    QCheckBox* wireframeCheck = new QCheckBox("线框模式");
    QCheckBox* axesCheck = new QCheckBox("显示坐标轴");
    displayLayout->addWidget(outlineCheck);
    displayLayout->addWidget(wireframeCheck);
    displayLayout->addWidget(axesCheck);
    layout->addWidget(displayGroup);

    // --- 数据信息 ---
    QGroupBox* infoGroup = new QGroupBox("数据信息");
    QVBoxLayout* infoLayout = new QVBoxLayout(infoGroup);
    QLabel* pointsLabel = new QLabel("点数: --");
    QLabel* cellsLabel  = new QLabel("单元数: --");
    QLabel* boundsLabel = new QLabel("范围: --");
    infoLayout->addWidget(pointsLabel);
    infoLayout->addWidget(cellsLabel);
    infoLayout->addWidget(boundsLabel);
    layout->addWidget(infoGroup);

    layout->addStretch();
    dock->setWidget(panel);
    return dock;
}
```

### 21.4.4 文件菜单与加载

```cpp
void CreateFileMenu() {
    QMenu* fileMenu = menuBar()->addMenu("文件(&F)");

    QAction* openAction = fileMenu->addAction("打开(&O)...");
    openAction->setShortcut(QKeySequence::Open);
    connect(openAction, &QAction::triggered, this, &MainWindow::OnFileOpen);

    QAction* saveAction = fileMenu->addAction("保存截图(&S)...");
    saveAction->setShortcut(QKeySequence("Ctrl+Shift+S"));
    connect(saveAction, &QAction::triggered, this, &MainWindow::OnSaveScreenshot);

    fileMenu->addSeparator();

    QAction* exitAction = fileMenu->addAction("退出(&X)");
    exitAction->setShortcut(QKeySequence::Quit);
    connect(exitAction, &QAction::triggered, this, &QWidget::close);
}

void OnFileOpen() {
    QString fileName = QFileDialog::getOpenFileName(
        this, "打开数据文件", QString(),
        "VTK 文件 (*.vtk *.vti *.vtu *.vts *.vtr *.vtp *.vtm);;"
        "所有文件 (*)");

    if (fileName.isEmpty()) return;

    LoadDataFile(fileName);
}
```

### 21.4.5 截图保存

```cpp
void OnSaveScreenshot() {
    QString fileName = QFileDialog::getSaveFileName(
        this, "保存截图", "screenshot.png",
        "PNG 图像 (*.png);;JPEG 图像 (*.jpg);;BMP 图像 (*.bmp)");

    if (fileName.isEmpty()) return;

    // 使用 VTK 的窗口到图像滤镜保存截图
    vtkNew<vtkWindowToImageFilter> w2if;
    w2if->SetInput(m_renderWindow);
    w2if->SetScale(1);
    w2if->SetInputBufferTypeToRGBA();
    w2if->ReadFrontBufferOff();
    w2if->Update();

    vtkNew<vtkPNGWriter> writer;
    writer->SetFileName(fileName.toStdString().c_str());
    writer->SetInputConnection(w2if->GetOutputPort());
    writer->Write();

    statusBar()->showMessage(
        QString("截图已保存: %1").arg(fileName), 3000);
}
```

## 21.5 代码示例

以下是一个完整的 Qt+VTK 应用程序，包含文件加载、等值面提取、实时参数调整和截图保存等功能。该示例演示了如何加载 VTK 支持的体数据文件，使用 Marching Cubes 算法提取等值面，并通过控制面板实时调整参数。

### 21.5.1 主头文件 MainWindow.h

```cpp
#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>
#include <QSlider>
#include <QDoubleSpinBox>
#include <QComboBox>
#include <QCheckBox>
#include <QPushButton>
#include <QLabel>
#include <QGroupBox>
#include <QProgressBar>
#include <QAction>

class QVTKOpenGLNativeWidget;
class vtkGenericOpenGLRenderWindow;
class vtkRenderer;
class vtkActor;
class vtkMarchingCubes;
class vtkColorTransferFunction;
class vtkLookupTable;
class vtkDataSet;
class vtkOutlineFilter;
class vtkPolyDataMapper;
class vtkImageData;
class vtkScalarBarActor;
class vtkAxesActor;

/**
 * @brief 主窗口类 - 完整的 Qt+VTK 科学可视化应用框架
 *
 * 功能:
 * - 基于 QMainWindow 的应用框架
 * - 通过 QVTKOpenGLNativeWidget 嵌入 VTK 渲染窗口
 * - 等值面值的实时滑块调整
 * - 颜色映射切换
 * - 文件加载 (支持 VTK 格式)
 * - 截图保存
 * - 状态栏显示数据统计信息
 */
class MainWindow : public QMainWindow {
    Q_OBJECT

public:
    explicit MainWindow(QWidget* parent = nullptr);
    ~MainWindow() override;

private slots:
    // === 文件操作 ===
    void OnFileOpen();
    void OnSaveScreenshot();

    // === 参数调整 ===
    void OnIsoValueChanged(double value);
    void OnColorMapChanged(int index);
    void OnShowOutlineToggled(bool checked);
    void OnShowWireframeToggled(bool checked);
    void OnShowAxesToggled(bool checked);
    void OnResetView();

private:
    // === UI 构建 ===
    void SetupUI();
    void CreateMenuBar();
    void CreateToolBar();
    void CreateStatusBar();
    QDockWidget* CreateControlPanel();

    // === VTK 管线 ===
    void SetupVTKPipeline();
    void LoadDataFile(const QString& filePath);
    void UpdatePipeline();
    void UpdateDataInfoDisplay();
    void ApplyColorMap(int index);

    // === 颜色映射预设 ===
    void SetPresetRainbow();
    void SetPresetGrayscale();
    void SetPresetCoolWarm();
    void SetPresetBlueRed();

    // === Qt Widgets ===
    QVTKOpenGLNativeWidget* m_vtkWidget = nullptr;
    QSlider*                m_isoSlider = nullptr;
    QDoubleSpinBox*         m_isoSpinBox = nullptr;
    QComboBox*              m_colorCombo = nullptr;
    QCheckBox*              m_outlineCheck = nullptr;
    QCheckBox*              m_wireframeCheck = nullptr;
    QCheckBox*              m_axesCheck = nullptr;
    QPushButton*            m_resetBtn = nullptr;
    QLabel*                 m_pointsLabel = nullptr;
    QLabel*                 m_cellsLabel = nullptr;
    QLabel*                 m_boundsLabel = nullptr;
    QLabel*                 m_scalarRangeLabel = nullptr;
    QProgressBar*           m_progressBar = nullptr;

    // === VTK 对象 ===
    vtkSmartPointer<vtkGenericOpenGLRenderWindow> m_renderWindow;
    vtkSmartPointer<vtkRenderer>                  m_renderer;
    vtkSmartPointer<vtkActor>                     m_mainActor;
    vtkSmartPointer<vtkActor>                     m_outlineActor;
    vtkSmartPointer<vtkMarchingCubes>             m_marchingCubes;
    vtkSmartPointer<vtkLookupTable>               m_lookupTable;
    vtkSmartPointer<vtkColorTransferFunction>     m_colorFunc;
    vtkSmartPointer<vtkPolyDataMapper>            m_mainMapper;
    vtkSmartPointer<vtkPolyDataMapper>            m_outlineMapper;
    vtkSmartPointer<vtkOutlineFilter>             m_outlineFilter;
    vtkSmartPointer<vtkScalarBarActor>            m_scalarBar;
    vtkSmartPointer<vtkAxesActor>                 m_axesActor;
    vtkSmartPointer<vtkDataSet>                   m_currentData;
    vtkSmartPointer<vtkImageData>                 m_imageData;

    // 当前数据范围
    double m_dataRange[2] = {0.0, 1.0};
    bool   m_hasData = false;
};

#endif // MAINWINDOW_H
```

### 21.5.2 主实现文件 MainWindow.cpp

```cpp
#include "MainWindow.h"

// === Qt 头文件 ===
#include <QApplication>
#include <QFileDialog>
#include <QMenuBar>
#include <QToolBar>
#include <QStatusBar>
#include <QDockWidget>
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QFormLayout>
#include <QMessageBox>
#include <QFileInfo>
#include <QScreen>

// === VTK Qt 集成 ===
#include <QVTKOpenGLNativeWidget.h>

// === VTK 核心 ===
#include <vtkActor.h>
#include <vtkAxesActor.h>
#include <vtkCamera.h>
#include <vtkColorTransferFunction.h>
#include <vtkDataSet.h>
#include <vtkDataSetMapper.h>
#include <vtkGenericOpenGLRenderWindow.h>
#include <vtkImageData.h>
#include <vtkLookupTable.h>
#include <vtkMarchingCubes.h>
#include <vtkNamedColors.h>
#include <vtkNew.h>
#include <vtkOutlineFilter.h>
#include <vtkPNGWriter.h>
#include <vtkPolyDataMapper.h>
#include <vtkProperty.h>
#include <vtkRenderWindow.h>
#include <vtkRenderer.h>
#include <vtkScalarBarActor.h>
#include <vtkSmartPointer.h>
#include <vtkSphereSource.h>
#include <vtkTextProperty.h>
#include <vtkWindowToImageFilter.h>
#include <vtkXMLImageDataReader.h>
#include <vtkXMLPolyDataReader.h>
#include <vtkXMLUnstructuredGridReader.h>
#include <vtkStructuredPointsReader.h>

// ============================================================================
// 构造函数与析构函数
// ============================================================================

MainWindow::MainWindow(QWidget* parent)
    : QMainWindow(parent) {
    SetupUI();
    SetupVTKPipeline();
}

MainWindow::~MainWindow() = default;

// ============================================================================
// UI 构建
// ============================================================================

void MainWindow::SetupUI() {
    // 窗口基本属性
    setWindowTitle("VTK + Qt 科学可视化应用");
    // 获取屏幕尺寸的 70% 作为初始窗口大小
    QScreen* screen = QGuiApplication::primaryScreen();
    QRect screenRect = screen->availableGeometry();
    int w = static_cast<int>(screenRect.width() * 0.7);
    int h = static_cast<int>(screenRect.height() * 0.7);
    resize(w, h);

    // 中央 Widget - VTK 渲染视口
    QWidget* central = new QWidget(this);
    QVBoxLayout* centralLayout = new QVBoxLayout(central);
    centralLayout->setContentsMargins(0, 0, 0, 0);

    m_vtkWidget = new QVTKOpenGLNativeWidget(central);
    centralLayout->addWidget(m_vtkWidget);
    setCentralWidget(central);

    // 控制面板 (右侧 Dock)
    QDockWidget* controlDock = CreateControlPanel();
    addDockWidget(Qt::RightDockWidgetArea, controlDock);

    // 菜单、工具栏、状态栏
    CreateMenuBar();
    CreateToolBar();
    CreateStatusBar();
}

void MainWindow::CreateMenuBar() {
    // === 文件菜单 ===
    QMenu* fileMenu = menuBar()->addMenu("文件(&F)");

    QAction* openAction = fileMenu->addAction("打开(&O)...");
    openAction->setShortcut(QKeySequence::Open);
    connect(openAction, &QAction::triggered, this, &MainWindow::OnFileOpen);

    QAction* saveAction = fileMenu->addAction("保存截图(&S)...");
    saveAction->setShortcut(QKeySequence("Ctrl+Shift+S"));
    connect(saveAction, &QAction::triggered, this, &MainWindow::OnSaveScreenshot);

    fileMenu->addSeparator();

    QAction* exitAction = fileMenu->addAction("退出(&X)");
    exitAction->setShortcut(QKeySequence::Quit);
    connect(exitAction, &QAction::triggered, this, &QWidget::close);

    // === 视图菜单 ===
    QMenu* viewMenu = menuBar()->addMenu("视图(&V)");

    QAction* resetAction = viewMenu->addAction("重置视图(&R)");
    resetAction->setShortcut(QKeySequence("R"));
    connect(resetAction, &QAction::triggered, this, &MainWindow::OnResetView);

    QAction* togglePanelAction = viewMenu->addAction("显示/隐藏控制面板");
    connect(togglePanelAction, &QAction::triggered, this, [this]() {
        // 找到控制面板 dock 并切换可见性
        for (auto* dock : findChildren<QDockWidget*>()) {
            dock->setVisible(!dock->isVisible());
        }
    });

    // === 帮助菜单 ===
    QMenu* helpMenu = menuBar()->addMenu("帮助(&H)");
    QAction* aboutAction = helpMenu->addAction("关于(&A)...");
    connect(aboutAction, &QAction::triggered, this, [this]() {
        QMessageBox::about(this, "关于 VTK+Qt 应用",
            "VTK 9.5.2 + Qt 集成示例\n\n"
            "功能:\n"
            "- 加载 VTK 数据文件\n"
            "- 等值面可视化\n"
            "- 实时参数调整\n"
            "- 截图保存\n");
    });
}

void MainWindow::CreateToolBar() {
    QToolBar* toolbar = addToolBar("主工具栏");
    toolbar->setMovable(false);

    QAction* openAction = toolbar->addAction("打开");
    openAction->setToolTip("打开数据文件");
    connect(openAction, &QAction::triggered, this, &MainWindow::OnFileOpen);

    QAction* saveAction = toolbar->addAction("截图");
    saveAction->setToolTip("保存当前视图截图");
    connect(saveAction, &QAction::triggered, this, &MainWindow::OnSaveScreenshot);

    toolbar->addSeparator();

    QAction* resetAction = toolbar->addAction("重置");
    resetAction->setToolTip("重置相机视图");
    connect(resetAction, &QAction::triggered, this, &MainWindow::OnResetView);
}

void MainWindow::CreateStatusBar() {
    m_progressBar = new QProgressBar;
    m_progressBar->setMaximumWidth(200);
    m_progressBar->setMaximumHeight(16);
    m_progressBar->setVisible(false);
    statusBar()->addPermanentWidget(m_progressBar);
    statusBar()->showMessage("就绪 - 请通过 '文件 > 打开' 加载数据文件");
}

QDockWidget* MainWindow::CreateControlPanel() {
    QDockWidget* dock = new QDockWidget("控制面板", this);
    dock->setFeatures(QDockWidget::DockWidgetMovable |
                      QDockWidget::DockWidgetFloatable);
    dock->setMinimumWidth(280);

    QWidget* panel = new QWidget(dock);
    QVBoxLayout* mainLayout = new QVBoxLayout(panel);
    mainLayout->setSpacing(12);

    // --- 等值面控制 ---
    QGroupBox* isoGroup = new QGroupBox("等值面设置");
    QVBoxLayout* isoLayout = new QVBoxLayout(isoGroup);

    QHBoxLayout* isoValueRow = new QHBoxLayout;
    QLabel* isoLabel = new QLabel("等值:");
    m_isoSpinBox = new QDoubleSpinBox;
    m_isoSpinBox->setRange(0.0, 1.0);
    m_isoSpinBox->setSingleStep(0.01);
    m_isoSpinBox->setDecimals(3);
    m_isoSpinBox->setValue(0.5);
    isoValueRow->addWidget(isoLabel);
    isoValueRow->addWidget(m_isoSpinBox);

    m_isoSlider = new QSlider(Qt::Horizontal);
    m_isoSlider->setRange(0, 1000);
    m_isoSlider->setValue(500);

    // 滑块与数值框双向绑定
    connect(m_isoSlider, &QSlider::valueChanged, this, [this](int v) {
        m_isoSpinBox->blockSignals(true);
        m_isoSpinBox->setValue(v / 1000.0 * m_dataRange[1]);
        m_isoSpinBox->blockSignals(false);
        OnIsoValueChanged(v / 1000.0 * m_dataRange[1]);
    });
    connect(m_isoSpinBox, QOverload<double>::of(&QDoubleSpinBox::valueChanged),
            this, [this](double v) {
        m_isoSlider->blockSignals(true);
        m_isoSlider->setValue(static_cast<int>(
            v / m_dataRange[1] * 1000));
        m_isoSlider->blockSignals(false);
        OnIsoValueChanged(v);
    });

    isoLayout->addLayout(isoValueRow);
    isoLayout->addWidget(m_isoSlider);
    mainLayout->addWidget(isoGroup);

    // --- 颜色映射 ---
    QGroupBox* colorGroup = new QGroupBox("颜色映射");
    QVBoxLayout* colorLayout = new QVBoxLayout(colorGroup);
    m_colorCombo = new QComboBox;
    m_colorCombo->addItems({"彩虹色", "灰度", "冷暖色", "蓝-红发散"});
    connect(m_colorCombo, QOverload<int>::of(&QComboBox::currentIndexChanged),
            this, &MainWindow::OnColorMapChanged);
    colorLayout->addWidget(m_colorCombo);
    mainLayout->addWidget(colorGroup);

    // --- 显示选项 ---
    QGroupBox* displayGroup = new QGroupBox("显示选项");
    QVBoxLayout* displayLayout = new QVBoxLayout(displayGroup);

    m_outlineCheck = new QCheckBox("显示轮廓框");
    connect(m_outlineCheck, &QCheckBox::toggled,
            this, &MainWindow::OnShowOutlineToggled);

    m_wireframeCheck = new QCheckBox("线框模式");
    connect(m_wireframeCheck, &QCheckBox::toggled,
            this, &MainWindow::OnShowWireframeToggled);

    m_axesCheck = new QCheckBox("显示坐标轴");
    connect(m_axesCheck, &QCheckBox::toggled,
            this, &MainWindow::OnShowAxesToggled);

    displayLayout->addWidget(m_outlineCheck);
    displayLayout->addWidget(m_wireframeCheck);
    displayLayout->addWidget(m_axesCheck);
    mainLayout->addWidget(displayGroup);

    // --- 操作按钮 ---
    m_resetBtn = new QPushButton("重置视图");
    connect(m_resetBtn, &QPushButton::clicked, this, &MainWindow::OnResetView);
    mainLayout->addWidget(m_resetBtn);

    // --- 数据信息 ---
    QGroupBox* infoGroup = new QGroupBox("数据信息");
    QVBoxLayout* infoLayout = new QVBoxLayout(infoGroup);

    m_pointsLabel = new QLabel("点数: --");
    m_cellsLabel  = new QLabel("单元数: --");
    m_boundsLabel = new QLabel("范围: --");
    m_scalarRangeLabel = new QLabel("标量范围: --");

    infoLayout->addWidget(m_pointsLabel);
    infoLayout->addWidget(m_cellsLabel);
    infoLayout->addWidget(m_boundsLabel);
    infoLayout->addWidget(m_scalarRangeLabel);
    mainLayout->addWidget(infoGroup);

    mainLayout->addStretch();
    dock->setWidget(panel);
    return dock;
}

// ============================================================================
// VTK 管线初始化
// ============================================================================

void MainWindow::SetupVTKPipeline() {
    // 获取 VTK Widget 的渲染窗口
    m_renderWindow = m_vtkWidget->renderWindow();

    // 创建渲染器
    m_renderer = vtkSmartPointer<vtkRenderer>::New();
    m_renderer->SetBackground(0.2, 0.2, 0.25);
    m_renderer->SetBackground2(0.1, 0.1, 0.15);
    m_renderer->GradientBackgroundOn();
    m_renderWindow->AddRenderer(m_renderer);

    // --- 创建一个默认的球体源作为占位符 ---
    vtkNew<vtkSphereSource> sphere;
    sphere->SetThetaResolution(50);
    sphere->SetPhiResolution(50);
    sphere->SetRadius(1.0);

    vtkNew<vtkPolyDataMapper> sphereMapper;
    sphereMapper->SetInputConnection(sphere->GetOutputPort());

    m_mainActor = vtkSmartPointer<vtkActor>::New();
    m_mainActor->SetMapper(sphereMapper);
    m_mainActor->GetProperty()->SetColor(0.6, 0.6, 0.8);
    m_mainActor->GetProperty()->SetOpacity(0.5);
    m_renderer->AddActor(m_mainActor);

    // --- 轮廓框 Actor ---
    vtkNew<vtkOutlineFilter> outlineFilter;
    outlineFilter->SetInputConnection(sphere->GetOutputPort());

    m_outlineMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
    m_outlineMapper->SetInputConnection(outlineFilter->GetOutputPort());

    m_outlineActor = vtkSmartPointer<vtkActor>::New();
    m_outlineActor->SetMapper(m_outlineMapper);
    m_outlineActor->GetProperty()->SetColor(0.8, 0.8, 0.8);
    m_outlineActor->GetProperty()->SetLineWidth(1.0);
    m_outlineActor->SetVisibility(false);
    m_renderer->AddActor(m_outlineActor);

    // --- 标量条 ---
    m_scalarBar = vtkSmartPointer<vtkScalarBarActor>::New();
    m_scalarBar->SetNumberOfLabels(5);
    m_scalarBar->GetTitleTextProperty()->SetColor(0.9, 0.9, 0.9);
    m_scalarBar->GetLabelTextProperty()->SetColor(0.9, 0.9, 0.9);
    m_scalarBar->SetVisibility(false);
    m_renderer->AddActor2D(m_scalarBar);

    // --- 坐标轴 ---
    m_axesActor = vtkSmartPointer<vtkAxesActor>::New();
    m_axesActor->SetTotalLength(1.5, 1.5, 1.5);
    m_axesActor->SetVisibility(false);
    m_renderer->AddActor(m_axesActor);

    // 初始化相机
    m_renderer->ResetCamera();
}

// ============================================================================
// 文件加载
// ============================================================================

void MainWindow::OnFileOpen() {
    QString fileName = QFileDialog::getOpenFileName(
        this, "打开数据文件", QString(),
        "VTK 文件 (*.vtk *.vti *.vtu *.vts *.vtr *.vtp *.vtm);;"
        "VTK ImageData (*.vti);;"
        "VTK StructuredPoints (*.vtk);;"
        "所有文件 (*)");

    if (fileName.isEmpty()) return;

    LoadDataFile(fileName);
}

void MainWindow::LoadDataFile(const QString& filePath) {
    statusBar()->showMessage("正在加载: " + filePath + " ...");
    m_progressBar->setVisible(true);
    m_progressBar->setValue(0);
    QApplication::processEvents();

    QFileInfo fileInfo(filePath);
    QString suffix = fileInfo.suffix().toLower();

    bool loaded = false;

    // 根据扩展名选择合适的 Reader
    try {
        if (suffix == "vti") {
            vtkNew<vtkXMLImageDataReader> reader;
            reader->SetFileName(filePath.toStdString().c_str());
            reader->Update();
            m_imageData = reader->GetOutput();
            m_currentData = m_imageData;
            loaded = true;
        } else if (suffix == "vtk") {
            // .vtk 文件可能是 ImageData (StructuredPoints) 或其他格式
            vtkNew<vtkStructuredPointsReader> reader;
            reader->SetFileName(filePath.toStdString().c_str());
            reader->Update();
            m_imageData = vtkImageData::SafeDownCast(reader->GetOutput());
            m_currentData = reader->GetOutput();
            loaded = true;
        } else if (suffix == "vtu") {
            vtkNew<vtkXMLUnstructuredGridReader> reader;
            reader->SetFileName(filePath.toStdString().c_str());
            reader->Update();
            m_currentData = reader->GetOutput();
            loaded = true;
        } else if (suffix == "vtp") {
            vtkNew<vtkXMLPolyDataReader> reader;
            reader->SetFileName(filePath.toStdString().c_str());
            reader->Update();
            m_currentData = reader->GetOutput();
            loaded = true;
        } else {
            // 尝试通用的读取方式
            vtkNew<vtkXMLImageDataReader> reader;
            reader->SetFileName(filePath.toStdString().c_str());
            reader->Update();
            m_imageData = reader->GetOutput();
            m_currentData = m_imageData;
            loaded = true;
        }
    } catch (...) {
        QMessageBox::warning(this, "加载失败",
            "无法加载文件: " + filePath + "\n请检查文件格式是否正确。");
        m_progressBar->setVisible(false);
        return;
    }

    if (loaded && m_currentData) {
        m_hasData = true;

        // 获取数据范围
        if (m_imageData && m_imageData->GetPointData()->GetScalars()) {
            double range[2];
            m_imageData->GetPointData()->GetScalars()->GetRange(range);
            m_dataRange[0] = range[0];
            m_dataRange[1] = range[1];
        }

        // 更新等值面控件的范围
        m_isoSpinBox->blockSignals(true);
        m_isoSpinBox->setRange(m_dataRange[0], m_dataRange[1]);
        m_isoSpinBox->setValue((m_dataRange[0] + m_dataRange[1]) * 0.5);
        m_isoSpinBox->blockSignals(false);

        m_isoSlider->blockSignals(true);
        m_isoSlider->setValue(500);
        m_isoSlider->blockSignals(false);

        // 构建管线
        UpdatePipeline();
        UpdateDataInfoDisplay();
        ApplyColorMap(m_colorCombo->currentIndex());

        // 启用控件
        m_isoSlider->setEnabled(true);
        m_isoSpinBox->setEnabled(true);
        m_colorCombo->setEnabled(true);
        m_outlineCheck->setEnabled(true);
        m_wireframeCheck->setEnabled(true);
        m_axesCheck->setEnabled(true);

        m_renderer->ResetCamera();
        m_renderWindow->Render();

        QFileInfo fi(filePath);
        statusBar()->showMessage(
            QString("已加载: %1 (点数: %2)")
                .arg(fi.fileName())
                .arg(m_currentData->GetNumberOfPoints()), 5000);
    }

    m_progressBar->setVisible(false);
}

// ============================================================================
// 管线更新
// ============================================================================

void MainWindow::UpdatePipeline() {
    if (!m_hasData || !m_currentData) return;

    // --- 构建等值面(Marching Cubes)管线 ---
    m_marchingCubes = vtkSmartPointer<vtkMarchingCubes>::New();

    if (m_imageData) {
        m_marchingCubes->SetInputData(m_imageData);
    } else {
        m_marchingCubes->SetInputData(m_currentData);
    }

    double isoValue = m_isoSpinBox->value();
    m_marchingCubes->SetValue(0, isoValue);
    m_marchingCubes->ComputeNormalsOn();
    m_marchingCubes->ComputeScalarsOn();
    m_marchingCubes->Update();

    // 映射器
    m_mainMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
    m_mainMapper->SetInputConnection(m_marchingCubes->GetOutputPort());
    m_mainMapper->ScalarVisibilityOn();
    m_mainMapper->SetScalarRange(m_dataRange);

    // 移除旧的 Actor (如果有)
    m_renderer->RemoveActor(m_mainActor);

    // 更新主 Actor
    m_mainActor = vtkSmartPointer<vtkActor>::New();
    m_mainActor->SetMapper(m_mainMapper);
    m_mainActor->GetProperty()->SetSpecular(0.3);
    m_mainActor->GetProperty()->SetSpecularPower(20);
    m_renderer->AddActor(m_mainActor);

    // --- 轮廓框 ---
    m_outlineFilter = vtkSmartPointer<vtkOutlineFilter>::New();
    m_outlineFilter->SetInputData(m_currentData);

    m_outlineMapper = vtkSmartPointer<vtkPolyDataMapper>::New();
    m_outlineMapper->SetInputConnection(m_outlineFilter->GetOutputPort());

    m_renderer->RemoveActor(m_outlineActor);
    m_outlineActor = vtkSmartPointer<vtkActor>::New();
    m_outlineActor->SetMapper(m_outlineMapper);
    m_outlineActor->GetProperty()->SetColor(0.8, 0.8, 0.8);
    m_outlineActor->GetProperty()->SetLineWidth(1.0);
    m_outlineActor->SetVisibility(m_outlineCheck->isChecked());
    m_renderer->AddActor(m_outlineActor);

    // --- 标量条 ---
    m_scalarBar->SetLookupTable(m_lookupTable);
    m_scalarBar->SetVisibility(true);

    ApplyColorMap(m_colorCombo->currentIndex());

    m_renderWindow->Render();
}

// ============================================================================
// 槽函数: 参数调整
// ============================================================================

void MainWindow::OnIsoValueChanged(double value) {
    if (!m_hasData || !m_marchingCubes) return;

    m_marchingCubes->SetValue(0, value);
    m_marchingCubes->Update();
    m_renderWindow->Render();

    statusBar()->showMessage(
        QString("等值面值: %1").arg(value, 0, 'f', 3), 2000);
}

void MainWindow::OnColorMapChanged(int index) {
    ApplyColorMap(index);
    if (m_hasData) {
        m_renderWindow->Render();
    }
}

void MainWindow::ApplyColorMap(int index) {
    if (!m_lookupTable) {
        m_lookupTable = vtkSmartPointer<vtkLookupTable>::New();
    }

    switch (index) {
        case 0: SetPresetRainbow();    break;
        case 1: SetPresetGrayscale();  break;
        case 2: SetPresetCoolWarm();   break;
        case 3: SetPresetBlueRed();    break;
    }

    if (m_mainMapper) {
        m_mainMapper->SetLookupTable(m_lookupTable);
    }
    if (m_scalarBar) {
        m_scalarBar->SetLookupTable(m_lookupTable);
    }
}

void MainWindow::SetPresetRainbow() {
    m_lookupTable->SetNumberOfTableValues(256);
    m_lookupTable->SetHueRange(0.0, 0.667);      // 红到蓝
    m_lookupTable->SetSaturationRange(1.0, 1.0);
    m_lookupTable->SetValueRange(1.0, 1.0);
    m_lookupTable->SetTableRange(m_dataRange);
    m_lookupTable->Build();
}

void MainWindow::SetPresetGrayscale() {
    m_lookupTable->SetNumberOfTableValues(256);
    m_lookupTable->SetHueRange(0.0, 0.0);
    m_lookupTable->SetSaturationRange(0.0, 0.0);
    m_lookupTable->SetValueRange(0.0, 1.0);
    m_lookupTable->SetTableRange(m_dataRange);
    m_lookupTable->Build();
}

void MainWindow::SetPresetCoolWarm() {
    m_lookupTable->SetNumberOfTableValues(256);
    m_lookupTable->SetHueRange(0.667, 0.0);       // 蓝到红（经青色）
    m_lookupTable->SetSaturationRange(0.8, 0.8);
    m_lookupTable->SetValueRange(0.8, 1.0);
    m_lookupTable->SetTableRange(m_dataRange);
    m_lookupTable->Build();
}

void MainWindow::SetPresetBlueRed() {
    m_lookupTable->SetNumberOfTableValues(256);
    m_lookupTable->SetHueRange(0.667, 0.0);       // 蓝到红
    m_lookupTable->SetSaturationRange(1.0, 1.0);
    m_lookupTable->SetValueRange(1.0, 1.0);
    m_lookupTable->SetTableRange(m_dataRange);
    m_lookupTable->Build();
}

void MainWindow::OnShowOutlineToggled(bool checked) {
    if (m_outlineActor) {
        m_outlineActor->SetVisibility(checked);
        m_renderWindow->Render();
    }
}

void MainWindow::OnShowWireframeToggled(bool checked) {
    if (m_mainActor) {
        if (checked) {
            m_mainActor->GetProperty()->SetRepresentationToWireframe();
        } else {
            m_mainActor->GetProperty()->SetRepresentationToSurface();
        }
        m_renderWindow->Render();
    }
}

void MainWindow::OnShowAxesToggled(bool checked) {
    if (m_axesActor) {
        m_axesActor->SetVisibility(checked);
        m_renderWindow->Render();
    }
}

void MainWindow::OnResetView() {
    if (m_renderer) {
        m_renderer->ResetCamera();
        m_renderWindow->Render();
        statusBar()->showMessage("视图已重置", 2000);
    }
}

// ============================================================================
// 截图保存
// ============================================================================

void MainWindow::OnSaveScreenshot() {
    QString fileName = QFileDialog::getSaveFileName(
        this, "保存截图", "screenshot.png",
        "PNG 图像 (*.png);;JPEG 图像 (*.jpg);;BMP 图像 (*.bmp)");

    if (fileName.isEmpty()) return;

    vtkNew<vtkWindowToImageFilter> w2if;
    w2if->SetInput(m_renderWindow);
    w2if->SetScale(1);
    w2if->SetInputBufferTypeToRGBA();
    w2if->ReadFrontBufferOff();
    w2if->Update();

    vtkNew<vtkPNGWriter> writer;
    writer->SetFileName(fileName.toStdString().c_str());
    writer->SetInputConnection(w2if->GetOutputPort());
    writer->Write();

    statusBar()->showMessage(
        QString("截图已保存: %1").arg(fileName), 3000);
}

// ============================================================================
// 数据信息更新
// ============================================================================

void MainWindow::UpdateDataInfoDisplay() {
    if (!m_currentData) return;

    vtkIdType numPoints = m_currentData->GetNumberOfPoints();
    vtkIdType numCells  = m_currentData->GetNumberOfCells();
    double bounds[6];
    m_currentData->GetBounds(bounds);

    m_pointsLabel->setText(
        QString("点数: %1").arg(numPoints));
    m_cellsLabel->setText(
        QString("单元数: %1").arg(numCells));
    m_boundsLabel->setText(
        QString("范围:\n  X: [%1, %2]\n  Y: [%3, %4]\n  Z: [%5, %6]")
            .arg(bounds[0], 0, 'f', 2).arg(bounds[1], 0, 'f', 2)
            .arg(bounds[2], 0, 'f', 2).arg(bounds[3], 0, 'f', 2)
            .arg(bounds[4], 0, 'f', 2).arg(bounds[5], 0, 'f', 2));
    m_scalarRangeLabel->setText(
        QString("标量范围: [%1, %2]")
            .arg(m_dataRange[0], 0, 'f', 3)
            .arg(m_dataRange[1], 0, 'f', 3));
}
```

### 21.5.3 main.cpp

```cpp
#include <QApplication>
#include <QSurfaceFormat>

#include "MainWindow.h"

/**
 * @brief 程序入口
 *
 * 启动 Qt 应用程序并显示主窗口。
 * 可以通过 QSurfaceFormat 设置全局的 OpenGL 参数。
 */
int main(int argc, char* argv[]) {
    // 设置全局 OpenGL Surface 格式 (可选)
    // 对于 VTK 渲染，通常建议使用默认格式
    QSurfaceFormat format = QVTKOpenGLNativeWidget::defaultFormat(true);
    QSurfaceFormat::setDefaultFormat(format);

    // 创建 Qt 应用
    QApplication app(argc, argv);
    app.setApplicationName("VTK Qt 可视化");
    app.setApplicationVersion("1.0.0");
    app.setOrganizationName("VTK Tutorial");

    // 创建并显示主窗口
    MainWindow window;
    window.show();

    return app.exec();
}
```

### 21.5.4 CMakeLists.txt

以下 CMake 配置文件同时支持 Qt5 和 Qt6，以及 VTK 9.x。根据系统中实际安装的版本，CMake 会自动选择合适的 Qt 版本。

```cmake
cmake_minimum_required(VERSION 3.20)
project(VTKQtApp VERSION 1.0.0 LANGUAGES CXX)

# ============================================================================
# C++ 标准
# ============================================================================
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ============================================================================
# 启用 Qt 的 AUTOMOC, AUTOUIC, AUTORCC
# ============================================================================
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)

# ============================================================================
# 查找 VTK (需要 GUISupportQt 模块)
# ============================================================================
find_package(VTK REQUIRED COMPONENTS
    # 核心模块
    CommonCore
    CommonDataModel
    CommonExecutionModel

    # 数据源与滤波器
    FiltersSources
    FiltersCore

    # 渲染
    RenderingCore
    RenderingOpenGL2

    # Qt 集成 (核心模块)
    GUISupportQt

    # IO
    IOImage
    IOLegacy
    IOXML

    # 交互
    InteractionStyle
)

# ============================================================================
# 查找 Qt (同时支持 Qt5 和 Qt6)
# ============================================================================
# 先尝试 Qt6，如果找不到则回退到 Qt5
find_package(Qt6 COMPONENTS Widgets OpenGLWidgets QUIET)
if(NOT Qt6_FOUND)
    find_package(Qt5 5.12 REQUIRED COMPONENTS Widgets)
    if(NOT Qt5_FOUND)
        message(FATAL_ERROR "需要 Qt5 (>= 5.12) 或 Qt6")
    endif()
    set(QT_VERSION_MAJOR 5)
else()
    set(QT_VERSION_MAJOR 6)
endif()

message(STATUS "使用 Qt 版本: ${QT_VERSION_MAJOR}")

# ============================================================================
# 源文件
# ============================================================================
set(SOURCES
    main.cpp
    MainWindow.cpp
)

set(HEADERS
    MainWindow.h
)

# ============================================================================
# 创建可执行目标
# ============================================================================
add_executable(${PROJECT_NAME}
    ${SOURCES}
    ${HEADERS}
)

# ============================================================================
# 链接库
# ============================================================================
target_link_libraries(${PROJECT_NAME} PRIVATE
    # VTK 库 (VTK:: 是 cmake 的命名空间)
    VTK::CommonCore
    VTK::CommonDataModel
    VTK::CommonExecutionModel
    VTK::FiltersSources
    VTK::FiltersCore
    VTK::RenderingCore
    VTK::RenderingOpenGL2
    VTK::GUISupportQt
    VTK::IOImage
    VTK::IOLegacy
    VTK::IOXML
    VTK::InteractionStyle
)

if(QT_VERSION_MAJOR EQUAL 6)
    target_link_libraries(${PROJECT_NAME} PRIVATE
        Qt6::Widgets
        Qt6::OpenGLWidgets
    )
else()
    target_link_libraries(${PROJECT_NAME} PRIVATE
        Qt5::Widgets
    )
endif()

# ============================================================================
# 编译器选项
# ============================================================================
if(MSVC)
    target_compile_options(${PROJECT_NAME} PRIVATE /W3 /utf-8)
else()
    target_compile_options(${PROJECT_NAME} PRIVATE -Wall -Wextra)
endif()

# ============================================================================
# 包含目录
# ============================================================================
target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${VTK_INCLUDE_DIRS}
)

# ============================================================================
# 安装规则 (可选)
# ============================================================================
install(TARGETS ${PROJECT_NAME}
    RUNTIME DESTINATION bin
)
```

### 21.5.5 构建与运行

构建此项目的典型步骤：

**Linux / macOS:**

```bash
mkdir build && cd build
cmake .. -DCMAKE_PREFIX_PATH=/path/to/Qt
cmake --build . --config Release
./VTKQtApp
```

**Windows (Visual Studio):**

```powershell
mkdir build; cd build
cmake .. -DCMAKE_PREFIX_PATH="C:\Qt\6.x.x\msvc2019_64"
cmake --build . --config Release
.\Release\VTKQtApp.exe
```

**常见构建注意事项:**

1. 确保 VTK 在编译时启用了 `VTK_MODULE_ENABLE_VTK_GUISupportQt` 选项（通常默认开启）。
2. Qt 和 VTK 必须使用相同的编译器编译（如 MSVC 2019、GCC 11+），否则会导致 ABI 不兼容。
3. 如果 VTK 和 Qt 使用不同的 OpenGL 后端，可能会出现渲染异常。推荐 VTK 使用 `RenderingOpenGL2` 模块。
4. 在 macOS 上运行时，可能需要设置环境变量 `QT_QPA_PLATFORM_PLUGIN_PATH`。

### 21.5.6 代码结构说明

上面的示例代码展示了一个功能完整的 Qt+VTK 应用，其设计遵循以下原则:

1. **关注点分离**: `MainWindow.h` 声明接口，`MainWindow.cpp` 实现逻辑，`main.cpp` 作为程序入口。

2. **信号槽驱动的参数更新**: 滑块 (`QSlider`) 的值变化通过信号传递给槽函数 `OnIsoValueChanged`，槽函数中直接修改 VTK 的 `vtkMarchingCubes` 滤波器参数并触发渲染。数值框 (`QDoubleSpinBox`) 与滑块双向同步，用户无论通过哪种方式输入都能即时得到反馈。

3. **模块化的颜色映射**: 四种颜色映射预设 (`SetPresetRainbow`, `SetPresetGrayscale`, `SetPresetCoolWarm`, `SetPresetBlueRed`) 被封装为独立函数，通过 `QComboBox` 的索引选择调用。`vtkLookupTable` 的 `HueRange`, `SaturationRange`, `ValueRange` 三元组决定了颜色在数据范围内的分布。

4. **智能文件加载**: `LoadDataFile` 函数通过文件扩展名自动选择合适的 VTK Reader。对于不支持的格式，程序会弹出警告对话框而非崩溃。

5. **UI 状态管理**: 在数据加载前，参数调节控件处于禁用状态 (`setEnabled(false)`)，防止用户进行无效操作。加载完成后才启用所有控件。

## 21.6 本章小结

本章系统地讲解了 VTK 9.5.2 与 Qt 框架集成的完整方案，从概念原理到可运行的完整代码示例。

**核心要点回顾:**

1. **两种集成方式**: 推荐使用现代的 `QVTKOpenGLNativeWidget`（基于 `QOpenGLWidget`），它是 VTK 9.x 的主要发展方向。传统的 `QVTKRenderWindowInteractor` 保留用于向后兼容。

2. **信号与槽桥接**: Qt 的信号槽机制是实现 UI 控件与 VTK 管线之间双向交互的核心。用户操作触发信号，槽函数中修改 VTK 参数并调用 `Render()` 实现即时视觉反馈。

3. **线程安全**: 所有 VTK 渲染操作必须在主线程（Qt GUI 线程）执行。文件 I/O 等耗时操作可以在后台线程完成，但结果必须传回主线程后再构建管线。

4. **完整应用架构**: 以 `QMainWindow` 为框架，`QVTKOpenGLNativeWidget` 占据中央视图区，`QDockWidget` 承载控制面板，`QMenuBar` 和 `QToolBar` 提供文件操作，`QStatusBar` 显示状态信息。这种布局是科学可视化应用程序的经典模式。

5. **CMake 构建**: 使用 `GUISupportQt` 模块，同时支持 Qt5 和 Qt6，`CMAKE_AUTOMOC` 自动处理 Qt 元对象编译。

**进一步学习方向:**

- 使用 `vtkEventQtSlotConnect` 直接从 VTK 事件连接到 Qt 槽函数，无需手动编写 Observer 回调。
- 将 VTK 的 `QVTKRenderWindowInteractor` 与 Qt 的多文档界面 (MDI) 结合，构建多视图可视化平台。
- 使用 Qt 的 Model/View 框架与 VTK 的数据集 (如 `vtkTable`) 整合，实现数据的表格化浏览与编辑。
- 探索 `ViewsQt` 模块中提供的 `vtkQtTableView` 等视图适配器，实现更深度集成。
- 将 QVTKOpenGLNativeWidget 与 Qt Quick (QML) 混合使用，构建更现代化的界面风格。

通过本章的学习，读者应该能够独立构建一个具备文件加载、三维渲染、实时参数调整和截图保存等功能的科学可视化桌面应用程序。这一模式是众多专业 VTK 应用（如 ParaView、3D Slicer、OpenFOAM 后处理工具）的架构基础。
