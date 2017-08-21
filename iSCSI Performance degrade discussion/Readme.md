一个关于`iSCSI Performance`的问题，记录如下讨论：

#### 问题
> 在Unity CNA 卡（iSCSI offload）上跑large sequential write IO 的时候发现performance有下降

#### 描述
> 当Cisco 5020 switch上多个port接受来自host的connection到Unity CNA的时候，产生congestion  
> 可以抓到unity CNA发给cisco switch的pause帧  
> 但在cisco switch连接host的port并没有抓到pause, 并且可以看到在ingress port处产生了dropped packets  
> 因过多的re-transmission导致了performance下降  
> 而换作是Juniper switch, 没有明显的performance降低的问题  

当host端连接多个 initiator 到 switch，并且开启多个线程 run IO, 而 storage 端却只有一个 port 连接到 iSCSI array 的情况非常多见，这就带来了congestion 的可能。传统 Ethernet 上面，如果交换机或者 array CNA 支持802.3x的话，那么会开启 standard pause 功能。如下图中 host 不断同时发起 large sequential write IO, storage 是可以感受到 congestion 的，于是 CNA 会向 switch 发 standard pause, 让 switch 停止向它发数据, 这些 pause 帧也被捕获。


![topo](iSCSI_pete.png)

#### 分析
首先，应该明确iSCSI 是要跑在传统ethernet上还是跑在DCB上。

从原理上分两种情况来讨论：

A）  
如果确定了iSCSI跑在传统以太网上，那么就要利用也只能利用[802.3x](https://en.wikipedia.org/wiki/Ethernet_flow_control)来做二层的流控，所以Host CNA, 中途的交换机，以及array的port/slic都应该打开802.3x (这是个老协议，这些设备都应该支持的)。另外，上面提到的:
```
“ 但是在cisco switch 连接host initiator的地方并没有抓到pause ”
```

这个值得再考虑原因，因为在传统以太网中，是没有协议来保证congestion control主动向上传递的，也就是说即使array扛不住了，发了pause到交换机，交换机虽然不会再发向array了，但是还是会继续收包，直到缓存被装满，才会往initiator发pause. 这两个pause是异步的，可能会间隔一段时间。

B）  
如果确定iSCSI跑在DCB上，那将是完全另外一套工作方式。交换机上需要设置DCB mapping，主要是三个方面：
1. 指定iSCSI流量的优先级  
2. 该优先级所匹配的服务等级（CoS）  
3. 以及为该等级指定带宽百分比。

这几点配置的理论依据来自于BDC框架下的[802.1Qbb](https://en.wikipedia.org/wiki/Ethernet_flow_control)(PFC)和 [802.1Qaz](https://docs.microsoft.com/en-us/windows-hardware/drivers/network/enhanced-transmission-selection--ets--algorithm)(ETS), 利用[DCBX](https://en.wikipedia.org/wiki/Link_Layer_Discovery_Protocol#Data_Center_Bridging_Capabilities_Exchange_Protocol), 通过TLV获取到与交换机相连的end设备的信息，并根据mapping, 将QoS同步到end设备上，以实现统一的QoS. 另外需要host CNA, array port/SLIC 都打开DCB. 然后在这样的环境中，流控是PFC来做的，而且做的是针对iSCSI所在的服务等级上的`精细流控`。

实验环境中，默认打开了PFC, 而针对 cos=3 的traffic进行pause并没有apply到其他的类型的traffic. 换句话说，iSCSI traffic并不会被识别为cos=3, 当然也就不会被map到所在的class, 因此PFC无法作用于iSCSI, DCB的流控失败。 同时，因为打开了PFC, 传统Ethernet下的pause，也没起作用，当ingress cache被装满，switch也并没有发standard pause给host, 而是直接将大量的数据帧**drop掉**，影响了性能。

解决的办法是:
- 打开802.3x的同时，要先关掉PFC，让全程有且仅有传统以太网流控，并且确认所有设备都能打开802.3x. 或者：
- 让全程跑在DCB上，为iSCSI配置相应的mapping, 让PFC做流控。同样需要事先确认所有设备（host CNA, switch, array port/SLIC）都能enable DCB并正常工作

