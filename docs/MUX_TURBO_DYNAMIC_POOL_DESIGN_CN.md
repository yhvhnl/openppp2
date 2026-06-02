# MUX Turbo 动态承载池设计与风险评估（方案 B：运行时动态增减）

> 状态：设计评审中（未实现）。本文件是 turbo "运行时按质量动态增减 per-connection 承载池" 的设计与风险清单，供拍板。配对实现前请逐条确认风险。技术术语保留英文。

## 需求

- `--tun-mux` 重新定位为**最初首包的 per-connection 连接池基数**（base）。
- turbo 开启时，按链路质量**运行时动态增减**承载链路数：
  - 池大小 = `ceil(需求数 / (有效空闲 + 带载倍数)) × factor`，`factor ∈ [1..3]`；
  - **质量越差，池越大**（多开链路扩竞争池）。
- 与方案 A（建链时一次性定池）的区别：B 要求**会话存活期间随质量变化加/减链路**。

## 当前架构的硬约束（为什么这是高风险）

`max_connections`（= `--tun-mux`）是会话**核心不变量**，运行时改它会连锁触发以下问题：

1. **双端协商绑定**（`VirtualEthernetExchanger::OnMux`，server）：
   - client 在 `DoMux(vlan, max_connections, ...)` 把 `max_connections` 发给 server；
   - server `OnMux` 用 `mux->get_max_connections() != max_connections` 作为**整会话重建判据**：一旦不等，立即 `mux_.reset(); mux->close_exec()` **销毁整个 mux 会话并重建**。
   - 含义：**运行时改变 max_connections，会被 server 当成"换了一个会话"而整体重建**——所有在途逻辑连接全断。这是 B 方案最致命的冲突点。

2. **建链完成判据**（`linklayer_established`）：`established_ = opened_connections >= max_connections`。运行时改 `max_connections` 会破坏 established 语义（已 established 的会话再改阈值，状态机不自洽）。

3. **一次性 spawn forwarding**（`add_linklayer` 尾部）：当 `rx_links_.size() == max_connections` 时，遍历 `rx_links_` 对**每条链路** spawn forwarding 协程。运行时再 `add_linklayer` 会对**已在 forwarding 的旧链路重复 spawn** → double-forwarding（同一链路两个读协程）→ 数据竞争 / 重复交付 / 崩溃。

4. **teardown 竞态未收尾**（D1/D2/D3）：运行时增减链路会增加 `rx_links_`/`tx_links_` 的并发变更点，与尚未收尾的 teardown 生命周期叠加，放大堆破坏风险。

5. **owner 约束**：issue #5 评审明确"连接数绑定 `--tun-mux`，不引入 `mux.max-connections`"。运行时动态池实质上引入了一个**会话级可变连接数**，需要 owner 重新确认是否接受这一语义扩张。

## 实现 B 必须做的改造（风险逐项）

### C-B1 协议：新增"运行时增减链路"控制帧（高风险）
- 不能复用 `DoMux` 改 `max_connections`（会触发 server 整会话重建，见约束 1）。
- 需新增 wire 命令（如 `cmd_pool_resize`），让一端请求"在现有会话上 +N / -N 条承载链路"，server 同意后双端各自增减，且**不重建会话、不改 established**。
- 风险：新控制帧 = 协议扩张 + 双端版本兼容（旧端不认 → 必须 fail-safe 忽略）；增加 double control 层（违背"控制平面尽量薄"）。

### C-B2 会话不变量：max_connections 改为可变上界（高风险）
- `max_connections` 从"固定建链数"拆成两个概念：`pool_hard_max`（硬上限，不变，防爆）+ `pool_current`（当前目标池大小，可变）。
- `established_` 判据、`add_linklayer` 配额、server `OnMux` 重建判据全部要改为基于 `pool_hard_max` 而非当前值。
- 风险：动 established 状态机，回归面大；改错会导致建链永不完成或提前完成。

### C-B3 运行时增链路：独立的单链路加入路径（高风险）
- 必须新增一条"运行时只加 1 条链路、只 spawn 1 个 forwarding 协程"的路径，**绝不复用** `add_linklayer` 尾部的"遍历全部 spawn"逻辑（否则 double-forwarding）。
- 新链路要走完整 handshake（client 端 `ConnectTransmission` + `ConnectMux`），在 vmux strand 上原子加入 `rx_links_`/`tx_links_`。
- 风险：这正是 teardown 竞态的高发区；新链路加入与 finalize/drain 并发，需要 C2/C3（strand-only finalize + inflight/epoch）作为前置，否则堆破坏概率上升。

### C-B4 运行时减链路：优雅移除（高风险）
- 减链路要等该链路上**在途发送 completion 全部回来**（否则 completion 踩已移除链路）→ 需要 per-link inflight 计数（C3 的子集）。
- 该链路上承载的逻辑连接帧要平滑迁移到竞争池其它链路——但竞争法本就不绑定，所以"减链路"只需停止在该链路发新帧 + 等 inflight 清零 + 关闭。
- 风险：依赖 C3；无 inflight 计数则无法安全减链路。

### C-B5 质量度量与池大小控制器（中风险）
- 需要 per-link 或会话级质量信号驱动 `factor`。现有信号：建链耗时、stall 看门狗触发、心跳活跃度（`last_active_`）——都是近似、滞后信号（云韵已指出滞后性问题）。
- 控制器要有**滞回/阻尼**（避免质量抖动导致池频繁增减 → 颠簸）；增减各有冷却时间。
- 风险：中。控制器设计不当会因滞后信号 + 频繁增减放大不稳定（恰是云韵警告的"流速度起搏压垮整体"）。

## 依赖关系

```
C-B3 / C-B4（运行时增/减链路）  ──依赖──>  C2（strand-only finalize）+ C3（inflight/epoch）
C-B1（协议帧）+ C-B2（不变量拆分）         ──需要──>  双端同时升级 + owner 确认连接数语义扩张
```

**结论：B 方案的安全前提是先完成 C2/C3（teardown 生命周期加固）。** 而 C2/C3 目前因缺 ASan 栈 + 容器竞态静态不成立而搁置。在 C2/C3 未落地时实现 B，等于在已知的生命周期薄弱区上叠加更多运行时并发变更——堆破坏风险显著上升。

## 建议的分期（若坚持 B）

1. **前置**：先做 C2（strand-only finalize）+ C3（per-link inflight 计数）。这两项是 C-B3/C-B4 安全的硬前提。
2. **协议**：C-B1（`cmd_pool_resize`，长度容忍、fail-safe、双端协商）。
3. **不变量**：C-B2（`pool_hard_max` / `pool_current` 拆分，改 established / OnMux 重建判据）。
4. **增减**：C-B3（运行时加单链路，独立 spawn）+ C-B4（运行时减链路，等 inflight 清零）。
5. **控制器**：C-B5（带滞回的质量→池大小控制器）。
6. 全程 ASan 构建验证，双端升级，owner 确认连接数语义。

## 与方案 A 的取舍（再次澄清）

- **A（建链时按质量一次性定池）**：拿到"质量差→多开链路"的核心收益，复用现有安全建链路径，零协议改动、零运行时并发新增、不依赖 C2/C3。风险接近零。
- **B（运行时动态增减）**：完整满足"存活期动态调整"，但需要协议扩张 + 不变量重构 + C2/C3 前置 + 控制器，且与 owner 约束冲突。风险高，工期长。

**工程建议仍是先做 A 验证收益，B 作为 A 之后、C2/C3 落地之后的演进。** 但方向由你定；本文件记录 B 的完整代价，便于知情决策。
