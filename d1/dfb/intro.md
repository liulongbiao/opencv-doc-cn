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


