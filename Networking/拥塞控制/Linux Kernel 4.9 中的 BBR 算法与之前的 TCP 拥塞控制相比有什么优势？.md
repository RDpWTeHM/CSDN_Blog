# Linux Kernel 4.9 中的 BBR 算法与之前的 TCP 拥塞控制相比有什么优势？

@[TOC]( )


> 转载自: http://zoufeng.net/2017/06/22/google-bbr/ **及补充/更新**

本文经[原作者](https://www.zhihu.com/people/li-bo-jie)授权转载。

来源[知乎](https://www.zhihu.com/question/53559433)

以下为原文



------
## [李博杰](https://www.zhihu.com/people/li-bo-jie) 的回答

> 中国科技大学微软亚洲研究院 博士在读
> 2,621 人赞同了该回答 (截至至 2018/12/17 转载) 

  中国科大 LUG 的[@高一凡](https://www.zhihu.com/people/f2efaaed2de4757d0ed478d50eee749a)在 LUG HTTP 代理服务器上部署了 Linux 4.9 的 TCP BBR 拥塞控制算法。从科大的移动出口到新加坡 DigitalOcean 的实测下载速度**从 647 KB/s 提高到了 22.1 MB/s**（截屏如下）。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-2218943eab7a6b70ad0f4a213cb1f393_b-1.jpg" />
</center>

（应评论区各位 dalao 要求，补充测试环境说明：是在新加坡的服务器上设置了 BBR，**新加坡的服务器是数据的发送方**。这个服务器是访问墙外资源的 HTTP 代理。科大移动出口到 DigitalOcean 之间不是 dedicated 的专线，是走的公网，科大移动出口这边是 1 Gbps 无限速（但是要跟其他人 share），DigitalOcean 实测是限速 200 Mbps。RTT 是 66 ms。实测结果这么好，也是因为大多数人用的是 TCP Cubic (Linux) / Compound TCP (Windows)，**在有一定丢包率的情况下，TCP BBR 更加激进，抢占了更多的公网带宽**。因此也是有些不道德的感觉。）

此次 Google 提交到 Linux 主线并发表在 ACM queue 期刊上的 TCP BBR 拥塞控制算法，继承了 Google “先在生产环境部署，再开源和发论文” 的研究传统。TCP BBR 已经在 Youtube 服务器和 Google 跨数据中心的内部广域网（B4）上部署。

**TCP BBR 致力于解决两个问题：**

1. **在有一定丢包率的网络链路上充分利用带宽。**
2. **降低网络链路上的 buffer 占用率，从而降低延迟。**

TCP 拥塞控制的目标是最大化利用网络上瓶颈链路的带宽。一条网络链路就像一条水管，要想用满这条水管，最好的办法就是给这根水管灌满水，也就是：
**水管内的水的数量 = 水管的容积 = 水管粗细 × 水管长度**
换成网络的名词，也就是：
**网络内尚未被确认收到的数据包数量 = 网络链路上能容纳的数据包数量 = 链路带宽 × 往返延迟**

TCP 维护一个**发送窗口**，估计当前网络链路上能容纳的数据包数量，希望在有数据可发的情况下，回来一个确认包就发出一个数据包，总是保持发送窗口那么多个包在网络中流动。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-c81d606af6f0cf75154687352af2de62_b-1.png" />
<p align="middle">TCP 与水管的类比示意（图片来源：Van Jacobson，Congestion Avoidance and Control，1988）</p> 
</center>

如何估计水管的容积呢？一种大家都能想到的方法是不断往里灌水，直到溢出来为止。标准 TCP 中的拥塞控制算法也类似：**不断增加发送窗口，直到发现开始丢包**。这就是所谓的 ”加性增，乘性减”，也就是当收到一个确认消息的时候慢慢增加发送窗口，当确认一个包丢掉的时候较快地减小发送窗口。

标准 TCP 的这种做法有两个问题：

首先，假定网络中的丢包都是由于拥塞导致（网络设备的缓冲区放不下了，只好丢掉一些数据包）。事实上网络中有可能存在传输错误导致的丢包，基于丢包的拥塞控制算法并不能区分**拥塞丢包**和**错误丢包**。在数据中心内部，错误丢包率在十万分之一（1e-5）的量级；在广域网上，错误丢包率一般要高得多。

更重要的是，“加性增，乘性减” 的拥塞控制算法要能正常工作，**错误丢包率需要与发送窗口的平方成反比**。数据中心内的延迟一般是 10-100 微秒，带宽 10-40 Gbps，乘起来得到稳定的发送窗口为 12.5 KB 到 500 KB。而广域网上的带宽可能是 100 Mbps，延迟 100 毫秒，乘起来得到稳定的发送窗口为 10 MB。广域网上的发送窗口比数据中心网络高 1-2 个数量级，错误丢包率就需要低 2-4 个数量级才能正常工作。因此标准 TCP 在有一定错误丢包率的**长肥管道（long-fat pipe，即延迟高、带宽大的链路）**上只会收敛到一个很小的发送窗口。这就是很多时候客户端和服务器都有很大带宽，运营商核心网络也没占满，但下载速度很慢，甚至下载到一半就没速度了的一个原因。

其次，网络中会有一些 *buffer*，就像输液管里中间膨大的部分，用于吸收网络中的流量波动。由于标准 TCP 是通过 “灌满水管” 的方式来估算发送窗口的，在连接的开始阶段，*buffer* 会被倾向于占满。后续 *buffer* 的占用会逐渐减少，但是并不会完全消失。客户端估计的水管容积（发送窗口大小）总是略大于水管中除去膨大部分的容积。这个问题被称为 **bufferbloat（缓冲区膨胀）**。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-47da56cba0cf246e65605ae986274d8b_b-1.png">

<p align="middle">缓冲区膨胀现象图示</p>
</center>

缓冲区膨胀有两个危害：

1. **增加网络延迟**。*buffer* 里面的东西越多，要等的时间就越长嘛。
2. 共享网络瓶颈的连接较多时，可能导致**缓冲区被填满而丢包**。很多人把这种丢包认为是发生了网络拥塞，实则不然。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-c623d2f88cc5f3ad568c2ce97e6ec9e1_b-1.png"/>

<p align="middle">往返延迟随时间的变化。红线：标准 TCP <br/>（可见周期性的延迟变化，以及 buffer 几乎总是被填满）；<br/>绿线：TCP BBR</p>
</center>

*（图片引自 Google 在 ACM queue 2016 年 9-10 月刊上的论文 [^1]，下同）*

有很多论文提出在网络设备上把当前缓冲区大小的信息反馈给终端，比如在数据中心广泛应用的 ECN（Explicit Congestion Notification）。然而广域网上网络设备众多，更新换代困难，需要网络设备介入的方案很难大范围部署。

TCP BBR 是怎样解决以上两个问题的呢？

1. **既然不容易区分拥塞丢包和错误丢包，TCP BBR 就干脆不考虑丢包。**
2. **既然灌满水管的方式容易造成缓冲区膨胀，TCP BBR 就分别估计带宽和延迟，而不是直接估计水管的容积。**

带宽和延迟的乘积就是发送窗口应有的大小。发明于 2002 年并已进入 Linux 内核的 TCP Westwood 拥塞控制算法，就是分别估计带宽和延迟，并计算其乘积作为发送窗口。然而**带宽和延迟就像粒子的位置和动量，是没办法同时测准的**：要测量最大带宽，就要把水管灌满，缓冲区中有一定量的数据包，此时延迟就是较高的；要测量最低延迟，就要保证缓冲区为空，网络里的流量越少越好，但此时带宽就是较低的。

TCP BBR 解决**带宽和延迟无法同时测准**的方法是：**交替测量带宽和延迟；用一段时间内的带宽极大值和延迟极小值作为估计值。**

在连接刚建立的时候，TCP BBR 采用类似标准 TCP 的**慢启动**，指数增长发送速率。然而标准 TCP 遇到任何一个丢包就会立即进入拥塞避免阶段，它的本意是填满水管之后进入拥塞避免，然而（1）如果链路的错误丢包率较高，没等到水管填满就放弃了；（2）如果网络里有 *buffer*，总要把缓冲区填满了才会放弃。

TCP BBR 则是根据收到的确认包，发现有效带宽不再增长时，就进入拥塞避免阶段。（1）链路的错误丢包率只要不太高，对 BBR 没有影响；（2）当发送速率增长到开始占用 *buffer* 的时候，有效带宽不再增长，BBR 就及时放弃了（事实上放弃的时候占的是 3 倍带宽 × 延迟，后面会把多出来的 2 倍 *buffer* 清掉），这样就不会把缓冲区填满。（应评论区各位 dalao 要求，补充测试环境说明：是在新加坡的服务器上设置了 BBR，**新加坡的服务器是数据的发送方**。这个服务器是访问墙外资源的 HTTP 代理。科大移动出口到 DigitalOcean 之间不是 dedicated 的专线，是走的公网，科大移动出口这边是 1 Gbps 无限速（但是要跟其他人 share），DigitalOcean 实测是限速 200 Mbps。RTT 是 66 ms。实测结果这么好，也是因为大多数人用的是 TCP Cubic (Linux) / Compound TCP (Windows)，**在有一定丢包率的情况下，TCP BBR 更加激进，抢占了更多的公网带宽**。因此也是有些不道德的感觉。）

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-1532609c46413137b0e7b7234b015dcb_b-1.png"/>
<p align="middle">发送窗口与往返延迟和有效带宽的关系。<br/>BBR 会在左右两侧的拐点之间停下，基于丢包的标准 TCP 会在右侧拐点停下<br/>（图片引自 TCP BBR 论文，下同）</p>
</center>

  在慢启动过程中，由于 *buffer* 在前期几乎没被占用，延迟的最小值就是延迟的初始估计；慢启动结束时的最大有效带宽就是带宽的初始估计。

慢启动结束后，为了把多占用的 2 倍带宽 × 延迟消耗掉，BBR 将进入**排空（drain）阶段**，指数降低发送速率，此时 *buffer* 里的包就被慢慢排空，直到往返延迟不再降低。如下图绿线所示。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-39bd5ebffb4b71256651390c1a1e9bf2_b-1.png">
<p align="middle">TCP BBR（绿线）与标准 TCP（红线）有效带宽和往返延迟的比较</p>
</center>

   排空阶段结束后，BBR 进入稳定运行状态，交替探测带宽和延迟。由于网络带宽的变化比延迟的变化更频繁，**BBR 稳定状态的绝大多数时间处于带宽探测阶段**。带宽探测阶段是一个正反馈系统：定期尝试增加发包速率，如果收到确认的速率也增加了，就进一步增加发包速率。

具体来说，以每 8 个往返延迟为周期，在第一个往返的时间里，BBR 尝试增加发包速率 1/4（即以估计带宽的 5/4 速度发送）。在第二个往返的时间里，为了把前一个往返多发出来的包排空，BBR 在估计带宽的基础上降低 1/4 作为发包速率。剩下 6 个往返的时间里，BBR 使用估计的带宽发包。

当网络带宽增长一倍的时候，每个周期估计带宽会增长 1/4，每个周期为 8 个往返延迟。其中向上的尖峰是尝试增加发包速率 1/4，向下的尖峰是降低发包速率 1/4（排空阶段），后面 6 个往返延迟，使用更新后的估计带宽。3 个周期，即 24 个往返延迟后，估计带宽达到增长后的网络带宽。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-a738f6a4660e99fc01ab8262cfecc4fa_b-1.png"/>
<p align="middle">网络带宽增长一倍时的行为。绿线为网络中包的数量，蓝线为延迟</p>
</center>

  当网络带宽降低一半的时候，多出来的包占用了 *buffer*，导致网络中包的延迟显著增加（下图蓝线），有效带宽降低一半。延迟是使用极小值作为估计，增加的实际延迟不会反映到估计延迟（除非在延迟探测阶段，下面会讲）。带宽的估计则是使用一段滑动窗口时间内的极大值，当之前的估计值超时（移出滑动窗口）之后，降低一半后的有效带宽就会变成估计带宽。估计带宽减半后，发送窗口减半，发送端没有窗口无法发包，*buffer* 被逐渐排空。*发送窗口与往返延迟和有效带宽的关系。BBR 会在左右两侧的拐点之间停下，基于丢包的标准 TCP 会在右侧拐点停下（图片引自 TCP BBR 论文，下同）*

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-d07a32627524703ebb1736091916975e_b-1.png"/>
<p align="middle">网络带宽降低一半时的行为。绿线为网络中包的数量，蓝线为延迟</p>
</center>

**当带宽增加一倍时，BBR 仅用 1.5 秒就收敛了；而当带宽降低一半时，BBR 需要 4 秒才能收敛。**前者由于带宽增长是指数级的；后者主要是由于带宽估计采用滑动窗口内的极大值，需要一定时间有效带宽的下降才能反馈到带宽估计中。

当网络带宽保持不变的时候，稳定状态下的 TCP BBR 是下图这样的：（我们前面看到过这张图）可见每 8 个往返延迟为周期的延迟细微变化。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-c623d2f88cc5f3ad568c2ce97e6ec9e1_b-1.png"/>
<p align="middle">
往返延迟随时间的变化。红线：标准 TCP；绿线：TCP BBR</p>
</center>

上面介绍了 BBR 稳定状态下的带宽探测阶段，那么什么时候探测延迟呢？在带宽探测阶段中，估计延迟始终是使用极小值，如果实际延迟真的增加了怎么办？TCP BBR 每过 10 秒，如果估计延迟没有改变（也就是没有发现一个更低的延迟），就进入延迟探测阶段。延迟探测阶段持续的时间仅为 200 毫秒（或一个往返延迟，如果后者更大），这段时间里发送窗口固定为 4 个包，也就是几乎不发包。这段时间内测得的最小延迟作为新的延迟估计。也就是说，**大约有 2% 的时间 BBR 用极低的发包速率来测量延迟**。

**TCP BBR 还使用 pacing 的方法降低发包时的 burstiness**，减少突然传输的一串包导致缓冲区膨胀。发包的 burstiness 可能由两个原因引起：

1. 数据接收方为了节约带宽，把多个确认（ACK）包累积成一个发出，这叫做 ACK Compression。数据发送方收到这个累积确认包后，如果没有 pacing，就会发出一连串的数据包。
2. 数据发送方没有足够的数据可传输，积累了一定量的空闲发送窗口。当应用层突然需要传输较多的数据时，如果没有 pacing，就会把空闲发送窗口大小这么多数据一股脑发出去。

下面我们来看 TCP BBR 的效果如何。

首先看 BBR 试图解决的第一个问题：在有随机丢包情况下的吞吐量。如下图所示，**只要有万分之一的丢包率，标准 TCP 的带宽就只剩 30%；千分之一丢包率时只剩 10%；有百分之一的丢包率时几乎就卡住了。而 TCP BBR 在丢包率 5% 以下几乎没有带宽损失，在丢包率 15% 的时候仍有 75% 带宽**。*网络带宽降低一半时的行为。**绿线为网络中包的数量，蓝线为延迟*

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-1df6e7ba0cae0147dddb01aac0d0f309_b-1.png"/>
</center>
<p align="middle">100 Mbps，100ms 下的丢包率和有效带宽（红线：标准 TCP，绿线：TCP BBR）</p>
</center>

  **异地数据中心间跨广域网的传输往往是高带宽、高延迟的，且有一定丢包率，TCP BBR 可以显著提高传输速度**。这也是中国科大 LUG HTTP 代理服务器和 Google 广域网（B4）部署 TCP BBR 的主要原因。
再来看 BBR 试图解决的第二个问题：**降低延迟，减少缓冲区膨胀**。如下图所示，**标准 TCP 倾向于把缓冲区填满，缓冲区越大，延迟就越高**。当用户的网络接入速度很慢时，这个延迟可能超过操作系统连接建立的超时时间，导致连接建立失败。使用 TCP BBR 就可以避免这个问题。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-78cabc327d487c325852ab96af1309e0_b-1.png"/>
<p align="middle">缓冲区大小与延迟的关系（红线：标准 TCP，绿线：TCP BBR）</p>
</center>

  Youtube 部署了 TCP BBR 之后，全球范围的中位数延迟降低了 53%（也就是快了一倍），发展中国家的中位数延迟降低了 80%（也就是快了 4 倍）。从下图可见，**延迟越高的用户，采用 TCP BBR 后的延迟下降比例越高，原来需要 10 秒的现在只要 2 秒了**。如果您的网站需要让用 GPRS 或者慢速 WiFi 接入网络的用户也能流畅访问，不妨试试 TCP BBR。

<center>
<img src="http://zoufeng.net/wp-content/uploads/2017/06/v2-febab4b4a48caaa8bf569e8d033757cf_b-1.png"/>
<p align="middle">标准 TCP 与 TCP BBR 的往返延迟中位数之比</p>
</center>

  综上，**TCP BBR 不再使用丢包作为拥塞的信号，也不使用 “加性增，乘性减” 来维护发送窗口大小，而是分别估计极大带宽和极小延迟，把它们的乘积作为发送窗口大小。**
BBR 的连接开始阶段由慢启动、排空两阶段构成。为了解决带宽和延迟不易同时测准的问题，BBR 在连接稳定后交替探测带宽和延迟，其中探测带宽阶段占绝大部分时间，通过正反馈和周期性的带宽增益尝试来快速响应可用带宽变化；偶尔的探测延迟阶段发包速率很慢，用于测准延迟。

BBR 解决了两个问题：

1. **在有一定丢包率的网络链路上充分利用带宽。非常适合高延迟、高带宽的网络链路。**
2. **降低网络链路上的 buffer 占用率，从而降低延迟。非常适合慢速接入网络的用户。**

看到评论区很多客户端和服务器哪个部署 TCP BBR 有效的问题，需要提醒：**TCP 拥塞控制算法是数据的发送端决定发送窗口，因此在哪边部署，就对哪边发出的数据有效。**如果是下载，就应在服务器部署；如果是上传，就应在客户端部署。

如果希望加速访问国外网站的速度，且下载流量远高于上传流量，在客户端上部署 TCP BBR（或者任何基于 TCP 拥塞控制的加速算法）是没什么效果的。需要在 VPN 的国外出口端部署 TCP BBR，并做 TCP Termination & TCP Proxy。也就是客户建立连接事实上是跟 VPN 的国外出口服务器建联，国外出口服务器再去跟目标服务器建联，使得丢包率高、延迟大的这一段（从客户端到国外出口）是部署了 BBR 的国外出口服务器在发送数据。或者在 VPN 的国外出口端部署 BBR 并做 HTTP(S) Proxy，原理相同。

大概是由于 ACM queue 的篇幅限制和目标读者，这篇论文并没有讨论（仅有拥塞丢包情况下）TCP BBR 与标准 TCP 的公平性。也没有讨论 BBR 与现有拥塞控制算法的比较，如基于往返延迟的（如 TCP Vegas）、综合丢包和延迟因素的（如 Compound TCP、TCP Westwood+）、基于网络设备提供拥塞信息的（如 ECN）、网络设备采用新调度策略的（如 CoDel）。期待 Google 发表更详细的论文，也期待各位同行报告 TCP BBR 在实验或生产环境中的性能。

本人不是 TCP 拥塞控制领域的专家，如有错漏不当之处，恳请指正。

[^1]: Cardwell, Neal, et al. “BBR: Congestion-Based Congestion Control.” *Queue*14.5 (2016): 50.

- 本文固定链接: <http://zoufeng.net/2017/06/22/google-bbr/>
- 转载请注明: [foam](http://zoufeng.net/author/foam/) 2017年06月22日 于 [foam](http://zoufeng.net/) 发表

------
------

## [tsetao](https://www.zhihu.com/people/tsetao) 的回答
> programmer
> 60 人赞同了该回答


在探讨这个问题之前，关于网络中的Bufferbloat问题需要了解，详细信息在这里（https://www.bufferbloat.net/projects/bloat/wiki/Introduction/），
@李博杰  的回答也说得比较清楚了。

在这里做一些补充吧。
流量控制分为两部分：

- 接收方的流量控制（即滑动窗口）-- 由接收方告知，只关注自身缓存情况，不关注网络，这里不讨论。
- 发送方的流量控制（即拥塞控制）
  现在广泛使用的CUBIC/(new)Reno都是基于丢包的，在算法上重点输出拥塞窗口（cwnd）；
  而BBR输出cwnd和pacing_rate，且pacing_rate为主，cwnd为辅，参考(https://groups.google.com/d/msg/bbr-dev/bIIdBU4feD8/sfmbh4W_DgAJ)

什么是pacing？看下图：

<center>
<img src="https://img-blog.csdnimg.cn/20181217170852698.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5NzU3Mjgz,size_16,color_FFFFFF,t_70"/>
<p align="middle">图片引用来自 QUIC @ Google Developers Live, February 2014</p>
</center>

也就是说早期的拥塞控制输出cwnd只是告诉tcp可以发多少数据，而没有说怎么发，恰好，

BBR输出的pacing_rate就是告诉TCP怎么发。



**BBR对TCP的大胆改动：**

<center>
<img src="https://img-blog.csdnimg.cn/20181217171040810.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5NzU3Mjgz,size_16,color_FFFFFF,t_70"/>
<p align="middle"></p>
</center>

红框中是BBR加入时添加的，这里很明显，BBR从TCP接管了充分的控制权。

之前的拥塞控制并不总是能起作用，换句话说就是会被夺权。TCP走进PRR算法（参考论文：Proportional Rate Reduction for TCP）后，拥塞控制无能为力。

从工程实现的角度来看，BBR这个小小的修改把TCP的 **可靠传输 / 拥塞控制** 解耦了 —— TCP你专注于自己的可靠性（当然还有很多其它细节），我BBR总是会负责任地告诉你（TCP）现在可以发多少数据，以什么速度发出这些数据。

**丢包对BBR有什么影响？**

<center>
<img src="https://img-blog.csdnimg.cn/20181217171225534.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI5NzU3Mjgz,size_16,color_FFFFFF,t_70"/>
<p align="middle">图片来自 BBR-TCP-Opportunities</p>
</center>

从算法上来讲，丢包对BBR的影响微乎其微。BBR只管告诉TCP cwnd/pacing_rate，**不管需要发送的包是重传的还是正常发送的**。看上图，丢包率在20%的时候BBR的输出骤降，导致这个现象其实是从实现层面考虑设置的一个**阈值**。

```
/* If lost/delivered ratio > 20%, interval is "lossy" and we may be policed: */
static const u32 bbr_lt_loss_thresh = 50;
```

这里还考虑了被限速的场景，不展开。

友好性测试

[@jamesr](https://www.zhihu.com/people/3ae27c8731f514f81c4dd3283eef8f3a)

 提到的友好性问题，google groups里有测试数据

https://groups.google.com/d/msg/bbr-dev/zyvKgjjOYr8/pN20-CmVBQAJ

\----------------

10 Mbps link with 40ms RTT

带宽，RTT固定的情况下，测试不同 Buffer 大小的情况：

```
buf以n倍BDP表示，CUBIC/Vegas/BBR输出单位Mbps
(1) 1 CUBIC flow vs. 1 Vegas flow:
buf  CUBIC  Vegas
---- -----  -----
0.5   6.81   2.73
1     8.49   1.06
2     9.12   0.43
4     9.15   0.4

(2) 1 CUBIC flow vs. 1 BBR flow:
buf  CUBIC  BBR
---- -----  -----
0.5   1.74   7.67
1     1.71   7.7
2     4.79   4.72
4     6.12   3.43
```

说明：
Vegas是基于延迟的拥塞控制，现在几乎不用了，因为它们跟基于丢包的拥塞控制发生竞争时会处于几乎饿死的状态。
另外，关于Ledbat（基于延迟的）用于BT，是理所应当的（挂BT下载的急迫性一般是低于常规网络流量的，当有常规流量介入时，主动退避是符合常理的）。

v------------2016.12.20更新--------------v
bbr-dev邮件组中一组测试数据（[@Leo Zhang](https://www.zhihu.com/people/844e274a5e578fac57e2d5d703b0863f) 的回答中也提供了）引起了大家的关注。

最开始，Leo Zhang的回答所提到的测试报告我并未详细阅读。今天抽时间详细看了一下，感到意外。

比如出现BBR饿死CUBIC，BBR饿死同类。这和其它测试数据出入很大。

我第一反应就是环境不对，BBR挑环境。然而这方面却没找到说明，BBR相关的文档都没有提到专用场景。

接着， [@NeterOster](https://www.zhihu.com/people/d69542bb0cae6da1283464303f4affca) 抛出了另一个问题（如何评价文章《令人躁动一时且令人不安的TCP BBR算法》？）。看过文章（该作者之前的文章也看了），写了简答，才发现自己似乎错过了什么，同时自己其实也对20%的丢包率阈值设置有疑问，偷懒的理解就是大量测试统计的结果吧。



这些天bbr-dev中关于BBR Report（https://groups.google.com/forum/#!topic/bbr-dev/Ps4zyM5AD9g）这份精彩的邮件列表我错过了，赶紧补一下！

Neal Cardwell 说到：

> We are actively working on reducing the *buffer* pressure exerted by BBR, which should help in the sorts of CUBIC vs BBR scenarios you describe (as well as BBR vs BBR scenarios).

这份刚刚诞生的拥塞控制的确还是存在不少缺陷，不过依然在持续改进中，期待美好的事情总会发生吧！
\^-------------2016.12.20更新-------------^

根据 [@李博杰](https://www.zhihu.com/people/02877c4fde3b2e9dbd938f1a8eef598d) 的回答中已经提到的实测延迟改善数据，我们可以看到目前互联网整体上 Bufferbloat 问题是比较严重的，即我们的网络请求延迟总体被拉高很多。

为了降低延迟，如果：

1、我们改用基于延迟的拥塞控制，因为不可能一时间全部更新，先替换的那部分网络体验会很糟糕；

2、逐步改用BBR这类拥塞控制，我们的网络高延迟问题将会逐渐得到改善。

以上并非说一定是BBR，但是BBR在拥塞控制上开辟了一条新的路（也许早就有人做过类似的尝试，但是因为工程实现，或者测试环境受限不能得到广泛验证，石沉大海），相信以后关于拥塞控制的研究改进更多会基于这条路去走。

**最后，BBR是基于什么的拥塞控制？**
根据论文，是基于拥塞的拥塞控制（Congestion-based Congestion Control），但是看起来感觉不好理解。
根据我的理解，我更倾向于称它为 基于带宽延迟的拥塞控制（**BDP-based Congestion Control**）。
因为，BBR总是在测量最小RTT（10s内），最大Bandwidth（10 Round Trips），并且尽量控制输出到网络的数据包（in-flight）靠近 BDP（without *buffer*），这样既能保证带宽利用率，又能避免*Bufferbloat*问题。

PS. BBR也已经被实现在QUIC协议中，参考（<https://cs.chromium.org/chromium/src/net/quic/core/congestion_control/bbr_sender.h>）。

我也并非完全理解了BBR的实现细节，并且其中还牵扯到TCP的实现，也不熟悉。
如果以上理解有错，或者不到位，欢迎指正。

完。

\---

XTao

[编辑于 2016-12-21](https://www.zhihu.com/question/53559433/answer/136002384)

**评论：**
> jamesr
> > 补充了不少资料
>
> 行功
> > 本质上，BBR也属于delay-based，只不过是RTTProp。因此，遇到Cubic后也会饿死，不会比Vegas好太多。上面的数据 就是例子。
> > 
> > 另外，Pacing就是大家原来说的rate-based congestion control, 给一个Rate发送而不是发送一个窗口，例如TFRC，WebRTC早已这样实现。其性能与控制间隔有关，调整慢了就会导致delay大甚至丢包。
> > :thumbsup: 2
> >
> > tsetao (作者) 回复行功
> > > “本质上，BBR也属于delay-based” 也说得过去，但是相对于典型的delay-based拥塞控制，BBR给了一个争取带宽的窗口（取10s内的最小RTT -- 实现上并不严格）。
> > > 
> > > 上面的测试数据来看，Vegas饿死太容易，BBR比起来还是好不少。
> > > 
> > > rate-based还没有了解过，mark，抽空看看！
>
> 李金峰
> > 我去，底层的东西都这么负复杂嘛
> > iLRainyday回复李金峰
> >
> > > TCP的拥塞控制应该是比较复杂的一块了
>
>  谢狄
>  > Congestion-based Congestion Control 翻译成「真正的拥塞控制」吧，意思是：你们之前都找错方向了，什么 loss-based，丢包根本就和拥塞没什么关系… 这标题就是一脸嘲讽啊！（我瞎说的
>  > ------
>  > BBR 这名字就自解释了，Bottleneck Bandwidth & Round-trip propagation time
>  > 
>  > 薛定谔的喵
>  人们的电脑可以配置使用BBR吗 电脑系统是Windows或Linux或Mac
>  > 
>  > tsetao (作者) 回复薛定谔的喵
>  > 
>  > > PC通常情况上行流量非常少，拥塞控制的差异并不会很明显地体现出来。当然如果你有较多上传需求是可以尝试的。Linux比较容易，自己配置编译4.9+ kernel；Windows/Mac不了解，应该有提供相应入口接入新的拥塞控制；另外一种就是把bbr作用在自定义的应用层协议上。

------
------

