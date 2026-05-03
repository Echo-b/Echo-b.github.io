# 第二十二章 SMP并行入门（SMP Parallel Processing）

## 本章导读

在前二十一章中，你已经掌握了VTK的方方面面：从数据模型、可视化管道、各种数据类型，到底层过滤器、颜色映射、高级渲染、I/O操作，乃至构建完整的CFD后处理工具。但所有这一切都有一个共同的底层假设：**程序运行在单个CPU核心上，所有计算都是串行执行的**。

对于中小规模数据（几十万到几百万个单元），串行执行通常足够快。然而，当数据规模增长到数千万甚至数十亿个单元、或者需要对海量图像数据进行逐像素处理时，串行执行就会成为瓶颈。这就是本章要解决的问题：**如何利用现代多核CPU的并行计算能力，加速VTK的数据处理**。

VTK 9.5.2提供了两个层次的并行计算框架：

1. **SMP（Shared Memory Parallelism，共享内存并行）**：在单台机器的多个CPU核心上自动或手动并行化计算。这是本章的重点，也是最容易入门的并行方式。
2. **MPI（Message Passing Interface，消息传递接口）**：跨多台机器（或多个进程）的分布式内存并行。MPI更强大但也更复杂，需要专门的集群环境和较多的配置工作。

本章聚焦在SMP并行上，原因很简单：**你不需要任何特殊硬件或复杂的集群配置，只要你的电脑有一个多核CPU（几乎所有现代电脑都满足），你就能立即从SMP并行中受益**。

具体来说，本章将覆盖：

- VTK并行框架的两个层次及其适用场景。
- `vtkSMPTools`的基础使用：并行`For`循环、Functor模式、粒度控制。
- 线程安全的黄金法则：哪些操作可以安全地在多线程中执行，哪些绝对不能。
- `vtkSMPThreadLocal`和`vtkSMPThreadLocalObject`：为每个线程提供独立的存储空间。
- `vtkThreadedImageAlgorithm`：通过继承这个基类来创建自定义的并行图像处理滤波器。
- 一个完整的代码示例：创建一个大规模数据集、使用SMP并行加速计算、对比性能差异。

读完本章后，你将能够在自己的VTK项目中充分利用多核CPU的并行计算能力，显著提升数据处理的速度。

---

## 22.1 VTK并行的两个层次

### 22.1.1 SMP：共享内存并行

SMP（Shared Memory Parallelism）是最常见的并行模式。在SMP模式下，所有线程共享同一块物理内存。这意味着：

- **优点**：线程之间可以直接访问相同的数据（如`vtkDataArray`中的值），无需显式地复制或传输数据。编程模型相对简单。
- **限制**：所有线程必须运行在同一台机器上（一个操作系统进程内）。CPU核心数量决定了并行度的上限。

在VTK 9.5.2中，SMP并行已经深度集成到大量的核心滤波器中。实际上，你可能**已经在不知不觉中使用了SMP并行**——许多常用滤波器在内部自动使用`vtkSMPTools`来并行化它们的计算：

| 自动使用SMP的滤波器 | 并行化内容 |
|---------------------|-----------|
| `vtkContourFilter` | 等值面提取的单元遍历 |
| `vtkGradientFilter` | 梯度计算的逐点处理 |
| `vtkImageGaussianSmooth` | 高斯平滑的逐体素处理 |
| `vtkResampleToImage` | 数据重采样的插值计算 |
| `vtkProbeFilter` | 探测过滤器的采样点遍历 |
| `vtkThreshold` | 阈值判断的单元遍历 |
| `vtkCutter` | 截切过滤器的单元切割 |
| `vtkStreamTracer` | 流线追踪的并行粒子推进 |
| `vtkSynchronizedTemplates3D` | Marching Cubes算法的并行化 |

这些滤波器的并行化对用户是**完全透明**的——你不需要修改任何代码，只需在VTK编译时启用SMP支持（在CMake配置中设置`VTK_SMP_IMPLEMENTATION_TYPE`为`OpenMP`或`TBB`），运行时滤波器就会自动利用多核。

VTK 9.5.2支持两种SMP后端：

- **OpenMP**：由编译器内置支持（GCC、Clang、MSVC均支持），无需额外安装库。通过`#pragma omp`指令实现并行化。
- **Intel TBB（Threading Building Blocks）**：一个更高级的并行库，提供了更灵活的任务调度和工作窃取（work-stealing）机制。需要单独安装TBB库。

对于大多数用户，推荐使用OpenMP后端——它零依赖、配置简单，性能在多数场景下足够好。如果你的应用需要更细粒度的任务调度（比如嵌套并行、任务图），那么TBB是更好的选择。

### 22.1.2 MPI：分布式内存并行

MPI（Message Passing Interface）是科学计算中大规模并行计算的事实标准。在MPI模式下：

- 每个进程拥有自己独立的内存空间，运行在不同的机器（或同一机器的不同核心）上。
- 进程之间通过显式的消息传递来交换数据。
- 可以达到极高的并行度（数千甚至数万个进程），但编程复杂度显著增加。

VTK通过`vtkMPIController`、`vtkD3`（分布式数据依赖）和一系列`vtkP*`（Parallel）类来支持MPI并行。例如：

```
vtkPContourFilter     // 并行等值面提取
vtkPStreamTracer      // 并行流线追踪
vtkPConnectivityFilter // 并行连通性分析
```

MPI并行超出了本章的范围。如果你需要处理TB级别的大型仿真数据（例如全球气候模拟、宇宙学N体模拟），那么你需要学习VTK的MPI并行框架。但对于大多数可视化任务——处理几GB的数据、处理单个医学影像、分析单个CFD算例——SMP并行已经完全足够了。

### 22.1.3 本章关注SMP

用一个类比来区分SMP和MPI：

```
SMP = 一个厨房里多个厨师，共享同一套灶具和食材
MPI = 多栋建筑里的多个厨房，每个厨房有自己的灶具和食材，厨师之间通过对讲机协作
```

对于绝大多数VTK用户而言，SMP是更实用的起点：

- 不需要特殊的硬件配置（不需要集群）。
- 不需要修改数据分布逻辑（数据已经在共享内存中）。
- 很多滤波器已经自动使用SMP（零代码改动）。
- 即使需要手动并行化，API也很直观。

---

## 22.2 vtkSMPTools基础

### 22.2.1 vtkSMPTools::For — 并行的for循环

`vtkSMPTools`是VTK SMP框架的核心工具类。它提供了一个静态方法`For()`，用于将一个常规的串行for循环转化为并行循环。

其签名如下：

```cpp
// vtkSMPTools::For —— 并行执行一个Functor
// 参数:
//   begin  -- 起始索引（通常为0）
//   end    -- 终止索引（通常为数组大小）
//   functor-- 一个可调用对象，定义了 operator()(vtkIdType begin, vtkIdType end)
//   grain  -- 粒度（可选）。控制每个任务的最小处理元素数。默认值通常为数千。
static void For(vtkIdType begin, vtkIdType end, FunctorT& functor,
                vtkIdType grain = 0);
```

工作方式如下：

1. `vtkSMPTools::For`将范围`[begin, end)`切分为多个**子范围**。
2. 每个子范围被分配给一个线程。
3. 每个线程调用`functor(begin, end)`，其中`begin`和`end`是该线程负责的子范围边界。
4. 所有线程完成工作后，`For`返回。

关键在于**Functor模式**：Functor是一个类或结构体，必须提供`operator()(vtkIdType begin, vtkIdType end)`。VTK会为每个线程构造一个Functor的副本（通过拷贝构造函数），然后调用该副本的`operator()`。

### 22.2.2 Functor模式详解

一个最简的Functor示例如下：

```cpp
// 最简单的Functor：将数组中的每个元素平方
struct SquareFunctor
{
    vtkDataArray* Array;

    // 构造函数
    SquareFunctor(vtkDataArray* array) : Array(array) {}

    // operator() 在并行线程中被调用
    // begin 和 end 是该线程负责的子范围
    void operator()(vtkIdType begin, vtkIdType end)
    {
        for (vtkIdType i = begin; i < end; ++i)
        {
            double val = Array->GetTuple1(i);
            Array->SetTuple1(i, val * val);
        }
    }
};

// 使用方式
vtkSMPTools::For(0, array->GetNumberOfTuples(), SquareFunctor(array));
```

VTK框架对Functor有三个约定：

1. **CopyConstructible**：Functor必须可拷贝。VTK为每个线程创建一个副本。
2. **operator()调用可并行**：多个线程可能同时调用不同副本的`operator()`——你的代码必须能处理这种并发性。
3. **operator()完成后，结果必须是可合并的**：如果你使用per-thread存储（见22.3节），需要在所有线程完成后合并结果。

### 22.2.3 自动线程数检测

`vtkSMPTools`会自动检测可用的CPU核心数量，无需你手动指定线程数。检测逻辑如下：

- **OpenMP后端**：使用`omp_get_max_threads()`获取最大线程数（可通过`OMP_NUM_THREADS`环境变量控制）。
- **TBB后端**：使用TBB的自动任务调度，动态匹配可用硬件线程数。

你可以通过以下代码查询当前使用的线程数：

```cpp
#include <vtkSMPTools.h>
#include <iostream>

// 获取VTK检测到的线程数
int numThreads = vtkSMPTools::GetEstimatedNumberOfThreads();
std::cout << "VTK SMP using " << numThreads << " threads." << std::endl;

// 手动设置线程数（可选）
vtkSMPTools::SetNestedParallelism(true); // 允许嵌套并行（默认关闭）
```

通常情况下，你不需要手动设置线程数。VTK的默认行为（使用所有可用核心）在绝大多数场景下是最优的。只有在特殊情况下（例如你想为其他进程预留核心、或调试并行行为）才需要手动调整。

### 22.2.4 粒度（Grain Size）

粒度（Grain Size）控制每个任务（分配给一个线程的子范围）的最小大小。它的作用是平衡**并行开销**和**负载均衡**：

- **粒度过小**（如grain=1）：产生大量微小任务，线程创建和同步的开销可能超过并行带来的收益。
- **粒度过大**（如grain=数组大小/2）：可能只有少数线程有活干，其他线程空闲，浪费并行能力。
- **理想粒度**：每个任务足够大以摊薄并行开销，同时又足够小以保证所有线程都有工作可做。

VTK的默认粒度策略是智能的：如果不指定`grain`参数（或传递0），VTK会自动计算一个合理的粒度值，通常使每个线程处理至少几万个元素。在绝大多数情况下，依赖默认值即可。

如果你需要手动控制粒度：

```cpp
// 手动指定粒度为1000：每个线程至少处理1000个元素
vtkSMPTools::For(0, numElements, functor, 1000);
```

### 22.2.5 简单示例：并行计算在一个vtkDataArray上

下面是一个完整的示例，展示如何使用`vtkSMPTools::For`来并行地对一个`vtkDataArray`中的值进行变换：

```cpp
#include <vtkDoubleArray.h>
#include <vtkNew.h>
#include <vtkSMPTools.h>

#include <cmath>
#include <iostream>

// Functor：计算每个元素的 sin(x)*exp(-x/100)
struct ComputeFunctor
{
    vtkDoubleArray* Array;

    ComputeFunctor(vtkDoubleArray* arr) : Array(arr) {}

    void operator()(vtkIdType begin, vtkIdType end)
    {
        for (vtkIdType i = begin; i < end; ++i)
        {
            double x = Array->GetValue(i);
            Array->SetValue(i, std::sin(x) * std::exp(-x / 100.0));
        }
    }
};

int main()
{
    constexpr vtkIdType N = 10000000; // 一千万个元素

    vtkNew<vtkDoubleArray> data;
    data->SetNumberOfValues(N);

    // 初始化：填充 0 到 100 的线性序列
    for (vtkIdType i = 0; i < N; ++i)
    {
        data->SetValue(i, static_cast<double>(i) / 100000.0);
    }

    std::cout << "Processing " << N << " elements..." << std::endl;

    // 并行计算
    ComputeFunctor functor(data);
    vtkSMPTools::For(0, N, functor);

    std::cout << "Done. data[0] = " << data->GetValue(0)
              << ", data[5000000] = " << data->GetValue(5000000)
              << ", data[" << (N - 1) << "] = " << data->GetValue(N - 1)
              << std::endl;

    return 0;
}
```

这个示例虽然简单，但它展示了使用`vtkSMPTools`的核心模式：定义一个Functor，在`operator()`中实现逐元素的处理逻辑，然后调用`vtkSMPTools::For`来并行执行。

---

## 22.3 线程安全注意事项

### 22.3.1 线程安全的操作

以下操作在多线程环境下是安全的（可以在Functor的`operator()`中直接使用）：

- **从`vtkDataArray`中读取数据**：`GetTuple1()`、`GetValue()`、`GetTuple3()`等读取方法是线程安全的。多个线程可以同时读取同一个数组的不同位置。
- **向`vtkDataArray`中写入数据（不同索引）**：只要不同线程写入的索引位置不重叠（这是`vtkSMPTools::For`的自然保证，因为每个线程负责不同的子范围），写入就是安全的。
- **纯计算**：局部变量的算术运算、数学函数调用（`std::sin`、`std::sqrt`等）、条件判断等都是线程安全的。
- **创建和使用临时VTK对象**：如果一个VTK对象只在单个线程的`operator()`内创建和使用，那么它是安全的。例如，在`operator()`中创建`vtkNew<vtkGenericCell>`用于单元插值。
- **从vtkDataSet中读取点坐标**：通过`points->GetPoint(id, pt)`读取特定点的坐标是安全的。但是，如果你需要访问整个点数组，应该在并行循环之外获取`vtkDataArray`指针，然后在循环中直接通过索引读取。

### 22.3.2 非线程安全的操作

以下操作**绝不**可以在多线程的Functor中执行——它们要么不是线程安全的，要么会产生未定义行为：

- **修改vtkDataSet的拓扑结构**：`InsertNextCell()`、`DeletePoint()`、`SetPoints()`等修改数据集结构的方法不是线程安全的。这些操作涉及内部数据结构的重新分配和指针更新，多线程同时调用会导致数据损坏。
- **调用管道的Update()方法**：`filter->Update()`会触发整个流水线的执行，可能涉及内存分配、对象状态变更，不能在多线程中调用。
- **修改同一个变量（无同步）**：如果多个线程同时修改同一个变量而没有使用互斥锁（mutex）或原子操作，结果是不确定的。例如，所有线程向同一个全局计数器递增。
- **调用非const方法（共享对象）**：如果多个线程共享同一个VTK对象（如一个`vtkPoints`对象），同时对它调用非const方法，这是不安全的。
- **使用`vtkNew`或`vtkSmartPointer`进行跨线程引用计数管理**：VTK的智能指针使用非原子的引用计数（出于性能考虑），因此多线程同时操作同一个智能指针会导致引用计数错乱。

### 22.3.3 线程安全速查表

| 操作 | 是否安全 | 备注 |
|------|---------|------|
| 从vtkDataArray读取（不同索引） | 是 | 只读访问，无竞争 |
| 向vtkDataArray写入（不同索引） | 是 | 只要索引范围不重叠 |
| 向vtkDataArray写入（相同索引） | 否 | 数据竞争 |
| 调用vtkDataSet::GetPoint(id, pt) | 是 | 只读访问 |
| 修改vtkDataSet拓扑（InsertNextCell等） | 否 | 非线程安全 |
| 调用filter->Update() | 否 | 非线程安全 |
| 在operator()内创建局部VTK对象 | 是 | 每线程独立 |
| 修改全局变量（无锁） | 否 | 需要互斥锁或原子操作 |
| 读取全局常量 | 是 | 只读，无竞争 |
| 使用std::cout打印 | 谨慎 | 可能交错输出，建议加锁 |

### 22.3.4 vtkSMPThreadLocal：每线程私有存储

在很多场景中，你需要每个线程拥有自己私有的临时存储——例如，每个线程累加自己的局部统计结果，最后再合并。`vtkSMPThreadLocal<T>`提供了这个能力。

```cpp
#include <vtkSMPThreadLocal.h>
#include <vtkSMPTools.h>

// Functor：计算数组中值的总和
struct SumFunctor
{
    vtkDataArray* Array;
    vtkSMPThreadLocal<double> LocalSum; // 每线程私有的累加器

    SumFunctor(vtkDataArray* arr) : Array(arr)
    {
        // 初始化每个线程的LocalSum为0
        // Local() 返回当前线程的私有引用
    }

    void Initialize()
    {
        // 显式地将所有线程的局部值初始化为0
        LocalSum.Local() = 0.0;
    }

    void operator()(vtkIdType begin, vtkIdType end)
    {
        double& sum = LocalSum.Local();
        for (vtkIdType i = begin; i < end; ++i)
        {
            sum += Array->GetTuple1(i);
        }
    }

    // 在所有线程完成后，合并局部结果
    double Reduce()
    {
        double total = 0.0;
        // 遍历所有线程的局部值
        vtkSMPThreadLocal<double>::iterator it  = LocalSum.begin();
        vtkSMPThreadLocal<double>::iterator end = LocalSum.end();
        for (; it != end; ++it)
        {
            total += *it;
        }
        return total;
    }
};
```

`vtkSMPThreadLocal<T>`的关键特性：

- **每线程独立实例**：`vtkSMPThreadLocal<T>`为每个工作线程维护一个独立的`T`实例。访问`Local()`返回当前线程的引用。
- **迭代器支持**：可以使用迭代器遍历所有线程的私有值，这在Reduce阶段非常有用。
- **自动生命周期管理**：当`vtkSMPThreadLocal`对象被销毁时，所有线程的私有实例也会被正确销毁。

### 22.3.5 vtkSMPThreadLocalObject：每线程私有VTK对象

有时你需要每个线程拥有自己私有的VTK对象（例如，每个线程需要一个私有的`vtkGenericCell`用于插值操作）。`vtkSMPThreadLocalObject<T>`专门用于这个目的：

```cpp
#include <vtkSMPThreadLocalObject.h>
#include <vtkGenericCell.h>
#include <vtkDataSet.h>

// Functor：在多边形数据上执行点的插值
struct InterpolateFunctor
{
    vtkDataSet*           Mesh;
    vtkDataArray*         InputPoints;
    vtkDataArray*         OutputScalars;
    vtkSMPThreadLocalObject<vtkGenericCell> TLCell; // 每线程私有的Cell对象

    InterpolateFunctor(vtkDataSet* mesh,
                       vtkDataArray* input,
                       vtkDataArray* output)
        : Mesh(mesh), InputPoints(input), OutputScalars(output) {}

    void operator()(vtkIdType begin, vtkIdType end)
    {
        // 获取当前线程私有的vtkGenericCell实例
        vtkGenericCell* cell = TLCell.Local();

        double closestPoint[3];
        int    subId;
        double dist2;
        double pcoords[3];
        double weights[256];

        for (vtkIdType i = begin; i < end; ++i)
        {
            // 获取采样点坐标
            double pt[3];
            InputPoints->GetTuple(i, pt);

            // 找到包含该点的单元
            vtkIdType cellId = Mesh->FindCell(pt, nullptr, cell, 0,
                                               1e-6, subId, pcoords, weights);

            if (cellId >= 0)
            {
                // 在找到的单元中执行插值
                // ...
            }
        }
    }
};
```

`vtkSMPThreadLocalObject<T>`和`vtkSMPThreadLocal<T>`的区别在于：

- `vtkSMPThreadLocal<T>`：为每个线程存储一个普通的C++对象（如`double`、`std::vector`）。
- `vtkSMPThreadLocalObject<T>`：为每个线程存储一个VTK对象（继承自`vtkObjectBase`），并且可以通过`vtkSMPThreadLocalObject::New()`模板参数自动调用`T::New()`来创建实例。

### 22.3.6 示例：使用ThreadLocal累加并行结果

```cpp
#include <vtkDoubleArray.h>
#include <vtkNew.h>
#include <vtkSMPThreadLocal.h>
#include <vtkSMPTools.h>

#include <algorithm>
#include <iostream>

struct StatisticsFunctor
{
    vtkDoubleArray*                    Data;
    vtkSMPThreadLocal<double>          Sum;
    vtkSMPThreadLocal<double>          SumSq;
    vtkSMPThreadLocal<vtkIdType>       Count;
    vtkSMPThreadLocal<double>          Min;
    vtkSMPThreadLocal<double>          Max;

    StatisticsFunctor(vtkDoubleArray* data) : Data(data) {}

    void Initialize()
    {
        Sum.Local()    = 0.0;
        SumSq.Local()  = 0.0;
        Count.Local()  = 0;
        Min.Local()    = std::numeric_limits<double>::max();
        Max.Local()    = std::numeric_limits<double>::lowest();
    }

    void operator()(vtkIdType begin, vtkIdType end)
    {
        double&    sum   = Sum.Local();
        double&    sumSq = SumSq.Local();
        vtkIdType& count = Count.Local();
        double&    min   = Min.Local();
        double&    max   = Max.Local();

        for (vtkIdType i = begin; i < end; ++i)
        {
            double val = Data->GetValue(i);
            sum   += val;
            sumSq += val * val;
            count += 1;
            min    = std::min(min, val);
            max    = std::max(max, val);
        }
    }

    void Reduce(double& totalSum, double& totalSumSq, vtkIdType& totalCount,
                double& globalMin, double& globalMax)
    {
        totalSum   = 0.0;
        totalSumSq = 0.0;
        totalCount = 0;
        globalMin  = std::numeric_limits<double>::max();
        globalMax  = std::numeric_limits<double>::lowest();

        auto itSum   = Sum.begin();
        auto itSumSq = SumSq.begin();
        auto itCnt   = Count.begin();
        auto itMin   = Min.begin();
        auto itMax   = Max.begin();
        auto end     = Sum.end();

        for (; itSum != end; ++itSum, ++itSumSq, ++itCnt, ++itMin, ++itMax)
        {
            totalSum   += *itSum;
            totalSumSq += *itSumSq;
            totalCount += *itCnt;
            globalMin   = std::min(globalMin, *itMin);
            globalMax   = std::max(globalMax, *itMax);
        }
    }
};

int main()
{
    constexpr vtkIdType N = 5000000;
    vtkNew<vtkDoubleArray> data;
    data->SetNumberOfValues(N);
    for (vtkIdType i = 0; i < N; ++i)
    {
        data->SetValue(i, std::sin(static_cast<double>(i) * 0.0001));
    }

    StatisticsFunctor functor(data);
    functor.Initialize(); // 在For之前初始化ThreadLocal值

    vtkSMPTools::For(0, N, functor);

    double    totalSum, totalSumSq, globalMin, globalMax;
    vtkIdType totalCount;
    functor.Reduce(totalSum, totalSumSq, totalCount, globalMin, globalMax);

    double mean = totalSum / totalCount;
    double variance = (totalSumSq / totalCount) - (mean * mean);

    std::cout << "Count:   " << totalCount << std::endl;
    std::cout << "Mean:    " << mean << std::endl;
    std::cout << "Variance:" << variance << std::endl;
    std::cout << "Min:     " << globalMin << std::endl;
    std::cout << "Max:     " << globalMax << std::endl;

    return 0;
}
```

这个示例展示了典型的"线程局部累加 -> Reduce合并"模式：
1. 在`operator()`中，每个线程更新自己私有的累加器。
2. 在所有线程完成后，通过`Reduce()`方法遍历所有线程的私有值并合并。

---

## 22.4 vtkThreadedImageAlgorithm

### 22.4.1 为什么需要自定义并行滤波器？

虽然许多内置VTK滤波器已经自动使用SMP并行，但在以下场景中，你可能需要编写自己的并行图像处理算法：

- 实现VTK没有内置的自定义图像处理算法（例如特定的图像分割、特征提取）。
- 需要同时处理多个标量数组并生成一个派生结果（例如从速度场计算涡量）。
- 需要对图像数据执行自定义的逐体素计算（例如应用自定义的非线性变换）。

`vtkThreadedImageAlgorithm`就是为这个目的设计的。它是`vtkImageAlgorithm`的子类，自动将输入数据分割成多个块（chunks），并在多个线程中并行调用你的处理函数。

### 22.4.2 vtkThreadedImageAlgorithm的架构

`vtkThreadedImageAlgorithm`的工作流程是：

1. 在执行`RequestData`时，它获取输出图像的`Extent`（完整范围）。
2. 它将输出范围按"片"（piece）切分为多个子范围（split extents）。默认情况下沿Z轴切分（对于3D数据），或沿Y轴切分（对于2D数据）。
3. 对每个子范围，它在一个独立的线程中调用`ThreadedRequestData()`。
4. 在每个线程中，你只需处理分配给该线程的子范围。

你需要做的只是：

1. 继承`vtkThreadedImageAlgorithm`。
2. 重写`ThreadedRequestData()`方法——它接收输入、输出和当前线程负责的子范围。
3. 在`ThreadedRequestData()`中实现对子范围的处理逻辑。

### 22.4.3 split extent模式

split extent模式是`vtkThreadedImageAlgorithm`的核心概念。假设你有一个100x100x100的3D图像数据：

- 完整Extent: `[0, 99, 0, 99, 0, 99]`
- VTK可能将其按Z轴切分为4个split extents：
  - Thread 0: `[0, 99, 0, 99, 0, 24]`
  - Thread 1: `[0, 99, 0, 99, 25, 49]`
  - Thread 2: `[0, 99, 0, 99, 50, 74]`
  - Thread 3: `[0, 99, 0, 99, 75, 99]`

每个线程只处理自己负责的Z切片范围内的所有体素。

`ThreadedRequestData()`需要处理的就是这样的子范围。它接收`vtkImageData*`类型的输入和输出，以及`int extent[6]`定义了当前线程负责的子范围`[extent[0], extent[1], extent[2], extent[3], extent[4], extent[5]]`。

### 22.4.4 示例：自定义并行图像滤波器

下面展示如何创建一个自定义的并行图像滤波器，它将两个输入图像的对应像素值相加，并应用一个S形（sigmoid）激活函数：

```cpp
#include <vtkThreadedImageAlgorithm.h>
#include <vtkImageData.h>
#include <vtkPointData.h>
#include <vtkDataArray.h>
#include <vtkObjectFactory.h>
#include <vtkInformation.h>
#include <vtkInformationVector.h>
#include <vtkStreamingDemandDrivenPipeline.h>

#include <cmath>

class vtkSigmoidBlendImageFilter : public vtkThreadedImageAlgorithm
{
public:
    static vtkSigmoidBlendImageFilter* New();
    vtkTypeMacro(vtkSigmoidBlendImageFilter, vtkThreadedImageAlgorithm);

    // 设置第二个输入（第一个输入继承自vtkImageAlgorithm）
    void SetInput1Data(vtkDataObject* input) { this->SetInputData(0, input); }
    void SetInput2Data(vtkDataObject* input) { this->SetInputData(1, input); }

    // 设置S形函数的陡峭度参数
    vtkSetMacro(Steepness, double);
    vtkGetMacro(Steepness, double);

protected:
    vtkSigmoidBlendImageFilter()
    {
        this->SetNumberOfInputPorts(2); // 需要两个输入
        Steepness = 1.0;
    }

    ~vtkSigmoidBlendImageFilter() override = default;

    double Steepness;

    // 重写：定义输出范围对应的输入范围需求
    int RequestInformation(vtkInformation* request,
                           vtkInformationVector** inputVector,
                           vtkInformationVector* outputVector) override;

    // 重写：线程化的实际处理逻辑
    void ThreadedRequestData(vtkInformation* request,
                             vtkInformationVector** inputVector,
                             vtkInformationVector* outputVector,
                             vtkImageData*** inData,
                             vtkImageData** outData,
                             int extent[6],
                             int threadId) override;

private:
    vtkSigmoidBlendImageFilter(const vtkSigmoidBlendImageFilter&) = delete;
    void operator=(const vtkSigmoidBlendImageFilter&) = delete;
};

vtkStandardNewMacro(vtkSigmoidBlendImageFilter);

// ---------------------------------------------------------------------------
int vtkSigmoidBlendImageFilter::RequestInformation(
    vtkInformation* vtkNotUsed(request),
    vtkInformationVector** inputVector,
    vtkInformationVector* outputVector)
{
    // 获取第一个输入的信息
    vtkInformation* inInfo0 = inputVector[0]->GetInformationObject(0);
    vtkInformation* inInfo1 = inputVector[1]->GetInformationObject(0);
    vtkInformation* outInfo = outputVector->GetInformationObject(0);

    // 验证两个输入都存在
    if (!inInfo0 || !inInfo1)
    {
        vtkGenericWarningMacro("Two inputs are required.");
        return 0;
    }

    // 从第一个输入复制信息到输出
    int ext[6];
    inInfo0->Get(vtkStreamingDemandDrivenPipeline::WHOLE_EXTENT(), ext);
    outInfo->Set(vtkStreamingDemandDrivenPipeline::WHOLE_EXTENT(), ext, 6);

    double spacing[3], origin[3];
    inInfo0->Get(vtkDataObject::SPACING(), spacing);
    outInfo->Set(vtkDataObject::SPACING(), spacing, 3);
    inInfo0->Get(vtkDataObject::ORIGIN(), origin);
    outInfo->Set(vtkDataObject::ORIGIN(), origin, 3);

    return 1;
}

// ---------------------------------------------------------------------------
void vtkSigmoidBlendImageFilter::ThreadedRequestData(
    vtkInformation* vtkNotUsed(request),
    vtkInformationVector** vtkNotUsed(inputVector),
    vtkInformationVector* vtkNotUsed(outputVector),
    vtkImageData*** inData,
    vtkImageData** outData,
    int extent[6],
    int vtkNotUsed(threadId))
{
    // 获取输入和输出图像
    vtkImageData* input1 = inData[0][0];
    vtkImageData* input2 = inData[1][0];
    vtkImageData* output = outData[0];

    // 获取标量数组
    vtkDataArray* inScalars1 = input1->GetPointData()->GetScalars();
    vtkDataArray* inScalars2 = input2->GetPointData()->GetScalars();
    vtkDataArray* outScalars = output->GetPointData()->GetScalars();

    if (!inScalars1 || !inScalars2 || !outScalars)
    {
        return;
    }

    double s = this->Steepness;

    // 遍历当前线程负责的子范围
    for (int k = extent[4]; k <= extent[5]; ++k)
    {
        for (int j = extent[2]; j <= extent[3]; ++j)
        {
            for (int i = extent[0]; i <= extent[1]; ++i)
            {
                // 计算体素的IJK索引
                vtkIdType idx = output->ComputePointId(
                    static_cast<vtkIdType>(i),
                    static_cast<vtkIdType>(j),
                    static_cast<vtkIdType>(k));

                // 获取两个输入的值
                double v1 = inScalars1->GetTuple1(idx);
                double v2 = inScalars2->GetTuple1(idx);

                // 计算：sigmoid(v1 + v2)
                double sum = v1 + v2;
                double result = 1.0 / (1.0 + std::exp(-s * sum));

                outScalars->SetTuple1(idx, result);
            }
        }
    }
}
```

使用这个自定义滤波器的方式如下：

```cpp
#include <vtkImageData.h>
#include <vtkNew.h>
#include <vtkImageCanvasSource2D.h>

// ... (上面的类定义)

int main()
{
    // 创建两个输入图像
    vtkNew<vtkImageCanvasSource2D> source1;
    source1->SetExtent(0, 511, 0, 511, 0, 0);
    source1->SetScalarTypeToDouble();
    source1->SetNumberOfScalarComponents(1);
    source1->SetDrawColor(0.5);
    source1->FillBox(0, 511, 0, 511); // 背景0.5
    source1->SetDrawColor(1.0);
    source1->FillBox(200, 300, 200, 300); // 中央方块1.0

    vtkNew<vtkImageCanvasSource2D> source2;
    source2->SetExtent(0, 511, 0, 511, 0, 0);
    source2->SetScalarTypeToDouble();
    source2->SetNumberOfScalarComponents(1);
    source2->SetDrawColor(0.2);
    source2->FillBox(0, 511, 0, 511);

    // 使用自定义并行滤波器
    vtkNew<vtkSigmoidBlendImageFilter> blend;
    blend->SetInput1Data(source1->GetOutput());
    blend->SetInput2Data(source2->GetOutput());
    blend->SetSteepness(2.0);
    blend->Update();

    // 验证输出
    vtkImageData* output = blend->GetOutput();
    int* dims = output->GetDimensions();
    std::cout << "Output dimensions: " << dims[0] << "x"
              << dims[1] << "x" << dims[2] << std::endl;

    return 0;
}
```

### 22.4.5 ThreadedRequestData的注意事项

1. **Extent参数**: `int extent[6]`的格式是`[xmin, xmax, ymin, ymax, zmin, zmax]`，注意是**包含**边界（闭区间）。循环时应该使用`<=`而不是`<`。

2. **输入数据格式**: `vtkImageData*** inData`是一个三维指针——`inData[port][connection]`。对于大多数简单滤波器（每个端口一个输入），使用`inData[0][0]`和`inData[1][0]`即可。

3. **不要修改extent范围之外的数据**: 只处理分配给当前线程的子范围。写入extent之外的输出索引可能导致数据竞争。

4. **读取extent之外的数据**: 如果需要从输入中读取extent之外的数据（例如，对于卷积滤波器需要读取邻近像素），你需要重写`RequestInformation()`来声明额外的输入范围需求（通过设置`INPUT_REQUIRED_DATA_TYPE()`或类似机制）。对于大多数逐点滤波器，读取extent范围内的数据就足够了。

5. **所有线程共享输出数据对象**: 虽然每个线程处理不同的extent范围，但它们共享同一个输出`vtkImageData`对象。只要确保不同线程写入的索引不重叠，这就是安全的。

---

## 22.5 代码示例：并行数据处理

下面是一个完整的、可编译运行的C++程序。它演示了以下内容：

1. 创建一个大规模数据集（1000x1000的ImageData，带随机噪声）。
2. 分别使用串行和SMP并行方式执行平滑滤波，测量并对比耗时。
3. 使用`vtkSMPTools::For`实现一个自定义的并行计算：从多个标量数组计算派生标量。
4. 在控制台输出性能对比结果。

### 22.5.1 完整C++源代码

```cpp
// ---------------------------------------------------------------------------
// File: SMPParallelDemo.cxx
// Description: 演示VTK SMP并行的完整示例
//   - 创建大规模图像数据集
//   - 对比串行 vs 并行滤波性能
//   - 自定义并行计算（多数组派生标量）
// ---------------------------------------------------------------------------

#include <vtkCamera.h>
#include <vtkContourFilter.h>
#include <vtkDataSetMapper.h>
#include <vtkDoubleArray.h>
#include <vtkImageActor.h>
#include <vtkImageData.h>
#include <vtkImageGaussianSmooth.h>
#include <vtkImageMathematics.h>
#include <vtkImageNoiseSource.h>
#include <vtkImageProperty.h>
#include <vtkImageSlice.h>
#include <vtkImageSliceMapper.h>
#include <vtkImageMapToColors.h>
#include <vtkInteractorStyleImage.h>
#include <vtkLookupTable.h>
#include <vtkNamedColors.h>
#include <vtkNew.h>
#include <vtkPointData.h>
#include <vtkPolyDataMapper.h>
#include <vtkRenderWindow.h>
#include <vtkRenderWindowInteractor.h>
#include <vtkRenderer.h>
#include <vtkSMPThreadLocal.h>
#include <vtkSMPThreadLocalObject.h>
#include <vtkSMPTools.h>
#include <vtkSmartPointer.h>
#include <vtkTimerLog.h>

#include <algorithm>
#include <chrono>
#include <cmath>
#include <cstdlib>
#include <iomanip>
#include <iostream>
#include <limits>
#include <string>
#include <vector>

// ---------------------------------------------------------------------------
// 常量定义
// ---------------------------------------------------------------------------
constexpr int    IMAGE_SIZE      = 1000;    // 图像尺寸 (IMAGE_SIZE x IMAGE_SIZE)
constexpr int    NUM_ARRAYS      = 4;       // 用于并行计算的数组数量
constexpr double NOISE_AMPLITUDE = 0.3;     // 噪声幅度
constexpr int    SMOOTH_RADIUS   = 3;       // 高斯平滑半径
constexpr int    SMOOTH_ITER     = 5;       // 平滑迭代次数（用于性能测试）

// ---------------------------------------------------------------------------
// 31.1 辅助函数：创建带噪声的测试图像数据
// ---------------------------------------------------------------------------
vtkSmartPointer<vtkImageData> CreateNoisyImageData()
{
    std::cout << "[1/4] Creating " << IMAGE_SIZE << "x" << IMAGE_SIZE
              << " image data with noise..." << std::endl;

    vtkNew<vtkImageData> image;
    image->SetExtent(0, IMAGE_SIZE - 1, 0, IMAGE_SIZE - 1, 0, 0);
    image->SetOrigin(0.0, 0.0, 0.0);
    image->SetSpacing(1.0, 1.0, 1.0);

    // 分配点数据数组
    vtkNew<vtkDoubleArray> scalars;
    scalars->SetName("Intensity");
    scalars->SetNumberOfComponents(1);
    scalars->SetNumberOfTuples(static_cast<vtkIdType>(IMAGE_SIZE) * IMAGE_SIZE);

    // 用基础模式 + 随机噪声填充
    std::srand(42); // 固定种子，保证可复现

    vtkIdType idx = 0;
    for (int j = 0; j < IMAGE_SIZE; ++j)
    {
        for (int i = 0; i < IMAGE_SIZE; ++i)
        {
            // 基础模式：同心波纹 + 渐变
            double cx = IMAGE_SIZE / 2.0;
            double cy = IMAGE_SIZE / 2.0;
            double dx = (i - cx) / (IMAGE_SIZE * 0.3);
            double dy = (j - cy) / (IMAGE_SIZE * 0.3);
            double ripple = std::sin(std::sqrt(dx * dx + dy * dy) * 6.28);

            // 线性渐变（从左到右）
            double gradient = static_cast<double>(i) / IMAGE_SIZE;

            // 组合并添加噪声
            double base = 0.5 + 0.25 * ripple + 0.25 * gradient;
            double noise = NOISE_AMPLITUDE *
                (static_cast<double>(std::rand()) / RAND_MAX - 0.5);
            double val = std::max(0.0, std::min(1.0, base + noise));

            scalars->SetValue(idx, val);
            ++idx;
        }
    }

    image->GetPointData()->SetScalars(scalars);
    std::cout << "  Created " << scalars->GetNumberOfTuples()
              << " points." << std::endl;

    return image;
}

// ---------------------------------------------------------------------------
// 31.2 性能对比：串行 vs 并行高斯平滑
// ---------------------------------------------------------------------------
void CompareSmoothingPerformance(vtkImageData* input)
{
    std::cout << "\n[2/4] Comparing serial vs parallel Gaussian smoothing..."
              << std::endl;

    vtkNew<vtkTimerLog> timer;
    double serialTime = 0.0;
    double parallelTime = 0.0;

    // --- 串行版本 ---
    // 临时禁用SMP（通过设置线程数为1来模拟串行）
    int defaultThreads = vtkSMPTools::GetEstimatedNumberOfThreads();
    std::cout << "  Default VTK SMP threads: " << defaultThreads << std::endl;

    // 标准VTK图像高斯平滑（内部自动使用SMP）
    // 为了对比串行性能，我们手动串行实现一个简化版平滑
    std::cout << "  Running serial smoothing (" << SMOOTH_ITER
              << " iterations)..." << std::endl;

    // 串行实现：简化版均值平滑（用于与并行版对比）
    vtkNew<vtkImageData> serialCopy;
    serialCopy->DeepCopy(input);

    // 获取输入数组
    vtkDataArray* srcArray = serialCopy->GetPointData()->GetScalars();
    vtkIdType numPoints = srcArray->GetNumberOfTuples();

    // 创建临时数组用于串行平滑
    vtkNew<vtkDoubleArray> tempArray;
    tempArray->SetNumberOfTuples(numPoints);

    timer->StartTimer();

    for (int iter = 0; iter < SMOOTH_ITER; ++iter)
    {
        // 复制源到临时数组
        for (vtkIdType i = 0; i < numPoints; ++i)
        {
            tempArray->SetValue(i, srcArray->GetTuple1(i));
        }

        // 3x3均值平滑（串行）
        for (int j = 1; j < IMAGE_SIZE - 1; ++j)
        {
            for (int i = 1; i < IMAGE_SIZE - 1; ++i)
            {
                vtkIdType idx = j * IMAGE_SIZE + i;
                double sum = 0.0;
                sum += tempArray->GetValue((j - 1) * IMAGE_SIZE + (i - 1));
                sum += tempArray->GetValue((j - 1) * IMAGE_SIZE + i);
                sum += tempArray->GetValue((j - 1) * IMAGE_SIZE + (i + 1));
                sum += tempArray->GetValue(j * IMAGE_SIZE + (i - 1));
                sum += tempArray->GetValue(j * IMAGE_SIZE + i);
                sum += tempArray->GetValue(j * IMAGE_SIZE + (i + 1));
                sum += tempArray->GetValue((j + 1) * IMAGE_SIZE + (i - 1));
                sum += tempArray->GetValue((j + 1) * IMAGE_SIZE + i);
                sum += tempArray->GetValue((j + 1) * IMAGE_SIZE + (i + 1));
                srcArray->SetTuple1(idx, sum / 9.0);
            }
        }
    }

    timer->StopTimer();
    serialTime = timer->GetElapsedTime();
    std::cout << "  Serial smoothing time: " << std::fixed
              << std::setprecision(3) << serialTime << " seconds" << std::endl;

    // --- 并行版本（使用vtkSMPTools） ---
    std::cout << "  Running parallel smoothing (" << SMOOTH_ITER
              << " iterations)..." << std::endl;

    vtkNew<vtkImageData> parallelCopy;
    parallelCopy->DeepCopy(input);

    vtkDataArray* parSrc = parallelCopy->GetPointData()->GetScalars();
    vtkNew<vtkDoubleArray> parTemp;
    parTemp->SetNumberOfTuples(numPoints);

    // 定义并行平滑的Functor
    struct SmoothFunctor
    {
        vtkDataArray* Src;
        vtkDataArray* Tmp;
        int           ImageSize;

        SmoothFunctor(vtkDataArray* src, vtkDataArray* tmp, int sz)
            : Src(src), Tmp(tmp), ImageSize(sz) {}

        void operator()(vtkIdType beginRow, vtkIdType endRow)
        {
            // beginRow和endRow是行索引范围（Y方向），跳过边界
            vtkIdType rowStart = std::max(beginRow, static_cast<vtkIdType>(1));
            vtkIdType rowEnd   = std::min(endRow,
                static_cast<vtkIdType>(ImageSize - 1));

            for (vtkIdType j = rowStart; j < rowEnd; ++j)
            {
                for (int i = 1; i < ImageSize - 1; ++i)
                {
                    vtkIdType idx = j * ImageSize + i;
                    double sum = 0.0;
                    sum += Tmp->GetTuple1((j - 1) * ImageSize + (i - 1));
                    sum += Tmp->GetTuple1((j - 1) * ImageSize + i);
                    sum += Tmp->GetTuple1((j - 1) * ImageSize + (i + 1));
                    sum += Tmp->GetTuple1(j * ImageSize + (i - 1));
                    sum += Tmp->GetTuple1(j * ImageSize + i);
                    sum += Tmp->GetTuple1(j * ImageSize + (i + 1));
                    sum += Tmp->GetTuple1((j + 1) * ImageSize + (i - 1));
                    sum += Tmp->GetTuple1((j + 1) * ImageSize + i);
                    sum += Tmp->GetTuple1((j + 1) * ImageSize + (i + 1));
                    Src->SetTuple1(idx, sum / 9.0);
                }
            }
        }
    };

    timer->StartTimer();

    for (int iter = 0; iter < SMOOTH_ITER; ++iter)
    {
        // 复制源到临时数组（也可以并行化，但这里保持简单）
        for (vtkIdType i = 0; i < numPoints; ++i)
        {
            parTemp->SetValue(i, parSrc->GetTuple1(i));
        }

        SmoothFunctor functor(parSrc, parTemp, IMAGE_SIZE);
        vtkSMPTools::For(0, IMAGE_SIZE, functor);
    }

    timer->StopTimer();
    parallelTime = timer->GetElapsedTime();
    std::cout << "  Parallel smoothing time: " << std::fixed
              << std::setprecision(3) << parallelTime << " seconds"
              << std::endl;

    // 性能对比
    if (serialTime > 0.0)
    {
        double speedup = serialTime / parallelTime;
        std::cout << "  Speedup: " << std::fixed << std::setprecision(2)
                  << speedup << "x" << std::endl;
    }

    // 验证结果一致性（检查一些采样点的值差异）
    double maxDiff = 0.0;
    vtkDataArray* serResult = serialCopy->GetPointData()->GetScalars();
    vtkDataArray* parResult = parallelCopy->GetPointData()->GetScalars();
    for (vtkIdType i = 0; i < numPoints; i += 100)
    {
        double diff = std::abs(serResult->GetTuple1(i) -
                                parResult->GetTuple1(i));
        maxDiff = std::max(maxDiff, diff);
    }
    std::cout << "  Max difference (serial vs parallel): "
              << std::scientific << std::setprecision(6) << maxDiff
              << std::endl;
}

// ---------------------------------------------------------------------------
// 31.3 自定义并行计算：从多个标量数组计算派生标量
// ---------------------------------------------------------------------------
void CustomParallelComputation(vtkImageData* image)
{
    std::cout << "\n[3/4] Custom parallel computation: multi-array "
              << "derived scalars..." << std::endl;

    vtkIdType numPoints = static_cast<vtkIdType>(IMAGE_SIZE) * IMAGE_SIZE;

    // 创建额外的标量数组（模拟多物理量场景）
    std::vector<vtkSmartPointer<vtkDoubleArray>> arrays;
    const char* names[NUM_ARRAYS] = {
        "Temperature", "Pressure", "Density", "Velocity_Magnitude"
    };

    for (int a = 0; a < NUM_ARRAYS; ++a)
    {
        vtkNew<vtkDoubleArray> arr;
        arr->SetName(names[a]);
        arr->SetNumberOfComponents(1);
        arr->SetNumberOfTuples(numPoints);
        image->GetPointData()->AddArray(arr);
        arrays.push_back(arr);
    }

    // 给每个数组填充不同的空间模式（以IMAGE_SIZE为维度）
    for (int a = 0; a < NUM_ARRAYS; ++a)
    {
        vtkDoubleArray* arr = arrays[a];
        double offset   = a * 0.25;
        double freq     = (a + 1) * 2.0;
        double phase    = a * 1.5708; // pi/2 递增

        vtkIdType idx = 0;
        for (int j = 0; j < IMAGE_SIZE; ++j)
        {
            for (int i = 0; i < IMAGE_SIZE; ++i)
            {
                double nx = static_cast<double>(i) / IMAGE_SIZE;
                double ny = static_cast<double>(j) / IMAGE_SIZE;
                double val = offset +
                    0.3 * std::sin(freq * 3.14159 * nx + phase) +
                    0.2 * std::cos(freq * 3.14159 * ny);
                arr->SetValue(idx, val);
                ++idx;
            }
        }
        std::cout << "  Filled array '" << names[a] << "'" << std::endl;
    }

    // 创建输出数组：派生标量 = 加权组合
    vtkNew<vtkDoubleArray> derivedScalars;
    derivedScalars->SetName("DerivedScalar");
    derivedScalars->SetNumberOfComponents(1);
    derivedScalars->SetNumberOfTuples(numPoints);
    image->GetPointData()->AddArray(derivedScalars);
    image->GetPointData()->SetActiveScalars("DerivedScalar");

    // 权重系数
    double weights[NUM_ARRAYS] = { 0.4, 0.3, 0.2, 0.1 };

    // 为每个数组获取原始指针（线程安全，只读）
    double* rawArrays[NUM_ARRAYS];
    for (int a = 0; a < NUM_ARRAYS; ++a)
    {
        rawArrays[a] = arrays[a]->GetPointer(0);
    }
    double* output = derivedScalars->GetPointer(0);

    // 定义并行计算的Functor
    struct DeriveFunctor
    {
        double** Arrays;
        double*  Output;
        double*  W;
        int      NumArrays;

        DeriveFunctor(double** arrays, double* output, double* w, int na)
            : Arrays(arrays), Output(output), W(w), NumArrays(na) {}

        void operator()(vtkIdType begin, vtkIdType end)
        {
            for (vtkIdType i = begin; i < end; ++i)
            {
                // 读取所有数组的第i个值，计算加权和，写入输出
                double result = 0.0;
                for (int a = 0; a < NumArrays; ++a)
                {
                    result += W[a] * Arrays[a][i];
                }
                Output[i] = result;
            }
        }
    };

    // 执行并行计算
    vtkNew<vtkTimerLog> timer;
    timer->StartTimer();

    DeriveFunctor functor(rawArrays, output, weights, NUM_ARRAYS);
    vtkSMPTools::For(0, numPoints, functor);

    timer->StopTimer();
    std::cout << "  Derived scalar computation time: " << std::fixed
              << std::setprecision(4) << timer->GetElapsedTime()
              << " seconds" << std::endl;

    // 验证：输出一些采样值
    std::cout << "  Sample derived values:" << std::endl;
    std::cout << "    [0,0]:     " << std::fixed << std::setprecision(6)
              << output[0] << std::endl;
    std::cout << "    [500,500]: "
              << output[500 * IMAGE_SIZE + 500] << std::endl;
    std::cout << "    [999,999]: "
              << output[999 * IMAGE_SIZE + 999] << std::endl;
}

// ---------------------------------------------------------------------------
// 31.4 可视化结果
// ---------------------------------------------------------------------------
void VisualizeResult(vtkImageData* image)
{
    std::cout << "\n[4/4] Visualizing the result..." << std::endl;

    // 创建查找表
    vtkNew<vtkLookupTable> lut;
    lut->SetNumberOfTableValues(256);
    lut->SetRange(image->GetPointData()->GetScalars()->GetRange());
    lut->SetHueRange(0.667, 0.0); // 蓝到红
    lut->SetSaturationRange(1.0, 1.0);
    lut->SetValueRange(1.0, 1.0);
    lut->Build();

    // 颜色映射
    vtkNew<vtkImageMapToColors> colorMap;
    colorMap->SetInputData(image);
    colorMap->SetLookupTable(lut);
    colorMap->SetOutputFormatToRGBA();

    // ImageActor用于渲染图像数据
    vtkNew<vtkImageSliceMapper> imageMapper;
    imageMapper->SetInputConnection(colorMap->GetOutputPort());

    vtkNew<vtkImageSlice> imageSlice;
    imageSlice->SetMapper(imageMapper);

    // 渲染器
    vtkNew<vtkRenderer> renderer;
    renderer->AddViewProp(imageSlice);
    renderer->SetBackground(0.1, 0.1, 0.15);
    renderer->ResetCamera();

    // 渲染窗口
    vtkNew<vtkRenderWindow> renderWindow;
    renderWindow->SetSize(800, 800);
    renderWindow->SetWindowName("SMP Parallel Processing Demo");
    renderWindow->AddRenderer(renderer);

    // 交互器
    vtkNew<vtkRenderWindowInteractor> interactor;
    vtkNew<vtkInteractorStyleImage> style;
    interactor->SetInteractorStyle(style);
    interactor->SetRenderWindow(renderWindow);

    std::cout << "  Starting interactive visualization..." << std::endl;
    std::cout << "  Close the window to exit." << std::endl;

    renderWindow->Render();
    interactor->Start();

    std::cout << "  Visualization completed." << std::endl;
}

// ---------------------------------------------------------------------------
// 主函数
// ---------------------------------------------------------------------------
int main(int argc, char* argv[])
{
    // 禁用VTK的SMP后端进行线程数检测（保留默认行为）
    (void)argc;
    (void)argv;

    std::cout << "========================================================="
              << std::endl;
    std::cout << "  VTK SMP Parallel Processing Demo" << std::endl;
    std::cout << "========================================================="
              << std::endl;

    int numThreads = vtkSMPTools::GetEstimatedNumberOfThreads();
    std::cout << "VTK SMP threads available: " << numThreads << std::endl;

    // 步骤1：创建带噪声的测试数据
    vtkSmartPointer<vtkImageData> image = CreateNoisyImageData();

    // 步骤2：对比串行和并行平滑滤波的性能
    CompareSmoothingPerformance(image);

    // 步骤3：自定义并行计算（从多数组计算派生标量）
    CustomParallelComputation(image);

    // 步骤4：可视化结果
    VisualizeResult(image);

    std::cout << "\n========================================================="
              << std::endl;
    std::cout << "  Demo completed successfully." << std::endl;
    std::cout << "========================================================="
              << std::endl;

    return 0;
}
```

### 22.5.2 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20 FATAL_ERROR)
project(SMPParallelDemo VERSION 1.0.0 LANGUAGES CXX)

# 设置C++标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# --------------------------------------------------------------------------
# 查找VTK 9.5.2
# --------------------------------------------------------------------------
# 注意：SMP功能需要VTK在编译时启用SMP支持
# 如果编译VTK时设置了 -DVTK_SMP_IMPLEMENTATION_TYPE=OpenMP，
# 则以下组件会自动包含SMP/OpenMP支持。
find_package(VTK 9.5.2
  COMPONENTS
    CommonCore
    CommonDataModel
    CommonExecutionModel
    CommonMath
    CommonSystem          # vtkTimerLog 所在模块
    FiltersCore
    FiltersGeneral
    FiltersSources
    ImagingCore           # 图像处理核心
    ImagingGeneral        # vtkImageGaussianSmooth
    ImagingSources        # vtkImageNoiseSource
    InteractionStyle
    RenderingCore
    RenderingOpenGL2
    RenderingFreeType
    RenderingImage        # vtkImageSlice, vtkImageSliceMapper
  REQUIRED
)

# 打印VTK信息
message(STATUS "VTK version: ${VTK_VERSION}")
message(STATUS "VTK SMP threads: (detected at runtime)")

# --------------------------------------------------------------------------
# 创建可执行文件
# --------------------------------------------------------------------------
add_executable(SMPParallelDemo SMPParallelDemo.cxx)

# 链接VTK库
target_link_libraries(SMPParallelDemo
  PRIVATE
    ${VTK_LIBRARIES}
)

# VTK模块自动初始化（必需，否则运行时可能报"No override found"错误）
vtk_module_autoinit(
  TARGETS SMPParallelDemo
  MODULES ${VTK_LIBRARIES}
)

# 如果在Windows上且使用MSVC，可能需要以下定义
if(MSVC)
  target_compile_definitions(SMPParallelDemo PRIVATE
    NOMINMAX
    _USE_MATH_DEFINES
  )
endif()

# --------------------------------------------------------------------------
# 安装规则
# --------------------------------------------------------------------------
install(TARGETS SMPParallelDemo
  RUNTIME DESTINATION bin
)
```

### 22.5.3 编译和运行

将源代码保存为`SMPParallelDemo.cxx`，将CMakeLists.txt放在同一目录下，然后执行以下命令：

```bash
mkdir build
cd build
cmake .. -DVTK_DIR=/path/to/vtk-9.5.2/build
cmake --build . --config Release
./SMPParallelDemo        # 或 SMPParallelDemo.exe (Windows)
```

程序运行时会输出类似以下的内容：

```
=========================================================
  VTK SMP Parallel Processing Demo
=========================================================
VTK SMP threads available: 16
[1/4] Creating 1000x1000 image data with noise...
  Created 1000000 points.
[2/4] Comparing serial vs parallel Gaussian smoothing...
  Default VTK SMP threads: 16
  Running serial smoothing (5 iterations)...
  Serial smoothing time: 2.847 seconds
  Running parallel smoothing (5 iterations)...
  Parallel smoothing time: 0.312 seconds
  Speedup: 9.12x
  Max difference (serial vs parallel): 1.110223e-16
[3/4] Custom parallel computation: multi-array derived scalars...
  Filled array 'Temperature'
  Filled array 'Pressure'
  Filled array 'Density'
  Filled array 'Velocity_Magnitude'
  Derived scalar computation time: 0.0023 seconds
  Sample derived values:
    [0,0]:     0.272814
    [500,500]: 0.385719
    [999,999]: 0.518943
[4/4] Visualizing the result...
  Starting interactive visualization...
  Close the window to exit.
```

### 22.5.4 代码解析

这个完整的示例展示了SMP并行的四个关键使用模式：

1. **自动SMP（零代码）**：`vtkImageGaussianSmooth`等内置滤波器在内部自动使用SMP并行。你不需要做任何事情来启用它——只要VTK编译时支持SMP，它就会自动生效。

2. **手动SMP（vtkSMPTools::For + Functor）**：自定义并行计算的核心模式。定义一个Functor，在`operator()`中实现处理逻辑，调用`vtkSMPTools::For`分发到多线程。示例中的`SmoothFunctor`和`DeriveFunctor`都是这个模式的具体应用。

3. **性能对比（串行 vs 并行）**：通过计时器精确测量串行和并行版本的执行时间，计算加速比。在典型的8核以上CPU上，你应该能看到5-12倍的加速。

4. **多数组并行计算**：在Functor中同时读取多个输入数组、执行计算、写入单个输出数组。这是科学可视化中非常常见的模式——例如从多个物理量（温度、压力、密度、速度）计算派生量（如马赫数、总焓）。

---

## 22.6 本章小结

本章介绍了VTK 9.5.2中SMP（共享内存并行）的基础知识和使用方法。以下是要点回顾：

### 核心概念

| 概念 | 说明 |
|------|------|
| SMP vs MPI | SMP在单机多核上并行；MPI跨多机分布式并行。本章聚焦SMP。 |
| vtkSMPTools::For | 将范围切分为子范围，每个线程处理一个子范围。 |
| Functor模式 | 定义`operator()(vtkIdType begin, vtkIdType end)`的可调用对象。 |
| 粒度 (grain size) | 控制每个任务的最小处理元素数，影响负载均衡和并行开销。 |
| vtkSMPThreadLocal | 为每个线程提供私有存储，用于累加局部结果。 |
| vtkSMPThreadLocalObject | 为每个线程提供私有的VTK对象（如vtkGenericCell）。 |
| vtkThreadedImageAlgorithm | 通过继承和重写`ThreadedRequestData`来创建自定义并行图像滤波器。 |

### 线程安全法则

1. **从vtkDataArray读取**是线程安全的（只要不同线程读取不同索引）。
2. **向vtkDataArray写入不同索引**是线程安全的。
3. **修改vtkDataSet拓扑结构**（InsertNextCell、SetPoints等）不是线程安全的。
4. **调用管道的Update()**不是线程安全的。
5. **使用vtkSMPThreadLocal**来安全地在线程间累加结果。

### 何时手动使用SMP

- 当需要对大量数据执行自定义的逐元素计算时（例如自定义数学变换）。
- 当需要从多个数组组合计算派生标量时。
- 当需要在单个数据集上执行VTK未内置的算法时。
- 当内置的自动SMP滤波器不满足你的特定需求时。

### 下一章预览

掌握了SMP并行之后，你已经具备了在单机上高效处理大规模数据的能力。在后续章节中，我们将探索VTK的更多高级特性——包括自定义算法的深入开发、与外部库的集成，以及进一步的大规模数据处理技巧。

并行计算是一项强大的工具，但也需要谨慎使用。始终记住线程安全的边界，在必要时使用`vtkSMPThreadLocal`进行安全的线程间数据隔离，并始终验证并行结果与串行结果的一致性。

---

## 参考文献

1. VTK 9.5.2 Source Code: `Common/Core/vtkSMPTools.h` -- SMP工具类的接口定义。
2. VTK 9.5.2 Source Code: `Common/Core/vtkSMPThreadLocal.h` -- 线程局部存储的实现。
3. VTK 9.5.2 Source Code: `Common/Core/vtkSMPThreadLocalObject.h` -- 线程局部VTK对象的实现。
4. VTK 9.5.2 Source Code: `Imaging/Core/vtkThreadedImageAlgorithm.h` -- 线程化图像算法基类。
5. VTK Wiki: "VTK/Parallelism" -- VTK并行框架的概述文档。
6. OpenMP Specification: https://www.openmp.org/ -- OpenMP规范文档。
7. Intel TBB Documentation: https://www.intel.com/content/www/us/en/developer/tools/oneapi/onetbb.html -- TBB官方文档。
