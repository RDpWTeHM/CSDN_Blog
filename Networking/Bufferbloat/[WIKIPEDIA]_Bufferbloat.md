# [WIKIPEDIA] Bufferbloat
From Wikipedia, the free encyclopedia



@[TOC]( )



Bufferbloat is a cause of high latency in packet-switched networks caused by excess buffering of packets. Bufferbloat can also cause packet delay variation (also known as jitter), as well as reduce the overall network throughput. When a router or switch is configured to use excessively large buffers, even very high-speed networks can become practically unusable for many interactive applications like voice over IP (VoIP), online gaming, and even ordinary web surfing.
> (**Google 自动翻译**) **Bufferbloat**是高的原因[延迟](https://en.wikipedia.org/wiki/Latency_(engineering))在[分组交换网络](https://en.wikipedia.org/wiki/Packet-switched_network)所造成的过量[缓冲](https://en.wikipedia.org/wiki/Buffer_(telecommunication))的[分组。](https://en.wikipedia.org/wiki/Network_packet)Bufferbloat还可能导致[数据包延迟变化](https://en.wikipedia.org/wiki/Packet_delay_variation)（也称为抖动），并降低整体网络[吞吐量](https://en.wikipedia.org/wiki/Throughput)。当[路由器](https://en.wikipedia.org/wiki/Router_(computing))或[交换机](https://en.wikipedia.org/wiki/Network_switch)配置为使用过大的缓冲区时，即使是非常高速的网络也几乎无法用于许多交互式应用，如[IP语音](https://en.wikipedia.org/wiki/Voice_over_IP)（VoIP），[在线游戏](https://en.wikipedia.org/wiki/Online_game)甚至普通的网上冲浪。

Some communications equipment manufacturers designed unnecessarily large buffers into some of their network products. In such equipment, bufferbloat occurs when a network link becomes congested, causing packets to become queued for long periods in these oversized buffers. In a first-in first-out queuing system, overly large buffers result in longer queues and higher latency, and do not improve network throughput.
> (**Google 自动翻译**) 一些通信设备制造商为其某些[网络产品](https://en.wikipedia.org/wiki/Network_equipment)设计了不必要的大缓冲区。在这样的设备中，当网络链路变得[拥挤](https://en.wikipedia.org/wiki/Network_congestion)时，会发生缓冲区，从而导致数据包在这些超大缓冲区中长时间排队。在[先进先出](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics))排队系统中，过大的缓冲区会导致更长的队列和更高的延迟，并且不会提高网络吞吐量。

The bufferbloat phenomenon was initially described as far back as in 1985.[^1] It gained more widespread attention starting in 2009.[^2]
> (**Google 自动翻译**) 缓冲浮现现象最初的描述早在1985年。它从2009年开始受到更广泛的关注。

[^1]: "[On Packet Switches With Infinite Storage :link:](http://tools.ietf.org/html/rfc970)". 1985-12-31.
[^2]: van Beijnum, Iljitsch (2011-01-07). "[Understanding Bufferbloat and the Network Buffer Arms Race :link:](https://arstechnica.com/tech-policy/news/2011/01/understanding-bufferbloat-and-the-network-buffer-arms-race.ars)". Ars Technica. Retrieved 2011-11-12.

@[toc]( )

## Buffering
*See also: [CoDel § Theoretical underpinnings](https://en.wikipedia.org/wiki/CoDel#Theoretical_underpinnings)*

An established rule of thumb for the network equipment manufacturers was to provide buffers large enough to accommodate at least 250 ms of buffering for a stream of traffic passing through a device. For example, a router's Gigabit Ethernet interface would require a relatively large 32 MB buffer.[^3] Such sizing of the buffers can lead to failure of the TCP congestion control algorithm. The buffers then take some time to drain, before congestion control resets and the TCP connection ramps back up to speed and fills the buffers again.[^4] Bufferbloat thus causes problems such as high and variable latency, and choking network bottlenecks for all other flows as the buffer becomes full of the packets of one TCP stream and other packets are then dropped.[^5]
> (**Google 自动翻译**) 网络设备制造商的既定[经验法则](https://en.wikipedia.org/wiki/Rule_of_thumb)是提供足够大的缓冲区，以便为通过设备的流量流提供至少250  [毫秒](https://en.wikipedia.org/wiki/Millisecond)的缓冲。例如，路由器的[千兆以太网](https://en.wikipedia.org/wiki/Gigabit_Ethernet)接口需要相对较大的32  [MB ](https://en.wikipedia.org/wiki/MiB)*缓冲区*。[[3\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-3) 缓冲区的这种大小调整可能导致[TCP拥塞控制算法的](https://en.wikipedia.org/wiki/TCP_congestion_control_algorithm)失败。在拥塞控制重置并且TCP连接恢复速度并再次填充缓冲区之前，缓冲区需要一些时间才能耗尽。[[4\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-codel-acmq-4)因此，Bufferbloat会导致诸如高延迟和可变延迟之类的问题，并且当缓冲区充满一个TCP流的数据包然后丢弃其他数据包时，会阻塞所有其他流的网络瓶颈。[[5\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-5)

A bloated buffer has an effect only when this buffer is actually used. In other words, oversized buffers have a damaging effect only when the link they buffer becomes a bottleneck. The size of the buffer serving a bottleneck can be measured using the ping utility provided by most operating systems. First, the other host should be pinged continuously; then, a several-seconds-long download from it should be started and stopped a few times. By design, the TCP congestion avoidance algorithm will rapidly fill up the bottleneck on the route. If downloading (and uploading, respectively) correlates with a direct and important increase of the round trip time reported by ping, then it demonstrates that the buffer of the current bottleneck in the download (and upload, respectively) direction is bloated. Since the increase of the round trip time is caused by the buffer on the bottleneck, the maximum increase gives a rough estimation of its size in milliseconds.[^6]
> (**Google 自动翻译**) 只有在实际使用此*缓冲区*时，膨胀*缓冲区*才有效。换句话说，超大缓冲区只有在它们缓冲的链接成为瓶颈时才会产生破坏性影响。可以使用大多数操作系统提供的[ping](https://en.wikipedia.org/wiki/Ping_(networking_utility))实用程序来测量服务于瓶颈的缓冲区的大小。首先，应该连续ping另一个主机; 然后，从它开始几秒钟的下载应该开始并停止几次。通过设计，TCP拥塞避免算法将迅速填补路由的瓶颈。如果下载（和分别上传）与ping报告的往返时间的直接和重要增加相关，则表明*缓冲区*下载（和分别上传）方向的当前瓶颈是臃肿的。由于往返时间的增加是由瓶颈上的*缓冲区*引起的，因此最大增加会粗略估计其大小（以毫秒为单位）。[[6\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-6)

In the previous example, using an advanced traceroute tool instead of the simple pinging (for example, MTR) will not only demonstrate the existence of a bloated buffer on the bottleneck, but will also pinpoint its location in the network. Traceroute achieves this by displaying the route (path) and measuring transit delays of packets across the network. The history of the route is recorded as round-trip times of the packets received from each successive host (remote node) in the route (path).[^7]
> (**Google 自动翻译**) 在前面的示例中，使用高级[traceroute](https://en.wikipedia.org/wiki/Traceroute)工具而不是简单的ping（例如，[MTR](https://en.wikipedia.org/wiki/MTR_(software))）将不仅证明瓶颈上存在膨胀*缓冲区*，而且还将精确定位其在网络中的位置。Traceroute通过显示路径（路径）和测量网络中数据包的传输延迟来实现此目的。将路由的历史记录为从路由（路径）中的每个连续主机（远程节点）接收的分组的往返时间。[[7\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-7)

[^3]: Guido Appenzeller; Isaac Keslassy; Nick McKeown (2004). "[Sizing Router Buffers :link:](http://conferences.sigcomm.org/sigcomm/2004/papers/p277-appenzeller1.pdf)" [![PDF](https://www.easyicon.net/download/png/26141/24/)(PDF)](http://conferences.sigcomm.org/sigcomm/2004/papers/p277-appenzeller1.pdf/___possible__unsafe__site__). ACM SIGCOMM. ACM. Retrieved 2013-10-15.

[^4]: [Nichols, Kathleen :link:](https://en.wikipedia.org/wiki/Kathleen_Nichols); [Jacobson, Van :link:](https://en.wikipedia.org/wiki/Van_Jacobson) (2012-05-06). ["Controlling Queue Delay" :link:](http://queue.acm.org/detail.cfm?id=2209336). *ACM Queue*. ACM Publishing. Retrieved 2013-09-27.

[^5]: Gettys, Jim (May–June 2011). ["Bufferbloat: Dark Buffers in the Internet" :link:](http://www.computer.org/portal/web/csdl/doi/10.1109/MIC.2011.56). IEEE Internet Computing. IEEE. pp. 95–96. [doi :link:](https://en.wikipedia.org/wiki/Digital_object_identifier):[10.1109/MIC.2011.56 :link:](https://doi.org/10.1109%2FMIC.2011.56). Retrieved 2012-02-20.

[^6]: Clunis, Andrew (2013-01-22). "Bufferbloat demystified". Retrieved 2013-09-27.[self-published source?]

[^7]: ["traceroute(8) – Linux man page" :link:](http://linux.die.net/man/8/traceroute). *die.net*. Retrieved 2013-09-27.



## Mechanism
*See also: [TCP tuning § Window size](https://en.wikipedia.org/wiki/TCP_tuning#Window_size), and [Slow-start](https://en.wikipedia.org/wiki/Slow-start)*

Most TCP congestion control algorithms rely on measuring the occurrence of packet drops to determine the available bandwidth between two ends of a connection. The algorithms speed up the data transfer until packets start to drop, then slow down the transmission rate. Ideally, they keep adjusting the transmission rate until it reaches an equilibrium speed of the link. So that the algorithms can select a suitable transfer speed, the feedback about packet drops must occur in a timely manner. With a large buffer that has been filled, the packets will arrive at their destination, but with a higher latency. The packets were not dropped, so TCP does not slow down once the uplink has been saturated, further filling the buffer. Newly arriving packets are dropped only when the buffer is fully saturated. Once this happens TCP may even decide that the path of the connection has changed, and again go into the more aggressive search for a new operating point.[^8]
> (**Google 自动翻译**) 大多数[TCP拥塞控制](https://en.wikipedia.org/wiki/TCP_congestion_control)算法依赖于测量分组丢弃的发生以确定连接两端之间的可用[带宽](https://en.wikipedia.org/wiki/Bandwidth_(computing))。算法加速数据传输，直到数据包开始下降，然后降低传输速率。理想情况下，他们不断调整传输速率，直到达到链路的平衡速度。为了使算法能够选择合适的传输速度，关于丢包的反馈必须及时发生。使用已填充的大[*缓冲区*](https://en.wikipedia.org/wiki/Buffer_(telecommunication))，数据包将到达目的地，但延迟时间更长。数据包未被丢弃，因此一旦上行链路饱和，TCP就不会减速，从而进一步填充*缓冲区*。只有当*缓冲区*完全饱和时，才会丢弃新到达的数据包。一旦发生这种情况，TCP甚至可以确定连接的路径已经改变，并再次进入更积极的搜索新的操作点。[[8\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-8)

Packets are queued within a network buffer before being transmitted; in problematic situations, packets are dropped only if the buffer is full. On older routers, buffers were fairly small so they filled quickly and therefore packets began to drop shortly after the link became saturated, so the TCP protocol could adjust and the issue would not become apparent. On newer routers, buffers have become large enough to hold several seconds of buffered data. To TCP, a congested link can appear to be operating normally as the buffer fills. The TCP algorithm is unaware the link is congested and does not start to take corrective action until the buffer finally overflows and packets are dropped.
> (**Google 自动翻译**) 数据包在传输之前在网络缓冲区中排队; 在有问题的情况下，仅在缓冲区已满时才丢弃数据包。在较旧的路由器上，缓冲区相当小，因此填充速度很快，因此在链路饱和后不久，数据包开始下降，因此TCP协议可以调整，问题不会明显。在较新的路由器上，缓冲区已经变得足够大，可以容纳几秒钟的缓冲数据。对于TCP，当缓冲区填充时，拥塞的链接似乎正在正常运行。TCP算法不知道链路拥塞，并且在缓冲区最终溢出并丢弃数据包之前不会开始采取纠正措施。

The problem also affects other protocols. All packets passing through a simple buffer implemented as a single queue will experience the same delay, so the latency of any connection that passes through a filled buffer will be affected. Available channel bandwidth can also end up being unused, as some fast destinations may not be reached due to buffers clogged with data awaiting delivery to slow destinations — caused by contention between simultaneous transmissions competing for some space in an already full buffer. This also reduces the interactivity of applications using other network protocols, including UDP or any other datagram protocol used in latency-sensitive applications like VoIP and games.[^9] In extreme cases, bufferbloat may cause failures in essential protocols such as DNS.
> (**Google 自动翻译**) 该问题还会影响其他协议。通过作为单个队列实现的简单*缓冲区的*所有数据包将经历相同的延迟，因此通过填充*缓冲区*的任何连接的延迟都将受到影响。可用的信道带宽也可能最终未被使用，因为由于缓冲区被等待传送到慢速目的地的数据阻塞而可能无法到达某些快速目的地 - 这是由于在已经完全缓冲区中竞争某些空间的同时传输之间的争用引起的。这也降低了使用其他[网络协议](https://en.wikipedia.org/wiki/Network_protocol)的应用程序的交互性，包括[UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol)或在延迟敏感的应用程序（如VoIP和游戏）中使用的任何其他[数据报](https://en.wikipedia.org/wiki/Datagram)协议。[[9\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-9) 在极端情况下，bufferbloat可能会导致[DNS](https://en.wikipedia.org/wiki/Domain_Name_System)等基本协议出现故障。


[^8]: Jacobson, Van; Karels, MJ (1988). ["Congestion avoidance and control" :link:](https://web.archive.org/web/20040622215331/http://www.cord.edu/faculty/zhang/cs345/assignments/researchPapers/congavoid.pdf) [![PDF](https://www.easyicon.net/download/png/26141/24/)(PDF)](https://web.archive.org/web/20040622215331/http://www.cord.edu/faculty/zhang/cs345/assignments/researchPapers/congavoid.pdf). *ACM SIGCOMM Computer Communication Review*. **18** (4). Archived from [the original :link:](http://www.cord.edu/faculty/zhang/cs345/assignments/researchPapers/congavoid.pdf) [![PDF](https://www.easyicon.net/download/png/26141/24/)(PDF)](http://www.cord.edu/faculty/zhang/cs345/assignments/researchPapers/congavoid.pdf) on 2004-06-22.

[^9]: ["Technical Introduction to Bufferbloat" :link:](http://www.bufferbloat.net/projects/bloat/wiki/TechnicalIntro). *Bufferbloat.net*. Retrieved 2013-09-27.

## Impact on applications

Any type of a service which requires consistently low latency or jitter-free transmission (whether in low or high traffic bandwidths) can be severely affected, or even rendered unusable by the effects of bufferbloat. Examples are voice calls, online gaming, video chat, and other interactive applications such as instant messaging and remote login. Latency has been identified as more important than raw bandwidth for many years.[citation needed]
> (**Google 自动翻译**) 任何类型的服务都需要始终如一的低延迟或无抖动传输（无论是在低流量带宽还是高流量带宽内）都会受到严重影响，甚至无法通过bufferbloat的影响使其无法使用。例如语音呼叫，在线游戏，[视频聊天](https://en.wikipedia.org/wiki/Video_chat)以及其他交互式应用，例如[即时消息](https://en.wikipedia.org/wiki/Instant_messaging)和[远程登录](https://en.wikipedia.org/wiki/Remote_login)。多年来，已确定延迟比原始带宽更重要。[ *引证需要* ]

When the bufferbloat phenomenon is present and the network is under load, even normal web page loads can take many seconds to complete, or simple DNS queries can fail due to timeouts.[^10]
> (**Google 自动翻译**) 当存在缓冲区浮动现象且网络负载不足时，即使正常的网页加载也可能需要很长时间才能完成，或者简单的DNS查询可能会因超时而失败。[[10\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-acm-10)


[^10]: ~~a b c d~~  Gettys, Jim; Nichols, Kathleen (January 2012). ["Bufferbloat: Dark Buffers in the Internet" :link:](http://cacm.acm.org/magazines/2012/1/144810-bufferbloat/fulltext). Communications of the ACM. **55** (1). ACM: 57–65. [doi :link:](https://en.wikipedia.org/wiki/Digital_object_identifier):[10.1145/2063176.2063196 :link:](https://doi.org/10.1145%2F2063176.2063196). Retrieved 2012-02-28.

## Diagnostic tools

The DSL Reports Speedtest[^11] is an easy-to-use test that includes a score for bufferbloat. The ICSI Netalyzr[^12] is another on-line tool that can be used for checking networks for the presence of bufferbloat, together with checking for many other common configuration problems.[^13] The bufferbloat.net web site also provides an easy procedure for determining whether a connection has excess buffering that will slow it down.[^14]
> (**Google 自动翻译**) DSL Reports Speedtest [[11\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-11)是一个易于使用的测试，包括缓冲区的分数。ICSI Netalyzr [[12\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-12)是另一种在线工具，可用于检查网络是否存在缓冲区，同时检查许多其他常见配置问题。[[13\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-13) bufferbloat.net网站还提供了一个简单的过程，用于确定连接是否具有过多的缓冲，从而减慢连接速度。[[14\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-14)


[^11]: ["Speed test - how fast is your internet?" :link:](http://dslreports.com/speedtest). *dslreports.com*. Retrieved 26 October 2017.

[^12]: ["ICSI Netalyzr" :link:](http://netalyzr.icsi.berkeley.edu/). *berkeley.edu*. Retrieved 30 January 2015.

[^13]: ["Understanding your Netalyzr results" :link:](https://www.newscientist.com/article/dn18953-understanding-your-netalyzr-results). Retrieved 26 October 2017.

[^14]: ["Tests for Bufferbloat" :link:](https://www.bufferbloat.net/projects/bloat/wiki/Tests_for_Bufferbloat/). *bufferbloat.net*. Retrieved 26 October 2017.


## Solutions and mitigations

Several technical solutions exist which can be broadly grouped into two categories: solutions that target the network and solutions that target the endpoints. These types of solutions are often complementary.
> (**Google 自动翻译**) 存在若干技术解决方案，其可以大致分为两类：针对网络的解决方案和针对端点的解决方案。这些类型的解决方案通常是互补的。

Network solutions generally take the form of queue management algorithms. This type of solution has been the focus of the IETF AQM working group[^15]. 
> (**Google 自动翻译**) 网络解决方案通常采用队列管理算法的形式。这种解决方案一直是[IETF](https://en.wikipedia.org/wiki/IETF) AQM工作组的重点[[15\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-ietf-aqm-15)。值得注意的

Notable examples include:

- AQM algorithms such as CoDel and PIE.[^16]
- Hybrid AQM and packet scheduling algorithms such as FQ-CoDel.[^17]
- Amendments to the DOCSIS standard[^18] to enable smarter buffer control in cable modems.[^10]
- Integration of queue management into the WiFi subsystem of the Linux operating system.[^19]

Notable examples of solutions targeting the endpoints are:

- The BBR congestion control algorithm for TCP.
- The µTP protocol employed by many BitTorrent clients.
- Techniques for using fewer connections, such as HTTP pipelining or HTTP/2 instead of the plain HTTP protocol.[^10]

For most end-users, the largest improvement can be had by fixing their home router.[^20] Most of the technical fixes are included in recent versions of the Linux operating system and the LEDE aftermarket router firmware.[^20] The problem may also be mitigated by reducing the buffer size on the OS[^10] and network hardware; however, this is often not configurable.
> (**Google 自动翻译**) 对于大多数最终用户来说，通过修复其家用路由器可以获得最大的改进。[[20\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-fix-bloat-20)大多数技术修复都包含在最新版本的[Linux](https://en.wikipedia.org/wiki/Linux)操作系统和[LEDE](https://en.wikipedia.org/wiki/LEDE)售后市场路由器固件中。[[20\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-fix-bloat-20)通过减少OS [[10\]](https://en.wikipedia.org/wiki/Bufferbloat#cite_note-acm-10)和网络硬件上的*缓冲区*大小，也可以缓解这个问题。但是，这通常是不可配置的。

[^15]: ["IETF AQM working group" :link:](https://datatracker.ietf.org/wg/aqm/about/). *ietf.org*. Retrieved 26 October 2017.

[^16]: Pan, Rong; Natarajan, Preethi; Piglione, Chiara; Prabhu, Mythili; Subramanian, Vijay; Baker, Fred; VerSteeg, Bill (2013). *PIE: A Lightweight Control Scheme To Address the Bufferbloat Problem*. 2013 IEEE 14th International Conference on High Performance Switching and Routing (HPSR). IEEE. [doi :link:](https://en.wikipedia.org/wiki/Digital_object_identifier):[10.1109/HPSR.2013.6602305 :link:](https://doi.org/10.1109%2FHPSR.2013.6602305). 

[^17]: Høiland-Jørgensen, Toke; McKenney, Paul; Taht, Dave; Gettys, Jim; Dumazet, Eric (2016-03-18). ["The FlowQueue-CoDel Packet Scheduler and Active Queue Management Algorithm" :link:](https://tools.ietf.org/html/draft-ietf-aqm-fq-codel-06). Retrieved 2017-09-28.

[^18]: ["DOCSIS "Upstream *Buffer* Control" feature" :link:](http://www.cablelabs.com/specs). CableLabs. pp. 554–556. Retrieved 2012-08-09.

[^19]: Høiland-Jørgensen, Toke; Kazior, Michał; Täht, Dave; Hurtig, Per; Brunstrom, Anna (2017). [*Ending the Anomaly: Achieving Low Latency and Airtime Fairness in WiFi* :link:](https://www.usenix.org/conference/atc17/technical-sessions/presentation/hoilan-jorgesen). 2017 USENIX Annual Technical Conference (USENIX ATC 17). USENIX - The Advanced Computing Systems Association. pp. 139–151. [ISBN :link:](https://en.wikipedia.org/wiki/International_Standard_Book_Number) [978-1-931971-38-6 :link:](https://en.wikipedia.org/wiki/Special:BookSources/978-1-931971-38-6). Retrieved 2017-09-28.

[^20]: ~~a b~~ ["What Can I Do About Bufferbloat?" :link:](https://www.bufferbloat.net/projects/bloat/wiki/What_can_I_do_about_Bufferbloat/). *bufferbloat.net*. Retrieved 26 October 2017.

