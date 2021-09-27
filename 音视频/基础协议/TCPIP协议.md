## 1. TCP/ IP 协议栈

![image-20210828163346420](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828163346420.png)

## 2. IP 协议头



![image-20210828160924239](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828160924239.png)

Version 版本号 	IHL (ip head length)   ip协议头长度 	Type of Service  		Total Length  总数据长度

拆包使用  Identification 拆包后唯一标识     flags 是否允许拆包（可以获取最大传输单元MTU）      fragment offset  拆包的偏移量

Time to Live  数据存活时间，现在常用作路由的跳数，最大跳跃次数，  protocol 数据协议类型    Header CheckSum 校验和，

source Address  		源地址 （ip)

destination address     	目的地址(ip)

Options  （可选）    padding 字节对齐



### MTU   Maximum Transmission Unit

* 指在网上可传输的最大尺寸

* 可以通过ICMP 查询最大传输单元， 它通过设置IP层的 Flags DF(Dont Fragment)不分片位，如果将这一比特置1 ，IP层将不对数据包进行分片

* 获取MTU的好处是传输过程中不拆包，提高传输效率，以太网的MTU默认是1500B





## 3. TCP协议头



![image-20210828164759676](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828164759676.png)

Source port			Destination port

Sequence Number  包序号，按字节增加，比如当前包序号1000，这个包数据长度1000 ，下个包的序号就是2001  **保证有序性**

Acknowledgment number  确认消息，确定某个数据包收到						**保证可靠性**

Data Offset 数据偏移量，因为协议头长度不固定，所以可以更具这个字段直接找到数据的偏移量

Reserved 保留位

URG			ACK    PSH    RST	SYN	FIN 

window  滑动窗口

CheckSum  校验和   urgent Point 紧急指针

Options  （可选）    padding  32位字节对齐

![52im_net_x1](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/52im_net_x1.png)



### Seq Number含义

* 在TCP 中数据不是按**包**排序，而是按**字节**排，每个包的Seq Number 代表的是发送字节的起始序号

* 发送的第一个包的初始序号是随机的，在创建连接的三次握手过程中交换

### Ack Number含义

* 在TCP中，Ack Number 代表的是希望对方发送数据的起始位置。

例如，A 向 B 发送一个数据包，其中Seq Number 为1000， 其大小为1000字节，则B收到该包后，返回的Ack Number为2001



## 4. TCP三次握手

![image-20210828171140923](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828171140923.png)



## 5. TCP四次挥手

![image-20210828171731545](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828171731545.png)



## 6. TCP ACK 机制

![image-20210828173954508](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828173954508.png)



![image-20210828174049708](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828174049708.png)







![image-20210828174300760](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828174300760.png)

![image-20210828174312770](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828174312770.png)



## 7. TCP 滑动窗口

![image-20210828174635518](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828174635518.png)

![image-20210828174901719](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828174901719.png)

![image-20210828175101248](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/image-20210828175101248.png)