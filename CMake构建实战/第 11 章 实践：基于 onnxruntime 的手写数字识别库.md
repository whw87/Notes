# 第 11 章 实践：基于 onnxruntime 的手写数字识别库

读者已经跟着本书实践了很多零零散散的实例，应该能够熟练使用CMake来构建 C 和 C++ 程序了吧！不过，前面的实例往往都是针对某个特定功能编写的，我们可能很难将它们综合起来实现一个完成度较高的项目。不必担心，本章就带领大家使用 C++ 语言实现一个完整的动态库，以及调用该库的可执行文件——手写数字识别库和手写数字识别命令行工具。相信经过本章的实践，读者一定可以将前面所学的知识融会贯通，应用于中大型项目中了！

## 11.1 前期设计

### 11.1.1 模块设计

本章不仅要实现一个手写数字识别库，还会同时编写一个 `recognize` 命令行工具，用户可以在命令行中调用该工具以识别图片中的手写数字。因此，项目中需要定义两个构建目标，分别是动态库目标 `num_recognizer` 和可执行文件目标 `recognize`。其中，可执行文件目标 `recognize` 将链接动态库目标 `num_recognizer`。

另外，我们希望构建的手写数字识别库是一个通用库，使其能够被 C++ 语言之外的其他编程语言调用，如 C 语言等。这就要求手写数字识别动态库在暴露 API 时，必须仅暴露符合C语言应用程序二进制接口（Application Binary Interface，ABI）的应用程序编程接口（Application Programming Interface，API）。简言之，就是暴露的接口都只能是纯 C 函数，函数的参数、返回值等都必须是 C 语言中支持的数据类型。在 C++ 编程语言中若想将一个函数定义为 C 语言函数，将一个结构体定义为 C 语言结构体，需要在函数定义和结构体定义前指定 `extern "C"` 修饰符，或将其置于 `extern "C"` 代码块中。

### 11.1.2 项目目录结构

设计好需要的模块后，就可以开始着手建立项目的目录结构了。本项目目录结构如下：

```text
ch011
├── CMakeLists.txt (目录程序)
├── cli (命令行工具的源文件目录)
│   └── recognize.c (命令行工具的源文件)
├── cmake (自定义CMake模块目录)
│   └── ...
├── include (头文件目录)
│   └── num_recognizer.h (手写数字识别库的头文件)
├── models (onnx模型文件目录)
│   └── mnist.onnx (手写数字识别模型文件)
└── src (手写数字识别库的源文件目录)
    └── num_recognizer.cpp (手写数字识别库的源文件)
```

命令行工具的源文件目录名称为 `cli`，这是命令行接口的英文 commandline interface 的缩写。另外，这里用到的 `mnist.onnx` 神经网络模型文件可以在 GitHub 的 `onnx/models` 代码仓库中下载。

### 11.1.3 接口设计

在实现之前，还需要对手写数字识别库具体提供什么功能作出定义，并将接口设计出来。

作为一个实践案例，该库不会涉及过于复杂的技术。尽管如此，笔者也希望这个手写数字识别库仍然是实用的：既支持用户传入二值化的图片像素数组，也支持用户传入一个 PNG 图片文件的路径来进行识别。可以说是麻雀虽小，五脏俱全！

#### 初始化

使用 `onnxruntime` 库，需要先初始化一个 `onnx` 环境（Ort::Env）供 `onnx` 会话（Ort::Session）使用。因此，手写数字识别库应当首先提供一个初始化的接口，如代码清单11.1所示。

**代码清单11.1 ch011/include/num_recognizer.h（第16行~第18行）**

```c
//! @brief 初始化手写数字识别库
//! @return void
NUM_RECOGNIZER_EXPORT void num_recognizer_init();
```

接口函数最前面的 `NUM_RECOGNIZER_EXPORT` 是导出宏，后面会在CMake目录程序中使用 [GenerateExportHeader](https://cmake.org/cmake/help/v3.30/module/GenerateExportHeader.html) 这一CMake模块定义它们。该CMake模块的具体用法参见9.2.3小节。

#### 创建和析构识别器

手写数字识别模型文件也许会更新迭代，因此接口应当能够灵活地根据用户指定的模型文件来创建识别器对象，同时提供用于析构识别器对象的接口，如代码清单11.2所示。

**代码清单11.2 ch011/include/num_recognizer.h（第20行~第28行）**

```c
//! @brief 创建识别器
//! @param model_path 模型文件路径
//! @param[out] out_recognizer 接受初始化的识别器指针的指针
NUM_RECOGNIZER_EXPORT void num_recognizer_create(const char *model_path,
                                                 Recognizer **out_recognizer);
//! @brief 析构识别器
//! @param recognizer 识别器的指针
NUM_RECOGNIZER_EXPORT void num_recognizer_delete(Recognizer *recognizer);
```

注意，`num_recognizer_create` 接口的第二个参数类型是 `Recognizer**`，即 `Recognizer` 结构体指针的指针。调用该接口后，程序会将创建好的识别器对象的指针赋值到该参数指向的变量中。

目前 `Recognizer` 类尚未定义，可以先在头文件中写一个前向声明，如代码清单11.3所示。

**代码清单11.3 ch011/include/num_recognizer.h（第11行）**

```c
struct Recognizer;
```

#### 识别二值化图片像素数组

手写数字识别库可以接受一个代表各像素颜色的数组作为被识别的图片对象。该数组是一个按行存储的 28×28 的 float 数组，即第一维索引对应列号，第二维索引对应行号。其中，元素的值若为 0，则代表白色，为 1 则代表黑色，因此它实际上表示了一个二值化后的图片。该接口如代码清单11.4所示。

**代码清单11.4 ch011/include/num_recognizer.h（第30行~第38行）**

```c
//! @brief 识别图片数据中的手写数字
//! @param recognizer 识别器的指针
//! @param input_image
//! 模型接受的输入图片数据（28×28的float数值数组，0代表白色，1代表黑色）
//! @param result 接受识别结果的数值的指针
//! @return 错误值，成功返回0
NUM_RECOGNIZER_EXPORT int num_recognizer_recognize(Recognizer *recognizer,
                                                  float *input_image,
                                                  int *result);
```

#### 识别 PNG 图片

当然，只提供接受数组参数的接口并不便于用户调用。这里还提供了一个可以直接识别指定路径的 PNG 图片中手写数字的接口，如代码清单11.5所示。

**代码清单11.5 ch011/include/num_recognizer.h（第40~第47行）**

```c
//! @brief 识别PNG图片中的手写数字
//! @param recognizer 识别器的指针
//! @param png_path PNG图片文件路径
//! @param result 接受识别结果的数值的指针
//! @return 错误值，成功返回0
NUM_RECOGNIZER_EXPORT int num_recognizer_recognize_png(Recognizer *recognizer,
                                                      const char *png_path,
                                                      int *result);
```

至此，手写数字识别库的全部接口声明完毕。

#### 接口功能实现思路

接口设计好后，不妨总结一下如果要实现这些接口的功能，需要有哪些具体的行为，借助哪些工具。

- 对二值化图片数组进行手写数字识别可以借助 `onnxruntime` 库来完成。
- 读取 PNG 图片像素可以借助 `libpng` 库来完成。
- 将 PNG 图片像素数据转换为 28×28 的二值化图片数组，即图片缩放及二值化算法。该功能由我们自行实现。

看起来我们能够站在巨人的肩膀上来完成这件事，应该能简单不少！

## 11.2 第三方库

正式编写程序之前，首先需要安装刚刚提到的第三方库。`onnxruntime` 库的安装已经在9.4.9节讲过，因此本节重点关注其他第三方库的安装： `libpng` 库及 `libpng` 依赖的 `zlib` 库。那么，首先一起来安装 `zlib` 库吧！

Linux 操作系统通常预装了 `zlib` 库，读者可以先尝试跳过 `zlib` 库的安装，看能否直接成功构建并安装 `libpng` 库。

### 11.2.1 安装 zlib 库

`zlib` 库的源程序可以从它的 GitHub 代码仓库中获取。将代码克隆或下载到本地后，按照以下步骤构建并安装：

```shell
> cd zlib
> mkdir build
> cd build
> cmake -DCMAKE_BUILD_TYPE=Release ..
> cmake --build . --config Release
> cmake --install . # 需要管理员权限
```

在执行 `cmake --install` 命令安装CMake项目时，CMake 会默认将其安装到系统目录中：在Windows中，默认安装目录前缀一般是 `C:\Program Files (x86)\zlib`；在 Linux 中，默认安装目录前缀一般是 `/usr/local`。如果想使用默认的安装目录，执行该命令时需要提供管理员权限。

当然，为 `cmake --install` 命令指定 `--prefix <安装目录>` 的参数，也可以自定义安装目录。不过采用这种方式，使用 [find_package](https://cmake.org/cmake/help/v3.30/command/find_package.html) 命令查找第三方库时，通常需要手动指定用于提示安装目录的参数或变量。

### 11.2.2 安装 libpng 库

`libpng` 库的源程序同样可以从其 GitHub 代码仓库中获取（本例采用 v1.6.40 版本）。其构建和安装步骤与构建和安装 `zlib` 库的步骤几乎完全相同：

```shell
> cd libpng
> mkdir build
> cd build
> cmake -DCMAKE_BUILD_TYPE=Release ..
> cmake --build . --config Release
> cmake --install . # 需要管理员权限
```

### 11.2.3 libpng 的查找模块

CMake 预置了 `zlib` 库的查找模块，不必自行实现；`onnxruntime` 库的查找模块在9.4.9小节中已经实现过，因此本小节也不再重复，这里仅介绍如何实现 `libpng` 的查找模块。

`libpng` 库自带了用于配置模式下的 [find_package](https://cmake.org/cmake/help/v3.30/command/find_package.html) 命令的配置文件，但其中缺失关于头文件目录等属性的设置，因此需要对其进行二次包装，编写一个自定义查找模块。模块程序的核心部分如 代码清单11.6所示。

**代码清单11.6 ch011/cmake/Findlibpng（第38行~第74行）**

```cmake
# 调用libpng库自带的配置文件来查找软件包，其自带配置文件会创建两个导入库目标：
# 1. 动态库导入目标png_shared
# 2. 静态库导入目标png_static
find_package(libpng CONFIG CONFIGS libpng16.cmake)
# 若成功查找，为两个库目标补上缺失的头文件目录属性
if(libpng_FOUND)
  # 获取png动态库导入目标对应动态库文件的路径，首先尝试其IMPORTED_LOCATION属性
  get_target_property(libpng_LIBRARY png_shared IMPORTED_LOCATION)
  # 若未能获得动态库文件路径，再尝试其IMPORTED_LOCATION_RELEASE属性
  if(NOT libpng_LIBRARY)
    get_target_property(libpng_LIBRARY png_shared IMPORTED_LOCATION_RELEASE)
  endif()
  # 根据png动态库的路径，设置libpng的根目录
  set(_png_root "${libpng_LIBRARY}/../..")
  # 查找png.h头文件所在目录的路径
  find_path(libpng_INCLUDE_DIR png.h
    HINTS ${_png_root}
    PATH_SUFFIXES include)
  # 为png_shared和png_static导入库目标设置头文件目录属性
  target_include_directories(png_shared INTERFACE ${libpng_INCLUDE_DIR})
  target_include_directories(png_static INTERFACE ${libpng_INCLUDE_DIR})
endif()
include(FindPackageHandleStandardArgs)
# 检查变量是否有效以及配置文件是否成功执行
find_package_handle_standard_args(libpng 
  REQUIRED_VARS libpng_LIBRARY libpng_INCLUDE_DIR
  CONFIG_MODE)
# 若一切成功，设置结果变量
if(libpng_FOUND)
  set(libpng_INCLUDE_DIRS ${libpng_INCLUDE_DIR})
  set(libpng_LIBRARIES ${libpng_LIBRARY})
endif()
```

本书暂未涉及 [find_package](https://cmake.org/cmake/help/v3.30/command/find_package.html) 命令配置模式的内容，因此没有对该查找模块的原理做更多解释，感兴趣的读者可以试着结合程序注释和官方文档自行理解。

## 11.3 CMake目录程序

终于完成了准备工作，可以开始手写数字识别库的部分了。首先编写好CMake目录程序，在项目根目录中创建 CMakeLists.txt，并把按照惯例要写的代码先写上去，如代码清单11.7所示。

**代码清单11.7 ch011/CMakeLists.txt（第1行~第4行）**

```cmake
cmake_minimum_required(VERSION 3.20)
project(num_recognizer)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")
set(CMAKE_CXX_STANDARD 11) # 设置C++标准为11
```

### 11.3.1 查找软件包

接着，查找即将用到的两个软件包，如代码清单11.8所示。

**代码清单11.8 ch011/CMakeLists.txt（第6行~第18行）**

```cmake
set(onnx_version 1.10.0) # 根据下载的版本进行设置，本例使用1.10.0版本
# 请下载onnxruntime库的压缩包，并解压至该目录中
if("$ENV{onnxruntime_ROOT}" STREQUAL "")
  if(WIN32)
      set(ENV{onnxruntime_ROOT} "${CMAKE_CURRENT_LIST_DIR}/onnxruntime-winx64-${onnx_version}")
  elseif(APPLE)
      set(ENV{onnxruntime_ROOT} "${CMAKE_CURRENT_LIST_DIR}/onnxruntime-osxuniversal2-${onnx_version}")
  else()
      set(ENV{onnxruntime_ROOT} "${CMAKE_CURRENT_LIST_DIR}/onnxruntime-linuxx64-${onnx_version}")
  endif()
endif()
find_package(onnxruntime 1.10 REQUIRED) # 指定依赖的最小版本
find_package(libpng REQUIRED)
```

其中的 `if` 条件只是为了设置 `onnxruntime_ROOT` 环境变量的值为 `onnxruntime` 软件包的安装目录，用于提示查找模块查找的路径。这里无须为 `libpng` 的查找模块提示查找的路径，因为我们将 `libpng` 安装到了默认的安装路径，而且 `libpng` 的查找模块能够在默认安装路径中找到它。

### 11.3.2 num_recognizer 动态库目标

下面创建本实例的第一个构建目标：`num_recognizer` 动态库目标。对应的 CMake 目录程序片段如代码清单11.9所示。

**代码清单11.9 ch011/CMakeLists.txt（第20行~第31行）**

```cmake
add_library(num_recognizer SHARED src/num_recognizer.cpp)
include(GenerateExportHeader)
generate_export_header(num_recognizer)
set_target_properties(num_recognizer PROPERTIES
  CXX_VISIBILITY_PRESET hidden
  VISIBILITY_INLINES_HIDDEN 1
)
target_include_directories(num_recognizer PUBLIC include ${CMAKE_BINARY_DIR})
target_link_libraries(num_recognizer PRIVATE onnxruntime::onnxruntime png_shared)
target_compile_definitions(num_recognizer PRIVATE ORT_NO_EXCEPTIONS num_recognizer_E
XPORTS)
```

这里除了调用 [add_library](https://cmake.org/cmake/help/v3.30/command/add_library.html) 命令创建动态库构建目标，还引用了 [GenerateExportHeader](https://cmake.org/cmake/help/v3.30/module/GenerateExportHeader.html) 模块，并调用了它提供的 `generate_export_header` 命令来为动态库生成导出头文件。同时，设置了该动态库目标的两个属性，用于默认隐藏符号并仅导出显式指定的符号。

为了让该库能够直接引用刚刚生成的导出头文件，这里要将当前二进制目录 `${CMAKE_ BINARY_DIR}` 加入库目标的头文件搜索目录。另外，为该库定义 `num_recognizer_EXPORTS` 宏以表示当前正在构建该库而非使用该库，确保导出头文件中的宏定义正确。

这里还将 `include` 目录加入该库的头文件搜索目录，并将 `onnxruntime` 和 `libpng` 第三方库目标链接到该库目标中。由于我们封装的是符合C语言 ABI 的动态库，不希望程序中有异常抛出，这里还定义了 `ORT_NO_EXCEPTIONS` 宏以禁用 `onnxruntime` 库中的异常。

### 11.3.3 recognize 可执行文件目标

配置好动态库目标后，创建 `recognize` 命令行工具的可执行文件目标。这非常简单，只需创建目标、指定源文件并链接刚刚创建好的动态库目标。CMake目录程序片段如代码清单11.10所示。

**代码清单11.10 ch011/CMakeLists.txt（第33行、第34行）**

```cmake
add_executable(recognize cli/recognize.c)
target_link_libraries(recognize PRIVATE num_recognizer)
```

## 11.4 代码实现

拖了这么久，下面就要施展真正的“魔法”了。不过这也体现了一个项目的成功，只靠代码写得漂亮还远远不够，还要依赖井井有条的项目结构和完善的基础设施。

### 11.4.1 全局常量和全局变量

首先，在手写数字识别库的源文件中定义一些全局的常量和变量，如代码清单11.11所示。

**代码清单11.11 ch011/src/num_recognizer.cpp（第17行~第27行）**

```c
static const char *INPUT_NAMES[] = {"Input3"}; // 模型输入参数名
static const char *OUTPUT_NAMES[] = {"Plus214_Output_0"}; // 模型输出参数名
static constexpr int64_t INPUT_WIDTH = 28;  // 模型输入图片宽度
static constexpr int64_t INPUT_HEIGHT = 28; // 模型输入图片高度
static const std::array<int64_t, 4> input_shape{
    1, 1, INPUT_WIDTH, INPUT_HEIGHT}; // 输入数据的形状（各维度大小）
static const std::array<int64_t, 2> output_shape{
    1, 10}; // 输出数据的形状（各维度大小）
static Ort::Env env{nullptr};                // onnxruntime环境
static Ort::MemoryInfo memory_info{nullptr}; // onnxruntime内存信息
```

其中的常量都是由手写数字识别库的 `onnx` 模型的神经网络结构决定的，如果要切换到具有不同神经网络结构的模型，可能需要做出相应修改。

其中 `env` 变量即 `onnxruntime` 的环境，由于它不能在静态初始化时构造，这里暂且将它定义为未初始化的状态（即使用 `nullptr` 初始化）。它会在暴露给用户的 `num_recognizer_init` 接口函数中初始化。表示 `onnxruntime` 内存信息的 `memory_info` 变量也是同理。

### 11.4.2 手写数字识别类

接下来编写一个手写数字识别类 `Recognizer`，用于封装 `onnxruntime` 会话，CMake目录程序片段如代码清单11.12所示。

**代码清单11.12 ch011/src/num_recognizer.cpp（第29行~第33行）**

```c++
//! @brief 手写数字识别类
struct Recognizer {
    //! @brief onnxruntime会话
    Ort::Session session;
};
```

每一个手写数字识别类都对应一个 `onnxruntime` 会话，每一个会话都可以加载一个 `onnx` 模型。

### 11.4.3 初始化接口实现

初始化接口是实现的第一个接口函数，在实现之前，先来编写一个 `extern "C"` 代码块，用于将其中的函数定义为 C 语言函数，如代码清单11.13所示。

**代码清单11.13 ch011/src/num_recognizer.cpp（第71行~第75行）**

```c++
extern "C" {
void num_recognizer_init() {
    env = Ort::Env{static_cast<const OrtThreadingOptions *>(nullptr)};
    memory_info = Ort::MemoryInfo::CreateCpu(OrtDeviceAllocator, OrtMemTypeCPU);
}
```

初始化接口的实现就是初始化两个与 `onnxruntime` 相关的全局变量：`env` 和 `memory_info`。

### 11.4.4 构造识别器接口实现

构造识别器接口的实现如代码清单11.14所示。

**代码清单11.14 ch011/src/num_recognizer.cpp（第77行~第90行）**

```c++
void num_recognizer_create(const char *model_path,
                           Recognizer **out_recognizer) {
    Ort::Session session{nullptr};
#if _WIN32
    // Windows中，onnxruntime的Session接受模型文件路径时需使用const
    // wchar_t*，即宽字符串。因此在这里做一下转换。
    wchar_t wpath[256];
    MultiByteToWideChar(CP_ACP, MB_PRECOMPOSED, model_path, -1, wpath, 256);
    session = Ort::Session(env, wpath, Ort::SessionOptions(nullptr));
#else
    session = Ort::Session(env, model_path, Ort::SessionOptions(nullptr));
#endif
    *out_recognizer = new Recognizer{std::move(session)};
}
```

这里的主要逻辑就是通过模型文件路径来构造 `onnxruntime` 会话，并将其赋值给在堆上创建的手写数字识别类，将这个类的指针作为结果传给用户。

这里有一处麻烦需要处理：Windows 中 `onnxruntime` 会话构造时接受的模型文件路径的编码不同，需要对传入的参数进行编码转换。为了使用 Windows 的编码转换 API，源文件最开始也做了条件编译以引用 `Windows.h`，如代码清单11.15所示。

**代码清单11.15 ch011/src/num_recognizer.cpp（第11行~第15行）**

```c++
#ifdef _WIN32
// 在Windows操作系统中，我们需要使用Windows API来帮助完成const char*到const
// wchar_t*的编码转换。因此需要引用Windows.h。
#include <Windows.h>
#endif
```

### 11.4.5 析构识别器接口实现

析构很简单，直接 `delete` 即可，如代码清单11.16所示。

**代码清单11.16 ch011/src/num_recognizer.cpp（第92行）**

```c++
void num_recognizer_delete(Recognizer *recognizer) { delete recognizer; }
```

### 11.4.6 识别二值化图片像素数组接口实现

我们使用的神经网络模型 `mnist.onnx` 本⾝就是接受一个 28×28 的 float 型数组作为输入，然后分别输出结果为 0 到 10 的可能性权重，因此只需在实现识别二值化图片像素数组的接口时通过 `onnxruntime` 库运行该模型的推理过程，最后取可能性最大的数值作为预测结果。CMake目录程序片段如代码清单11.17所示。

**代码清单11.17 ch011/src/num_recognizer.cpp（第94行~第114行）**

```c++
int num_recognizer_recognize(Recognizer *recognizer, float *input_image,
                             int *result) {
    std::array<float, 10> results{};
    auto input_tensor = Ort::Value::CreateTensor<float>(
        memory_info, input_image, INPUT_WIDTH * INPUT_HEIGHT,
        input_shape.data(), input_shape.size());
    auto output_tensor = Ort::Value::CreateTensor<float>(
        memory_info, results.data(), results.size(), output_shape.data(),
        output_shape.size());
    recognizer->session.Run(Ort::RunOptions{nullptr}, INPUT_NAMES,
                            &input_tensor, 1, OUTPUT_NAMES, &output_tensor, 1);
    *result = static_cast<int>(std::distance(
        results.begin(), std::max_element(results.begin(), results.end())));
    return 0;
}
```

### 11.4.7 识别 PNG 图片接口实现

识别 PNG 图片的重点，就是如何将 PNG 图片读取并转换为 28×28 的二值化的图片像素数组。首先，借助 `libpng` 第三方库来完成 PNG 图片的读取，如代码清单11.18所示。

**代码清单11.18 ch011/src/num_recognizer.cpp（第116行~第186行）**

```c++
int num_recognizer_recognize_png(Recognizer *recognizer, const char *png_path,
                                 int *result) {
    int ret = 0;
    std::array<float, INPUT_WIDTH * INPUT_HEIGHT> input_image;
    FILE *fp;
    unsigned char header[8];
    png_structp png_ptr;
    png_infop info_ptr;
    png_uint_32 png_width, png_height;
    png_byte color_type;
    png_bytep *png_data;
    // 打开PNG图片文件
    fp = fopen(png_path, "rb");
    if (!fp) {
        ret = -2;
        goto exit3;
    }
    // 读取PNG图片文件头
    fread(header, 1, 8, fp);
    // 验证文件头确实是PNG格式的文件头
    if (png_sig_cmp(reinterpret_cast<unsigned char *>(header), 0, 8)) {
        ret = -3;
        goto exit2;
    }
    // 创建PNG指针数据结构
    png_ptr = png_create_read_struct(PNG_LIBPNG_VER_STRING, nullptr, nullptr,
                                     nullptr);
    if (!png_ptr) {
        ret = -4;
        goto exit2;
    }
    // 创建PNG信息指针数据结构
    info_ptr = png_create_info_struct(png_ptr);
    if (!info_ptr) {
        ret = -5;
        goto exit2;
    }
    // 设置跳转以处理异常
    if (setjmp(png_jmpbuf(png_ptr))) {
        ret = -6;
        goto exit2;
    }
    // 初始化PNG文件
    png_init_io(png_ptr, fp);
    png_set_sig_bytes(png_ptr, 8);
    // 读取PNG信息
    png_read_info(png_ptr, info_ptr);
    png_width = png_get_image_width(png_ptr, info_ptr);   // PNG图片宽度
    png_height = png_get_image_height(png_ptr, info_ptr); // PNG图片高度
    color_type = png_get_color_type(png_ptr, info_ptr); // PNG图片颜色类型
    // 设置跳转以处理异常
    if (setjmp(png_jmpbuf(png_ptr))) {
        ret = -7;
        goto exit2;
    }
    // 读取PNG的数据
    png_data = (png_bytep *)malloc(sizeof(png_bytep) * png_height);
    for (unsigned int y = 0; y < png_height; ++y) {
        png_data[y] = (png_byte *)malloc(png_get_rowbytes(png_ptr, info_ptr));
    }
    png_read_image(png_ptr, png_data);
```

代码执行到这就已经成功将 PNG 图片中每个像素的颜色值读取到了 `png_data` 这个 `png_bytep *` 类型，也就是字节指针的指针类型的变量中，它用于表示一个字节类型的二维数组。

至于这些字节是如何表示图片各个像素的颜色值的，需要根据 PNG 图片采用的颜色类型灵活判断：若图片采用 RGB 颜色类型，那么文件中每三个字节表示一个颜色值，这三个字节分别对应颜色的 RGB 值；若图片采用 RGBA 颜色类型，那么它就需要四个字节表示一个颜色值。为了方便地获取 PNG 图片数据中指定像素的颜色值，并将其二值化，不妨在源文件中创建一些帮助函数，如代码清单11.19所示。

**代码清单11.19 ch011/src/num_recognizer.cpp（第35行~第62行）**

```c++
//! @brief 将byte类型的颜色值转换为模型接受的二值化后的float类型数值
//! @param b byte类型的颜色值
//! @return 模型接受的二值化后的float类型值，0代表白色，1代表黑色。
static float byte2float(png_byte b) { return b < 128 ? 1 : 0; }
//! @brief 获取PNG图片指定像素的二值化后的float类型颜色值
//! @param x 像素横坐标
//! @param y 像素纵坐标
//! @param png_width 图片宽度
//! @param png_height 图片高度
//! @param color_type 图片颜色类型
//! @param png_data 图片数据
//! @return 对应像素的二值化后的float类型颜色值
static float get_float_color_in_png(unsigned int x, unsigned int y,
                                    png_uint_32 png_width,
                                    png_uint_32 png_height, png_byte color_type,
                                    png_bytepp png_data) {
    if (x >= png_width || x < 0)
        return 0;
    if (y >= png_height || y < 0)
        return 0;
    switch (color_type) {
    case PNG_COLOR_TYPE_RGB: {
        auto p = png_data[y] + x * 3;
        return byte2float((p[0] + p[1] + p[2]) / 3);
    } break;
    case PNG_COLOR_TYPE_RGBA: {
```

`get_float_color_in_png` 函数仅支持了较为常见的两种 PNG 图片颜色类型：RGB 和 RGBA。感兴趣的读者可以自行扩充其支持的颜色类型。有了该函数的帮助，再回到刚才代码清单11.18中尚未完成的接口实现，其后续CMake目录程序片段如代码清单11.20所示。

**代码清单11.20 ch011/src/num_recognizer.cpp（第187行~第208行）**

```c++
// 将PNG图片重新采样，缩放到模型接受的输入图片大小
for (unsigned int y = 0; y < INPUT_HEIGHT; ++y) {
    for (unsigned int x = 0; x < INPUT_WIDTH; ++x) {
        float res = 0;
        int n = 0;
        for (unsigned int png_y = y * png_height / INPUT_HEIGHT;
             png_y < (y + 1) * png_height / INPUT_HEIGHT; ++png_y) {
            for (unsigned int png_x = x * png_width / INPUT_WIDTH;
                 png_x < (x + 1) * png_width / INPUT_WIDTH; ++png_x) {
                res += get_float_color_in_png(png_x, png_y, png_width,
                                              png_height, color_type,
                                              png_data);
                ++n;
            }
        }
        input_image[y * INPUT_HEIGHT + x] = res / n;
    }
}
// 识别图片数据中的手写数字
ret = num_recognizer_recognize(recognizer, input_image.data(), result);
```

这里完成了对 PNG 图片的重采样，即缩放图片到 28×28 这个尺寸，并将最终满足输入要求的数据存入 `input_image` 数组。到此，如果未发生错误，程序将通过复用 `num_recognizer_recognize` 接口来完成最终的识别。

除了正常的代码路径，代码中还有一些异常处理的分支，用于分别跳转到不同的标签。这些标签对应不同的退出路径，它们的代码就在函数的末尾做一些资源清理的工作，如代码清单11.21所示。

**代码清单11.21 ch011/src/num_recognizer.cpp（第209行~第222行）**

```c++
exit1:
    // 释放存放PNG图片数据的内存空间
    for (unsigned int y = 0; y < png_height; ++y) {
        free(png_data[y]);
    }
    free(png_data);
exit2:
    // 关闭文件
    fclose(fp);
exit3:
    return ret;
}
```

不同的退出路径需要对应不同程度的清理工作。而如果程序正常退出，那么全部退出路径都会执行到，也就会对全部使用过的资源进行清理释放。

至此，全部接口实现完毕，最后不要忘记用于结束 `extern "C"` 代码块的花括号。

### 11.4.8 完善手写数字识别库的头文件（以同时支持 C 语言）

在进行接口设计时，实际上就是在编写头文件的核心部分——接口函数。不过，只是声明这些函数，并不足以构成一个完善的公开头文件。另外，手写数字库暴露的都是 C 语言接口，这个头文件应当能够同时被 C++ 语言和 C 语言引用。下面一起来看看应当如何完善这个头文件。

首先，头文件引用卫哨必不可少，如代码清单11.22所示。

**代码清单11.22 ch011/include/num_recognizer.h（第1行、第2行）**

```c
#ifndef NUM_RECOGNIZER_H
#define NUM_RECOGNIZER_H
```

其次，要引用导出头文件，这样才能使用 `num_recognizer_EXPORTS` 等宏定义，以便为动态库导出符号，如代码清单11.23所示。

**代码清单11.23 ch011/include/num_recognizer.h（第4行）**

```c
#include "num_recognizer_export.h"
```

再次是声明接口相关的类和函数。由于接口函数都是 C 语言的函数，当采用 C++ 编译器时需要将接口涉及的结构体和函数声明用 `extern "C"` 包括起来。这里借助 `__cplusplus` 宏来判断是否采用 C++ 编译器，如代码清单11.24所示。

**代码清单11.24 ch011/include/num_recognizer.h（第6行~第8行）**

```c
#ifdef __cplusplus
extern "C" {
#endif
```

下面开始声明涉及的类或结构体，如代码清单11.25所示。

**代码清单11.25 ch011/include/num_recognizer.h（第10行~第14行）**

```c
#ifdef num_recognizer_EXPORTS
struct Recognizer;
#else
typedef struct _Recognizer Recognizer;
#endif
```

这里涉及两种情况：该头文件被实现该库的源文件（即 num_recognizer.cpp）引用，以及该头文件被用户的外部程序（包括即将编写的 `recognize` 命令行工具的源文件）引用。`num_recognizer_EXPORTS` 宏就可以用于判断当前是在构建还是使用该库。还记得吗？它是在代码清单11.9所示的目录程序中定义的。

当该宏被定义，也就意味着当前正在构建该库。此时会前向声明 `Recognizer` 结构体，以避免声明接口函数时编译器不认识这个结构体。这个结构体的具体定义会在源文件中给出。

当该宏未被定义，也就是说该库被用户使用时，这里不能仅包含一个前向声明，否则编译器会报告找不到定义的错误。我们需要将 `Recognizer` 结构体定义为一个不透明结构体（opaque strcture），即没有具体定义的结构体，这种结构体仅能出现在指针类型中，正符合接口中 `Recognizer` 类的使用场景。

接下来是在头文件中声明最开始设计的接口函数，如代码清单11.26所示。

**代码清单11.26 ch011/include/num_recognizer.h（第16行~第-47行）**

```c
//! @brief 初始化手写数字识别库
//! @return void
NUM_RECOGNIZER_EXPORT void num_recognizer_init();
//! @brief 创建识别器
//! @param model_path 模型文件路径
//! @param[out] out_recognizer 接受初始化的识别器指针的指针
NUM_RECOGNIZER_EXPORT void num_recognizer_create(const char *model_path,
                                                 Recognizer **out_recognizer);
//! @brief 析构识别器
//! @param recognizer 识别器的指针
NUM_RECOGNIZER_EXPORT void num_recognizer_delete(Recognizer *recognizer);
//! @brief 识别图片数据中的手写数字
//! @param recognizer 识别器的指针
//! @param input_image
//! 模型接受的输入图片数据（28×28的float数值数组，0代表白色，1代表黑色）
//! @param result 接受识别结果的数值的指针
//! @return 错误值，成功返回0
NUM_RECOGNIZER_EXPORT int num_recognizer_recognize(Recognizer *recognizer,
                                                   float *input_image,
                                                   int *result);
//! @brief 识别PNG图片中的手写数字
//! @param recognizer 识别器的指针
//! @param png_path PNG图片文件路径
//! @param result 接受识别结果的数值的指针
//! @return 错误值，成功返回0
NUM_RECOGNIZER_EXPORT int num_recognizer_recognize_png(Recognizer *recognizer,
                                                       const char *png_path,
                                                       int *result);
```

最后，还要记得将前面的 `extern "C"` 代码块，以及头文件引用卫哨的闭合！如代码清单11.27所示。

**代码清单11.27 ch011/include/num_recognizer.h（第48行~第52行）**

```c
#ifdef __cplusplus
} // extern "C"
#endif
#endif // NUM_RECOGNIZER_H
```

现在，这个头文件已经相当完善了。不论用户采用 C 语言还是 C++ 语言，不论是用于构建手写数字识别库，还是分发给用户使用，它都能够胜任。

### 11.4.9 命令行工具的实现

手写数字识别库的代码实现已经完成，下面着手命令行工具的编写。在此之前，我们需要确定命令行工具的调用方式。

命令行接口的设计，也就是命令行的参数设计十分重要。友好的参数设计可以极大地方便用户。本例配套提供的 `recognize` 命令行工具的参数设计十分简单，只需依次接收两个参数：模型文件路径和 PNG 图片文件路径。调用示例如下：

```shell
recognize models/mnist.onnx 2.png
```

为了展现 C 语言接口作为编程界 “通用语言” 的魅力，该命令行工具将采用 C 语言而非 C++ 语言编写，它会调用 C++ 编写的手写数字识别库。这也能验证刚刚编写的手写数字识别库的头文件是否完善。

代码实现相当简单，完整的源文件如代码清单11.28所示。

**代码清单11.28 ch011/cli/recognize.c**

```c
#include <num_recognizer.h>
#include <stdio.h>
//! @brief 主函数
//!
//!   命令行参数应有3个：
//!   1. 命令行程序本⾝的文件名，即recognize；
//!   2. 模型文件路径；
//!   3. 将要识别的PNG图片文件路径。
//!
//!   例如：recognize models/mnist.onnx 2.png
//!
//! @param argc 命令行参数个数
//! @param argv 命令行参数值数组
//! @return 返回码，0表示正常退出
int main(int argc, const char **argv) {
    // 检查命令行参数个数是否为3个
    if (argc != 3) {
        printf("Usage: recognize mnist.onnx 3.png\n");
        return -1; // 返回错误码-1
    }
    int ret = 0;            // 返回码
    int result = -1;        // 识别结果
    Recognizer *recognizer; // 识别器指针
    num_recognizer_init(); // 初始化识别器
    // 使用模型文件创建识别器，argv[1]即模型文件路径
    num_recognizer_create(argv[1], &recognizer);
    // 识别图片文件中的手写数字，argv[2]即图片文件路径
    if (ret = num_recognizer_recognize_png(recognizer, argv[2], &result)) {
        // 返回值非0，识别过程发生错误
        printf("Failed to recognize\n");
        goto exit_main;
    }
    printf("%d\n", result); // 输出识别结果
exit_main:
    num_recognizer_delete(recognizer); // 析构识别器
    return ret;                        // 返回正常退出的返回码0
}
```

## 11.5 构建和运行

代码实现终于告一段落，是不是迫不及待地想要构建并运行它了呢？跟着下面的步骤开始构建吧！

```shell
> cd CMake-Book/src/ch011
> mkdir build
> cd build
> cmake -DCMAKE_BUILD_TYPE=Debug ..
...
> cmake --build . --config Debug
...
```

构建成功后，画一张手写数字的图片，如图11.1所示。

![]()

图11.1 手写数字2的图片

然后，调用构建好的 `recognize` 命令行工具尝试识别这幅图，命令调用方式如下：

```shell
> ./build/recognize ../models/mnist.onnx ../2.png 
2
```

成功识别！

如果使用 MSVC 构建，`recognize` 应该在 `Debug` 子目录中，即 `.\Debug\recognize.exe`。另外，在 Windows 中执行 `recognize.exe` 前，还要记得复制 `zlib.dll`、`libpng16d.dll`（如果采用 `Release` 构建模式，则应复制 `libpng16.dll`）、`onnxruntime.dll` 到 `recognize.exe` 的同一目录中。

## 11.6 小结

本章借助CMake组织项目结构和构建流程，引入了多个第三方库，使用 C 和 C++ 语言实现了一个完整且实用的手写数字识别库项目。相信读者通过本章的实践过程，对CMake的能力有了更加深入的理解，同时也对 C 和 C++ 程序从设计到实现的完整流程有了一定把握。

本书内容已近尾声，但尚未涉及的CMake相关内容其实还有很多。在项目测试、安装、打包发布等流程中，都可以有CMake发光的地方。希望本书能够作为读者学习使用CMake的一个开始，带领读者踏进 C 和 C++ 主流开发实践的大门。同时也衷心祝愿读者在将来的学习和工作中，能够用好CMake这一利器，共同建设更加高效的 C 和 C++ 编程社区！