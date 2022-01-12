## linux 下的终端调试



###  启用调试

1. cmake 模式

   > //  首先 启用cmake下的调试编译模式
   >
   > SET(CMAKE_BUILD_TYPE "Debug")

2. gcc 模式

   > // 编译时候启用调试模式 ,需要在gcc编译过程中加上`-g`选项
   >
   > gcc gdb.c -g -o gdb 