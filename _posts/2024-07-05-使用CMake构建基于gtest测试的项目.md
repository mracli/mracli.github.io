---
title: GTest单元测试
description: 剪短的描述
author: momo
categories:
- 笔记
- CMake
tags:
- 笔记
- GTest
- CMake
layout: post
pin: false
math: true
date: 2024-07-05 15:13 +0800
---
## 使用GoogleTest

常用的方法：
- 从源码中构建 `Googletest` 并加入 `/usr/local`, 这样后续所有本机用户都可以直接使用GTest:
- 直接将 GoogleTest 嵌入到自己的项目中，并编写相关代码进行适配

## 准备工作
### 从源码中构建

不知道如何使用CMake 构建可以参考：[CMake如何安装项目](https://mracli.github.io/posts/cmake%E4%BD%BF%E7%94%A8%E8%AE%B0%E5%BD%95/)

我们需要拿到 `Googletest` 源码，然后使用CMake进行编译，最后将其安装到默认或我们自己指定的位置。

然后在我们的项目中就可以通过 find_package() 指令来找到 Googletest 了：

### 远程获取

使用 CMake 中的 ContentFetch()，不需要手动获取项目并构建，即可自行从远程获取并执行对应的构建编译，具体:

```cmake
cmake_minimum_required(VERSION 3.14)
project(my_project)

# GoogleTest requires at least C++14
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
# FetchContent 要求 CMake 版本不低于 3.14
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/03597a01ee50ed33e9dfd640b249b4be3799d395.zip
)
# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
```

## 在项目中使用 GTest

```cmake
# 顶层 CMakeLists.txt
cmake_minimum_required(VERSION 3.10)
project(myproject)

# GoogleTest requires at least C++14
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

enable_testing()
# 添加测试源文件目录
add_subdirectory(${CMAKE_SOURCE_DIR}/test)

# test/CMakeLists.txt
# 从 CMake Modules 中搜索我们已经安装好的 Googletest
find_package(GTest REQUIRED)
# 后面就可以使用 GTest 进行我们自己设定的单元测试
add_executable(hello_test hello_test.cc)
target_link_libraries(hello_test GTest::gtest_main)

add_executable(yet_another_test yet_another_test.cc)
target_link_libraries(yet_another_test GTest::gtest_main)

find_package(GTest REQUIRED)

add_executable(hello_test hello_test.cc)
target_link_libraries(hello_test GTest::gtest_main)

add_executable(yet_another_test yet_another_test.cc)
target_link_libraries(yet_another_test GTest::gtest_main)

# 两种加入 test 的方式
# 使用 CMake 的 add_test() 命令
add_test(NAME "SimpleTest" COMMAND hello_test)
add_test(NAME "SimpleTest2" COMMAND yet_another_test)

# 使用 GoogleTest 的 gtest_discover_tests() 命令
include(GoogleTest)
gtest_discover_tests(hello_test)
gtest_discover_tests(yet_another_test)
```

链接 `GTest::GTest_main` , 该库提供主函数，我们自己不需要额外定义。

```cpp
// test/hello_test.cc
#include <gtest/gtest.h>

// Demonstrate some basic assertions.
TEST(HelloTest, BasicAssertions) {
  // Expect two strings not to be equal.
  EXPECT_STRNE("hello", "world");
  // Expect equality.
  EXPECT_EQ(7 * 6, 42);
}

TEST(HelloTest, AnotherBasicAssertions) {
  EXPECT_NE(0, 1);
}

// test/yet_another_test.cc
#include <gtest/gtest.h>

// Demonstrate some basic assertions.
TEST(YetAnotherTest, SomeBasicAssertions) {
  // Expect two integers to be equal.
  EXPECT_EQ(2, 2);

  // Expect a string to contain a substring.
  EXPECT_STRCASEEQ("world", "WORld");
  
  // Expect a floating-point value to be within a certain range.
  EXPECT_NEAR(3.14159, 3.141, 0.001);
}

// 因为链接到的是 GTest::GTest_main, 所以不需要 main 函数
```

项目结构：

```shell
.
├── CMakeLists.txt
└── test
    ├── CMakeLists.txt
    ├── hello_test.cc
    └── yet_another_test.cc
```

尝试构建项目并进行测试：

```shell
> mkdir build && cd build
> cmake ..
> make 
> make test
```
结果：
```shell
Running tests...
Test project /home/momo/fun_stuff/learning/googletest/build
    Start 1: HelloTest.BasicAssertions
1/5 Test #1: HelloTest.BasicAssertions ............   Passed    0.00 sec
    Start 2: HelloTest.AnotherBasicAssertions
2/5 Test #2: HelloTest.AnotherBasicAssertions .....   Passed    0.00 sec
    Start 3: YetAnotherTest.SomeBasicAssertions
3/5 Test #3: YetAnotherTest.SomeBasicAssertions ...   Passed    0.00 sec
    Start 4: SimpleTest
4/5 Test #4: SimpleTest ...........................   Passed    0.00 sec
    Start 5: SimpleTest2
5/5 Test #5: SimpleTest2 ..........................   Passed    0.00 sec

100% tests passed, 0 tests failed out of 5

Total Test time (real) =   0.02 sec
```

使用两种方式都可正常执行单元测试，但使用 gtest_discover_tests() 命令给出的结果更细致，具体到每一个 `TEST()` 项. 

结果日志也都在 `build/Testing/Temporary/LastTest.log` 文件中，可以查看详细的测试结果。

## 参考链接

1. [GoogleTest Primer](https://google.github.io/googletest/primer.html)
