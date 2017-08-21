# Brocade Flow Vision
最近 CMCNE 有新版本14.2.1发布，功能大致跟以前无异，注意到flow vision这一部分比较有意思，简单列举一下  
Flow Vision是Brocade上的用来诊断FC SAN 工具，主要有三个功能
- 监测traffic flow
- 复制traffic flow用以分析bottleneck, bandwidth utilization, slow drain 之类
- 主动generate flow用以检测SAN的各种connectivity

### 关于Flow
`Flow`在这里指的是具有相同特征的一系列frame，包含两方面信息：一是port，比如他们都要通过同一个ingress port 或 egress port，二是frame内容信息，比如他都具有相同的source ID，Des ID，LUN …
![flow](http://www.brocade.com/content/html/en/administration-guide/fos-740-flow/GUID-26564051-E3AD-4911-A1E8-9F3FC994D607-output_low.png)

### Flow Vision操作
Flow Vision 监测的对象是flow，一般有下面操作：
1. 通过指定上面所说的这些参数，create flow
2. 指定flow的属性，是monitor IO，是copy流量，还是制造流量
3. Activate flow
4. Show出collected到的data

<br/>

#### `Flow Monitor`
Flow monitor 最为直观，主要是monitor flow上的各种statistics, 包括：  
- Frame statistics(比如 frame count & rate)
- Throughput statistics(byte/sec)
- I/O statistics(I/O count, IOPS)

##### eg：
监测特定端口上SCSI frame 的数量：
```
flow --create scsicsflow -feature monitor -egrport 9 –frametype scsicheckstatus
```
监测端到端（某个initiator - target）的throughput
```
flow --create endtoendflow -feature monitor -ingrport 5 -srcdev 0x010500 -dstdev 0x040900 –bidir
```
监测经过某条路径上的指定LUN的I/O
```
flow –-create lunFlow1 -feature monitor –ingrport 5 -srcdev 0x010502 -dstdev 0x030700 -lun 4
```
<br/>

#### `Flow Generator`
可在Fabric中配置多个sim port，并在sim port之间模拟流量，不需要连接真实的设备。
```
flow –-create flowCase1 –feature generator -ingrPort 1/1 –srcDev 0x040100 –dstDev 0x050200
```
<br/>

#### `Flow Mirror`
把指定flow上的流量mirror一份出来，不影响原来的流量，相当于抓包。
##### eg:
在fc device 0x040500 和 0x010200 之间的 通过port 1/5的双向流量mirror下来 
```
flow --create flow_slowdrain -feature mirror –egrport1/5 –dstdev 0x040500 –srcdev 0x010200 -bidir
```
然后根据scsi的 request/response 情况判断 slow drain device


