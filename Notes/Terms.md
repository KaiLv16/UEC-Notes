| 类别 Class | 术语 Term（英文） | 术语 Term（中文） | 描述 Description（中文） |
|------------|------------|------------------|-------------|
| Operating System  | CPU   | 中央处理器 | 通用的处理器，用于执行各种计算功能。 |
|  | Memory | 存储器 | 用于存放 CPU 和加速器所使用的指令和数据的电子存储装置。 |
|  | Operating system instance (OSI) | 操作系统实例  | 操作系统的一个实例（例如虚拟机）。  |
|  | Process | 进程   | 在特定用户拥有的 OSI 上执行的程序实例，拥有私有虚拟地址空间（VAS）。进程由操作系统分配唯一的进程ID标识。 |
|  | Process address space ID (PASID) | 进程地址空间ID | 每个 OSI 内虚拟地址空间的唯一标识。目标 PASID 可基于 JobID 和 PIDonFEP 的组合确定。 |
|  | Process ID (PID)  | 进程ID | 操作系统分配给进程的标识符。   |
|  | User  | 用户   | 有权限访问集群节点并能执行计算进程的实体。   |
|  | Virtual address space (VAS) | 虚拟地址空间  | 特定进程用于内存分配的虚拟地址空间。 |
| Fabric Communication  | Absolute addressing | 绝对寻址 | 客户端/服务器 任务模型中定位资源的一种寻址方式。包括目的地址的三部分：（1）标识 FEP 的 FA，（2）无 JobID 的 PIDonFEP，（3）资源索引。  |
|  | Accelerator | 加速器 | 专门设计用于高效执行特定功能的计算模块或设备。   |
|  | Acknowledgment packet (ACK) | 确认包（ACK） | UET 用于实现可靠性的包，由目的 PDC 发送给源 PDC，表示在 PDS 层成功接收包。ACK 可以携带语义响应。 |
|  | Best-effort network | 尽力网络 | 与无损网络相对，网络中的包在某些链路上发送时，发送方与链路端点之间没有显式的缓冲区可用性通信，可能因缓冲不足而丢包。 |
|  | Cluster | 集群   | 由一个或多个 fabric 平面连接的节点集合。 |
|  | **Congestion control context (CCC)** | 拥塞控制上下文  | 用于控制两个 FEP 之间 RUD 和 ROD 流量的数据传输速率，有时一个 CCC 由多个 PDC 共享。 |
|  | Congestion management sublayer (CMS) | 拥塞管理子层   | Ultra Ethernet Transport (UET) 协议中负责管理拥塞的部分。   |
|  | Entropy value (EV) | 熵值 | 包头中的字段值（如 UDP 源端口），用于在 fabric 内负载均衡包的路径选择。 |
|  | Fabric | Fabric 平面 | 一个或多个 fabric 平面。 |
|  | Fabric address (FA) | Fabric 地址 | IPv4（RFC 791）或 IPv6（RFC 8200）地址。 |
|  | Fabric endpoint (FEP)   | Fabric 端点 | 通过单一（分配的）FA 可寻址的逻辑实体。UE 传输协议及可选的安全上下文终止于 FEP。FEP 使用端口连接到 fabric，且仅能被单个 OSI 使用。FEP可以使用单个FA拥有多个端口，前提是每个端口都连接到完全隔离的 fabric 平面。一个节点可有多个 FEP。   |
|  | Fabric path or path | Fabric 路径 | 包从源 FEP 到目的 FEP 传输经过的有序链路集合（节点和交换机跳数）。包可沿多条路径或多 fabric 平面传输。   |
|  | Fabric plane or plane   | Fabric 平面 | 由 FEP、链路和交换机（可选）组成的集合，允许集合内任意 FEP 之间通信。不同 fabric 平面间通信超出本规范范围。 |
|  | Folded Clos | 折叠 Clos | 一种由交叉开关组成的多级网络拓扑，也称为 fat tree。  |
|  | Frame | 帧   | Ethernet 网络中使用第2层封装的数据传输单元，以 MAC 地址开头，以 CRC 结尾。   |
|  | Initiator | 发起端 | 发起 RUD 和 ROD 模式下 PDC 创建的 FEP。  |
|  | Link | 链路   | 两个端口之间的物理连接。 |
|  | Lossless network  | 无丢包网络 | 所有网络设备避免因缓冲溢出导致丢包，只在确认链路端有可用缓冲区时才发送包的网络。  |
|  | Message   | 消息   | 具有相同消息 ID 的一个或多个包。消息在源端被划分为空间有序的包集合。一个或多个消息及其支持包组成一个事务。 |
|  | Negative acknowledgement packet (NACK) | 否认包（NACK） | UET 用于实现可靠性的包，由目的 PDC 发送给源 PDC，显式指出包丢失。 |
|  | Node | 节点   | 拥有一个或多个 FEP 的计算设备，可能含一个或多个 CPU 和/或加速器。 |
|  | Packet | 包 | 在网络上传输的 IPv4 或 IPv6 数据报。包沿路径路由，且同一 PDC 的包经不同路径可能乱序到达。   |
|  | Packet delivery context (PDC) | 包传输上下文  | 两个 FEP 之间的逻辑单向实体（由传输建立），存在于发起和目标 FEP，控制包的成功传输。   |
|  | Packet delivery sublayer (PDS) | 包传输子层   | UET 协议负责在 IP/Ethernet 网络上传递包并保证排序和可靠性的部分。  |
|  | Packet window | 包窗口 | 两个 FEP 间拥塞控制上下文中最大未确认字节数。  |
|  | Port | 端口   | 单个 IEEE 802.3 定义的 MAC 及 UE 扩展。  |
|  | Relative addressing | 相对寻址  | 并行作业模型中用于资源寻址以确保扩展性的方式。包括四部分：（1）标识 FEP 的 FA，（2）唯一标识集群中作业的 JobID，（3）本地 PIDonFEP（0 到 P-1），（4）资源索引。   |
|  | Semantic sublayer (SES)   | 语义子层  | UET 协议中实现 OFI libfabric API 的部分。 |
|  | Switch | 交换机 | 具有两个或以上端口，基于包的 FA 或其它信息转发包的设备。  |
|  | Target | 目标端 | 响应发起端 PDC 创建的 FEP。目标端不发起消息，仅响应发起端。   |
|  | Traffic class (TC) | 流量类别  | 网络流量分类，标识端点和交换机中用于隔离包传输的机制和资源（如队列、缓冲、调度器）。流量类别彼此区分，可优先级调度。包字段用于识别流量类别。  |
|  | Transaction   | 事务   | 实现 libfabric 请求并传送用户请求载荷所需的一个或多个消息及其支持包。 |
|  | UE Transport Protocol (UET) | UE 传输协议   | FEP 之间通信所用的方法，包括协议、包格式和 FEP 策略。  |
| Parallel Communication | Job | 作业   | 由一个或多个 rank 组成的作业。 |
|   | JobID | 作业ID   | 集群中并行作业的唯一标识，用于寻址和授权。 |
|   | Parallel job | 并行作业 | 同一用户在集群上运行的多个进程集合，可互相通信。  |
|   | Parallel job model   | 并行作业模型 | 采用 MPI/*CCL 或 SHMEM 的集群运行模式，特点为“运行至完成”模型（checkpoint/restart 是简单的可靠性技术）。 |
|   | PIDonFEP | FEP 进程ID   | 关联 FEP 的进程标识符，编号从 0 到 P-1。如果每个 FEP 关联相同数量进程，则每个端点可轻松计算特定 RankID 的 PIDonFEP。 |
|   | Rank | Rank | 参与计算特定工作负载的进程。一个作业可在每个 OSI 上产生多个 rank，且 OSI 可托管多个作业的 rank。   |
|   | RankID   | Rank ID | 每个作业分配的 rank 编号，从0开始。  |
| Client-Server Communication | Client   | 客户端 | 节点上的软件实体，通过 FEP 与服务器通信。 |
| | Client/server job model  | 客户端/服务器作业模型 | 集群运行模式，客户端连接服务器（如存储或函数即服务 FaaS）。服务器通常长时间运行，服务无限客户端，具有复杂的可靠性和可用性保证。  |
| | Discovery | 发现   | 通过静态 fabric 地址或发现服务（如 DNS 或 LLDP）查找服务器的过程。 |
| | Resource Index (RI)  | 资源索引  | 标识进程内的资源（如服务、库或其他实体，例如 MPI 与 *CCL）。 |
| | Server   | 服务器 | 节点上的软件实体，为一个或多个客户端提供服务。 |
| | Server PIDonFEP | 服务器 PIDonFEP  | 结合资源索引标识特定 FEP 上可用的服务。同一 FEP 可能被多个服务器使用，且单个服务器可通过多个 PIDonFEP 和资源索引提供多项服务。  |
| Security Threat Model | Attacker | 攻击者 | 想从通信中窃取信息或篡改数据的实体。  |
| | Ciphertext   | 密文   | 在发送方和接收方之间传输的加密数据包内容。 |
| | Information  | 信息   | 参与者交换的数据或数据属性，可被攻击者利用造成危害（如加密密钥、FEP 决策等）。   |
| | In-scope threat | 范围内威胁   | TSS 明确应对并定义了缓解措施的威胁。 |
| | Intermediary/switch  | 中间体/交换机 | 路由或转发包至接收方的实体。  |
| | Out-of-scope threat  | 范围外威胁   | 本规范未考虑或未应对的威胁。  |
| | Plaintext | 明文   | 发送方加密前和接收方解密后的原始数据。 |
| | Protocol secrets | 协议秘密 | UET 中为维护可信连接保护的协议秘密，免于用户或攻击者获取。 |
| | Side channel | 旁路通道 | 攻击者在发送方或接收方不知情的情况下提取信息的方法。   |
| | Threat   | 威胁   | 可能导致协议秘密泄露、包数据泄漏或网络完整性下降的危险。 |
| | Threat mitigation | 威胁缓解 | TSS 针对威胁的具体应对措施。  |
| | Trusted entity  | 可信实体 | FEP 中负责处理密钥材料和执行密码学功能的部分。   |
| | Privileged entity   | 特权实体 | FEP 和内核驱动中负责分配关键传输信息（如 JobID 和安全上下文）的部分。 |
| | User entity  | 用户实体 | 使用 UET 传输服务的用户应用。 |
| Transport Security  | Additional authentication data (AAD) | 附加认证数据（AAD） | 与密文一起认证的附加数据，结合 AEAD 密码使用。   |
| | Association number (AN)  | 关联编号 | 在 TSS 头部携带，选择两个活动密钥（SDK）之一，支持密钥轮换。  |
| | Authenticated Encryption with Associated Data (AEAD) | 带附加认证数据的认证加密 | 一种结合机密性和真实性的对称加密方案。 |
| | Advanced Encryption Standard (AES) | 高级加密标准  | 一种对称加密算法，常与 AES-GCM 组合使用。  |
| | Cryptographic key | 密钥   | 由密码算法定义长度的真正随机或伪随机二进制字符串，符合 NIST SP800-108 定义。 |
| | Differential power analysis (DPA) | 差分功率分析 | 一种通过统计分析密码系统功耗进行的旁路攻击。   |
| | Epoch | 关键纪元 | 安全关联变更间的子间隔，由 SDME 管理，确保 IV 唯一，并可用于自动生成新的 SDK。 |
| | Initial secure domain key (SDKi) | 初始安全域密钥   | 来自 SDK 数据库的对称密钥，可直接用作 SDK 或通过 KDF 生成。 |
| | Galois/Counter Mode (GCM) | 伽罗瓦计数模式 | 对称密钥密码分组加密的操作模式。  |
| | Galois message authentication code (GMAC) | 伽罗瓦消息认证码 | 与 AES-GCM 结合使用的认证算法。 |
| | Initialization vector (IV) | 初始化向量   | 分组密码的初始区块或状态。 |
| | Integrity check value (ICV) | 完整性校验值 | 发送方计算的 AAD 和密文校验和，接收方用以验证包的密码完整性。 |
| | IPv4SIP  | IPv4 源地址  | IPv4（RFC 791）源地址。  |
| | IPv6SIP  | IPv6 源地址  | IPv6（RFC 8200）源地址。 |
| | IPv4DIP  | IPv4 目的地址 | IPv4（RFC 791）目的地址。 |
| | IPv6DIP  | IPv6 目的地址 | IPv6（RFC 8200）目的地址。 |
| | Key derivation function (KDF) | 密钥派生函数 | 使用伪随机函数从输入密钥派生新密钥的过程（Ko = KDF(Ki, label, context)）。  |
| | SDK database (SDKDB) | SDK 数据库  | 以 SDI 为索引的 SD 数据库，用于存储/检索安全参数。 |
| | Secure domain (SD)   | 安全域  | 使用 TSS 安全服务通信的一组 FEP，成员共享安全参数，SD 在包中用 SDI 表示。 |
| | Secure domain management entity (SDME) | 安全域管理实体 | 抽象的安全域管理者。  |
| | Secure domain identifier (SDI) | 安全域标识  | 包中携带的 SD 标识，与 AN 一起定位 SDKDB 密钥槽，用于重密钥。 |
| | Secure domain key (SDK)  | 安全域密钥  | 用于包的 AEAD 密码或 KDF 的对称密钥，符合 NIST 定义。  |
| | Secure source identifier (SSI) | 安全源标识  | 包中显式携带或源 IP 头地址的包来源唯一标识。  |
| | Timestamp counter (TSC)  | 时间戳计数器 | 单调递增计数器，每包不同。 |
| | Transport Security Sublayer (TSS) | 传输安全子层 | 本规范定义的 UE 传输安全子层。   |
| Libfabric 映射  | Application binary interface (ABI) | 应用二进制接口（ABI）  | 两个二进制程序模块之间的接口（例如用户程序和库或操作系统）。该接口定义了如何以底层硬件相关格式访问数据结构和计算例程。 |
|  | Application programming interface (API) | 应用程序接口（API） | 一种软件接口，API 向其他软件提供服务，并提供两个或多个程序或组件间通信的方式。 |
|  | Collective Communications Library (*CCL) | 集体通信库（*CCL）  | 实现并行计算中常见集体操作（如广播、全规约、全收集等）的一类集体通信库。加速器厂商通常提供支持其加速器功能的专有 *CCL 实现。 |
|  | Kernel mode driver (KMD)  | 内核模式驱动（KMD） | 运行在内核特权模式的操作系统组件，允许代码直接访问系统内存和硬件。 |
|  | Libfabric address vector (AV) | Libfabric 地址向量（AV） | 将更高层地址映射为 fabric 专用地址，方便应用使用。详见 libfabric 文档。 |
|  | Libfabric completion queue (CQ) | Libfabric 完成队列（CQ） | 用于报告数据传输完成的高性能事件队列。详见 libfabric 文档。 |
|  | Libfabric endpoint (EP)   | Libfabric 端点（EP）   | 使用 libfabric API 的通信端点，能监听连接请求并执行数据传输，配置有特定的通信能力和数据传输接口。详见 libfabric 文档。 |
|  | Libfabric event queue (EQ) | Libfabric 事件队列（EQ） | 用于收集和报告异步操作及事件完成的队列，报告与数据传输操作无直接关联的事件。详见 libfabric 文档。 |
|  | Message passing interface (MPI) | 消息传递接口（MPI） | 一种标准化且可移植的消息传递通信库接口，设计用于并行计算（如 MPI-4.1）。 |
|  | Partitioned global address space (PGAS) | 分区全局地址空间（PGAS） | 一种并行编程模型，使用逻辑分区的全局内存地址空间以提升分布式系统的性能和效率。 |
|  | Shared memory / symmetric hierarchical memory parallel programming library (SHMEM) | 共享内存 / 对称分层内存并行库（SHMEM） | 一种面向分布式内存环境的通信库，侧重于一侧通信，允许应用读取和写入彼此的内存。 |
| 信用基流控   | Best-effort VC | 最佳努力虚拟通道（VC） | 配置为不使用 CBFC 信用机制的虚拟通道。 |
|  | CBFC message | CBFC 消息 | CBFC 发送方与接收方间的链路层消息，不同 CBFC 消息格式为 CtlOS 或完整以太网包。 |
|  | Cell | 单元（Cell）   | 数据缓冲区中的存储单位。数据包通常分割为一个或多个单元进行存储。接收方缓冲区的单元数直接影响发送方可用的信用数。 |
|  | Control ordered set | 控制有序集 | CBFC 和 LLR 用于链路伙伴间传递信息的 8 字节消息格式。 |
|  | Credit  | 信用  | 代表接收方数据存储单位的令牌。信用允许发送方传输数据包，发送时消耗信用，接收方释放缓冲区资源后将信用返还发送方。 |
|  | Lossless VC  | 无损虚拟通道（Lossless VC） | 需要 CBFC 信用保证传输的虚拟通道，接收方保证有缓冲。无损 VC 可在单链路上与其他无损 VC 分开流控。 |
|  | Receiver | 接收方   | 链路伙伴功能，负责接收数据包并发送 CBFC 信用。 |
|  | Sender  | 发送方   | 链路伙伴功能，负责发送数据包并接收 CBFC 信用。 |
|  | Virtual channel (VC) | 虚拟通道（VC） | 包含具有相似流量特征、专用缓冲和流控管理的端口流量子集的实体。 |
| 链路层发现协议  | Link Layer Discovery Protocol (LLDP) | 链路层发现协议（LLDP）  | IEEE 标准 IEEE Std 802.1AB 的媒体无关协议，能在所有 IEEE 802® LAN 站点运行，允许端口学习邻近设备的连接和管理信息。 |
|  | Company ID (CID) | 公司标识符（CID）  | IEEE 标准中的唯一 24 位标识符，用于标识组织。CID 不能用来生成全球唯一 MAC 地址。 |
|   Management   | gNMI  | 管理 gNMI | 基于 gRPC 的标准网络管理接口，由 OpenConfig 项目定义，用于获取和修改网络设备配置及提供控制和遥测生成。 |
|  | gNOI   | gNOI | 基于 gRPC 的标准网络操作接口，由 OpenConfig 项目定义，用于在网络设备上执行操作命令。 |
|  | gRPC   | gRPC | 高性能开源框架，实现通用远程过程调用（RPC）。 |
|  | Yet Another Next Generation (YANG) | Yet Another Next Generation (YANG) | 数据建模语言，用于定义通过网络管理协议（如 NETCONF 和 RESTCONF）传输的数据。 |

