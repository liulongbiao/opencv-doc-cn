# 简介

OpenCV (开源计算机视觉类库: http://opencv.org) 是一个包含数百种计算机视觉算法的开源 BSD 许可的类库。
本文档描述了称为 OpenCV 2.x 的 API，它主要是一个 C++ API， 而以前的基于 C 的 API 为 OpenCV 1.x API。
后者在 `opencv1x.pdf` 中定义。

OpenCV 具有模块化的结构，即它的包由多个共享或者静态类库组成。
当前有以下模块：

* [核心功能](../../d0/de1/group__core.md) - 一个简洁的模块，它定义了基本数据结构，包括
稠密多维数组 `Mat` 和由其它模块所使用的基础函数。
* [图像处理](../../d7/dbd/group__imgproc.md) - 一个图像处理模块，包含了线性和非线性图像过滤，
几何图像转换(改变尺寸、仿射和透视弯曲，通用基于表格的重映射)，颜色空间转换、直方图等等。
* `video` - 一个视频分析模块，包含运动估计、背景减除和对象跟踪算法
* `calib3d` - 基本多视图几何算法，单摄像机和立体摄像机标定，目标姿态估计，立体匹配算法和三维重构要素
* `features2d` - 显著特征检测器、描述符和描述符匹配器
* `objdetect` - 目标和预定义类的实例(如脸部、眼睛、马克杯、人物、汽车等)的检测
* `highgui` - 简单 UI 功能的易用的接口
* `videoio` - 视频捕获和视频编解码的易用接口
* `gpu` - 针对不同 OpenCV 模块的 GPU 加速算法
* ... 一些其它帮助模块，如 `FLANN` 和 Google 测试封装器，Python 绑定等等。

本文档的后续章节描述了每个模块的功能。
但首先请确保熟悉在该类库中通用的 API 概念。

## API 概念

### `cv` 命名空间

所有 OpenCV 类和函数都放置在 `cv` 命名空间下。
因此要在你的代码中访问这些功能，你需要使用 `cv::` 限定符或 `using namespace cv;` 指令：

```c++
#include "opencv2/core.hpp"
...
cv::Mat H = cv::findHomography(points1, points2, CV_RANSAC, 5);
...
```

或者：

```c++
#include "opencv2/core.hpp"
using namespace cv;
...
Mat H = findHomography(points1, points2, CV_RANSAC, 5);
...
```

其中某些当前或未来版本的 OpenCV 的外部名称可能和 STL 或其它类库的名称相冲突。
这时，需要使用命名空间限定符来解决命名冲突：

```c++
Mat a(100, 100, CV_32F);
randu(a, Scalar::all(1), Scalar::all(std::rand()));
cv::log(a, a);
a /= std::log(2.);
```

### 自动内存管理

OpenCV 会自动管理所有的内存。

首先， `std::vector`、`Mat` 和其它由其函数和方法所使用的数据结构都具有解构器，会在需要的时候
释放底层的内存缓冲。这意味着这些解构器并不会总是释放缓冲，如 Mat。
它们会尽可能地共享数据。一个结构其会减少其关联的矩阵数据缓冲的引用计数。
其缓冲区会且仅会在其引用计数到达 0 时被释放，即没有其它结构引用该缓冲区。
类似的，当一个 `Mat` 实例被拷贝时，并没有实际地拷贝任何数据。
相反，其引用次数被增加以表示存在另一个相同数据的拥有者。
也存在一个 `Mat::clone` 方法来创建该矩阵数据的完整拷贝。示例如下：

```c++
// create a big 8Mb matrix
Mat A(1000, 1000, CV_64F);
// create another header for the same matrix;
// this is an instant operation, regardless of the matrix size.
Mat B = A;
// create another header for the 3-rd row of A; no data is copied either
Mat C = B.row(3);
// now create a separate copy of the matrix
Mat D = B.clone();
// copy the 5-th row of B to C, that is, copy the 5-th row of A
// to the 3-rd row of A.
B.row(5).copyTo(C);
// now let A and D share the data; after that the modified version
// of A is still referenced by B and C.
A = D;
// now make B an empty matrix (which references no memory buffers),
// but the modified version of A will still be referenced by C,
// despite that C is just a single row of the original A
B.release();

// finally, make a full copy of C. As a result, the big modified
// matrix will be deallocated, since it is not referenced by anyone
C = C.clone();
```

可以看到 `Mat` 和其它数据结构的使用还是很简单的。
但是高层级的类或者甚至用户创建的数据类型是否能够自动内存管理呢？
对于它们， OpenCV 提供了类似于 C++11 中 `std::shared_ptr` 的 `Ptr` 模板类。
因此，取之使用原生的指针：

```c++
T* ptr = new T(...);
```

你可以使用：

```c++
Ptr<T> ptr(new T(...));
```

或者：

```c++
Ptr<T> ptr = makePtr<T>(...);
```

`Ptr<T>` 封装了一个到某个 T 实例的指针和关联该指针的引用计数器。详见 `Ptr` 描述部分。

### 自动对输出数据进行分配

OpenCV 会自动释放内存，同时在大多数时候也会对其输出函数的参数自动分配内存。
因此如果某个函数具有一或多个输入数组(`cv::Mat` 实例)和一些输出数组，
其输出数组会被自动分配或重分配。
这些输出数组的大小和类型由输入数组的大小和类型所决定。
如果需要的话，这些函数也接收额外的参数来帮助确定输出数组的属性。

如：

```c++
#include "opencv2/imgproc.hpp"
#include "opencv2/highgui.hpp"

using namespace cv;
int main(int, char**)
{
    VideoCapture cap(0);
    if(!cap.isOpened()) return -1;

    Mat frame, edges;
    namedWindow("edges",1);
    for(;;)
    {
        cap >> frame;
        cvtColor(frame, edges, COLOR_BGR2GRAY);
        GaussianBlur(edges, edges, Size(7,7), 1.5, 1.5);
        Canny(edges, edges, 0, 30, 3);
        imshow("edges", edges);
        if(waitKey(30) >= 0) break;
    }
    return 0;
}
```

其数组 `frame` 由 `>>` 操作符所自动地分配，因为视频帧的分辨率和位深度在视频捕获模块中是已知的。
其数组 `edges` 由 `cvtColor` 函数自动分配。它具有和输入数组一样的大小和位深度。
频道的数量为 1 ，因为传入了颜色转换代码 `COLOR_BGR2GRAY`，它意味着将颜色进行灰度转换。
注意 `frame` 和 `edges` 在整个循环体中仅在第一次执行时进行了分配，因为所有后续的视频帧都具有相同的分辨率。
若你在某些地方改变了视频的分辨率，则数组会被自动重新分配。

这个技术的关键组件是 `Mat::create` 方法。
它接收所需的数组大小和类型。
若数组已经具有指定大小和类型，则该方法什么都不做。
否则它释放以前(如果有的话)分配的数据，(这部分设计减少引用计数并和 0 做比较)，
然后分配一个和所需大小一样的新的缓冲区。
大多数函数对每个输出函数调用了 `Mat::create` 方法，这样自动输出数据的分配被实现。

这种模式的一些显著的例外是 `cv::mixChannels`、 `cv::RNG::fill` 和一些其它的函数和方法。
它们不能分配输出数组，因此你需要预先分配。

### 饱和度算术

作为一个计算机视觉类库，OpenCV 需要处理大量以紧凑的每个频道 8 或 16 位来编码的像素点，
这种形式并且因此具有有限的值的范围。
更进一步，在图像上的特定的操作，如色度空间转换、亮度/对比度调整、锐化、复杂插值(双立方、Lanczos)
可能会产生在可用范围之外的值。
若你仅存储结果的最低的 8(16) 位，它会导致人为的可视工件并且可能会进一步影响图像分析。
要解决这个问题，这里使用了一种称为 **饱和度** 的算术。
如，要将某个操作的结果 `r` 存储为一个 8 位图像，你需要找到 0~255 范围内最接近的值。

```
I(x,y)=min(max(round(r),0),255)
```

类似的规则同样应用在 8位有符号、 16 位有符号和无符号类型。
这种语义被类库中的各种地方所使用。
在 C++ 代码中，它通过使用类似于标准 C++ 转型操作的 `saturate_cast<>` 函数来完成。
如下对以上提供公式的实现：

```c++
I.at<uchar>(y, x) = saturate_cast<uchar>(r);
```

其中 `cv::uchar` 是一个 OpenCV 8位无符号整型。
在优化的 SIMD 代码中，会使用诸如 `paddusb`、`packuswb` 等等这样的 SSE2 指令。
它们帮助得到和 C++ 代码中的相同行为。

> 注： 饱和度不会应用在 32 位整型的结果上。

### 固定的像素类型，有节制地使用模板

模板是 C++ 中的一个强大特性，使得可以实现强大、有效且安全的数据结构和算法。
然而对模板的过度使用会导致编译时间和代码尺寸的显著增加。
另外，当模板被专有地使用时也很难分离接口和实现。
这对于基础算法很好用，但不适合计算机视觉类库，我们的单个算法很可能跨数千行代码。
因此，同时也为了简化对其他语言(如Python、Java、Matlab)的绑定的开发，
(这些语言可能没有模板或者仅有有限的模板能力)，
当前的 OpenCV 实现上基于多态，且运行时通过模板分发。
在那些运行时分发可能过慢(如像素访问操作符)、不可能(泛型 Ptr<> 实现) 
或者只是非常不方便(saturate_cast<>()) 的地方，当前的实现引入了较小的模板类、方法和函数。
当前 OpenCV 版本的任何其他地方，对模板的使用还是很节制的。

因此，该类库只能操作少量的原生数据类型。
即，数组元素应该是以下类型之一：

* 8位无符号整型(uchar)
* 8位有符号整型(schar)
* 16位无符号整型(ushort)
* 16位有符号整型(short)
* 32位有符号整型(int)
* 32位浮点数(float)
* 64位浮点数(double)
* 多个元素的元组，其中所有的元素具有相同的类型(上述类型的一种)。
一个元素为这种元组的数组被称为多通道数组，相对应的单通道数组的元素为标量值。
最大可能的通道数量由 `CV_CN_MAX` 常量定义，其值当前为 512.

对这些基础类型，对应于以下枚举：

```c++
enum { CV_8U=0, CV_8S=1, CV_16U=2, CV_16S=3, CV_32S=4, CV_32F=5, CV_64F=6 };
```

多通道(n-channel) 类型可通过以下选项来指定：

* `CV_8UC1` ... `CV_64FC4` 常量(通道数量从 1 到 4)
* `CV_8UC(n)` ... `CV_64FC(n)` 或 `CV_MAKETYPE(CV_8U, n)` ... `CV_MAKETYPE(CV_64F, n)` 宏，
其中通常数量大于 4 或者在编译时是未知的。

> 注：`CV_32FC1 == CV_32F`, `CV_32FC2 == CV_32FC(2) == CV_MAKETYPE(CV_32F, 2)`, 且 `CV_MAKETYPE(depth, n) == ((depth&7) + ((n-1)<<3)`。
> 这意味着这些常量类型由 depth 的低三位和通道数量减 1，接收后 `log2(CV_CN_MAX)` 位构造而成。

示例：

```c++
Mat mtx(3, 3, CV_32F); // make a 3x3 floating-point matrix
Mat cmtx(10, 1, CV_64FC2); // make a 10x1 2-channel floating-point
                           // matrix (10-element complex vector)
Mat img(Size(1920, 1080), CV_8UC3); // make a 3-channel (color) image
                                    // of 1920 columns and 1080 rows.
Mat grayscale(image.size(), CV_MAKETYPE(image.depth(), 1)); // make a 1-channel image of
                                                            // the same size and same
                                                            // channel type as img
```

具有更复杂类型元素的数组不能被 OpenCV 所构造或处理。
更进一步，每个函数或方法仅能处理所有可能数组类型的一个子集。
算法越复杂，支持的格式越少。这种限制的典型示例如：

* 脸部检测仅支持 8 位灰度或彩色图像
* 线性代数函数和大多数机器学习算法仅适用于浮点类型数组
* 基础函数，如 `cv::add` 支持所有类型
* 颜色空间转换函数支持 8位无符号、16位无符号和32位浮点类型

每个函数所支持的类型子集是从实际所需来定义，且后续可能会根据用户需要进行扩展。

### InputArray 和 OutputArray

很多 OpenCV 函数处理的是稠密2维或多维数值数组。
通常这些函数接收 `cppMat` 作为参数，但在某些情况下，
使用 `std::vector<>` (如对于点集) 或 `Matx<>` (对3x3单应矩阵等) 会更方便。
为避免在 API 中的重复定义，会引入特殊的 "代理" 类。
基础的 "代理" 类是 `InputArray`。它用于给函数输入传入只读数组。
派生自 `InputArray` 的 `OutputArray` 类用于给函数指定一个输出数组。
通常你不需要关注这些中间类型(并且不应该明确地声明这些类型的变量) - 它自己会运作。
你可以假设在任何 `InputArray/OutputArray` 的地方，
你总是可以使用 ` Mat`, `std::vector<>`, `Matx<>`, `Vec<>` 或 `Scalar`。
当函数具有一个可选的输入或输出数组，而你又不需要或不想传入时，你可以传入 `cv::noArray()` 。

### 错误处理

OpenCV 使用异常来发出重要错误的信号。
当输入数据具有正确的格式且属于指定的值范围，但算法因某些原因没有成功时(如优化算法没有收敛)，
它返回特殊的错误代码(通常为一个布尔值)。

异常可以是 `cv::Exception` 类或其派生类的实例。而 `cv::Exception` 派生自 `std::exception`。
因此它可以被其它标准 C++ 类库组件优雅地处理。

异常通常使用 `CV_Error(errcode, description)` 宏或者其类似于 printf 的变体
`CV_Error_(errcode, printf-spec, (printf-args))` 来抛出，
或者使用 ` CV_Assert(condition)` 宏来检验条件并在不满足时抛出一个异常。
对性能关键的代码，其 `CV_DbgAssert(condition)` 仅会在调试配置中被保留。
由于有自动内存管理，所有中间缓冲会在突发错误时被自动释放。
你只需要在需要的地方添加一个 `try` 语句来捕获异常：

```c++
try
{
    ... // call OpenCV
}
catch( cv::Exception& e )
{
    const char* err_msg = e.what();
    std::cout << "exception caught: " << err_msg << std::endl;
}
```

### 多线程和可重入性

当前的 OpenCV 实现是完全可重入的。即对于相同的函数、某个类实例的相同的 **常量** 方法
或不同类实例的相同 **非常量** 方法都可以从不同的线程来调用。
同样，相同的 `cv::Mat` 可用在不同的线程中，因为引用计数操作符使用了架构特有的原子指令。
