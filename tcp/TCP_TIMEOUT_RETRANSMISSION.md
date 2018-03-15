##TCP超时重传

> TCP提供可靠数据传输服务，为保证传输正确性，TCP重传其认为已经丢失的包。TCP有两套重传机制，一是基于定时器（超时），二是基于确认信息的构成（快速重传）。

#### 简单的超时重传

![](http://image.littlechao.top/20180315023638000003.jpg)

图中黑色那条就是因为定时器超时仍没有收到ACK，所以引起了发送方超时重传。实际上TCP有两个阈值来决定如何重传同一个报文段：一是愿意重传的次数R1、二是应该放弃当前连接的时机R2。R1和R2的值应分别至少为3次和100秒，如果超过任何一个但还没能重传成功，会放弃该连接。当然这两个值是可以设置的，在不同系统里默认值也不同。

那么如何设定一个合适的超时的值呢？假设TCP工作在静态环境中，这很容易，但真实网络环境不断变化，需要根据当前状态来设定合适的值。

####超时时间 RTO

RTO（retransmission timeout）一般是根据RTT（round trip time）也就是往返时间来设置的。若RTO小于RTT，则会造成很多不必要的重传；若RTO远大于RTT，则会降低整体网络利用率，RTO是保证TCP性能的关键。并且不同连接的RTT不相同，同一个连接不同时间的RTT也不相同，所以RTO的设置一直都是研究热点。

所以凭我们的直觉，RTO应该比RTT稍大：

​									**RTO=RTT+Δt**

那么，RTT怎么算呢：

​							**SRTT=(1−α)×SRTT+α×RTTnew**

SRTT(smooth RTT)，RTTnew是新测量的值。如上，为了防止RTT抖动太大，给了一个权值 **a** ，也叫平滑因子。a的值建议在10%~20%。举个例子，当前 RTTs=200ms，RTTs=200ms，最新一次测量的 RTT=800ms，RTT=800ms，那么更新后的 RTTs=200×0.875+800×0.125=275ms，RTTs=200×0.875+800×0.125=275ms.



**Δt**如何得到呢？RFC 2988 规定：

​								**RTO=SRTT+4×RTTD**

因此，按照上面的定义，**Δt=4×RTTD**. 而 RTTD 计算公式如下：

​							**RTTD=(1−β)×RTTD+β×|SRTT−RTTnew|**

实际上，RTTDRTTD 就是一个均值偏差，它就相当于某个样本到其总体平均值的距离。这就好比你的成绩与你班级平均成绩差了多少。RFC 推荐 β=0.25β=0.25.

##### 退避指数

根据前面的公式，我们可以得到RTO。一旦超过RTO还没收到ACK，就会引起发送方重传。但如果重传后还是没有在RTO时间内收到ACK，这时候会认为是网络拥堵，会引发TCP拥塞控制行为，使RTO翻倍。则第 n 次重传的 RTOnRTOn 值为：

​								**RTOn=2^(n−1)×RTO1**

下图是一个例子：

![图片来源：http://blog.csdn.net/q1007729991/article/details/70196099](http://img.blog.csdn.net/20170422182458581?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcTEwMDc3Mjk5OTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如上，在时间为0.22531时开始第一次重传，此后重传时间开始指数增大，（在尝试了8次后停下，说明其R2的值可能为8）。

