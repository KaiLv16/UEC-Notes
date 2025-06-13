
## 3.6.12 总体拥塞控制拥塞控制（CCC）伪代码

CCC算法分为与选择哪种CC算法无关的通用部分，以及针对NSCC和RCCC的特定部分。本节规定了处理通用状态和负载均衡的通用部分。


### 通用CCC状态：

|state| explanation|
|:----|:-----|
| ccc.backlog = 0 | ccc.backlog保存了PDS指示CCC在与该CCC相关联的所有PDC上通过此CCC发送的总字节数（包括根据标称数据包大小计算得出的包头）。|
|ccc.waiting_rtx = 0|ccc.waiting_rtx保存当前标记为需要重传的数据包数量。|
|ccc.rtx_backlog = 0|ccc.rtx_backlog 表示 PDS 在与该 CCC 相关联的所有 PDC 上，针对此 CCC 有待重传的总字节数（包括根据标称数据包大小计算得出的包头）。|
|ccc.inflight_pkts = 0|ccc.inflight_pkts保存了此CCC上与该CCC相关联的所有PDC上正在传输的数据包总数。|

3.6.12.3节中的伪代码针对许多值引用了数据包的大小。在有说明的地方，这些使用标称数据包大小（nominal_pktsize）。即使数据包长度在传输过程中因（例如）网络遥测等原因发生变化，源端和目的端得出相同的值也很重要。标称数据包大小（nominal_pktsize）是实际数据包大小的近似值，对于拥塞控制的目的而言足够准确，但在传输过程中不能改变。
```
nominal_pktsize = transport_pktsize + 40
```
如果使用UDP封装，transport_pktsize是数据包的UDP长度；如果不使用UDP封装，它是从UET熵头开始到UET尾部结束的数据包大小。

### 3.6.12.3 总体CCC伪代码

``` py
# 3.6.12.3 Overall CCC Pseudocode
OnACK(newly_rcvd_bytes, Entropy, M_Flag, pkt_tx_state, ack_arrival_time, Service_Time, Retx_Flag, Rcv_Cwnd_Pend, Restore_Cwnd):
    #Each packet that was ACKed for the first time (e.g., by a cumulative ACK or a SACK bit being set) should not be retransmitted. Remove it from the retransmit list if it is there.
    foreach newly acked packet:
        unmark_packet_for_rtx(acked_pkt)

    if M_Flag == 1: #pkt was ECN marked
        process_ev(Entropy, ECN)
    else: 
        process_ev(Entropy, NO_ECN)

    if nscc:
        NSCC.OnACK(newly_rcvd_bytes, M_Flag, pkt_tx_state, ack_arrival_time, Service_Time, Retx_Flag, Rcv_Cwnd_Pend, Restore_Cwnd)

    if rccc:
        RCCC.OnACK()
    #Each ACK may acknowledge multiple packets (cumulative ACK, SACK, etc)
    ccc.inflight_pkts -= number_of_newly_acked_packets
    update_state()


OnNACK(nominal_pktsize, NACK_PSN, Entropy, M_Flag, pkt_tx_state, nack_arrival_time, reason, Retx_Flag, pdc_present):
    if reason == TRIM_NACK: # Non-last hop trim
        process_ev(Entropy, NACK)
    else if M_Flag == 1: # pkt was ECN marked before it was trimmed
        process_ev(Entropy, ECN)
    else: #last hop trim, or other cause – no need to load balance
        process_ev(Entropy, NO_ECN)
    
    if nscc:
        NSCC.OnNACK(nominal_pktsize, pkt_tx_state, nack_arrival_time, Retx_Flag, reason, rccc)
    
    mark_packet_for_rtx(NACK_PSN, nominal_pktsize)
    if rccc:
        RCCC.OnNACK(pdc_present)
    
    ccc.inflight_pkts--
    update_state()


OnCreditUpdate(Credit):
    if rccc:
        RCCC.OnCreditUpdate(Credit)
    update_state()


OnInferredLoss(pkt_state, nominal_pktsize, pdc_present):
    if nscc:
        NSCC.OnInferredLoss(nominal_pktsize)
    mark_packet_for_rtx(pkt_state.psn, nominal_pktsize)
    if rccc:
        RCCC.OnInferredLoss(pdc_present)
    process_ev(pkt_state.entropy, TIMEOUT)
    ccc.inflight_pkts--
    update_state()


OnSend(nominal_pktsize, is_rtx):
    if is_rtx:
        ccc.waiting_rtx -= 1
        ccc.rtx_backlog -= nominal_pktsize
    else:
        ccc.backlog -= nominal_pktsize

    if nscc:
        NSCC.OnSend(nominal_pktsize)
    if rccc:
        RCCC.OnSend(nominal_pktsize)
    ccc.inflight_pkts++
    update_state()


OnNewData(delta_backlog):
    ccc.backlog += delta_backlog
    if rccc:
        RCCC.OnNewData()
    update_state()


GetSendParams(free_port_list) -> port, entropy, credit_target, ack_req:
    determine entropy and NIC port
    if rccc:
        credit_target = RCCC.computeCreditTarget()
    ack_req = FALSE
    if nscc:
        #ack_req is TRUE if the CCC would like AR to be set. 
        #PDS may also set AR for other reasons.
        ack_req = NSCC.AckRequest()
    return port, entropy, credit_target, ack_req


mark_packet_for_rtx(psn, nominal_pktsize):
    # marking the packet for retransmission is a PDC function, but there
    # are CC side effects below. Mark the psn for retransmission as soon
    # as allowed by cwnd
    ccc.waiting_rtx += 1
    ccc.rtx_backlog += nominal_pktsize
    update_state()


unmark_packet_for_rtx(acked_pkt):
    #check if we were going to retransmit a packet that was just acked
    if acked_pkt is marked for retransmission:
        unmark the packet for retransmission
        ccc.waiting_rtx -= 1
        ccc.rtx_backlog -= acked_pkt.size


update_state():
    if ccc.backlog == 0 and ccc.waiting_rtx == 0: #no more data to send
        if ccc.inflight_pkts == 0:
            ccc.state = IDLE
        else:
            ccc.state = PENDING #still waiting for Acks

    else if ccc.state == IDLE or ccc.state == PENDING:
        ccc.state = ACTIVE

    can_send = TRUE #does each active CC algorithm allow sending?
    if ccc.state == ACTIVE or ccc.state == READY:
        if nscc: 
            can_send &= NSCC.CanSend()
        if rccc:
            can_send &= RCCC.CanSend()
    
    if ccc.state == ACTIVE and can_send:
        ccc.state = READY #add CCC to scheduler ready list
    else if ccc.state == READY and not can_send:
        ccc.state = ACTIVE #remove CC from scheduler ready list


process_ev(entropy, reason):
    # UET-CC并未规定如何使用从确认（ACK）和否定确认（NACK）接收的反馈信息来进行负载均衡。如3.6.16节所述，盲目负载均衡和自适应负载均衡都是允许的。一些负载均衡方案在process_ev() 中几乎不需要（或根本不需要）采取行动，而其他方案可能会根据原因采取不同的行动。对于给定原因不需要处理熵的负载均衡方案，无需针对该原因处理熵。如果CCC用于ROD或TFC PDC，process_ev() 不执行任何操作。 
    pass
```

#### receiver algorithm

以下伪代码定义了在CCC目的地的通用CCC行为。发送确认（ACK）的条件在PDS第3.5.12节中有规定。RCCC还维护一个信用计时器，用于计算信用超时。这在第3.6.14.5节中有详细说明。

``` py
OnRX(pkt):
    if rccc:
        RCCC.OnRX(pkt)
    if nscc:
        NSCC.OnRX(pkt)
    if pkt.IsTrimmed:
        lastHopTrim = (pkt.ip.dscp == DSCP_TRIMMED_LASTHOP)
        sendNACK(pkt.pds.psn, Entropy, lastHopTrim, pkt.ip.ecn.ce)
    else if conditions are met to send an ACK:
        sendACK(pkt.pds.psn, Entropy, pkt.ip.ecn.ce) 
        #other ACK CC_STATE fields will be filled in by the relevant CC algorithm
```

## 3.6.13 NSCC (基于网络信号的拥塞控制)

**本节指定了NSCC算法，该算法基于SMaRTT [9]和Strack [10]算法。**

NSCC使用拥塞窗口cwnd来限制未完成的飞行中数据量。NSCC仅在\(cwnd > (inflight + MTU<sup>[^1]</sup>)\)时才允许发送。与基于窗口的拥塞控制协议一样，NSCC依靠ACK时钟来调节允许进入网络的数据量，ACK的到达表明数据已离开网络，从而将新数据输入网络。在NSCC中，减少飞行中数据量的ACK时钟由目的地在ACK数据包中返回的Rcvd_Bytes字段驱动，该字段表示已到达目的地的数据量。当目的地告知源一定数量的数据已离开网络时，飞行中数据量将减少该数量，从而允许发送更多数据。同样，如果已知或推断某个数据包丢失，飞行中数据量也会减少。这些机制能够抵抗多路径喷射导致的乱序。请注意，由于Rcvd_Bytes向上取整为256字节的单位以及ACK乱序，飞行中数据量可能会暂时略微为负；这并不影响正确性，但确实要求将飞行中数据量维护为有符号值。

为跨多条路径发送的数据包维护单个拥塞窗口（cwnd）；实际上，它跟踪的是网络中多条路径聚合后的可用容量。核心的网络状态拥塞控制（NSCC）算法根据观察到的网络状况调整cwnd。主要控制循环由显式拥塞通知（ECN）和测量的往返时间（RTT）共同驱动。排队延迟近似为RTT - base_RTT，其中base_RTT是无队列时的预期延迟。RTT的测量方法是在源端存储每个数据包的发送时间，然后从确认（ACK）数据包中报告的ACK到达时间中减去该发送时间以及服务时间（Service_Time）。目标是将延迟控制在一定范围内，但只有在数据包花费时间通过队列后才能测量到高延迟。相比之下，在现代以太网交换机中，当数据包从队列中出队时，会根据当时的队列大小为其设置ECN。这意味着单比特的ECN信号是拥塞的先行指标，而延迟是滞后的多比特指标。目的端将接收到的ECN值回显给源端。这就产生了ECN和延迟的四种组合，用于确定适当的响应：

- **未设置 ECN，且延迟 < 目标队列延迟（target_qdelay）：**  
  - 网络未发生拥塞  
  - 执行 `proportional_increase()`  

- **未设置 ECN，且延迟 ≥ 目标队列延迟：**  
  - 网络曾发生拥塞，但拥塞已经缓解  
  - 执行 `fair_increase()`  

- **设置了 ECN，且延迟 < 目标队列延迟：**  
  - 网络开始对当前数据包后的数据变得拥塞，但尚未被视为真正拥塞  
  - 无需增加或减少  

- **设置了 ECN，且延迟 ≥ 目标队列延迟：**  
  - 网络已发生拥塞  
  - 执行 `multiplicative_decrease()`  

针对每种情况，系统将采取不同的拥塞响应策略。

在网络未拥塞时执行的`proportional_increase()`操作，默认情况下会根据延迟与目标延迟之间的差值按比例增加拥塞窗口（cwnd）。因此，如果网络负载较轻，其增长速度会比接近拥塞阈值时更快。如果网络在一段时间内负载不足，例如在竞争流终止时可能出现这种情况，`proportional_increase()`可以触发`fast_increase()`，以便更快地收敛到新的工作点。`fast_increase()`将执行指数增长，并在出现任何初期拥塞迹象时终止。

在数据包经历拥塞后执行`fair_increase()`操作，但显式拥塞通知（ECN）表明其后方队列已排空至低于ECN设置阈值。`fair_increase()`操作启动以防止传输速率过低。它执行加法增加，这有助于竞争的拥塞控制算法（CCC）趋向公平：所有接收到相同信号的CCC将把拥塞窗口（cwnd）增加相同的量，因此，拥塞窗口较小的CCC相比拥塞窗口较大的CCC将获得更大的比例增长。

当延迟超过阈值且显式拥塞通知（ECN）未表明队列正在减少时，将执行`multiplicative_decrease()`操作。在这种情况下，平均延迟直接反映了入队数据比预期多出的量。然后，`multiplicative_decrease()`操作会根据这种队列超额情况按比例直接减少拥塞窗口（cwnd）。如果在这种情况下所有流都执行相同的操作，目标是在略多于一个往返时间（RTT）内将队列减少到目标值。

最后，当检测到丢包或多条路径的延迟过高时，会执行`quick_adapt()`。`quick_adapt()`使用测量得到的吞吐量直接设置拥塞窗口（cwnd）。这意味着，例如，如果发生了 incast，在incast的分组数据汇聚（PDC）的第二个往返时间（RTT）中，拥塞窗口将合并，以避免目的链路过载。`quick_adapt()`操作并不能保证即时公平性，但在后续的往返时间中，公平增长（`fair_increase()`）和乘法减小（`multiplicative_decrease()`）将导致收敛。由于`quick_adapt()`依赖于对已实现带宽的测量，所以每个往返时间只能触发一次。当`quick_adapt()`函数被触发时，可能会有许多数据包排队，并且其中大多数数据包在确认时会设置ECN（显式拥塞通知）。在这些数据包被确认之前，`multiplicative_decrease()`函数会被抑制。

NSCC旨在能够与基于主动式ECN的负载均衡协同工作，在此情况下，ECN信号除了用作NSCC拥塞信号外，还用于平衡各路径间的拥塞情况。因此，NSCC不会对仅由ECN指示而无延迟体现的低程度拥塞做出反应，因为这往往是负载均衡不完善的一种表现。如果使用基于ECN的负载均衡，除了在ECN信号饱和且仅用于降低总负载的高拥塞情况外，最好在总速率降低之前先让负载均衡有机会做出反应。

如果有需要，NSCC将处理被排斥情况（例如，当多个CCC处于活动状态，且飞行中的数据量从未接近拥塞窗口（cwnd）限制时）的任务留给实现者。在被排斥期间，多个发送CCC的拥塞窗口（cwnd）有可能增加到超出传出FEP发送能力的最大值。出现这种情况是因为每个CCC都在其标称允许速率以下运行，而对NSCC来说网络延迟似乎较低。

#### 3.6.13.1 计算往返时间

NSCC 需要了解从 ACK 到达测量的 RTT 样本。分组数据汇聚协议（PDC）应存储每个已发送数据包的传输时间戳，并在每次数据包重传时更新此时间戳。通常，源端将从确认到达时间中减去存储的传输时间戳，并减去目的端的服务时间，以计算往返时间。但是，对于重传的数据包必须谨慎处理，因为存在确认是针对原始数据包还是重传数据包的潜在歧义。为了解决这种歧义，源端应存储数据包是否已被重传。数据分组携带`pds.flags.retx`位，指示该分组为重传分组。此`pds.flags.retx`标志在确认消息中回显。因此，如果 ACK 中的`pds.flags.retx`位未设置，且数据包已被重传，则绝不能使用该确认来更新往返时间测量值。

一个实现可以是，当 ACK 中的`pds.flags.retx`位被设置时，选择不更新 RTT 的测量值。然而，基于重传的确认来更新 RTT 是更可取的，但同样必须注意，当一个数据包第二次被重传时，要避免任何潜在的歧义。建议源端记录数据包是否进行了第二次（或后续）重传。如果数据包仅重传过一次，并且在相应的 ACK 中设置了`pds.flags.retx`位，那么更新 RTT 是安全的。否则，更新 RTT 是不安全的，不应进行更新。实现此功能的一种方法是维护一个2 bit 的`rtx_count`，用于记录数据包是否 a) 从未重传、b) 重传过一次，或 c) 重传过不止一次。为清晰起见，3.6.13.6 节中的伪代码假定采用这种实现方式，但也允许其他实现方式。

#### 3.6.13.2 目的端流量控制（DFC）

目的端可以使用接收窗口惩罚值`Rcv_Cwnd_Pend`来调节源端的传输速率。如果目的端自身发生拥塞，无法跟上流量的到达速率，则应使用这种流控制方式。`pds.ack_cc_state.rcvr_cwnd_pend`字段会在ACK中返回，并且必须设置为0到127之间的值。例如：
- 将 `pds.ack_cc_state.rcvr_cwnd_pend` 设置为 **0** 不会对源节点产生流控作用，即该字段**无效**。
- 将其设置为 **127** 会以最大幅度减慢所有源节点的速率，相当于每个 RTT 只能发送一个数据包的窗口。
- 若在一个 RTT 的所有 SACK 报文中将其设置为 **64**，则会将所有源节点的拥塞窗口（cwnd）**减半**，从而降低接收端的带宽压力。

ACK 中的 `pds.ack_cc_state.rc` 标志表示源节点是否应该：
- 在 `rcvr_cwnd_pend` 指示接收端流控开始时，**保存当前的 cwnd 值**；
- 并在接收端流控结束时，**恢复原始的 cwnd 值**。

接收端可以根据目的端拥塞的程度**动态设置 `rcvr_cwnd_pend`**，该值的具体计算方式依赖于实现。若接收端不支持该功能，该字段可设为 0。

源节点在接收到非零的 `rcvr_cwnd_pend` 时，将通过修改拥塞控制策略做出响应，具体行为在 NSCC 的伪代码中的 `apply_cwnd_penalty()` 函数中定义。






### 注释

[^1]: 在一些实现中，有可能使用实际的数据包大小而非最大传输单元（MTU）。