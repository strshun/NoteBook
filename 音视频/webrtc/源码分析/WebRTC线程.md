## 1. 线程基础知识

## 2. 常见的线程模型

## 3. 源码分析-WebRTC线程的创建



## 4. WebRTC线程模型



## 5. 源码分析- WebRTC线程间切换

## 6. 源码分析 - 从接口层到实现层

## 7. 源码分析 - 信号处理

## 8 理解WebRTC的关键

### 1. 理解WebRTC的关键点是线程

### 2. 有哪些关键线程

### 3. 线程之间是如何协调工作的



### Thread类中的数据

~~~c++
{
    ...
    MessageList messages_;
    PriorityQueue delayed_messages_;
    CriticalSection crit_;
    
    //事件
    SocketServer* const ss_;
    //thread
    HANDLE thread_ = nullptr;
}
~~~

### Thread类中的重要方法

~~~c++
{
    ...
    //队列的获取
    Get(...);
    Peek(...);
    //线程的切换
    Post(...);
    PostTask(...);
    Send(...);
    Invoke(...);
    //线程的控制
    start();
    run();
    stop();
}
~~~

### ThreadManager类

~~~c++
{
    ...
    static ThreadManager* Instance();
    ...
    vector<Thread*> message_queues_;
    CriticalSection crit_;
    ...
    const DWORD key_;
    
}
~~~

### CreateThread(windows)

~~~c++
HANDLE CreateThread(
	LPSECURITY_ATTRIBUTES lpThreadAttributes,
    SIZE_T dwStackSize,
    LPTHREAD_START_ROUTINE lpStartAddress,
    LPVOID lpParameter,
    DWORD dwCreatioFlags, // 0标识创建好立即调度
    LPDWORD lpThreadId
)
eg： CreateThread(nullptr,0,prerun,this,0,&thread_id);
~~~

### pthread_create(类Unix)

~~~c++
int pthread_create(pthread_t *tidp,
                  const pthread_attr_t *attr,
                  void *(*start_rtn)(void*),
                  void *arg);
eg:	pthread_t thread_;
	pthread_arrt_t attr;
	pthread_create(&thread_, &attr, prerun,this);
~~~

### 线程运行的基本逻辑

~~~c++
while(true)
{
    ...
    Get(&msg, ...);
    ...
    Dispatch(&msg);
    ...
}
Dispatch(Message* pmsg){
    ...
    pmsg->handler->OnMessage(pmsg);
}
~~~

### windows下的两种不同事件

#### 1. 普通事件 CreateEvent

#### 2. 异步IO事件 WSACtreatEvent



### webrtc下的事件处理类

#### 1. NullSocketServer（处理无socket的事件）

	1. CreateEvent 创建事件句柄
 	2. WaitForSingleObject 等待事件
 	3. SetEvent 发送事件

~~~C++
HANDLE CreateEvent(
	LPSECURITY_ATTRIBUTES lpEventAttributes,
    BOOL bManualReset, //是否手动复位
    BOOL bInitialState, // 初始状态
    LPCTSTR lpName // 是否是匿名事件
);
DWORD WaitForSingleObject(
	HANDLE hHandle, // CreateEvent 获得
    DWORD dwMilliseconds// 等待时间，INFINITE 一直等待
)
WAIT_OBJECT_0 事件已经收到
WAIT_TIMEOUT 超时
WAIT_FAILED 失败了，可以通过GetLastError获取错误。
~~~



#### 2. PhysicalSocketServer (处理有socket事件)

 	1. WSACreateEvent 创建事件句柄
 	2. WSAWaitForMultipleObjects 等待事件
 	3. WSASetEvent 触发事件
 	4. WSAResetEvent 重置事件

### 3. Socket与事件绑定

1. WSAEventSelect, 将socket与事件绑定

2. WSAEnumNetworkEvents, 枚举发生事件的socket

~~~c++
DWORD WSAAPI WSAWaitForMultipleObjects(
	DWORD cEvents, // events数组个数
    const WSAEVENT *lphEvents, // event数组
    BOOL fWaitAll， // true 当所有等待事件都触发时才触发
    				// false 返回值减WSA_WAIT_EVENT_0
    				// 是触发事件的索引值
    				// 如果有多个事件，则是最小的索引值
    DWORD dwTimeout, // 超时 -1永远等待
    BOOL fAlertable // 通常设置为false
)

int WSAAPI WSAEventSelect(
	SOCKET s, //与事件绑定的Socket
    WSAEVENT hEventObject, // 绑定的事件
    long INetworkEvents //对那些事件感兴趣，读、写、连接等
)
int WSAAPI WSAEnumNetworkEvents(
	SOCKET s, //与事件绑定的Socket
     WSAEVENT hEventObject, // 绑定的事件
    LPWSANETWORKEVENTS lpNetworkEvents // 输出参数，获取读写连接等事件
)
~~~

#### 4. PhysicalSocketServer 中两种事件源

1. socket读写事件
2. 普通事件 WSAEvent（Win)，  pipe（POSIX）



