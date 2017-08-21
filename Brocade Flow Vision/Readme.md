# Brocade Flow Vision
最近 CMCNE 有新版本14.2.1发布，功能大致跟以前无异，注意到flow vision这一部分比较有意思，简单列举一下  
Flow Vision是Brocade上的用来诊断FC SAN 工具，主要有三个功能
- 监测traffic flow
- 复制traffic flow用以分析bottleneck, bandwidth utilization, slow drain 之类
- 主动generate flow用以检测SAN的各种connectivity

### 关于flow
`Flow`在这里指的是具有相同特征的一系列frame，包含两方面信息：一是port，比如他们都要通过同一个ingress port 或 egress port，二是frame内容信息，比如他都具有相同的source ID，Des ID，LUN …
![flow](http://www.brocade.com/content/html/en/administration-guide/fos-740-flow/GUID-26564051-E3AD-4911-A1E8-9F3FC994D607-output_low.png)

### Flow Vision操作
Flow Vision 监测的对象是flow，一般有下面操作：
1. 通过指定上面所说的这些参数，create flow
2. 指定flow的属性，是monitor IO，是copy流量，还是制造流量
3. Activate flow
4. Show出collected到的data
