
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

```python
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
```

```python
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
```

```python
OnCreditUpdate(Credit):
    if rccc:
        RCCC.OnCreditUpdate(Credit)
    update_state()
```

```python
OnInferredLoss(pkt_state, nominal_pktsize, pdc_present):
    if nscc:
        NSCC.OnInferredLoss(nominal_pktsize)
    mark_packet_for_rtx(pkt_state.psn, nominal_pktsize)
    if rccc:
        RCCC.OnInferredLoss(pdc_present)
    process_ev(pkt_state.entropy, TIMEOUT)
    ccc.inflight_pkts--
    update_state()
```

```python
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
```

```python
OnNewData(delta_backlog):
    ccc.backlog += delta_backlog
    if rccc:
        RCCC.OnNewData()
    update_state()
```

```python
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
```

```python
mark_packet_for_rtx(psn, nominal_pktsize):
    # marking the packet for retransmission is a PDC function, but there
    # are CC side effects below. Mark the psn for retransmission as soon
    # as allowed by cwnd
    ccc.waiting_rtx += 1
    ccc.rtx_backlog += nominal_pktsize
    update_state()
```

```python
unmark_packet_for_rtx(acked_pkt):
    #check if we were going to retransmit a packet that was just acked
    if acked_pkt is marked for retransmission:
        unmark the packet for retransmission
        ccc.waiting_rtx -= 1
        ccc.rtx_backlog -= acked_pkt.size
```

```python
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
```

```python
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

### 3.6.13.1 计算往返时间

NSCC 需要了解从 ACK 到达测量的 RTT 样本。分组数据汇聚协议（PDC）应存储每个已发送数据包的传输时间戳，并在每次数据包重传时更新此时间戳。通常，源端将从确认到达时间中减去存储的传输时间戳，并减去目的端的服务时间，以计算往返时间。但是，对于重传的数据包必须谨慎处理，因为存在确认是针对原始数据包还是重传数据包的潜在歧义。为了解决这种歧义，源端应存储数据包是否已被重传。数据分组携带`pds.flags.retx`位，指示该分组为重传分组。此`pds.flags.retx`标志在确认消息中回显。因此，如果 ACK 中的`pds.flags.retx`位未设置，且数据包已被重传，则绝不能使用该确认来更新往返时间测量值。

一个实现可以是，当 ACK 中的`pds.flags.retx`位被设置时，选择不更新 RTT 的测量值。然而，基于重传的确认来更新 RTT 是更可取的，但同样必须注意，当一个数据包第二次被重传时，要避免任何潜在的歧义。建议源端记录数据包是否进行了第二次（或后续）重传。如果数据包仅重传过一次，并且在相应的 ACK 中设置了`pds.flags.retx`位，那么更新 RTT 是安全的。否则，更新 RTT 是不安全的，不应进行更新。实现此功能的一种方法是维护一个2 bit 的`rtx_count`，用于记录数据包是否 a) 从未重传、b) 重传过一次，或 c) 重传过不止一次。为清晰起见，3.6.13.6 节中的伪代码假定采用这种实现方式，但也允许其他实现方式。

### 3.6.13.2 目的端流量控制（DFC）

目的端可以使用接收窗口惩罚值`Rcv_Cwnd_Pend`来调节源端的传输速率。如果目的端自身发生拥塞，无法跟上流量的到达速率，则应使用这种流控制方式。`pds.ack_cc_state.rcvr_cwnd_pend`字段会在ACK中返回，并且必须设置为0到127之间的值。例如：
- 将 `pds.ack_cc_state.rcvr_cwnd_pend` 设置为 **0** 不会对源节点产生流控作用，即该字段**无效**。
- 将其设置为 **127** 会以最大幅度减慢所有源节点的速率，相当于每个 RTT 只能发送一个数据包的窗口。
- 若在一个 RTT 的所有 SACK 报文中将其设置为 **64**，则会将所有源节点的拥塞窗口（cwnd）**减半**，从而降低接收端的带宽压力。

ACK 中的 `pds.ack_cc_state.rc` 标志表示源节点是否应该：
- 在 `rcvr_cwnd_pend` 指示接收端流控开始时，**保存当前的 cwnd 值**；
- 并在接收端流控结束时，**恢复原始的 cwnd 值**。

接收端可以根据目的端拥塞的程度**动态设置 `rcvr_cwnd_pend`**，该值的具体计算方式依赖于实现。若接收端不支持该功能，该字段可设为 0。

源节点在接收到非零的 `rcvr_cwnd_pend` 时，将通过修改拥塞控制策略做出响应，具体行为在 NSCC 的伪代码中的 `apply_cwnd_penalty()` 函数中定义。

### 3.6.13.3 NSCC 配置参数

以下配置参数和常量用于调优 NSCC 行为。实现**应**允许配置表 3-76 中列出的所有参数。

**实现说明：**  
表 3-76 中将若干参数的类型定义为浮点数。浮点运算的精度未作规定，由具体实现决定。可以为浮点运算选择任意有效的舍入模式。

参数 `config_base_rtt` 定义为在无其他流量情况下，一个 MTU 大小的数据包跨整个网络结构的最长路径的往返时间（单位：秒），由配置统一设置于所有 CCCs 中。为保持公式一致性，这里使用秒作为单位。实现时应将 `base_rtt` 保持在不超过 128 纳秒的精度，以匹配 `tx_timestamp` 的精度。

**BDP** 的默认值如下：
```
BDP = min(sender.linkspeed, receiver.linkspeed) * config_base_rtt
```


BDP 的单位为字节，因此上述 `linkspeed` 应使用字节/秒为单位。目标速率通常与源速率相同，如未知，则使用源速率。

**alpha** 的默认值如下：
```
alpha = 4.0 * scaling_a * scaling_b * MTU / target_qdelay
```

参数 `target_qdelay`、`qa_threshold` 和 `adjust_period_threshold` 均表示时间，在这些公式中使用秒为单位以保持一致性。实现时应将 `target_qdelay` 保持在不超过 128 纳秒的精度，以匹配 `tx_timestamp` 的精度。

`target_qdelay` 的默认值：

- 使用 trimming 时：`config_base_rtt * 0.75`
- 不使用 trimming 时：`config_base_rtt * 1.0`

参数 `qa_threshold` 在禁用 trimming 时使用；当启用 trimming 时，应设置为较大值以使其失效。`quick_adapt()` 动作需要在网络发生 tail dropping 前被触发，因此 `qa_threshold` 是 tail drop 阈值的导数。推荐设置如下：
```
qa_threshold = (drop threshold / Plane_BDP - 1) * config_base_rtt
```

使用推荐的 `tail drop threshold = 5 * Plane_BDP`，可得：
```
qa_threshold = 4 * target_qdelay
```

有关推荐交换机设置的详细信息见 3.6.17 节。

---

### 3.6.13.4 NSCC 源状态

当某个 NSCC CCC 被初始化时，以下 per-CCC 状态变量将被初始化：

#### 表 3-77 - NSCC 源状态

| 名称 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `ccc.cwnd` | 无符号整数 | MaxWnd | 拥塞窗口，限制在途数据（单位：字节） |
| `ccc.inflight` | 有符号整数 | 0 | 尚未确认的在途数据量（单位：字节），见下文说明 |
| `ccc.saved_cwnd` | 无符号整数 | 0 | 目的地流控启动时保存的原始 `cwnd` |
| `ccc.base_rtt` | 无符号整数 | config_base_rtt | CCC 的基础 RTT，初始为 `config_base_rtt`，若后续 RTT 样本更小则取更小值（单位：128 纳秒） |
| `ccc.achieved_bytes` | 无符号整数 | 0 | 用于快速调整的上一个 RTT 接收字节数（单位：字节） |
| `ccc.received_bytes` | 无符号整数 | 0 | 用于触发 `fulfill_adjustment()` 的接收字节数（单位：字节） |
| `ccc.fi_count` | 无符号整数 | 0 | 在无 ECN-CE 且 RTT 约为 base_rtt 的情况下看到的字节数（单位：字节） |
| `ccc.trigger_qa` | 布尔值 | FALSE | 到达快速调整触发时间后设为 TRUE，触发 `quick_adapt()` |
| `ccc.qa_endtime` | 无符号整数 | 0 | Quick_Adapt 时间窗口的结束时间（单位：本地时间单位） |
| `ccc.bytes_to_ignore` | 无符号整数 | 0 | `quick_adapt()` 后需忽略的数据量（单位：字节） |
| `ccc.bytes_ignored` | 无符号整数 | 0 | 已忽略的数据字节数（单位：字节） |
| `ccc.inc_bytes` | 无符号整数 | 0 | 累积窗口增长值，待后续应用 |
| `ccc.last_adjust_time` | 无符号整数 | now | 上次应用 `cwnd` 累积变动的时间（单位：本地时间单位） |
| `ccc.increase_mode` | 布尔值 | FALSE | 进入 `fast_increase` 模式时设为 TRUE，退出时清除 |
| `ccc.last_dec_time` | 无符号整数 | now | 上次执行 `multiplicative_decrease` 的时间（单位：本地时间单位） |
| `ccc.max_wnd` | 无符号整数 | MaxWnd | 基于实际 RTT 缩放后的 MaxWnd |

---

`ccc.inflight` 表示 CCC 记录的仍在途且尚未收到 ACK、NACK、超时，或其他推测丢失的所有 PDC 上的总字节数（包括包头，按标准包大小计算）。  
在某些情况下，`inflight` 可能会暂时出现略小于 0 的情况，因此该字段应使用**有符号整数**存储。

---

### 3.6.13.5 NSCC 源端算法

```python
def NSCC.OnACK(newly_rcvd_bytes, M_Flag, pkt_tx_state, ack_arrival_time, 
               Service_Time, Retx_Flag, Rcv_Cwnd_Pend, Restore_Cwnd):
    ccc.inflight -= newly_rcvd_bytes
    ccc.bytes_ignored += newly_rcvd_bytes
    ccc.received_bytes += newly_rcvd_bytes
    ccc.achieved_bytes += newly_rcvd_bytes

    rtt_sample = calculate_rtt(pkt_tx_state, ack_arrival_time, Service_Time, Retx_Flag)
    rcv_limit_mode = apply_cwnd_penalty(Rcv_Cwnd_Pend, Restore_Cwnd, newly_rcvd_bytes)

    if rtt_sample != INVALID_RTT:
        update_base_rtt(rtt_sample)
        delay = rtt_sample - ccc.base_rtt
        update_delay(delay)
    else:
        return

    if quick_adapt(is_loss=False, M_Flag, delay):
        return

    # ACK 中的 pds.flags.m 比特 表示请求中标记了 ECN
    # 以下为四种核心情况（其中三种有行为）：
    if M_Flag == 0 and delay >= target_qdelay and not rcv_limit_mode:
        fair_increase(newly_rcvd_bytes)
    elif M_Flag == 0 and delay < target_qdelay and not rcv_limit_mode:
        proportional_increase(newly_rcvd_bytes, delay)
    elif M_Flag == 1 and delay >= target_qdelay:
        multiplicative_decrease()

    # 我们已经积累了足够的窗口变化，现在应用它们
    if (now - ccc.last_adjust_time) >= adjust_period_threshold or ccc.received_bytes > adjust_bytes_threshold:
        fulfill_adjustment()
```

```python
def NSCC.OnInferredLoss(nominal_pktsize):
    cwnd = max(cwnd - nominal_pktsize, MTU)
    ccc.bytes_ignored += nominal_pktsize
    ccc.inflight -= nominal_pktsize
```

```python
def NSCC.OnNACK(nominal_pktsize, pkt_tx_state, nack_arrival_time, Retx_Flag, reason, rccc):
    adjust_cwnd = False
    ccc.inflight -= nominal_pktsize

    rtt_sample = calculate_rtt(pkt_tx_state, nack_arrival_time, service_time=0, Retx_Flag)
    if rtt_sample != INVALID_RTT:
        update_base_rtt(rtt_sample)

    # 如果 NACK 是由于 trimming 生成的，则使用 trimmed queue delay 的估计更新延迟
    if reason == UET_TRIMMED or reason == UET_TRIMMED_LASTHOP:
        update_delay(config_base_rtt)
        ccc.bytes_ignored += nominal_pktsize

    if reason == UET_TRIMMED or (reason == UET_TRIMMED_LASTHOP and rccc == False):
        # 仅当 RCCC 未启用时，才对 last hop trim 调整 cwnd
        adjust_cwnd = True
        ccc.trigger_qa = True

        # is_loss 会导致延迟被忽略，因此将延迟设置为0；
        # 一个剪切的数据包会被标记为ECN以实现快速适应，因此需要设置 M_Flag。
        if quick_adapt(is_loss=True, M_Flag=1, delay=0):
            adjust_cwnd = False  # 若 quick_adapt 已执行，则不再调整 cwnd

    if adjust_cwnd:
        cwnd = max(cwnd - nominal_pktsize, MTU)
```

```python
def NSCC.OnSend(nominal_pktsize):
    ccc.inflight += nominal_pktsize
```

```python
def NSCC.CanSend():
    # 如果需要，一个实现可以是准确的，并考虑 inflight + pktsize。注意：当CWND为一个MTU时，使用pktsize可能会导致小包的PDC饿死大包的PDC。这是调度器实现中需要处理的问题。
    return ccc.inflight + MTU <= ccc.cwnd
```

```python
def NSCC.AckRequest():
    # 如果目标在源的cwnd满时生成ACK，NSCC的性能会更好。如果cwnd不足以大到触发ACK，则请求ACK。
    return (ccc.cwnd - ccc.inflight) < MTU or ccc.cwnd < pds.ACK_Gen_Trigger
```

**实现说明：**
在NSCC.AckRequest()的测试中使用ACK_Gen_Trigger参数，这要求前端处理器（FEPs）对该参数进行对称配置。也允许采用其他实现方法，例如特定设备配置，以便向传输逻辑提供系统中ACK_Gen_Trigger的最小值。

此外，NSCC.AckRequest() 测试假定在调用 NSCC.AckRequest() 之前，此数据包的大小已计入 ccc.inflight 中。

---

### 3.6.13.6 NSCC Internal Functions
```python
# 计算 RTT
calculate_rtt(pkt_tx_state, ack_arrival_time, Service_Time, Retx_Flag):
    rtt_sample = INVALID_RTT
    # 其他算法可以决定当包被重传时 RTT 是否有效
    if (pkt_tx_state.rtx_count == 0 and Retx_Flag == FALSE)
     or (pkt_tx_state.rtx_count == 1 and Retx_Flag == TRUE):
        rtt_sample = ack_arrival_time – (pkt_tx_state.tx_timestamp + Service_Time)
    return rtt_sample

# 执行窗口调整
fulfill_adjustment():
    ccc.cwnd += ccc.inc_bytes / ccc.cwnd
    if (now - ccc.last_adjust_time) >= adjust_period_threshold:
        ccc.last_adjust_time = now
        ccc.cwnd += eta
    if ccc.cwnd > ccc.max_wnd:
        ccc.cwnd = ccc.max_wnd
    ccc.inc_bytes = 0
    ccc.received_bytes = 0

# 公平增长
fair_increase(newly_rcvd_bytes):
    #inc_bytes在被添加到cwnd之前会被除以cwnd
    ccc.inc_bytes += fi * newly_rcvd_bytes

# 比例增长
proportional_increase(newly_rcvd_bytes, delay):
    fast_increase(newly_rcvd_bytes, delay)
    if ccc.increase:
        return
    ccc.inc_bytes += alpha * newly_rcvd_bytes * (target_qdelay - delay)


# 乘性下降
multiplicative_decrease():
    ccc.increase = FALSE    # turn off fast increase
    ccc.fi_count = 0
    avg_delay = get_avg_delay()
    if avg_delay > target_qdelay:
        if (now - ccc.last_dec_time) > ccc.base_rtt:
            ccc.cwnd *= max(1 - gamma * (avg_delay - target_qdelay) / avg_delay, max_md_jump)
            ccc.cwnd = max(ccc.cwnd, MTU)
            ccc.last_dec_time = now

# 快速自适应
quick_adapt(bool is_loss, bool M_Flag, qdelay):
    if disable_quick_adapt == TRUE:
        return FALSE
    qa_done_or_ignore = FALSE

    if ccc.bytes_ignored < ccc.bytes_to_ignore and M_Flag == 1:
        # 我们仍处于“忽略字节”的阶段
        # 不要运行quick_adapt，而是重置 fulfill-adjustment 计数器
        qa_done_or_ignore = TRUE
    else if now >= ccc.qa_endtime:
        if ccc.qa_endtime != 0
         and (ccc.trigger_qa or is_loss or qdelay > qa_threshold)
         and (ccc.achieved_bytes < (ccc.max_wnd >> qa_gate)):
            # we have a trim packet or very large RTT
            ccc.cwnd = max(ccc.achieved_bytes, MTU)
            ccc.bytes_to_ignore = ccc.inflight
            ccc.bytes_ignored = 0
            ccc.trigger_qa = FALSE
            qa_done_or_ignore = TRUE
        ccc.achieved_bytes = 0
        ccc.qa_endtime = now + ccc.base_rtt + target_qdelay
    # If we are either in the bytes to ignore phase or ran quick adapt,
    # reset fulfill-adjustment counters
    if qa_done_or_ignore == TRUE:
        ccc.inc_bytes = 0
        ccc.received_bytes = 0

    return qa_done_or_ignore

# 快速增长
fast_increase(newly_rcvd_bytes, delay):
    if delay ~= 0:
        ccc.fi_count += newly_rcvd_bytes
        if ccc.fi_count > ccc.cwnd or ccc.increase:
            ccc.cwnd += newly_rcvd_bytes * fi_scale
            if ccc.cwnd > ccc.max_wnd:
                ccc.cwnd = ccc.max_wnd
            ccc.increase = TRUE
            return
    else:
        ccc.fi_count = 0
    ccc.increase = FALSE


# 更新 base RTT
update_base_rtt(raw_rtt):
    if ccc.base_rtt > raw_rtt:
        ccc.base_rtt = raw_rtt
        ccc.max_wnd = 1.5 * sender.linkspeed * (ccc.base_rtt in seconds)

# 应用 cwnd 惩罚
apply_cwnd_penalty(Rcv_Cwnd_Pend, Restore_Cwnd, newly_rcvd_bytes):
    if Rcv_Cwnd_Pend > 0:
        if ccc.saved_cwnd == 0:
            ccc.saved_cwnd = ccc.cwnd
        window_decrease = (Rcv_Cwnd_Pend * newly_rcvd_bytes) >> 7
        ccc.cwnd = min(ccc.cwnd, ccc.inflight)
        ccc.cwnd = max(mtu, ccc.cwnd - window_decrease)
    else if Restore_Cwnd and ccc.saved_cwnd > 0:
        ccc.cwnd = ccc.saved_cwnd
        ccc.saved_cwnd = 0
    return Rcv_Cwnd_Pend > 0


# 平均延迟用于乘性减 和 quick_adapt()。如何计算此值是特定于实现的决策。例如，在调用 update_delay() 时更新存储的 running average，并在调用 get_avg_delay() 时返回该值的实现就足够了。还允许使用其他方法来降低每 ACK 计算成本。

# 更新延迟（实现由具体实现者定义）
update_delay(delay):
    # 将样本加入平均延迟状态
    pass

get_avg_delay():
    # 返回最近 base_rtt 时间窗口内的平均延迟
    pass
```

### 3.6.13.7 初始化 base RTT

NSCC的公平性取决于获取良好的低往返时间（RTT）样本以作为基础往返时间（base_rtt）的初始值。当本地流启动并与预先存在的长距离流竞争时，情况尤其如此。在这些情况下，本地流可能无法很好地测量基础往返时间（base_rtt），从而导致对长距离流不公平。

当建立到某个目的地的新NSCC CCC时，源端应使用DSCP_CONTROL区分服务代码点，在控制流量类别上发送探测控制分组（Probe CP），以获取良好的基本往返时间（base_rtt）样本。由于探测控制分组无法建立分组数据连接（PDC），如果立即使用控制流量类别发送此探测控制分组，它很可能在PDC的第一个数据包之前到达，进而被丢弃。因此，源端应等待，直到接收到第一个确认（ACK）或否定确认（NACK）后，再发送探测控制分组。如果某个实现方案知道在第二个往返时间之后不会再发送更多数据，则可以安全地省略此探测控制分组。 

如果源端具有来自先前连接建立尝试（CCC）的缓存基本往返时间（base_rtt）、能够可靠地估计基本往返时间，或者正在处理已知传输数据量较少（通常约为一个带宽时延积，即BDP）的路径诊断码（PDC），则发送探测控制包（Probe CP）是不必要的。


### 3.6.13.8 NSCC目标状态

NSCC 主要是一种基于发送方的算法。它依靠在接收端计算出的 Rcvd_Bytes 值来驱动其 ACK 时钟。Rcvd_Bytes 是按每个 PDC 进行维护的。

一个NSCC目的端维护以下拥塞控制（CC）状态：

#### 表 3-78 - NSCC 目的端状态

| 名称 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `pdc.rcvd_bytes` | 无符号整数 | 0 | 该PDC接收的总的新字节数量（单位：字节） |

### 3.6.13.9 NSCC目的端算法

```python
OnRX(pkt):
    if pkt contains data and pkt is not trimmed and pkt is not a duplicate:
        pdc.rcvd_bytes += pkt.nominal_pktsize
```
从目的地发送的 ACK 中的`pds.ack_cc_state.rcvd_bytes`字段以256字节为单位。它是根据以下方式从pdc.rcvd_bytes得出的：
```
Rcvd_Bytes = ceil(pdc.rcvd_bytes / 256.0)
```
该值需要向上取整，以避免当剩余未确认字节小于256字节时，永远不确认小数据包。接收字节（Rcvd_Bytes）值也可以通过整数运算等效计算为：
```
Rcvd_Bytes = (pdc.rcvd_bytes + 255) >> 8
```

## 3.6.14 UET接收方信用拥塞控制

### TODO

## 3.6.15 Transport Flow Control (TFC)

### TODO






------------

### 注释

[^1]: 在一些实现中，有可能使用实际的数据包大小而非最大传输单元（MTU）。