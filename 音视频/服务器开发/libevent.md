## libevent

### libevent 重要函数

1. event_base_new
2. event_base_dispatch
3. evconnlisttener_new_bind
4. event_new
5. event_add
6. event_del
7. event_free

### evconnlisttener_new_bind

~~~c++
typedef void (*evconnlistener_cb)(struct evconnlistener *, 
                                 evutil_socket_t,
                                 struct sockaddr *
                                 int socklen,
                                 void *);
struct evconnlistener* evconnlistener_new_bind(struct event_base *base,
                                              evconnlistener_cb cb,
                                              void *ptr,
                                              unsigned flags,
                                              int backlog,
                                              const struct sockaddr *sa,
                                              int socklen);
~~~



