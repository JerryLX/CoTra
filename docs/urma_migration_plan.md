# CoTra 代码架构与 RDMA → URMA 迁移方案

## 1. 结论

CoTra 的通信层可以替换为 URMA，但不建议直接在 `RdmaContext` 中逐个替换 verbs 调用。当前代码把 verbs 的资源模型和完成语义暴露到了 `RdmaCommunication`，并且上层协议依赖 `RDMA_WRITE_WITH_IMM`。推荐先稳定一个与设备无关的传输接口，再实现 URMA backend；算法、序列化格式、调度器和图索引基本可以保持不变。

迁移的实际边界如下：

- **无需重写**：索引构建/搜索算法、`QueryMsg`、`TaskManager`、图数据布局、CoroMem 协程调度、`ControlType` 业务含义。
- **需要适配**：内存注册与远端描述、端点建立、单边 Read、消息 Write、完成轮询、buffer 归还、错误处理与启动参数。
- **需要重新设计或先验证能力**：当前 `RDMA_WRITE_WITH_IMM + 零长度 Receive WQE` 通知协议。参考 URMA 实现展示了普通 `URMA_OPC_READ/WRITE`，没有展示等价的 write-with-immediate 用法。

建议分两阶段交付：先以兼容 facade 保持 `RdmaCommunication` 上层 API 不动，完成 URMA 可运行版本；随后把类型和命名泛化为 `TransportCommunication`，保留 RDMA/URMA 双 backend 以便回归和性能对比。

## 2. 当前代码架构

### 2.1 分层

```text
tests/scala_anns.cc, tests/scala_index.cc
                 │
                 ▼
ScalaSearch / ScalaBuild
  ├─ QueryMsg / TaskManager / ScalaScheduler
  ├─ ScalaGraphIndex / B1Graph / TopIndex
  └─ RdmaCommunication                业务通信 API 与消息分发
       ├─ VectorBuffer                异步 Read 本地缓存
       ├─ SendWriteBuffer             每目标机的发送槽和空闲槽
       ├─ RecvWriteBuffer             接收槽和按 ControlType 分类的队列
       ├─ ServerConnect               memcached 元数据交换与 barrier
       └─ PerThreadStorage<RdmaContext>
              └─ verbs: device/PD/MR/CQ/QP/WR/WC
```

主要目录职责：

| 目录/文件 | 职责 | URMA 影响 |
|---|---|---|
| `include/anns`, `src/anns` | 分布式搜索、查询状态、任务序列化与调度 | 低；只依赖通信 facade |
| `include/index`, `src/index` | 分片、建图、布局转换、构建期数据交换 | 低到中；调用大量 `rdma_comm.*` |
| `include/rdma/rdma_comm.h`, `src/rdma/rdma_comm.cc` | Read/Write/Send API、完成处理、控制消息分类 | 高；迁移核心 |
| `include/rdma/rdma_context.h`, `src/rdma/rdma_context.cc` | verbs 资源、MR/QP/CQ、post/poll | 极高；由 URMA backend 替代 |
| `server_connect.*` | memcached 机器发现、端点/MR 元数据、barrier | 中；机制可复用，元数据格式需替换 |
| `rdma_param.*` | 集群参数与 verbs 设备参数混合 | 高；应拆成公共参数和 backend 参数 |
| `CMakeLists.txt` | 强制链接 libibverbs/librdmacm 和绝对 include | 高；需要 backend 构建选项 |

### 2.2 数据平面

1. **远端向量读取**：`post_read()` 为多个 `BufferCache` 组装链式 Read WR；完成后 `poll_send()` 根据 `wr_id = query_id | buffer_id` 将 buffer 放入 `VectorBuffer::recv_list`。`poll_read()`/上层协程持续推进，`release_cache()` 归还槽位。
2. **控制和任务消息**：发送端从 `SendWriteBuffer` 取槽，将序列化数据写入远端 `RecvWriteBuffer` 的固定槽，使用 `WRITE_WITH_IMM` 携带 `ControlType` 和 `buf_id`。
3. **接收分发**：接收 CQ 完成给出 `byte_len`、`imm_data` 和 `wr_id`；`poll_recv()` 将消息复制到 `TASK/RESULT/QUERY/SYNC/...` 队列，并重新 post receive。
4. **buffer 流控**：接收方把消费完的槽号通过 `RELEASE/CLEAR` 消息返还发送方。这个协议必须原样保留或等价实现，否则高并发时会覆盖尚未消费的接收槽。

### 2.3 控制平面与并发模型

- `ServerConnect` 使用 memcached 发布/获取每线程连接和 MR 元数据，并提供 barrier；URMA 迁移不要求更换 memcached。
- 每个工作线程拥有独立 Context、端点、完成队列和缓冲池，上层没有专用网络 progress thread。
- 协程能否继续运行依赖业务循环频繁调用 `poll_send()`、`poll_recv()` 和 `poll_read()`。URMA backend 必须保持非阻塞 poll 语义，不能在 post 或 poll 中长时间阻塞。

## 3. verbs 与 URMA 对照

以下映射来自参考目录 `Mooncake/mooncake-transfer-engine/src/transport/kunpeng_transport`：

| 当前 verbs | URMA 参考方式 | 迁移说明 |
|---|---|---|
| `ibv_get_device_list/open_device` | `urma_get_device_list`、`urma_get_eid_list`、`urma_create_context` | 配置由 IB device/port/GID 改为 URMA device/active port/EID index |
| `ibv_alloc_pd` | 无直接一一对应的上层对象 | 不要在公共接口暴露 PD |
| `ibv_reg_mr` | `urma_register_seg` | 记录 `urma_target_seg_t`；远端需序列化 `urma_seg_t` |
| 交换 `vaddr + rkey` | 交换序列化的 `urma_seg_t`，远端 `urma_import_seg` | 地址仍参与 SGE，但授权对象从 rkey 变为 imported target segment |
| `ibv_create_cq` | `urma_create_jfc`（可选 `urma_create_jfce`） | 首版建议 busy-poll JFC，以匹配当前协程推进模型 |
| `ibv_create_qp` | `urma_create_jfr` + `urma_create_jetty` | 参考实现共享 JFR，并为 endpoint 建多个 jetty |
| QP INIT/RTR/RTS | `urma_import_jetty`；RC 下再 `urma_bind_jetty` | 交换 EID、jetty id、transport mode，替代 LID/GID/QPN/PSN |
| `ibv_post_send` Read/Write | `urma_post_jetty_send_wr`，opcode 为 `URMA_OPC_READ/WRITE` | `urma_sge_t` 同时持地址、长度和 segment handle |
| `ibv_poll_cq` | `urma_poll_jfc` | 用 `urma_cr_t.user_ctx/status` 恢复请求上下文并更新槽状态 |
| `wr_id` | `urma_jfs_wr_t.user_ctx` | 建议指向稳定的 `CompletionToken`，不要把短生命周期栈对象作为上下文 |
| `ibv_dereg_mr/destroy_*` | `urma_unregister_seg/unimport_seg/delete_jetty/delete_jfr/delete_jfc/delete_context` | 严格按“停止 post → drain → 解绑/反导入 → 删除本地资源”的逆序释放 |

参考实现还包含 runtime 初始化、异步设备事件、端点断线和失败重试。CoTra 首版至少要实现 context/segment/jetty/JFC 生命周期和完成状态检查；设备 fatal、port down、EID change 的处理可在第二阶段补齐，但不能默默忽略错误完成。

## 4. 推荐目标架构

```text
ScalaSearch / ScalaBuild
          │
          ▼
TransportCommunication              保持现有业务级 API
  ├─ read()/postRead()/pollRead()
  ├─ sendMessage()/pollMessages()
  ├─ barrier()/registerMemory()
  └─ buffer pools + ControlType dispatcher
          │
          ▼
ITransportBackend                   只表达传输原语
  ├─ initialize/shutdown
  ├─ registerRegion/export/import
  ├─ connectPeer
  ├─ postRead/postWrite/postNotify
  └─ pollCompletions
       ├───────────────┐
       ▼               ▼
UrmaBackend         VerbsBackend
context/JFC/       PD/CQ/QP/MR
JFR/jetty/segment
```

建议的公共对象：

```cpp
struct RegionDescriptor { std::vector<std::byte> opaque; };
struct EndpointDescriptor { std::vector<std::byte> opaque; };

enum class CompletionKind { Read, Write, Notify, Receive, Error };
struct Completion {
  CompletionKind kind;
  uint64_t request_id;
  uint32_t bytes;
  int status;
};

class ITransportBackend {
 public:
  virtual RegionDescriptor registerRegion(void*, size_t, Access) = 0;
  virtual void importRegion(PeerId, const RegionDescriptor&) = 0;
  virtual EndpointDescriptor localEndpoint(PeerId) const = 0;
  virtual void connect(PeerId, const EndpointDescriptor&) = 0;
  virtual bool postRead(const ReadRequest&) = 0;
  virtual bool postWrite(const WriteRequest&) = 0;
  virtual bool postNotify(const NotifyRequest&) = 0;
  virtual size_t poll(std::span<Completion>) = 0;
  virtual ~ITransportBackend() = default;
};
```

`RegionDescriptor` 和 `EndpointDescriptor` 必须使用带 `version/backend/length` 的显式 wire header，不能继续依赖 `sprintf/sscanf` 固定格式；opaque payload 内可以存 `urma_seg_t`、EID、jetty id 和 transport mode。所有机器必须使用相同 URMA ABI；如果直接序列化厂商 struct，应至少校验 ABI 版本、结构长度和字节序。

## 5. 消息通知协议：迁移的关键设计

当前 `WRITE_WITH_IMM` 同时完成三件事：写 payload、通知远端、携带 `control + slot`。普通 URMA Write 只覆盖第一件事。推荐改成：

```text
发送方                         接收方
  URMA WRITE payload ─────────► 固定 recv slot
  URMA WRITE doorbell ────────► doorbell ring[producer]
                                  │
                                  └─ poll doorbell
                                     acquire fence
                                     按 control/slot/len 分发
  ◄──────── RELEASE message / credit batch
```

`Doorbell` 建议至少包含：

```cpp
struct Doorbell {
  uint32_t magic;
  uint16_t version;
  uint16_t control;
  uint32_t source_machine;
  uint32_t source_thread;
  uint32_t slot;
  uint32_t bytes;
  uint64_t sequence;
};
```

实现选择按优先级为：

1. **若部署环境的 URMA API 确认支持等价 Write-with-notify/immediate**：封装为 `postNotify`，保留现有协议，改动最小。
2. **doorbell ring（推荐通用方案）**：先写 payload，再写带 sequence 的 doorbell；接收方轮询本地 doorbell。必须确认同一 jetty 上写入顺序，必要时使用 fence/完成后再发 doorbell，并用 release/acquire 内存序保证 CPU 可见性。
3. **双边 Send/Receive**：用 URMA send opcode 和 JFR 传控制头，payload 仍走单边 Write。语义清晰，但需要维护 receive WQE、JFR 完成和额外队列；参考主路径没有提供可直接照搬的双边消息实现。

不要只轮询 payload 中的 `ControlType` 字段：槽复用时会产生 ABA、半写入和旧消息重读问题。sequence、明确的生产/消费索引及 credit 是必需的。

## 6. 文件级修改方案

### 6.1 第一阶段：兼容式 URMA backend

新增：

- `include/transport/transport_backend.h`：公共请求、完成、region/endpoint descriptor。
- `include/transport/urma/urma_context.h`、`src/transport/urma/urma_context.cc`：URMA runtime、device/EID、JFC/JFR、segment 生命周期。
- `include/transport/urma/urma_endpoint.h`、`src/transport/urma/urma_endpoint.cc`：jetty 创建、导入、RC bind、post 和完成统计。
- `include/transport/urma/urma_backend.h`、`src/transport/urma/urma_backend.cc`：实现 `ITransportBackend`。
- `include/transport/wire_protocol.h`：版本化元数据和 doorbell wire 格式。

修改：

- `rdma_comm.*`：首版可保留类名以减少上层改动，但内部持有 `ITransportBackend`，不再访问 `ibv_wc`、QP 或 MR；完成分类改用公共 `CompletionKind`。
- `server_connect.*`：key 名从 RDMA 专用改为 transport session；发布 URMA segment/endpoint descriptor。barrier 逻辑保留。
- `rdma_param.*`：拆出 `ClusterParameter` 和 `UrmaParameter`。新增 device name、EID index、active port、transport mode（建议先 RC）、JFC depth、jetty count、doorbell ring size。
- `vec_buffer.h`：buffer pool 可保留；把隐含于 `wr_id` 的状态改成显式 `CompletionToken`，并为消息槽增加 sequence/doorbell 状态。
- `CMakeLists.txt`：添加 `COTRA_TRANSPORT=urma|verbs`；URMA 模式查找头文件和 `liburma.so`，不再全局链接 verbs。使用 target 级 include/link，避免 `link_libraries()` 污染所有目标。
- `tests/rdma`：泛化为 `tests/transport`；测试程序参数和输出不再硬编码 RDMA。

### 6.2 第二阶段：清理上层命名与依赖

- `RdmaCommunication` → `TransportCommunication`。
- `RdmaParameter` → `ClusterParameter + TransportConfig`。
- `init_rdma()` → `init_transport()`，`rdma_comm` → `transport`。
- 将 `include/anns/*.h`、`include/index/*.h` 对 `rdma_param.h` 的包含替换为轻量公共配置头。
- 将 `build_sync`、任务发送、结果发送等重复的“取槽—序列化—post”收敛为 `sendMessage(peer, ControlType, span)`，降低 backend 语义再次渗透的风险。

## 7. 分步实施顺序

### P0：能力验证（必须先做）

1. 在目标 Kunpeng/URMA 环境编译最小程序，确认头文件、`liburma.so`、设备名、EID 和端口。
2. 两机验证 register/export/import segment、jetty import/bind、单次 Read/Write、JFC completion。
3. 查询安装版本的 URMA 头文件，确认是否有 write-with-immediate/notify 或可靠的 Send/JFR 能力。
4. 验证同 jetty 连续两次 Write 的远端可见顺序以及 CPU cache coherent 行为。这决定 doorbell 是否需要额外 fence 或 cache 操作。
5. 验证单个 segment 最大长度、SGE 数、JFC/jetty depth，确保覆盖 `for_read` 分区、读缓存和消息池。

P0 不通过时不要进入全量改造；尤其不能假设 Mooncake 的配置上限等于当前设备能力。

### P1：抽象和元数据

1. 引入公共 backend 接口和 completion token，不改变上层调用签名。
2. 将 memcached 中的 `MessageContext` 替换为版本化 descriptor。
3. 先实现 verbs backend 适配并跑现有测试，证明抽象没有改变行为。

### P2：URMA Read

1. 创建每线程 URMA context/JFC/JFR/jetty，按机器导入远端 segment/jetty。
2. 将 `for_read` 与 `read_to` 注册为 segment。
3. 实现 sync/async Read 和 `user_ctx → query_id/buffer_id` 完成映射。
4. 先跑 `test_read`，再跑只使用远端向量读取的搜索小数据集。

### P3：URMA 消息 Write

1. 实现 payload slot + 选定的通知机制。
2. 迁移 `ControlType` 分发，保持序列化内容不变。
3. 恢复 RELEASE/CLEAR credit 协议并做槽耗尽、乱序、环回测试。
4. 迁移 scatter/gather；若设备/URMA SGE 上限不足，首版合并到连续 staging buffer。

### P4：构建流程和全链路

1. 依次恢复 `BUILD_SYNC/PART_INFO/DISPATCH_*`，完成多机构建。
2. 恢复 `QUERY/TASK/RESULT/NODE_*/COMPUTE/SCHEDULE/TERMINATION`，完成分布式搜索。
3. 加入超时和诊断信息，避免当前无限 poll 在设备错误时永久挂起。
4. 在相同数据集、线程数和机器数下对比 recall、结果一致性、吞吐、P50/P99、CPU 占用和远端读放大。

## 8. 主要风险与控制措施

| 风险 | 后果 | 控制措施 |
|---|---|---|
| URMA 无等价 immediate 通知 | 消息接收侧无法获知类型/长度/槽 | P0 能力验证；采用 versioned doorbell ring |
| segment descriptor 直接跨机序列化不兼容 | import 失败或错误访问 | wire header 带 ABI/version/size；集群启动时 fail-fast |
| 远端写后 CPU 可见性/顺序假设错误 | 读到半包或旧包 | sequence + producer/consumer；验证排序；必要时 fence/两阶段提交 |
| 每线程、每机器 jetty 数量过多 | 超过设备资源或初始化很慢 | 先查询容量；可改为每线程少量 jetty、多 peer imported target，或按 NUMA 共享 context/JFC |
| JFC depth 与软件在途数不一致 | post 失败、死锁或吞吐抖动 | backend 显式 credit；post 前限流；失败 WR 回滚计数 |
| `user_ctx` 生命周期错误 | 完成后 UAF | token 存在固定 request/slot 表中，完成或取消后才复用 |
| 当前无限 busy poll | 链路故障时永久挂起 | 状态机、deadline、异步事件和错误 completion 统一上报 |
| scatter/gather 能力差异 | 构建/查询消息无法发送 | 启动时检查 max_sge；连续 staging buffer 作为保底 |
| 一次性删除 verbs 路径 | 难以定位算法还是传输回归 | 双 backend、同一 wire protocol、逐阶段 A/B 测试 |

## 9. 测试与验收标准

### 单元测试（无硬件或 mock）

- descriptor 和 doorbell 的 encode/decode、版本和长度拒绝。
- `ControlType` 全枚举分发。
- slot allocator：分配、完成、RELEASE batch、wrap-around、sequence ABA。
- completion token：Read/Write/Notify 成功、post 失败回滚、错误完成。
- SGE 超限时 staging 合并。

### 两机集成测试

- 1B、缓存行、4KiB、最大向量、最大消息、跨页 Read/Write。
- 并发到 JFC/jetty depth 上限，验证无覆盖、无泄漏、无永久等待。
- 主动断链、进程重启、端口 down，要求请求失败可观测而不是挂死。
- `test_read`、`test_write`、`test_bw` 对应的 transport 版本。

### 业务验收

- 小数据集构建产物与 verbs 版本的节点/邻接/分片元信息一致。
- 相同查询的 top-k 和 recall 不因传输迁移下降。
- 连续压力运行无 slot 泄漏，所有机器终止检测可完成。
- 性能基线至少记录：QPS、P50/P99、有效网络带宽、每查询远端 Read 数、post/poll 批量、CPU 使用率。

## 10. 工作量判断

这不是只修改 `CMakeLists.txt + rdma_context.cc` 的小改动。若目标 URMA 支持等价的 write-with-immediate，兼容迁移主要集中在资源对象、连接元数据和 WR/CQ 映射；若不支持，则消息通知和流控协议是最大工作项。

推荐的最小可交付范围是：

1. P0 两机 URMA 探针；
2. 公共 backend 接口及 verbs 回归；
3. URMA async Read；
4. doorbell 消息通道；
5. 两机小数据集构建与查询；
6. 再扩展到当前机器数和线程数。

这样可以把硬件/API 能力风险、协议正确性风险和业务回归风险分开验证，而不是在最终多机运行时同时排查。
