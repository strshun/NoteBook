## Epoll

### 使用Epoll的好处

1. 没有文件描述符的限制
2. 工作效率不会随着文件描述符的增加而下降
3. Epoll经过系统优化更高效

### Epoll 事件的触发模式

1. Level Trigger 没有处理反复发送（水平触发）
2. Edge Trigger 只发送一次（边缘触发）

### Epoll 重要的API

1. int epoll_create() 参数无意义，可忽略
2. int epoll_ctl(epfd, op, fd, struct epoll_event *event)
3. int epoll_wait(epfd, events, maxevents, timeout)

### Epoll的事件

1. EPOLLET 设置边缘模式
2. EPOLLIN
3. EPOLLOUT
4. EPOLLPRI
5. EPOLLERR
6. EPOLLHUP

### epoll_ctl的相关操作

1. EPOLL_CTL_ADD
2. EPOLL_CTL_MOD
3. EPOLL_CTL_DEL

### Epoll重要结构体

~~~c++
typedef union epoll_data{
	void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;

struct epoll_event{
    uint32_t events; /*Epoll events */
    epoll_data_t data; /* user data variable */
}
~~~

