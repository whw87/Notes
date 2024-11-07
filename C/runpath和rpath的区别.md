# runpath和rpath的区别

## 背景

需有简单的 Linux 编程知识，了解动态库是什么。了解 `LD_LIBRARY_PATH` 的作用。

## RPATH 是什么？

什么是运行时（run-time）？运行时就是程序运行的时候。我们知道，在程序运行的时候，会依赖一些动态库，只有所依赖的库文件在运行的机器上存在，才能运行程序。问题是如何找到这些库？这些库可能在不同的目录中，每个人的操作系统中的目录结构可能都不一样。运行时搜索路径即提供了一些路径，程序运行的时候去这些路径下搜索程序所依赖的库。这些路径被称为 `rpath`（run-time search path，即运行时搜索路径）。

## 运行时动态库搜索路径有哪些？

一般情况下，我们可以知道的路径有如下几个：

1. 环境变量 `LD_LIBRARY_PATH` 指定的目录列表。
2.  缓存文件，通常包含 `/etc/ld.so.conf` 文件编译出的二进制列表（比如 CentOS 上，该文件会使用 `include` 指令从而使用 `ld.so.conf.d` 目录下面所有的 `*.conf` 文件，这些都会缓存在 `ld.so.cache` 中）
3. 系统的默认路径。如 `/lib`，`/usr/lib` 等。

是否还可以设置别的路径呢？当然可以，而且可以在编译的时候就可以指定。

什么情况下需要在编译时就指定呢？一种场景是：我们在自己的设备上编译好一个程序及其所有依赖的库，如果另一个人（假设和我们的系统相同）想使用我们的程序，直接拷贝我们的程序到对方电脑上就可以运行。这个时候，如果我们依赖的库都是我们自己编写的，那么除非我们显式指定这些 `.so` 的路径（或者拷贝到系统的默认路径中），否则就无法运行。是否可以让用户直接拷贝程序到自己系统上，而不用修改别的内容呢？这个时候就可以通过在编译时指定 `rpath` 路径来实现。这种程序一般称为便携式软件（portable software）。

## RPATH 的设置

`RPATH` 有两个比较相近的名称：`rpath` 和 `runpath`。这两个往往容易让人搞混。在最初的时候，`ELF` 文件只有一个 `DT_RPATH` 标签用来表示 `rpath` 列表，后来 `ELF` 规范弃用了 `DT_RPATH`，并引入了一个新的标签 — `DT_RUNPATH` 用于 `rpath` 列表。这两个标签的区别是它们相对于 `LD_LIBRARY_PATH` 环境变量的相对优先级。当链接器在程序运行时搜索动态库时，`DT_RPATH` 有较高的优先级，`DT_RUNPATH` 有较低的优先级。即搜索顺序如下：

1. `DT_RPATH` 指定的目录列表。
2. 环境变量 `LD_LIBRARY_PATH` 指定的目录列表。
3. `DT_RUNPATH` 指定的目录列表
4. `/etc/ld.so.cache` 缓存文件，通常包含 `/etc/ld.so.conf `文件编译出的二进制列表（比如 CentOS 上，该文件会使用 `include` 指令从而使用 `ld.so.conf.d` 目录下面所有的 `*.conf` 文件，这些都会缓存在 `ld.so.cache` 中）
5. 系统的默认路径。如 `/lib`，`/usr/lib` 等。

## 查看 RPATH

对于任意的 `ELF`文件（如可执行程序和 `.so`），可以使用 `readelf -d xxx | grep PATH` 来查看，如果有 `RUNPATH` 或者 `RPATH`，则表示 `ELF` 文件中设置了 `RPATH` 路径。如
```shell
readelf -d libtest.so
0x000000000000001d (RUNPATH) Library runpath: [$ORIGIN:/home/test]
```

## 如何设置 RPATH

`gcc` 编译器有 `-Wl,-rpath` 选项可以设置 `RPATH`，如下所示：

```shell
gcc -Wl,-rpath,dir1 test.c
```

如想设置多个目录，则每个目录之间用冒号 `:` 分割即可，如下所示：

```shell
gcc -Wl,-rpath,dir1:dir2:...:dirN test.c
```

### 例子

头文件 liba.h 如下：

```c
#ifndef __LIBA_FUNC__
#define __LIBA_FUNC__
int liba_func(int a, int b);
#endif
```

头文件 libb.h 如下：

```c
#ifndef __LIBB_FUNC__
#define __LIBB_FUNC__
int libb_func(int a, int b);
#endif
```

源文件 liba.c 如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include "libb.h"

int liba_func(int a, int b)
{
    int sum = 0;

    sum = a + b;
    printf("%d + %d = %d.\n", a, b, sum);

    a = a * a;
    b = b * 2;
    sum = a * b;
    printf("sum = %d.\n", sum);

    libb_func(6, 6);

    return 0;
}
```

源文件 libb.c 如下：

```c
#include <stdio.h>
#include <stdlib.h>

int libb_func(int a, int b)
{
    printf("this is libb_func, sum is %d.\n", a + b);

    return 0;
}
```

源文件 main.c 如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include "liba.h"

int main()
{
    int main_a = 3;
    int main_b = 5;
    int main_sum = 0;

    main_a = main_a * main_a;
    main_b = main_b * 2;
    main_sum = main_a * main_b;
    printf("main_sum = %d.\n", main_sum);
    liba_func(1, 2);
    return 0;
}
```

```shell
[zy@fedora rpath]$ gcc --version
gcc (GCC) 13.2.1 20231205 (Red Hat 13.2.1-6)
Copyright © 2023 Free Software Foundation, Inc.
[zy@fedora rpath]$ tree .
.
├── liba.c
├── liba.h
├── libb.c
├── libb.h
└── main.c
[zy@fedora rpath]$ gcc main.c -L. -I. -la -g -Wl,-rpath,/home/zy/rpath:/home/zy -o test
[zy@fedora rpath]$ readelf -d test |grep -i path
 0x000000000000001d (RUNPATH)            Library runpath: [/home/zy/rpath:/home/zy]
```

可以看到，使用  `gcc` 编译后，生成的二进制文件中添加了 `RUNPATH`。上文说到其实还有一个是 `RPATH`，为什么传给链接器的是 `-rpath` 选项，而编译出来的却是 `RUNPATH` 呢？这个和链接器有关系，编译的时候添加了 `-rpath` 选项后，链接器默认使用 `runpath`，而不是 `rpath`。编译的时候，可以添加 `--disable-new-dtags` 选项给链接器，表示使用 `rpath`。如下所示：

```typescript
[zy@fedora rpath]$  gcc main.c -L. -I. -la -g -Wl,-rpath,/home/zy/rpath:/home/zy -Wl,--disable-new-dtags -o test
[zy@fedora rpath]$ readelf -d test |grep -i path
 0x000000000000000f (RPATH)              Library rpath: [/home/zy/rpath:/home/zy]
```

如果有些链接器默认使用 `rpath`，而不是 `runpath`，想强制使用 `runpath` 的话，可以添加 `--enable-new-dtags` 选项。如下所示：

```typescript
[zy@fedora rpath]$  gcc main.c -L. -I. -la -g -Wl,-rpath,/home/zy/rpath:/home/zy -Wl,--enable-new-dtags -o test
[zy@fedora rpath]$ readelf -d test |grep -i path
 0x000000000000001d (RUNPATH)            Library runpath: [/home/zy/rpath:/home/zy]
```

> [!NOTE]
>
> 摘自 GNU LD 文档
>
> --enable-new-dtags
>
>  --disable-new-dtags
> 	This linker can create the new dynamic tags in ELF. But the older ELF systems may not understand them. If you specify
> 	--enable-new-dtags, the new dynamic tags will be created as needed and older dynamic tags will be omitted.  If you
> 	specify --disable-new-dtags, no new dynamic tags will be created. By default, the new dynamic tags are not created.
> 	Note that those options are only available for ELF systems.



## $ORIGIN 问题

使用 `rpath` 或者 `runpath` 指定一个路径后，会存在一个问题。比如我们编译的时候设定了一个 `/home/test` 路径，但我们将程序打包给其他人用的时候，其他人的环境不一定将包放到这个目录，那么依然会报找不到库。为了解决这个问题，编译器提供了一个特殊的目录，`$ORIGIN`，它在动态链接时表示文件所在的当前路径。这是一个非常有用的功能，尤其是当我们需要创建一个可移植的应用程序或库时。需要注意的是，`$ORIGIN` 需要加引号，如果不加引号，会当成普通的字符串。编译时传递的 `flags`  为 `-Wl,-rpath,'$ORIGIN'` 或 `'-Wl,-rpath,$ORIGIN'`，如下所示：

```elixir
[zy@fedora rpath]$  gcc main.c -L. -I. -la -g -Wl,-rpath,'$ORIGIN' -o test
[zy@fedora rpath]$ readelf -d test |grep -i path
 0x000000000000001d (RUNPATH)            Library runpath: [$ORIGIN]
[zy@fedora rpath]$  gcc main.c -L. -I. -la -g '-Wl,-rpath,$ORIGIN' -o test
[zy@fedora rpath]$ readelf -d test |grep -i path
 0x000000000000001d (RUNPATH)            Library runpath: [$ORIGIN]
```

## rpath 和 runpath 其它方面的不同

上面说了 rpath 和 runpath 在加载程序时，搜索路径的优先级是不同的，除了这个区别外，还有以下区别：

1. 如果同时有 `rpath` 和 `runpath`，那么 `rpath `是失效的，只有 `runpath` 有效。
2. `rpath` 和 `runpath` 对间接依赖库的作用。

在搜索程序或库的间接依赖时，`rpath` 和 `runpath` 是不同的，`rpath` 设置的路径对间接库的搜索也生效，即搜索间接库时，也会优先从 `rpath` 指定的路径中搜索。而 `runpath` 设置的路径在对间接库搜索时是不起作用的。所谓的间接库是指一个库所依赖的库中又依赖的别的库，如 test 程序依赖 liba.so，而 liba.so又依赖 libb.so，那么对于 test 程序来说，libb.so 就是一个间接依赖库。那么加载器在加载 test 程序时，寻找 libb.so 的时候，`rpath` 和 `runpath` 的作用是不同的，如下所示:

```crystal
[zy@fedora rpath]$  gcc main.c -L. -I. -la -g -Wl,-rpath,'$ORIGIN' -o test
[zy@fedora rpath]$ 
[zy@fedora rpath]$ ldd test 
 linux-vdso.so.1 (0x00007fff3b538000)
 liba.so => /home/zy/rpath/./liba.so (0x00007f09a9203000)
 libc.so.6 => /lib64/libc.so.6 (0x00007f09a900a000)
 /lib64/ld-linux-x86-64.so.2 (0x00007f09a920a000)
 libb.so => not found
[zy@fedora rpath]$ readelf -d test |grep -i path
 0x000000000000001d (RUNPATH)            Library runpath: [$ORIGIN]
[zy@fedora rpath]$  gcc main.c -L. -I. -la -g -Wl,-rpath,'$ORIGIN' -Wl,--disable-new-dtags -o test
[zy@fedora rpath]$ readelf -d test |grep -i path
 0x000000000000000f (RPATH)              Library rpath: [$ORIGIN]
[zy@fedora rpath]$ ldd test 
 linux-vdso.so.1 (0x00007fffd1272000)
 liba.so => /home/zy/rpath/./liba.so (0x00007f6fcac3b000)
 libc.so.6 => /lib64/libc.so.6 (0x00007f6fcaa42000)
 libb.so => /home/zy/rpath/./libb.so (0x00007f6fcaa3d000)
 /lib64/ld-linux-x86-64.so.2 (0x00007f6fcac42000)
```

从上面可以看出，当使用 `runpath` 时，libb.so 是无法找到的。
查看 `rpath`

```shell
readelf -d xxx
```

可以看到类似这一行:

```shell
0x000000000000001d (RUNPATH) Library runpath: [$ORIGIN:/home/test]
```

## 该用哪一个？

如果必须要使用 `rpath` 或 `runpath`，建议还是使用 `runpath`。因为最开始是只有 `rpath` 的，为什么后来又增加了 `runpath` 呢？而且 `runpath` 和 `rpath` 同时存在的时候，只有 `runpath` 生效。因为只有 `rpath` 的情况下，一旦设置了 `rpath`，那么在运行时，其优先级是最高的，且我们无法通过其它手段（如通过设置 `LD_LIBRARY_PATH` 等）覆盖默认的库路径，我们必须重新编译程序才能加载其它路径下的库，这对于某些情况下是很不方便的。而使用 `runpath` 的时候，我们可以很方便的通过 `LD_LIBRARY_PATH` 变量去覆盖默认的路径。

## 注意事项

`runpath` 和 `rpath` 的行为，和 Linux 发行版中链接器的实现有关，不同的发行版，可能实现并不相同。

## 参考资料



