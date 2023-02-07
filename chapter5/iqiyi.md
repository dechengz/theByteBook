# 某艺2022年春晚故障分析

2022年的春晚刚开始，使用小米电视、或者某艺投屏的用户观看春晚出现了持续的 卡顿、无法播放现象，服务不正常持续两个小时左右，后面大量的用户跑到微博at小米...。

笔者当时也在观看春晚，通过电视端出现的 502 错误提示，笔者猜测应该是直播源服务出现了故障

后面爱奇艺内部复盘了此次故障，我申请了该篇文章的公开权限，带读者一起分析此次故障的原因，以及后续给工程人员带来的思考和改善。

## 事故回顾

笔者在第一章讲过CDN的一些技术，CDN一个很重要的作用是卸载源站访问压力，比如直播服务，会通过大量的 L2、L1 Cache节点缓解 带宽请求压力。

<div  align="center">
	<img src="/assets/chapter5/iqiyi.png" width = "550"  align=center />
</div>

在春晚19:59 左右，内部的运维人员发现直播 CDN 的回源陡增， 20：10分左右，出现大量的直播pct、pht文件回源请求。请求量特别大，但因为文件较小，没有产生带宽影响

到20：25左右，大量的请求穿透Cache导致爱奇艺内直播源服务器过载，各代理Cache 请求源站 出现 网关 timeout 等 gateway 错误。 运维监控的错误期间开始进行大量的排查修复工作：包括扩容CDN 商业代理、调整回源链路等。到 22:10 左右 直播终于恢复正常。

## 事故的原因

事故的直接原因是某商业CDN没有缓存 PCT文件，这就导致 商业CDN和 直播Cache之间 大量的连接，笔者也在前面章节讲过 连接数的问题，大量的连接 TIME WAIT等问题 会引起 整个服务的不可用

当爱奇艺直播Cache不可用之后，继而影响所有的商业CDN。 

并且在事故发生的前期没有找到直接问题，进行的抢救措施没有起到预想的作用，导致服务宕机出现的时长较长。

## 事故的经验教训

从笔者的角度总结这次事故的教训：首先要对重大的活动有清晰的认识，像春晚直播、双11这类活动 在开始的阶段会有海量的用户瞬时涌入，处理不及时或者没有预备的方案，出现问题会造成严重的负面影响。

对于此类的系统要进行完成地链路检测、以及系统可用性的评测。

比如对各个服务QPS、TPS指标性的建立，当达到指标阈值时进行横向扩展，极端不可用时，启用限流保障部分用户可用。第二个教训是是监控 : 业务性的指标 QPS、TPS、缓存穿透率， 服务器相关的指标监控：出口带宽的承载、内网带宽的承载、CPU load、内存占用等，第三个教训是：故障演练，在团队内部，通过模拟流量，演练观察员通过主动触发个故障，增加技术团队对故障的敏感度以及排查能力。

还有一个做好第三方服务的评测、监控以及备份的方案。

<div  align="center">
	<img src="/assets/chapter5/iqiyi-2.png" width = "500"  align=center />
</div>

故障之后内部团队从服务自我保护、指标监控、故障拆除、连接数限制、商业CDN管理、优化源站层面进行了优化整改。让我们学习这些整改的思路和措施。

**自我保护整改**

主要观点为通过对业务隔离，熔断和限流保障服务的可用性。

- 各个关键节点的并发数预警及限制，避免穿透扩大影响
- 内部 Cache proxy 对 第三方商业 CDN进行分组，防止第三方服务异常，引发全局故障
- 直播、点播的服务分离 使用 K8S等 实现动态伸缩能力

**数据指标检测整改**

建立更完善、更实时的指标检测业务的状态。

- 内部Cache proxy 回源数、资源句柄数等快速发现
- 文件异常增加敏感度：4xx、5xx的监控之外，也关注2xx的指标
- 商业CDN的指标性监控，对于各商业CDN的可用性、调度量
- 直播分层监控：不同层级监控 由不同的专业的人员监控

**故障拆除整改**

调度服务对商业CDN的智能摘除，实现对 商业CDN 实时、精准监控以及摘除能力，增强操作的可用性，完善系统后台的管理能力。

**连接数的限制**

- 找到各级别服务器的瓶颈：比如 1ms的延迟下能达到多少并发
- 后续推进全链路业务压测
- 配置保护
- 日志回顾缓存性能
- 功能拨测常态化，及时发现商业CDN策略变更

**商业CND管理**

- 每场直播比赛后各商业CDN边缘与云端Proxy间各类文件回源收敛性分析，回溯商业CDN的配置问题
- 对CDN供应商的问题需求做到闭环管理落地 
- 监测各商业CDN不对类型文件的带宽、请求回源比变化