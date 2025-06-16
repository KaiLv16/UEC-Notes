### 数据包传输模式（Delivery Modes）

可靠性和有序性之间的四种组合定义了数据包传输模式。每一种组合称为一种传输模式：

- **可靠无序传输（Reliable unordered delivery, RUD）**（详见第 3.5.7.1 节）
- **可靠有序传输（Reliable ordered delivery, ROD）**（详见第 3.5.7.2 节）
- **幂等操作的可靠无序传输（Reliable unordered delivery of idempotent operations, RUDI）**（详见第 3.5.7.3 节）
- **不可靠无序传输（Unreliable unordered delivery, UUD）**（详见第 3.5.7.4 节）

其中，RUD 和 ROD 这两种可靠传输模式是在动态建立的 PDC（Packet Delivery Channel）上下文中定义的。PDC 的相关内容详见第 3.5.8 节。而 RUDI 和 UUD 模式则不使用 PDC。

一旦建立了 PDC，该 PDC 上的所有数据包必须使用相同的传输模式，直到该 PDC 被关闭。SES（Session Endpoint Subsystem）决定每个 SES 请求应使用的传输模式。数据包如何被分配到 PDC 的方式详见第 3.5.8.1 节。必须支持在一对 FEP（Fabric Endpoint）之间建立多个 PDC。决定何时建立多个 PDC 的标准由具体实现自行决定。不同 PDC 之间的数据包不提供有序性保证。

对于一个 SES 消息，其所有数据包必须通过同一个 PDC 发送。但某些 SES 事务 —— 如 rendezvous（集合）或 deferrable send（可延迟发送）—— 可能包含多个消息，而这些消息可能会分布在不同的 PDC 上。

若应用希望允许将大型数据传输分散到多个 PDC 上，必须在 SES 层之上将传输拆分成多个消息，因为多个消息可能被映射到不同的 PDC。

- 在使用 **rendezvous** 时，消息中的 eager 部分可以使用与消息其他部分不同的 PDC。
- 在使用 **deferrable send** 时，RTR（Restart transmission request）将使用不同的 PDC（由 deferrable send 的目标端发起），数据传输也可能使用与原始 deferrable send 不同的 PDC。

对上述两种情况的处理方式由实现自行定义。

> **实现说明（Implementation Note）**
> 
> 由于在两个 FEP（Fabric endpoint）之间可能存在多个 PDC（Packet delivery context），PDS（Packet delivery sublayer）需要追踪某条消息的第一个数据包是通过哪个 PDC 发送的。后续属于同一消息的所有数据包必须映射到同一个 PDC。
> 
> PDS 利用 `ses.som`（Start of Message）和 `ses.eom`（End of Message）字段来确保同一消息的所有数据包使用相同的 PDC，并防止在消息尚未传输完成时关闭该 PDC。
> 
> 一旦某个 **deferrable send（可延迟发送）** 被推迟并通过 RTR（Restart transmission request）重新启动，该事务将被视为一个**新消息**，因此可以使用**不同的 PDC**。



**不同消息的数据包** **不得** 在同一个 PDC 上交错传输。具体而言：在另一条消息的数据包上链路之前，**当前消息的所有数据包必须已传输完毕**。整条消息会在一个 PDC 上完成传输，然后再开始传输下一条消息。为了避免阻塞短消息，PDS 可以在两个 FEP 之间建立额外的 PDC。

对于被标记为 **需要保证交付（guaranteed delivery）** 的 SES 响应，携带它的 **PDS ACK** 必须确保这些响应被成功交付。特别是，当位于 RUD 或 ROD PDC 上的 SES 响应被标记为“需要保证交付”时，PDS 有责任确保该响应被交付并清除。相关的“哪些响应需要保证交付”的定义，详见 **语义（Semantics）章节 3.4.3.3**。有关 ACK 的保证交付与清除的更多细节，请参阅 **第 3.5.11 节**。

- 对于**不可靠传输模式（UUD）**，**不会进行任何确认（acknowledgement）**。

PDC 和 CCC 是一对一或者多对一的关系。