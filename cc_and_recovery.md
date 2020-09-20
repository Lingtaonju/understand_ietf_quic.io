# 0.前言
本系列是我们团队在落地 IETF QUIC 时通过对 QUIC 相关的 Draft 进行阅读和跟踪整理而来，目的是在保证本团队保持对 Draft 精确理解基础上，将相关内容贡献到社区，便于需要使用 QUIC 协议的同学有所参阅。
同时，相关内容的准确性可能会受限于本团队同学知识水平的影响，所以也欢迎对内容有异议的同学，跟我们联系，共同讨论进步成长。

本文是本系列的第一篇，主要介绍 QUIC 中的丢包检测和拥塞控制，内容来自于 draft 29 版本 

https://tools.ietf.org/pdf/draft-ietf-quic-recovery-29.pdf 

后序的 changelog 以 29 为基础进行更新。

# 1.背景理论部分
为了避免有些读者在可靠传输这块没有太多基础，这块先介绍一下需要的背景。本文的理论部分，主要介绍 “丢包检测与重传” 以及 “拥塞控制”，这部分的内容是可靠传输协议的通用理论知识，不区分 TCP 协议和 QUIC 协议，如果对这部分的理论很熟悉，则可以跳过。

## 丢包检测与重传
丢包检测，是可靠传输协议用来对网络中可能存在的丢包进行检测，并及时进行重传的一种手段，之所以要设计一个好的丢包检测算法，是因为：

+ 在传输过程中，丢包基本上都发生在中间的网络设备上，比如路由器或者交换机的队列溢出、或者队列主动丢包 (Active Queue Management, 例如 RED，以减少时间延迟)。而这些设备对于发送端或者接收端来说相当于黑盒，在丢包的过程中，不会有任何明确的丢包信息告诉发送端或者接收端，丢了就丢了；

+ 丢包信息在四层协议上的反应就是，sequence number 不连续了，举个例子，如发送端依次发送的 seq 是：s1, s2, s3, s4, s5 这五个报文，如果中间的一个报文，比如 s2 被丢弃了，那么接收端会依次收到 s1, s3, s4, s5, 这个时候，接收端为了通知发送端有丢包，会在 ack 报文中，对于接收到的 s3, s4, s5  依次 ack = s2, 这就是重复 ACK 的含义，发送端就知道 s2 之后的报文被收到了，但是 s2 没有被收到。 这个时候如果发送端能快速重传 s2 倒也没事，但是事情往往没有这么简单，因为丢包检测的过程中，还有些麻烦的事情需要处理：

  1. 序列号没有按序到达还有可能是因为乱序(reordering)，即有些包因为网路原因等走错路了，或者因为其他原因被缓存了，导致报文没有按顺到达
  2. 检查到丢包还需要进入拥塞控制，即降速，这个是比较要命的；所以不能轻易的发现有不连续的报文，就判断是丢包，降速（当然如果你违背了契约精神，不降速也行， 但是这往往会玉石俱焚）
  3. 即使根据某种算法判断出丢包，但是重传哪些报文也存在问题，例如是否需要重传当前发送窗口内的所有的数据包？

所以基于以上分析，我们可以发现，丢包检测算法掣肘于两个方面：

1. 网络设备是黑盒，即便可以大概率的通过报文的序列号判断出丢包，但是准确的丢包信息却无从获知，所以只能靠猜测，并需要在快速重传与容忍乱序上找到 tradeoff。
无法完全得知丢失的报文，从而无法高效的重传所有的报文
所以这使得丢包检测算法异常困难，也导致了大量演进算法的出现，比如: Fast Retransmit  、SACK、 FACK、RACK 等，到目前为止，丢包检测算法主要有两个手段：

2. 基于不连续被 ack 的 seq 的数量或者间隔的时间，例如传统的快速重传和快速恢复在收到3个重复 ACK 的时候就会开始重传
基于 RTO 等一些定时器进行判断

## 拥塞控制
拥塞控制用于发送端调节发送速率从而逼近网络带宽。之所以是逼近，因为网络设备的带宽对于发送端来说也是黑盒，发送速率只能是通过试探的方式来逼近，即：如果发送速率小于可用带宽，则增加，如果超过则降低；这里速率的控制是通过一个叫拥塞窗口的变量，我们知道的那个经典的滑动窗口，就是滑动这个拥塞窗口，而如何根据拥塞情况计算这个拥塞窗口，就是我们说的拥塞控制算法做的事情。

为了完成拥塞控制，则存在三个状态机，这三个状态机分别完成下面拥塞控制将要完成的几件事情：
1. 如何快速逼近可用带宽并避免拥塞，对应的状态机是: Slow Start 和 Congestion Avoidance
2. 如何快速的重传和恢复丢失的报文, 对应的状态机是:   Fast Retransmit/Fast Recovery

其中第一个问题：Slow Start 用于连接建立或者 RTO 之后快速的提高发送速率，典型的算法是每一个 RTT，发送窗口增加一倍； Congestion Avoidance 即拥塞避免，首选要解决如何检测拥塞，即我们说的拥塞信号的问题，拥塞信号可以分为：

1. 丢包；如果发现丢包了那肯定就是拥塞了，所以上一讲提到的丢包检测可以作为拥塞控制的一个输入，用来判断拥塞，直观上说，丢包越多，拥塞越大；
2. RTT；拥塞直观的反应就是 RTT 变大，网路设备的队列上产生拥堵，所以 RTT 也是作为拥塞的一种反馈，但是 RTT 的组成有很多部分，如何精确的判断出是传输中的 RTT 是比较重要的（这也是后面 QUIC 设计了 host delay 这个参数的目的;
3. ECN；这个就是网络设备主动的一个行为了，一般网络设备在队列长度超过阈值的时候，会打 ECN 标志，典型的应用如 DCTCP 这个算法；

检测到拥塞之后，不同的拥塞控制算法有不同的降速策略，例如：
1. NewReno 就会在检测到丢包后，将发送窗口减半
2. Cubic 则降窗 70%，之后根据丢包经过的时间，以三次曲线的形式调整窗口
3. BBR 则根据拥塞程度在 probe rtt , draing 等状态切换；

第二个问题，Fast Retransmit/Fast Recovery 即快速重传/快速恢复，不同的算法也有不同的实现，例如经典的 NewReno 在检测到三次重复 ACK 之后就会回重传丢失的报文，另外会严格遵循包守恒原则，每收到一个重复的 ACK，即将一个重传报文或者新的 packet 发送出去，从而达到可以快速重传的目的。

以上就是这篇文章的理论基础部分，接下来我们看看 QUIC 本身基于丢包检测和拥塞控制做了哪些创新，以及如何基于 QUIC 本身的特性实现这些功能。下文主要是对 ietf-draft 29 版本的一些理解，有些是一些直白的翻译，可以中英文对照阅读


# 2. QUIC 设计理念
QUIC 的丢包检测和拥塞控制 follow the spirit of TCP，借鉴了各种 RFCs 以及一些经典的论文，但是 QUIC 也有一些自己特殊的地方，比如：

+ 所有的传输都有一个包级别的 header, header 中的加密等级和 packet number space 对等，packet number 在一个 space 内严格递增以防止 pkt number 的二义性
+ 一个 QUIC 包可以包含多个不同类型的 QUIC Frame，frame 的类型决定了不同的恢复和拥塞控制逻辑：
  1. 所有的 Packets 都是需要被 acked 的，尽管携带有 no ack-eliciting frame 的报文是跟着 ack-eliciting 报文一起被 acked 的
  2. 带有 CRYPTO frames 的 long header packets 影响握手完成时间，所以需要 ACK 的更快一些
  3. 除了携带 ACK Frame 以及 Conn Close Frame 的 packtets 以外，其他所有的 Packets 都受 congestion control 的控制，并且认为是 packets in flight
+ none ack-eliciting frames ，即不会触发 ack 的frmae, 但是算在 packet in flight 里, 主要有三种：
  1. ACK
  2. PADDING
  3. CONNECTION_CLOSE

# 3. 与TCP的不同
1. packet number spaces 有多个
+ quic 里有三种不同的 pakcet number space, packet number 在不同的 space 是独立增长和回复 ACK 的，这个是区别 TCP 的地方。但是 congestion control 和 RTT 的测量在不同的 space 是统一、全局计算的
2. packet number 是严格单调递增的
+ 每个 packet number 空间内，每次发送报文，即使是重传报文，packet number 也是递增的
+ packet nunmber 只是 transmission order 的含义，值越大表示越晚发送；用于表示保序的 delivery order 的是 stream 层的 offsets
+ 这样就消除了 TCP 之前的重传也用相同的 seq 的二义性问题
+ 没有了二义性，带来的好处就是：RTT 计算更加准确、spurious retransmissions 很容易被 detect、一些 fast retranmit 的算法更加准确
3. 丢包的周期更加清晰
+ 还是跟 tcp 进行对比的，在 tcp 里，在 recv pkt num 不连续有 gap 期间，无论重传报文丢了几次，都认为是一次，对应的拥塞控制也是一次
+ 而 QUIC 因为每次重传都是用的新的 pkt number , 所以有多次重复丢包的情况下，都可以被检测出来，会进行多次的拥塞控制
4. No Regeing
+ 即被不允许 ACK 反悔. 这块 draft 里写的很简单，靠一些猜测：
+ 就是说，被接收端 SACK 的报文(例如 paceket K)，接收端会接收下来，这个时候，发送端就不需要继续维护 packet K 对应的内存了；
+ 有什么好处呢？这个其实要对比下 TCP，因为在 TCP 中，被 SACK 的报文，在接收端其实属于不连续的报文，无法被应用层接收走，所以缓存在内核，而在内核内存压力大的时候，这个被 SACK 的报文可能被丢失了，所以发送端为了避免出现这种情况，是无法将被 SACK 的报文清除的，因为这种情况下需要发送端重传，这样就带来了，发送端会使用较多的内存；
+ 所以 QUIC 这个机制的好处，就是可以减少发送端内存的使用；
+ 那 QUIC 为什么可以这样做呢，就是因为 UDP 在内核里不需要按序被接收
5. 携带更多的 sack 字段
+ 即 QUIC 的 SACK 是没有像 TCP 一样的限制的
+ 因为 TCP 是受限制于 TCP Header 的 60B，所以 SACK 的数量是固定最多有 3 个；
+ 在存在大量丢包的情况下，更多的 SACK，可以提高重传的效率，减少不必要的重传
6. 更加准确的 delayed ack以计算rtt
+ 因为 quic 的接收端会将 “发送 ack 的时间 - 收到报文的时间” 这个时间差放到 ack 报文中返回给发送端
+ 所以发送端，可以根据 “接收到 ack 的时间 - 发送时间 - delay ack 的时间” 获取到准确的 rtt
7. 使用 PTO 取代 RTO 和 TLP
+ PTO 将 RTO 和 TLP 结合起来， timer 是基于 RTO 的计算公式
+ PTO 在超时的时候，不会立刻进入 cwnd 雪崩，而是可以超过 cwnd 一些继续发送报文
+ 可以避免 F-RTO 以及 TLP
+ 在检测到持续丢包的情况下，才会进入 cwnd 雪崩状态
8. 将最小的 congestion cwnd 设置为 2个 packets
+ TCP 的最小 cwnd 为 1，这样就会有尾丢弃的可能，导致恢复时间要超过一个 RTT
+ 所以 quic 将 cwnd 最小值设置为 2， 虽然会带来一些网络的压力，但是如果可以检测到持续拥塞，然后将窗口置指数下降的话，也是安全的

# 4.RTT 测量
一般 RTT 的计算方式是： 收到 ack 的时间 - 发送数据包的时间；因为 QUIC 里可以携带了 ack delay, 所以可以更加准确，计算主要是下面三个变量：
1. min_rtt:  整个 conn lifetime 里的最小 rtt
2. smoothed_rtt: EWMA rtt, 是每次测量到 rtt 的指数加权滑动平均
3. rttvar: 每次计算得到的 rtt 的方差
计算上面三个变量依赖 lastest_rtt

## latest_rtt（RTT sample)
计算公式：
```
latest_rtt = ack_time - send_time_of_largest_acked
```
ACK 需要满足下面的条件才会让 RTT sample：
1. 当前 ACK 了 largest packet number(即 ack 有更新)
2. 且对这个 larget pakcet number 的 ack 是最早的
3. largest acked packet number 中包含 ack-eliciting 报文

+ 条件 1 是因为：ack delay 是只针对 largest acked packet 的，尽管 RTT sample 这里不使用 ack delay, 但是 latest_rtt 对于后序计算 smoothed_rtt 以及 rttvar 是有作用的，而这些变量是需要 ack delay 的 
+ 条件2 是因为：防止对一个报文，多次计算 rtt, 导致不准 
+ 条件3 是因为：只有 ack-eliciting 的 ack 才是有效的

## min_rtt
1. 用途：loss detection, 作为 Lower band, 排除一些 rtt 计算异常过小的情况
2. 计算方法：
```
初始：min_rtt = sample rtt
之后：min_rtt = min(min_rtt, latest_rtt)
```

## smoothed_rtt
1. 含义：是对 RTT samples 做 EWMA，即指数加权移动平均，功能就是对 rtt sample 做一些滤波的能力来过滤大的波动。
2. 计算方法：
```
初始：smoothed_rtt = rtt_sample
之后：
// 1. 先计算 ack_delay， 从 ack frame 中获取到合法的 ack delay, 不超过设置的 max ack 
       delayack_delay = min(Ack Delay in ACK Frame, max_ack_delay)
// 2. 更新 SRTT
      adjusted_rttadjusted_rtt = latest_rtt 
      if (min_rtt + ack_delay < latest_rtt): 
         adjusted_rtt = latest_rtt - ack_delay
      smoothed_rttsmoothed_rtt = 7/8 * smoothed_rtt + 1/8 * adjusted_rtt
```

## rttvar
1. 含义：就是 rtt 的平均变化值
2. 计算方法：
```
// 1. 在 smoothed_rtt 的计算中，获取到 adjusted_rtt 之后，更新
      rrtvat_sample：rttvar_sample = abs(smoothed_rtt - adjusted_rtt)       
// 2. 计算 
      rttvar rttvar = 3/4 * rttvar + 1/4 * rttvar_sample​
```

# 5.丢包检测
QUIC 使用 ack 来判断丢包;  
probe timer 来保证 ack 被收到;  
如果判断一个 packet loss, 则 quic transport 需要从 loss 中 recovery, 包括：
1. 重传报文
2. 发送 updated frame
3. 放弃这个 frame

## 基于 ack 的丢包检测
QUIC 的 ack-based loss detection ，集成了如下算法的精神，每个算法都是一篇论文，比较牛逼:  
TCP Fast Retransmit,  
Early Retransmit,  
FACK,  
SACK,  
RACK,  

首先算法的核心内容：基于 ACK 的检测，如果满足以下两个条件，则认为，数据包丢失了：  
条件 1： 如果一个发出去的报文[A]，比他晚发出去的报文[B]被 acked, 而自己没有被 acked 的时候，且依然是 in-flight 状态  
条件2：  B - A 的 seq 相差一定的数量(packet thres)，或者发送间隔在一定时间(time thres)以上  
其中 条件1 表明比 A 发送晚的报文 ACK 了，那么没到的情况下，只能有两个可能：要么是有乱序、要么是这个报文丢了；  
所以 条件2 给了这个报文一定的乱序容忍，如果超过了这个则表明可能真的丢包了。但是，注意，这里只是可能，并不能很准确的判断。所以阈值很重要。

## 聊聊阈值选择
1. 很显然，如果丢包判断的阈值小了，那么，会有大量的非必要重传，英文叫：spuriously retransmit,  带来的危害包括：浪费带宽，然后拥塞控制会降速，导致 performace 下降
2. 如果丢包判断大了， 则会导致重传不及时，带来的危害就是，传输时间延迟增大
3. 同时，QUIC 会比 TCP 有更多的乱序，因为 QUIC 的 packet number 被隐藏了，所以中间设备没法基于 pakcet number 来做一些排序工作了

所以，阈值是个 tradeoff, 但是研究算法的人就很聪明，他们想了如下的方法：一开始在没有历史经验的情况下，将阈值设的小一些，然后随着判断无效重传多了，再将阈值太高。另外还有一些方法是，使用之前连接一些值作为参考。但是，这个方法依赖的是对 spuriously retransmit 的判断，究竟怎么判断这个？读者可以思考下。后面我们可以提示下。下面我们看这两个阈值的选择方法：

## packet threshold
1. 参考了 TCP 中的经验值，使用 packet thres = 3，这个文章里写到是 tcp 的 best practice。另外，这个值 should not less than 3, 以与 TCP 保持一致
2. 可以根据一些算法，例如 TCP-NCR 这个 RFC 中定义的一些方法来提高鲁棒性

## time threshold
计算公式:
```
time thres = max( kTimeThreshold * max(SRTT, latest_RTT), kGranularity)
 ```
在 largest acked packnum 收到之后，在这之前没有收到 ack 的 packet, 就要将定时器打开。  
kTimeThreshold 推荐值为 9/8；取 SRTT 和 lastest_RTT 中的较大值，还是出于对 reordering 的容忍来考虑的，RTT 的计算见上文；  
另外，由于一些 rtt 很小的网络环境，计算出来的 srtt 等可能为 0，所以如果直接使用上面的公式，会导致对于reordering  的 0 容忍，为了解决这个问题，draft 中定义了一个  kGranularity 的变量，这个变量约束了重传定时器的粒度必须要达到 ms 以上。这里  kGranularity  设置为 1ms

## probe timeout(PTO)
这里我更想称为  PTO timer,  本质还是一个定时器。  
PTO 参考了 TCP 的 TLP（Tail Loss Probe）、RTO、F-RTO.他的作用是在超时后发送一个或者两个 probe packet, 人如其名，目的就是为了探测对端的情况。探测什么呢？  
其实就是为了防止发送端最后一个报文丢了，那么因为没有比他再大的报文了，所以如果这样的话，就只能等待超时重传(RTO)，为什么这样，可以想象判断丢包的原理。而 RTO 的时间又比较长，所以为了尽快恢复丢弃的数据，需要有这个能力。  同理，也可以防止接收端最后一个回复的 ACK 丢了。

另外 PTO 也是每个 packet number space 有一个，这个也好理解，因为毕竟是跟丢包检测配合使用的，而丢包检测是基于 pakcet number.PTO 超时之后，只是发送 probe 报文，不能判断报文是否丢弃。当这个 probe 报文触发了新的 ack 报文到达后，可以根据这个 ack 来进行上述的 loss detection.

## PTO 值的计算
计算方法：
```
 PTO timer = SRTT + max(4*rttvar, kGranularity) + max_ack_delay
 ```

可以近似为一个 packet 从发送到收到 ACK 应该经历的最大时间，包括：评估得到的 rtt (srtt), rtt 的误差(rttvar),服务端可能  delay ack 的最大时间(max_ack_delay)

对 Init packet 或者 hanshake packet, max_ack_delay 因为还没有设置，所以这个值为 0. 另外之前提过，kGranularity 为 1ms,所以可以保证 PTO 的时间至少在 1ms.

这里看看 timer reset 的几种情况

1. 在握手完成的情况下，如果 ack-eliciting packet 被发送或者收到对应的 ack，这 pto timer reset.
2. 因为 pto 在不同的 space 中是独立的，所以各自管理。但是有一个特殊的，即 applicatipn space 的 timer 要在 handshake space 握手完成之后，再 work,主要就是为了防止接收端还没发解密 1rtt 报文，重传也是无用的

如果 pto timer 超时了，下次的 pto timer 的值对应翻倍。之所以翻倍是因为连续 pto 超时，可能是网络发生持续的拥塞，这个时候宁愿 RTO， 也不想 PTO 来增加拥塞。

如果对应的 ack 收到了，则之前 double 的值被重新 reset 掉。但是有一个地方特殊，就是如果没有 handshake done，还是不重置，理由是，这期间服务端可能在校验客户端地址，不需要频繁发送 probe.

最后连续 PTO 被终止的条件是触发了 RTO 超时。

另外，如果基于 time threshold 来检测丢包的 timer 开启了， pto 也可以不用开启。因为这个 timer 同样可以发现丢包，并且比 pto 会更加及时。

## Sending Probe Packet
关于发送报文的选择：

当 PTO timer 超时后，在对应的 packet number space, 发送端必须发送至少一个 ack-eliciting packet ，除非没有数据来发送。同时，为了避免 probe packet 的丢弃，尽量可以发送达到两个 packet. 注意这里的报文一定是 ack-eliciting

发 new data 还是 retransmitted unacked packet , 这里可以设置不同的策略，根据优先级，还是根据是否需要能快速重传。另外，new data 是受拥塞控制的，如果不允许发送 new data, 那就只能 retransmit 那些没有被 ack 的  packet.

如果极端情况碰到没有新数据也没有需要重传的数据的话，

另外，在具体的实践中，不同 packet number space 的 pto 可以相互作用，比如 application space 的 pto 超时，在握手没有完成期间，可以发送 handshake space 的数据，原因就是这个时候，服务端的 key 可能没有在所有 space 都获取到。

在 PTO 超时后，发送端也可以仅仅是标记报文丢了，不再发送 probe packet, 这带来的好处是可以少发送报文，但是可能会导致一些没必要的重传。

## 处理 retry packets
当收到 retry packets 的时候，sender 会：
1. reset congestion control
2. reset loss recovery state 包括一些 pending timers.
3. sender 会根据 retry 报文估算 rtt, 作为下次使用

## 丢弃 keys 与 packets state
1. 如果 keys discarded, 那么与这个 keys 相关的 packets 的 acks 便不能再解开，这样发送端需要将这些报文的 recovery state discard 掉，并从 bytes in flight 中去掉
2. 状态从 Initial -> handshake 的时候， Initial 报文的 recocy states 也可以去掉
3. 0rtt 报文被拒绝的时候，对于所有的 0rtt packets 的 recovery state is discard

# 6. 拥塞控制
只是提到一种算法，类似于 NewReno.

在拥塞信号的选择上，QUIC 本身提供了一种可扩展的机制，以适配不同的拥塞控制算法。

与 TCP 类似，只包含 ack frames 的 packets 不计算到 bytes in flight 里，也不受 congestion 的控制。

QUIC 的拥塞控制以 bytes 为调节，TCP 以 packets.

当 bytes in flight 大于 congestion window 的时候，发送端不允许(Must not)再发送报文。但是有个例外就是我们上文提到的，PTO timer 超时后，可以 retranstmit unacked packets. 但是不能发新报文。

## ECN
QUIC 也支持是否将 ECN 标志作为拥塞信号。如果 ECN 被置位，可以做相应的拥塞调节。
![image](https://github.com/Lingtaonju/understand_ietf_quic.io/blob/master/cc_recovery_images/ECN_IP.png)
![image](https://github.com/Lingtaonju/understand_ietf_quic.io/blob/master/cc_recovery_images/ECN_TCP.png)

## 初始/最小窗口
### 初始
依然是参考经典值，使用 10*max_datagram_size ，来自[RFC 6928]，如果考虑 Ip/udp 共 28字节的 overhead 的话，最多只有：14720 bytes 的长度

### 最小
推荐值是：2*max_datagram_size

我们知道在最开始的 reno 算法里，在 RTO 后是从 1 开始的，这里是2，就是这个值。

### Slow start
new reno 中的慢启动状态；

在这个状态中，congestion window 每次增加 ack 的都数据量大小，所以是 double.

出现丢包或者遇到 ENC-CE 标志的时候，才退出。

退出的时候，cwnd 减半或者，将 new cwnd 设置为 slow start threshold.

当出现持续拥塞进入 RTO, 再次会进入 Slow Start

## 拥塞避免
进入条件：slow start 退出后进行这个阶段

退出阶段：拥塞避免期间，再发现丢包或者 ECN-CE, 则进入恢复阶段

特点：使用基于 AIMD 的增长方式。当一个发送窗口内的数据都被 ack 后，新的发送窗口加1个packet size.

## 恢复阶段
进入条件：拥塞避免阶段，报文被判断为 loss/ecn-ce

退出条件：recovery 阶段发送的报文被 ack，区别于 tcp:  被判断为 Lost 的重传报文被 ack

特点：在 recovery 中再次发现丢包后，不再降速；且开始时，为了加速重重允许发送超过 cwnd 的报文，之后不再允许

## probe 超时
probe packets must not blocked by congestion cotroller.

但是 probe packets 还是要计入 in flight 的，这些报文也有可能导致网络的拥塞.

## 持续拥塞
When an ACK frame is received that establishes loss of all in-flight packets sent over a long enough period of time, the network is considered to be experiencing persistent congestion.

注意持续拥塞的判断这里有两点：

1.所有的 in-flight packet 都没有被 ack

2.持续的时间要足够的久

表现在连续的 PTO，kPersistentCongestionThreshold 建议值为 3

比如下图中的，在 t=8 时刻，即可以判断出是持续拥塞

因为在 t=8 时刻，已经发生了三次 pto，且从 t=0 到 t=7 没有报文被 ack.满足持续时间和丢包的条件

![image](https://github.com/Lingtaonju/understand_ietf_quic.io/blob/master/cc_recovery_images/Persiset_Cong.png)

持续拥塞之后，跟 TCP 的 RTO 一样，sender 将拥塞窗口降为最小值，我们前文提到过是 2.

## Pacing
draft 里并没有标准化到底使用什么 pacer.

主要可以将报文打散均匀发送，避免 burst 的手段都可以认为是 Pacer.

pacer 和拥塞控制配合使用, 比如 pacer 可以控制 cc 窗口中数据发送的顺序

及时的传输 ACK frames 对于有效的 loss recovery 是帮助的，所以如果 packets 中只有 ack frame,应该避免被 pacing,从而可以快速 delivery 到对方

如下所示，典型的 pacer 就是就 cc 窗口中的数据按照 srtt 的时间传输完，所以发送速率或者每个报文的发送间隔计算如下：

```
rate = N * congestion_window / smoothed_rtt
或者
interval = smoothed_rtt * packet_size / congestion_window / N 
```

这里的 N 是一个比 1 稍大的值，比如 1.25，目的是为了能够充分利用这里拥塞控制的窗口

## congestion window 低估
这种情况的表现形式为：

bytes in flight 小于 cc，但是 pacing 是不限制的

导致这一现象的原因一般是应用层数据不够充足，或者被 flow control 限制

## 安全特性
这一块主要是介绍拥塞相关的特性可能会被攻击者恶意伪造攻击

## Congestion signals
拥塞控制在很大程度上是跟拥塞信号请绑定的，比如丢包或者 ECN 信号。

onpath 攻击者，通过 drop 数据包或者篡改 ECN 标志都可以让对端判断出丢包，然后发送降速

## Traffic Analysis
因为 ACK frames 通常比较小，所以通过观察包大小攻击者可以判断出 ack 包的信息，针对 ack 的话，可以做一些恶意丢弃的攻击。

为了避免这种情况，ack frame 可以带一些 padding 或者跟其他的 frame 一起发送

## Misreporting ECN Markings
误报 ECN 标志位

因为数据接收端可能不希望发送端降速，所以可能会误报 ECN-CE 这个标志，或者将这个标志进行一些合并。

但是这样只会造成发送端发送的拥塞

为了避免这个行为，发送端可以刻意构造一些 ip 层的 ecn ce, 看接收端是否会回复 ecn-ce, 如果没有的话，则关闭 ECN
