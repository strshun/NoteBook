#### DLL调用
dll调用有两种方式。分为显示调用和隐式调用。
##### 隐式调用
动态链接库的隐式调用依赖.lib和.dll两个文件。lib文件存储导出符号和相关定位代码，通过它找到DLL中执行函数的地址。
隐式调用时将lib所在路径设置到附加库目录，同时在附加库中添加库名。之后再工程中包含.h文件即可像正常调用函数一样使用dll。
其在程序加载时即导入了dll,导致程序运行时内存占用增加。但使用起来方便。
##### 显式调用
仅有dll，无lib文件的库，可以进行显式调用。这种方式的前提是已经知道了dll中所提供的函数名称，及其参数列表，返回值。使用示例如下：
````c++

typedef uint32_t(*pfun)(uint8* pu8Para，uint32_t u32Length);  
  
int main()  
{  
  
    HINSTANCE hDLL;  
    pfun fun;  
    hDLL=LoadLibrary(TEXT("fundll.dll"));  //加载DLL文件  
    if(hDLL == NULL)
    {
        //error
        return -1;
    }
    fun=(pfun)GetProcAddress(hDLL,"fun");  //取DLL中的函数地址，fun为dll中提供函数的真实名称 
    uint32_t u32Ret = fun(NULL, 0); //函数调用
    FreeLibrary(hDLL);   //卸载动态库
    
    return u32Ret;
}
```
显式调用在执行中随时可以加载和卸载DLL文件，更为灵活，更省内存。