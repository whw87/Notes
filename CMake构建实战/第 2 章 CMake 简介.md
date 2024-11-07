# 第 2 章 CMake 简介

CMake 官网给出了如下的定义：CMake 是一个跨平台开源工具家族，用于构建、测试和打包软件。CMake 通过简单的平台无关且编译器无关的配置文件来控制软件的编译流程，并能够生成原生的 Makefile 和工作空间，以便用于用户所选择的编译环境。为了满足开源项目对强大的跨平台构建工具的需求，Kitware 公司创建了 CMake 工具套装。

定义中，“跨平台” 和 “开源” 这两个特性不必多说，要注意的是“工具家族”这个说法。狭义的 CMake 一般特指用于构建项目的 CMake 工具及其使用的 CMake 脚本语言，而广义的 CMake 指的是一系列工具的组合，包括用于构建的狭义的 `CMake`、用于测试的 `CTest`、用于打包的 `CPack` 等，它们都属于 CMake 家族。本书的主要内容仅涉及狭义的 CMake。

“平台无关且编译器无关的配置文件” 指的是类似 Makefile 的配置文件。但 CMake 的配置文件是平台无关且编译器无关的，能够做到一次编写，到处编译。

“能够生成原生的 Makefile 和工作空间” 是在说 CMake 本身并不实际调用编译器和链接器等，而是根据配置生成 Makefile 或者其他构建工具的配置文件，通过它们来实际调用各种命令完成构建。工作空间指与 IDE 相关的构建工具的配置，如 Visual Studio 中基于 `MSBuild` 的解决方案和项目工程文件，以及 Xcode、CodeBlocks 等其他 IDE 的项目工程文件。可见，将 CMake 称为构建工具不够准确，它更像是一个 “元构建工具”，或者说是 “构建工具的构建工具”。毕竟，它自己不会构建程序，而是指导其他构建工具来构建程序。这样的好处也很明显，就是 “以便用于用户所选择的编译环境”。即便 IDE 不支持打开使用 CMake 组织的项目，但总可以打开通过 CMake 生成的 IDE 原生项目。

简单介绍一下 Kitware 公司。这是一家软件研发公司，专注于计算机视觉、数据分析、科学计算、医学计算、软件开发流程等领域。Kitware 采用开源的商业模型，提供定制解决方案、技术支持和培训等商业服务。针对 CMake，Kitware 公司提供了 “迁移到CMake”、“CMake现场培训”、“CMake技术支持”、“CMake定制开发” 等多项商业服务。希望本书能够帮助读者或企业节约上述业务的开销。

## 2.1 为什么使用 CMake

从 CMake 的定义中，已经多多少少能够看出 CMake 的优势。先来看看其官网的说法。

- CMake是高效的。

  - CMake 帮助开发者花更多的时间在写代码上，花更少的时间在搞定构建系统上。

  - CMake 是开源的，可自由用于任何项目。

- CMake是强大的。

  - CMake 支持在同一个项目中使用多种开发环境和编译器，如 Visual Studio、QtCreator、 JetBrains、Vim、Emacs、GCC、MSVC、Clang、Intel 等。
  - CMake 支持多种编程语言，如 C、C++、CUDA、Fortran、Python 等，还支持在构建过程中执行任意的自定义命令。
  - CMake 支持持续集成，通过 CTest 可在Jenkins、Travis、CircleCI、GitlabCI 等几乎任意持续集成系统中完成测试。测试结果可以在CDash中展示。
  - CMake 支持将第三方库集成到个人项目中。

- CMake是研发团队的首选。

  - CMake几乎已经成为构建 C 和 C++ 项目的业界标准工具。
  - 很多 C++ 项目都开始使用 CMake 构建。根据 2018 Octoverse 报告，CMake 脚本语言在 GitHub 上的增长速度排在第六位。
  - CMake 成熟且经过良好测试，拥有广泛的开发者社区。它从 2000 年开始不断被改进。

CMake 的优势很多，甚至可以说有不少都是独一无二的优势，如生态活跃。这也是为什么即便 CMake 功能复杂、学习成本高，大家仍然选择使用它。另外，CMake 在很多方面都与 C++ 语言很类似：

- 具有庞大的用户群体；
- 稳定地向后兼容；
- 强大的功能特性，支持多范式，较为复杂。

下面从几个方面具体总结一下 CMake 的优势。

### 2.1.1 平台无关和编译器无关

向别人推荐 CMake 时，总会有人问，为什么不用 `Makefile` 或者 `Autoconf` 等工具。

编译器不同，很多命令参数也不同。`Makefile` 和 `Autoconf` 对命令参数的抽象几乎不存在，这就导致一份 `Makefile` 或 `Autoconf` 配置文件常常只能用于命令参数所兼容的编译器。当不仅需要跨编译器，还需要跨操作系统时，这个弊端就更为凸显了。首先，`GCC` 和 `MSVC` 的编译参数差距很大；其次，如果构建配置中还涉及调用其他一些命令，也很难保证这些命令在各个操作系统平台中是通用的。

使用一个平台无关和编译器无关的构建工具对于构建跨平台程序而言至关重要，CMake 当然在此列。除此之外，CMake 还支持交叉编译，可以满足更多样的构建需求。

### 2.1.2 开源自由和优秀的社区生态

若想一个产品被广泛接纳，将其开源作为自由软件是一个很好的办法。谁都不希望支撑着自己成千上万行程序代码的构建工具突然某一天就停止支持了。而开源意味着项目在理论上获得 “永生”，最不济也可以自己接盘维护嘛！另外，CMake 作为自由软件没有对商业用途设限，这进一步促进了来自社区的青睐。

一旦产品被社区广泛采纳，就意味着它的生态建立起来了。当有一个定制需求时，社区中很可能有人已经通过 CMake 实现了它，那么又有什么理由不直接用它呢？更重要的是，很多第三方库已经在使用 CMake 来构建它，若想在自己的程序中使用它们，同样使用 CMake 自然是最方便的。

另外，现在也有很多 C 和 C++ 包管理工具支持 CMake，甚至直接使用 CMake 来管理包的构建和安装流程。如果使用 CMake，这些包管理工具所支持的第三方软件包基本上能够做到一键安装，这又何乐而不为呢？

最后，开源社区的共同维护能够提高软件缺陷的发现和修复速度，提高整个工具的鲁棒性。这对于 CMake 这类构建流程中的核心基础设施尤为重要。

### 2.1.3 强大通用的脚本语言

CMake 的脚本语言经常因为其古板的语法被人诟病，但其功能的强大是毋庸置疑的。它的脚本语言可以对标 Shell 脚本，提供了很多方便操作字符串、文件等的命令。同时，CMake 脚本语言是领域特定语言（Domain Specific Language，`DSL`），即专注于某个应用程序领域的计算机语言。对于 CMake 来说，它所专注的便是构建这个领域，自然也提供了很多用于构建过程的命令，如下载并构建外部的项目、生成配置文件等。

CMake 脚本语言既然是能够对标Shell脚本的图灵完备的语言，当然不仅限于描述构建过程。也可以用于编写测试脚本，甚至编写持续集成的整个流程。

第 3 章会详细讲解 CMake 脚本语言的具体用法，但不涉及任何构建过程。另外，本书的第一个实践项目将使用 CMake 脚本语言实现快速排序算法，这充分体现了 CMake 脚本语言的通用性。

### 2.1.4 稳定地向后兼容

CMake 相当重视向后兼容。毕竟，使用 C 和 C++ 构建的项目往往偏底层，会存在并被使用很长时间。经过漫长的时间，可能早已无人更新维护它了。因此，必须保证数年前的代码在今天依然能够被正常构建。在这一点上，CMake 与 C 和 C++ 很相似。

CMake 提出了策略机制以保证稳定的向后兼容。一方面，保证原有配置的可用性；另一方面，为过时的配置提出警告和建议。这极大地方便了对古老的代码库遗产的维护和升级。

### 2.1.5 持续不断地改进和推出新特性

得益于 CMake 的策略机制，CMake 可以在提供相当稳定的兼容性的前提下不断更新。 CMake 也确实在持续不断地改进中。

实际上，CMake 曾有很多不便之处，CMake 3.0 版本的推出改善了这一缺陷。因此人们也常把此后版本的 CMake 称为 “现代CMake”。CMake 2 时代采用过程式脚本程序描述构建过程，封装性较差，很难厘清项目结构。而现代 CMake 为开发者提供了面向构建目标的构建配置，很多构建要求或使用要求通过绑定在构建目标上的属性来实现。这一改变为项目各组件的解耦提供了极大的便利。

截至本书写作之时，CMake 3 已经相继推出了二十多个版本，每个版本都伴随相当多的问题修正和功能更新等，“新陈代谢” 速度极快。本书将以 CMake 3.20 版本为例，介绍现代 CMake 与构建相关的方方面面。对于其早期版本，这里不做介绍。毕竟，CMake 2 是早于 C++11 的产物。

## 2.2 安装CMake

安装 CMake 非常容易。访问其官方网站的下载页面，即可找到针对各个平台的安装文件的下载链接。

### 2.2.1 在Windows中安装CMake

如无特别说明，本书的讲解默认使用 64 位操作系统。

针对 Windows x64 平台，有两个安装文件可供下载 ：

- Installer（安装器），运行后如图2.1所示；

![]()

图2.1 Windows中的CMake安装器

- Zip压缩包，打开后如图2.2所示。

![]()

图2.2 Windows中的CMake安装压缩包

安装器可以双击运行，按照说明一步步完成安装过程；压缩包则需要手动解压到某一目录，同时相关的系统配置也需要人工完成（如设置 `PATH` 系统环境变量等）。读者可以根据实际需要选择。

### 2.2.2 在 Linux 中安装 CMake

针对 `Linux x86_x64` 平台，也有两个安装文件可供下载：

- sh 自解压脚本；
- tar.gz 压缩包，用户可以自行解压到任意目录并配置环境变量等。

这里仅演示自解压脚本的安装过程：

```shell
$ ./cmake-3.20.0-Linux-x86_64.sh
CMake Installer Version: 3.20.0, Copyright (c) KitwareThis is a self-extracting archive.
The archive will be extracted to: ~/
...（此处省略了操作说明和许可协议等文本）
Do you accept the license? [yn]: y
By default the CMake will be installed in:
  "~/cmake-3.20.0-Linux-x86_64"
Do you want to include the subdirectory cmake-3.20.0-Linux-x86_64?
Saying no will install in: "~" [Yn]: y
...
```

中文部分不属于终端输出内容。

直接执行自解压脚本会默认将 CMake 文件解压到当前目录中。如果需要定制安装过程，可以通过调用自解压脚本命令的 `--help` 参数查看可配置的选项 ：

```shell
$ ./cmake-3.20.0-Linux-x86_64.sh  --help
Usage: ./cmake-3.20.0-Linux-x86_64.sh [options]Options: [defaults in brackets after descriptions]
  --help            print this message
                    打印帮助
  --version         print cmake installer version
                    打印安装脚本的版本号
  --prefix=dir      directory in which to install
                    设定安装目录
  --include-subdir  include the cmake-3.20.0-Linux-x86_64 subdirectory
                    解压到cmake-3.20.0-Linux-x86_64⼦目录
  --exclude-subdir  exclude the cmake-3.20.0-Linux-x86_64 subdirectory
                    不要解压到上述⼦目录
  --skip-license    accept license
                    默认接受许可协议
```

另外，也可以使用 Linux 发行版自带的包管理器安装 CMake，这里不再赘述。

### 2.2.3 在 macOS 中安装 CMake

针对 macOS，也有两个安装文件可供下载：

- dmg 安装包，这是常规的 macOS 安装方式，即拖动 CMake 应用程序到系统应用程序目录中；
- tar.gz 压缩包，这与 Linux 中的压缩包类似。

在这里仅介绍第一种安装方法。拖动 CMake 应用程序到应用程序目录后，就可以在启动台中找到CMake应用程序图标了。通过启动台启动 CMake 实际上打开的是 CMake GUI，即 CMake 的可视化界面。如果想在命令行中能够调用 cmake 等命令，可以在 "Tools" 菜单中找到 "How to Install For Command Line Use" 选项，如图2.3所示。

![]()

图2.3 macOS中CMake的How to Install For Command Line Use菜单项

选择该菜单项，会弹出一个对话框，提示如何安装CMake命令行工具。一般来说，只需执行下面这段命令：

```shell
sudo "/Applications/CMake.app/Contents/bin/cmake-gui" --install
```

该命令将会在 `usr/local/bin` 中生成一系列指向 CMake 命令行工具等的符号链接。

另外，除了从官网下载安装包外，还可以使用 `Homebrew` 包管理器在 macOS 中安装 CMake：

```shell
brew install cmake
```

## 2.3 您好，CMake!

终于来到了 CMake 版本的 “您好，世界” 程序，如代码清单2.1所示。

**代码清单2.1 ch002/您好CMake.cmake**

```cmake
message(您好，CMake！)
```

就像 C 语言中的 printf 命令一样，使用 CMake 中的 message 命令即可输出一行文本。该 CMake 脚本程序的运行过程如下：

```shell
> cd CMake-Book/src/ch002
> cmake -P 您好CMake.cmake
您好，CMake！
```

本书对不同平台下的提示符和输入指令中的目录分隔符采用不同的方式展示，对于 Linux 的 Shell，分别以 `$` 和 `/` 展示；对于 Windows 的PowerShell，分别以 `>` 和 `\` 展示；对于不强调平台的跨平台操作，分别以 `>` 和 `/` 展示。

CMake 默认是用于构建任务的，如果想让它像脚本语言一样执行需要指定 `-P` 参数。另外，运行 CMake 脚本程序时不再区分不同的操作系统，毕竟 CMake 是跨平台的嘛！需要注意的是，对于跨平台的命令行中的目录分隔符，统一采用 `/`。在Windows操作系统中，推荐使用 PowerShell 运行上述命令。命令提示符（cmd）当然也可以，只不过它只支持 `\` 作为目录分隔符 。

至此，我们成功运行了第一个 CMake 程序。在后续章节中，我们会继续领略 CMake 的强大本领。