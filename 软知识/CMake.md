## 1. 设置cmake最小版本

```
cmake_minimum_required (VERSION 2.8)
```

## 2. 设置项目名称

```
project(demo)
```

## 3. 设置编译目标类型

- `add_executable`：生成可执行文件
- `add_library`：生成库文件

`add_library`默认生成静态库，可以显示指定生成库的类型：

```
#静态库
add_library(test STATIC test.cpp)

#动态库
add_library(test SHARED test.cpp)
```

windows下生成的可执行文件是`*.exe`, 静态库是`*.lib`，动态库是`*.lib`和`*.dll`都有，但是`*.lib`文件很小，只是指向`*.dll`文件，相当于Linux下的软连接。

Linux下的可执行程序没有后缀名，静态库是`lib*.a`文件，动态库是`lib*.so`文件。

## 4. 指定编译包含的源文件

### 1. 明确指明包含的源文件

```
add_executable(demo main.cpp test.cpp util.cpp)
```

### 2. 搜索指定目录的所有的cpp文件

```
aux_source_directory(. SRC_LIST) #搜索当前目录下的所有.cpp文件
add_executable(demo ${SRC_LIST})
```

### 3. 自定义搜索规则

```
file(GLOB SRC_LIST "*.cpp" "*.cc")
add_executable(demo ${SRC_LIST})
GLOB`不支持递归遍历子目录，若想实现递归遍历子目录，请使用`GLOB_RECURSE
```

### 4. 包含多个文件夹里的文件

```
file(GLOB SRC_LIST "*.cpp" "protocol/*.cpp")
add_executable(demo ${SRC_LIST})
# 或者
file(GLOB SRC_LIST "*.cpp")
file(GLOB SRC_PROTOCOL_LIST "protocol/*.cpp")
add_executable(demo ${SRC_LIST} ${SRC_PROTOCOL_LIST})
# 或者
aux_source_directory(. SRC_LIST)
aux_source_directory(protocol SRC_PROTOCOL_LIST)
add_executable(demo ${SRC_LIST} ${SRC_PROTOCOL_LIST})
```

## 5. 设置包含目录

```
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
```

Linux下还可以通过flag的方式设置：

```
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -I${CMAKE_CURRENT_SOURCE_DIR}")
```

## 6. 设置链接库搜索目录

```
link_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/libs64
)
```

Linux下还可以通过flag的方式设置：

```
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -L${CMAKE_CURRENT_SOURCE_DIR}/libs64")
```

## 7. 设置需要链接的库

### 链接库目录搜索

```
target_link_libraries(demo test)
```

该指令会在链接库目录（包括系统默认库目录和指定的自定义库目录）下搜索文件：

- Windows下会搜索`test.lib`文件；
- Linux下会搜索`libtest.a`和`libtest.so`。

当动态库和静态库同时存在时，默认会优先链接动态库，可以在链接时指定动态库或静态库：

```
target_link_libraries(demo test.a)  # 链接静态库
target_link_libraries(demo test.so) # 链接动态库
```

### 指定完整路径

```
target_link_libraries(demo ${CMAKE_CURRENT_SOURCE_DIR}/lib/libtest.a)
target_link_libraries(demo ${CMAKE_CURRENT_SOURCE_DIR}/lib/test.so)
```

### 指定多个链接库

`target_link_libraries`可以一次添加多个链接库：

```
target_link_libraries(demo 
    ${CMAKE_CURRENT_SOURCE_DIR}/libs64/libtest.a 
    pthread)
```

## 8. 设置变量

### 1. set直接设置变量的值

```
set(SRC_LIST main.cpp test.cpp)
add_executable(demo ${SRC_LIST})
```

### 2. set追加变量的值

```
set(SRC_LIST main.cpp)
set(SRC_LIST ${SRC_LIST} test.cpp)
add_executable(demo ${SRC_LIST})
```

### 3. list追加或删除变量的值

```
set(SRC_LIST main.cpp)
list(APPEND SRC_LIST test.cpp)
list(REMOVE_ITEM SRC_LIST main.cpp)
add_executable(demo ${SRC_LIST})
```

## 9. 条件控制

### if...else...elseif...endif

```
if(MSVC)
    set(LINK_LIBS common)
else()
    set(boost_thread boost_log.a boost_system.a)
endif()

target_link_libraries(demo ${LINK_LIBS})


if(${CMAKE_BUILD_TYPE} MATCHES "debug")
    ...
else()
    ...
endif()
```

### while break continue foreach endwhile endforeach

```
while(TRUE)
  message(STATUS "While true")
  break()
endwhile()

foreach(project_file ${COMMON_PROJECT_FILES})
        message(STATUS "project file found -- ${project_file}")
        include("${project_file}")
    endforeach()
```

## 10. 打印消息

```
message(${MY_VAR})
message("build with debug mode")
message(WARNING "this is warnning message")
message(FATAL_ERROR "this build has many error") # 会导致生成失败
```

## 11. 包含其他cmake文件

```
include(./common.cmake) #指定包含文件的全路径
include(def) #在搜索路径中搜索def.cmake文件
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake) #设置include的搜索路径
```

## 12. 多目录

可以使用`add_subdirectory`的方式添加子目录，注意子目录下也需要一个`CMakeLists.txt`文件。

拿 https://www.cnblogs.com/hbccdf/p/introduction_of_cmake.html 中的例子说明，项目目录是

```
./demo
    |
    +--- main.cc
    |
    +--- math/
          |
          +--- MathFunctions.cc
          |
          +--- MathFunctions.h
```

在添加cmake文件后结构如下：

```
./demo
    |
    +--- CMakeLists.txt
    |
    +--- main.cc
    |
    +--- math/
          |
          +--- CMakeLists.txt
          |
          +--- MathFunctions.cc
          |
          +--- MathFunctions.h
```

demo下的CMakeLists.txt文件如下：

```
cmake_minimum_required (VERSION 2.8)
project(demo)
aux_source_directory(. DIR_SRCS)

# 添加math子目录
add_subdirectory(math)

# 指定生成目标
add_executable(demo ${DIR_SRCS})

# 添加链接库
target_link_libraries(demo MathFunctions)
```

math目录下的CMakeLists.txt文件如下：

```
aux_source_directory(. DIR_LIB_SRCS)
# 生成链接库
add_library(MathFunctions ${DIR_LIB_SRCS})
```

在采用这种方式时，注意在写代码时就要规划好模块。

## 13. 常用变量

### 1. 构建类型

| CMAKE_BUILD_TYPE | 对应的c编译选项变量          | 对应的c++编译选项变量          |
| ---------------- | ---------------------------- | ------------------------------ |
| None             | CMAKE_C_FLAGS                | CMAKE_CXX_FLAGS                |
| Debug            | CMAKE_C_FLAGS_DEBUG          | CMAKE_CXX_FLAGS_DEBUG          |
| Release          | CMAKE_C_FLAGS_RELEASE        | CMAKE_CXX_FLAGS_RELEASE        |
| RelWithDebInfo   | CMAKE_C_FLAGS_RELWITHDEBINFO | CMAKE_CXX_FLAGS_RELWITHDEBINFO |
| MinSizeRel       | CMAKE_C_FLAGS_MINSIZEREL     | CMAKE_CXX_FLAGS_MINSIZEREL     |

### 2. 指定编译类型

#### widnows

1. windows中可以在VS中进行选择配置， 默认都会生成四中配置。
2. 要改变默认配置，需在cmake文件中配置：

```
set(CMAKE_CONFIGURATION_TYPES "Debug;RelWithDebInfo")
```

1. 默认生成的平台类型为Win32，如果需要x64类型，则需要按如下命令执行:

```
cmake -G "Visual Studio 14 2015 Win64" ..
```

#### linux

1. Linux下默认会生成none类型的配置；
2. 可通过命令行指定配置：

```
cmake -DCMAKE_BUILD_TYPE=Debug ..
```

1. 也可以在CMake文件中指定配置：

```
set(CMAKE_BUILD_TYPE DEBUG)
```

### 3. 变量

- 自定义变量使用set来定义，如：

```
set(OBJ_NAME, xxxxx)
```

- 使用时使用`${}`，和shell类似，如`${OBJ_NAME}`
- 在if命令里，直接使用变量名即可，不需要加`${}`。

### 4. 内置变量

- CMAKE_BINARY_DIR,PROJECT_BINARY_DIR,_BINARY_DIR：这三个变量内容一致，如果是内部编译，就指的是工程的顶级目录，如果是外部编译，指的就是工程编译发生的目录。
- CMAKE_SOURCE_DIR,PROJECT_SOURCE_DIR,_SOURCE_DIR： 这三个变量内容一致，都指的是工程的顶级目录。
- CMAKE_CURRENT_BINARY_DIR: 外部编译时，指的是target目录，内部编译时，指的是顶级目录
- CMAKE_CURRENT_SOURCE_DIR: CMakeList.txt所在的目录
- CMAKE_CURRENT_LIST_DIR: CMakeList.txt的完整路径
- CMAKE_CURRENT_LIST_LINE: 当前所在的行
- CMAKE_MODULE_PATH: 如果工程复杂，可能需要编写一些cmake模块-，这里通过SET指定这个变量
- LIBRARY_OUTPUT_DIR,BINARY_OUTPUT_DIR: 库和可执行的最终存放目录

### 5. 环境变量

- 使用环境变量：`$ENV{Name}`
- 写入环境变量：`set(ENV{Name} value`) #这里没有“$”符号

## 14. 系统信息

- CMAKE_MAJOR_VERSION，CMAKE 主版本号，比如2.4.6 中的2
- CMAKE_MINOR_VERSION，CMAKE 次版本号，比如2.4.6 中的4
- CMAKE_PATCH_VERSION，CMAKE 补丁等级，比如2.4.6 中的6
- CMAKE_SYSTEM ，系统名称，比如Linux-2.6.22
- CMAKE_SYSTEM_NAME ，不包含版本的系统名，比如Linux
- CMAKE_SYSTEM_VERSION ，系统版本，比如2.6.22
- CMAKE_SYSTEM_PROCESSOR，处理器名称，比如i686
- UNIX ，在所有的类UNIX平台为TRUE，包括OS X 和cygwin
- WIN32 ，在所有的win32 平台为TRUE，包括cygwin

## 15. 开关选项

- `BUILD_SHARED_LIBS`： 用来控制默认的库编译方式，如果 不设置，使用`add_library`在没有指定库类型的情况下，默认生成的都是静态库。如果设置了`set(BUILD_SHARED_LIBS ON)`后，默认生成动态库。
- `CMAKE_C_FLAGS`设置C编译选项，也可以通过`add_definitions()`添加
- `CMAKE_CXX_FLAGS`设置C++编译选项，也可以通过指令`add_definitions()`添加。
- `option`可以添加cmake编译选项。
  如我们想在代码中添加一个宏：

```
．．．

＃ifdef USE_MACRO

．．．

＃endif
```

我们可以通过在项目中的CMakeLists.txt 中添加如下代码控制代码的开启和关闭．

```
option(USE_MACRO
"Build the project using macro"
OFF)

IF(USE_MACRO)

    add_definitions("-DUSE_MACRO")

endif(USE_MACRO)
```

`add_definitions`的作用是添加一个代码中可以使用的宏，而`option`的作用是添加一个cmake可以使用的参数，在构建项目时，你就可以使用如下指令进行控制：

```
cmake 　-DUSE_MACRO＝on ..
cmake 　-DUSE_MACRO＝off ..
```