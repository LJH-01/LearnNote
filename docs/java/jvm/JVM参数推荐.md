[TOC]

# JVM调优

## JVM推荐配置



| 参数                                                         | 备注/参数释义                                                | 规范                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| JVM版本                                                      | 1.8.0_60以下，无法使用Pfinder  1.8.0_131之前，jvm无法感知docker的真实核数和内存大小，而是使用的宿主机的核数和内存 | 必须1.8.0_60以上  推荐1.8.0_191以上                          |
| JVM GC方法                                                   | ParallelGC：1.8默认，高吞吐量，响应时间不敏感  CMS：响应优先，堆内存8G以下优先选择 G1：响应优先，堆内存8G及以上选择 | C端应用：8G及以上选择G1，8G以下选择CMS  B端应用：推荐使用ParallelGC |
| Xmx                                                          | 堆的最大值                                                   | 必须配置  小于8G的，不超过50% 8G的，最多可以设置6G 12G的，最多设置为8G 16G的，最多设置为12G 32G的，最多设置为24G |
| Xms                                                          | 初始堆的大小，也是堆大小的最小值                             | 必须配置 与Xmx一致                                           |
| MaxDirectMemorySize                                          | 堆外内存大小                                                 | 一般无需配置  使用了OHC等堆外缓存的需要配置，配置时需与架构师评审并压测 |
| ParallelGCThreads                                            | 并行GC时的线程数（ParallelGC、CMS、G1均适用）  此值过小，则stw时间变长，此值过大，影响吞吐量，CPU过高 | 必须配置 =容器核数                                           |
| ConcGCThreads                                                | 并发标记时的线程数  并发标记时并没有stw，CPU密集型任务（CMS、G1才有并发标记步骤） | 限CMS、G1必须配置  ParallelGCThreads的20%~50% 一般为ParallelGCThreads/4或ParallelGCThreads/2 |
| CICompilerCount                                              | JIT进行热点编译的线程数 CPU密集型任务                        | 必须配置(值要大于2)  推荐值如下： 1C容器 : 2 <br />2C容器：2 <br />4C容器 : 2~4 <br />8C容器：2~4 <br />16C容器 : 4~12 |
| MetaspaceSize MaxMetaspaceSize                               | 元空间初始大小、元空间最大大小  如果未指定初始大小，默认是20m，应用启动时如果不够就会gc来扩容 元空间并不在虚拟机中，而是使用本机内存，因此受本机内存限制 | jdk1.8适用，必须配置  需要大于256M                           |
| Xmn NewRatio                                                 | Xmn：新生代内存大小  NewRatio：老年代与新生代与内存容量的比例 x:1 这2个参数只需设置其中1个即可，若都设置了，以Xmn为准 | CMS必须设置，只需配置其中1个参数；G1不要设置<br />Xmn=堆内存Xmx的 1/3 <br />NewRatio=2 |
| UseCMSInitiatingOccupancyOnly  CMSInitiatingOccupancyFraction=x | CMSInitiatingOccupancyFraction为堆内存占用率达到百分比时开始GC的阈值  默认不设置时，由JVM自动计算垃圾回收的周期 | CMS必须设置  <br />推荐值70~80                               |
| UseCMSCompactAtFullCollection  CMSFullGCsBeforeCompaction=x  | CMSFullGCsBeforeCompaction为配置fullGC时，进行了多少次fullGC之后对老年代进行压缩整理处理碎片 | CMS专用，不强制 推荐值1                                      |
| HeapDumpOnOutOfMemoryError  HeapDumpPath=/export/Logs/       | 首次遭遇OOM时导出此时堆中相关信息  路径带“/”则为目录，否则为文件 | 必须配置  文件目录需要为已存在的目录，若配置为具体文件，其所属目录也需要为已存在的目录 |
| InitiatingHeapOccupancyPercent                               | 混合回收触发时机是由参数`InitiatingHeapOccupancyPercent`控制，默认值为45，含义是老年代占用空间大小与堆的总大小比值超过此数便会触发混合回收。 | 默认值为45，G1使用                                           |
| G1HeapRegionSize                                             | G1默认将堆内存分成2048块，如果容器堆内存不大，则每个Region也会比较小，当应用创建对象大于Region一半大小时，G1认为是巨型对象，会直接在老年代创建，因此需要根据容器情况，配置G1HeapRegionSize | 设置为8m，G1使用                                             |
| ParallelRefProcEnabled                                       | -XX:+ParallelRefProcEnabled  如果GC时Reference处理时间较长，例如大量使用WeakReference对象，可以通过此参数开启并行处理 | 例如大量使用WeakReference对象，可以通过此参数开启并行处理，CMS以及G1都适用 |



## JVM配置示例

以下为应用容器规格分布和JVM参考配置：

| 序号 | 容器规格  |                         JVM配置样例                          |
| :--: | :-------: | :----------------------------------------------------------: |
|  1   | **8C16G** | **使用G1：** export JAVA_OPTS="-Djava.library.path=/usr/local/lib -server -Xms8192m -Xmx8192m -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m -XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:ParallelGCThreads=8 -XX:ConcGCThreads=4 -XX:CICompilerCount=4 -XX:G1HeapRegionSize=8m -XX:MaxGCPauseMillis=150  -XX:+ParallelRefProcEnabled -Djava.awt.headless=true -Dsun.net.client.defaultConnectTimeout=60000 -Dsun.net.client.defaultReadTimeout=60000 -Djmagick.systemclassloader=no -Dnetworkaddress.cache.ttl=300 -Dsun.net.inetaddr.ttl=300 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/export/Logs/" |

## 如何判断GC是否正常

1）GC是否频繁：YoungGC频率一般几十秒钟一次，FullGC一般每天几次，注意G1回收器不应该出现FullGC；

2）GC耗时：耗时主要取决于堆内存大小及垃圾对象数量。YoungGC时间通常应在几十毫秒，FullGC通常在几百毫秒；

3）每次GC内存是否下降：应用刚启动时，每次YoungGC内存应该回收到较低水位，随着时间推移老年代逐步增多，内存水位会逐步上涨，直到FullGC/MixedGC（G1），内存会再次回到较低水位，否则可能存在内存泄漏；

4）如果使用ParallelGC，堆内存耗尽才会触发FullGC，所以不用配置堆内存使用率告警，但需关注GC频率；

