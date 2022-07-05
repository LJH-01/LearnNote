# TCP连接请求过程(含keepalive)

## 使用短连接的方式验证


### 三次握手
![9249fca280b7ffad008c586102afd2af.png](../_resources/9249fca280b7ffad008c586102afd2af.png)
* * *

### 数据传输

本次请求：
![89f67236f4029f8eabd6db60c6875a88.png](../_resources/89f67236f4029f8eabd6db60c6875a88.png)

![4d078cb29d84d63230d65c1005185dde.png](../_resources/4d078cb29d84d63230d65c1005185dde.png)

服务端：
***ack=seq+len***

客户端
***seq=seq+len***
* * *

### 四次挥手

![e415367725e61c1aed3a2f866fc067b5.png](../_resources/e415367725e61c1aed3a2f866fc067b5.png)

### keepalive保活机制

如图：下次请求的seq应该为3910+232=4142，
但是keepalive保活包的seq=应该的seq-1=4141，如下图所示，
总之保活报文不在窗口控制范围内
![475ecc685a73c41c5cc387b48ad8b50d.png](../_resources/475ecc685a73c41c5cc387b48ad8b50d.png)

![3a2257fb8fbbd6ba90295f94112b32b3.png](../_resources/3a2257fb8fbbd6ba90295f94112b32b3.png)