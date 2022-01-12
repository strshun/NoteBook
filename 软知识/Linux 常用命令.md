## Linux 常用命令



### g++ 编译

> -L  指定依赖库路径
>
> -l (小写L)   指定依赖库名称
>
> -I(大写i )  指定依赖头文件路径
>
> -Wl,-rpath  指定库的搜索路径

~~~shell
g++ main.cpp -L./3rd/lib/ -lppip -los -ljson -lsdk_rtsp -lvss_protocol -liflytekPlatSdk -lprotobuf -liconv -I./3rd/include/ -Wl,-rpath ./3rd/lib
~~~



### ln 建立软连接

~~~ shell
 ln -s /iflytek/shuntao/iPlayer/3rd/lib/libprotobuf.so        /iflytek/shuntao/iPlayer/3rd/lib/libprotobuf.so.15
~~~



### 打包压缩

 **打包及压缩**：tar -czvf  打包并压缩后的文件名.tar.gz 欲打包及压缩的文件夹名

​            例子：tar  -czvf  news.tar.gz ./java 或 tar  -czvf  news.tar.gz java/  或 tar  -czvf  news.tar.gz java

​    **解压及拆包：** tar  -xzvf  打包及压缩后的文件名.tar.gz

​            例子：tar  -xzvf  news.tar.gz



ffmpeg -rtsp_transport tcp - i "" 