---
title: miniVim 问题合集
date: 2024-10-29 21:01:33
categories:
  - [C++]
  - [助教工作]
tags:
  - C++
  - 助教工作
---

{% note::实际上，这篇文档同时起到测试本网站是否能访问交大图床的功能。 %}

<!-- more -->

> 如果你的问题没有出现在这篇文档中，请到 **[John班论坛](https://john.sjtu.app/t/topic/258)** 指定帖子进行提问
> 声明：**不允许**线下对同学代码直接修改来解决问题

## Update List
24.10.19 更新 demo(minivim) 无法打开的情况
24.10.19 更新 cmake无法找到的情况
24.10.27 更新关于如何debug的笼统方法
24.10.29 更新关于如何配置Cmakelist的方法


## Issue1: 下发的minivim程序无法打开

**下面的情况均建立在已经完成了cmake和ncurses库的安装的基础上**

### 情况1

![](https://notes.sjtu.edu.cn/uploads/upload_3849da5874e553a79276fc7653a466ed.png)

此时是可执行文件权限不够，应当在命令行中输入： ```chmod 777 minivim```

然后再执行命令 ```./minivim words_alpha.txt```

Note: minivim需改成可执行文件的名字，并且要在可执行文件的同级目录下执行这些指令

### 情况2

![](https://notes.sjtu.edu.cn/uploads/upload_05d94a55412cf89f15d206417b45c9b5.png)

可能是gcc版本太低，请使用汪助教下发的第二个版本，或在链接中下载 mock_gcc11。

### 情况3

对于其他更罕见的情况，请使用本文提供的下载链接中的 mock_static 文件。

**注：下载完记得覆盖原来的minivim文件。**

### 下载链接：[demo(minivim) -SJTU云盘](https://jbox.sjtu.edu.cn/l/c1ZJpm)

## Issue2: 无法找到cmake

**关于Cmakelist的提问前，请务必充分阅读github page中的README！**

**特别是Project Layout部分**

### clion编辑器

如果使用clion进行编程，可能在编译过程中发现以下问题：

![](https://notes.sjtu.edu.cn/uploads/upload_972adfe7f117545ed0d163ca9a6853b5.png)

如果出现了同样的错误，说明你的clion并没有正确识别到你的cmake的位置，请你进行如下处理：

打开相应设置，确认无法找到CMake位置：

![](https://notes.sjtu.edu.cn/uploads/upload_5c4af99318dc1efc09bb13dc7fe650bd.png)

打开wsl，输入以下指令：

![](https://notes.sjtu.edu.cn/uploads/upload_3a8e4b08d0d6aca6242e75024f7bd4b0.png)

命令的结果就是cmake环境位置所在，用这个输出修改CMake位置。

### vscode编辑器 (实际上是命令行)

找到环境位置同上

然后输入命令：

```nano ~/.bashrc```

移动到文件的末尾，输入：

类似指令：```export PATH="<CMake_path>:$PATH"```

其中```<CMake_path>```用上面得到的结果替代，在我的例子中，应该改成：

```bash
export PATH="/usr/local/bin/cmake:$PATH"
```

保存并关闭 .bashrc 文件。

具体按键为 Ctrl + X，然后按 Y 确认保存更改，最后按 Enter 键。

通过命令 `source ~/.bashrc` 进行重新加载。

最终输入命令 `cmake --version` 检查是否能够检测到。



## Advice3: 利用 fstream 库进行DEBUG

该方法实际上和**输出调试法本质相同**，但是考虑到这一次大作业我们的 stdscr 被 ncurses 库占用了，所以将输出信息输入到文本中或许是一种不错的方法。

我们需要在cpp程序开头 `#include <fstream>`

这只是我们提供的一个简单DEBUG的思路，如果需要这个方法的具体方法和实现可以自己去询问gpt或者找对应的网页进行学习。

下图是一种写法（不一定具有普适性）：

![](https://notes.sjtu.edu.cn/uploads/upload_2723bec743e3746a3acb8e875c06062b.png)

**CR的时候需要删除、注释掉这些和输出错误相关的命令和文件（图中即out.txt），不然酌情扣分**

## Issue4: Cmakelist 配置相关问题

### CMakelist 里面究竟写了什么

我们在此不介绍CMakelists的原理，本次作业的目的是让大家学会如何使用。

我们虽然不需要了解具体原理，但是我们还是要对CMakelists内部的部分行在干什么有一个大致的了解。

下面是我们下发的样例CMakelists:

``` cmake
# specify the minimum version of CMake that is supported
cmake_minimum_required(VERSION 3.10)

# include a project name
project(minivim)

# set C++ Version & executable output path
set(CMAKE_CXX_STANDARD 17)
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

# make your compiler aware of header directory
include_directories(${PROJECT_SOURCE_DIR}/include)

# find & include ncurses library
find_package(Curses REQUIRED)
include_directories(${CURSES_INCLUDE_DIR})

# create a variable which includes needed source files
set(MY_SOURCES
    ${PROJECT_SOURCE_DIR}/src/ncurses_demo.cpp
)

# specify an executable, 
# build from the specified source files
add_executable(minivim ${MY_SOURCES})

# link ncurses library with your executable
target_link_libraries(minivim ${CURSES_LIBRARY})
```

我们逐步分析：
```cmake
# specify the minimum version of CMake that is supported
cmake_minimum_required(VERSION 3.10)

# include a project name
project(minivim)

# set C++ Version & executable output path
set(CMAKE_CXX_STANDARD 17)
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
```
这一部分的主要功能为：
* 确定了最低要求的cmake的版本（3.10）
* 给出了项目(project)的名字，在我们的样例文件里面叫做minivim
* 指定可执行文件的位置，在源代码目录下的`bin`目录下，这里我们需要指出的是：`${PROJECT_SOURCE_DIR}$` 是一个预定义的变量，它指向包含项目顶级 CMakeLists.txt 文件的目录。

**简而言之，这一段你其实不用更改，你只需要知道你在不更改的情况下项目的名字叫做minivim，所在位置在源代码目录下的 `bin` 目录下即可**


然后我们来看下面这段：
```cmake
# make your compiler aware of header directory
include_directories(${PROJECT_SOURCE_DIR}/include)

# find & include ncurses library
find_package(Curses REQUIRED)
include_directories(${CURSES_INCLUDE_DIR})

```
这一段的主要功能是：
* 将项目源代码目录下的`include`子目录添加到编译器的头文件搜索路径中
* 在系统中搜索 `nucurses` 库并且添加到头文件搜索目录中

所以我们希望：
* 所有的头文件均存放在 `include` 子目录下

这样的好处是：

在main.cpp（类似名字）中，如果要引用**自己写的头文件**，可以直接这么写：`#include "myheader.h"`

而不需要使用相对路径：`#include "../include/my_header.h"`

需要注意的是，对于 `ncurses` 库，要这样引入：`#include <ncurses.h>`，因为这个库不是你写的。

我们再来看最后一部分：
```cmake
# create a variable which includes needed source files
set(MY_SOURCES
    ${PROJECT_SOURCE_DIR}/src/ncurses_demo.cpp
)

# specify an executable, 
# build from the specified source files
add_executable(minivim ${MY_SOURCES})

# link ncurses library with your executable
target_link_libraries(minivim ${CURSES_LIBRARY})
```

这部分的作用如下：
* 我们定义了一个变量，`${MY_SOURCES}$`，每次当我们新建了一个会被使用到的.cpp文件时，我们应当把这个文件的位置放入`{MY_SOURCES}`中，例如：

    ```cmake
    set(MY_SOURCES
        ${PROJECT_SOURCE_DIR}/src/ncurses_demo.cpp
        ${PROJECT_SOURCE_DIR}/src/my_header.cpp
    )
    ```

    我们希望：
    * 所有.cpp的文件（包括.h(pp)的视线文件都放在源文件的`src`目录下）
* 通过和各种文件的链接形成可执行文件`minivim`（这部分大家目前不需要了解怎么做到的）

值得注意的是，如果你希望可执行文件名字不叫 `minivim`，那么请在上述CMakelists中把所有minivim的位置全部改成相对应的可执行文件的名字。

### 遇到的情况

#### 默认情况0：

如果你还没有动手实践，可以按照这个步骤走：

1. 到Cmakelist所在目录下：
2. 建立build文件夹（可以通过命令行输入`mkdir build`）
3. 到build目录下（可以通过命令行输入`cd build`）
4. 在命令行输入 `cmake ..`，结果如下则说明成功
    ![](https://notes.sjtu.edu.cn/uploads/upload_b7aa2555fb9cb27f9d55d52a7b0a2ac1.png)
5. 回到原来的目录（可以通过命令行输入`cd ..`）
6. 命令行输入 `cmake --build build --target minivim`（最后一个minivim要换成可执行文件的名字，如果你在CMakelist中进行了更改了）出现如下结果则意味着成功：
    ![](https://notes.sjtu.edu.cn/uploads/upload_b015c061f55c68e8e3a3625371ed149a.png)

在以后没有修改CMakelists的情况下，只用输入步骤6中的指令即可。

如果修改了CMakelists，那么请从步骤3全部重新做一遍。

**有空可以拷打gpt以上的命令是什么意思，相信你会对cmake和命令行指令有更多的理解**

#### 情况1：

![](https://notes.sjtu.edu.cn/uploads/upload_7097227a5893c7a189c6279b1640fc51.png)

错误特征： `undefined reference to 'main'`

说明你的 `main.cpp` 里面没有main函数。

保存后重新编译即可通过。

#### 情况2：

![](https://notes.sjtu.edu.cn/uploads/upload_27811f48ced6319adb1463b4804f5e9b.png)

错误特征： `Cannot find source file:`

检查你的CMakelist里面的 `set (MY_SOURCES ...)`部分

目录下是不是出现了实际上不存在的（.cpp）文件

在本样例中，CMakelist出现了不存在的 hello.cpp 和 world.cpp，解决方法是把这两行删掉：

![](https://notes.sjtu.edu.cn/uploads/upload_cde67fb89f76bd280178edfcd82bd4da.png)

修改这部分即可通过：
```cmake
set(MY_SOURCES
        ${PROJECT_SOURCE_DIR}/src/main.cpp
    )
```

#### 情况3：

![](https://notes.sjtu.edu.cn/uploads/upload_9ef49fa490a8efcd1276c61b6e3ef8a8.png)

说明没有到相对应的目录下。

通过命令行访问对应目录，例如：

```bash
cd /mnt/c/Users/wwx/Desktop/miniVim
```

**注：第二个参数要改成自己电脑中的对应文件在的位置**

#### 情况4：

如果在修改文件之后重新编译程序没有预期效果，且编译结果类似下图：

![](https://notes.sjtu.edu.cn/uploads/upload_e6be579f6793dad24c51c1b55eeebd32.png)

**重点：没有出现 [50%] 的情况。**

请确认有没有手动保存，即 `Crtl+S`

**不要相信自动保存**

---

~~看样子是SJTU图床是可行的~~