# Agent Note: 工作区浏览器支持取消归档、「已归档」视图与持久删除

Status: implemented

[English](2026-08-14-unarchive-archived-sessions.md) | 中文

## Problem

归档一个会话（`workspace.archiveSession`）会把它加入注册表级全局归档集合，并从所有分组视图（分组树、单列表、搜索）中隐藏，却没有任何办法把它带回来：既没有取消归档 RPC，也没有列出已归档会话的界面。被误归档、或在用户尚未结束前就被归档的会话因此既看不见也够不着，这让「归档」变成了一次性操作，而不是清静列表的手段。唯一的出路是永久删除，而只追加的持久化层根本没有 `delete` 原语。

## Decision

`workspace.unarchiveSession({ sessionId })` 把一个会话移出注册表级全局归档集合并应答完整的更新后集合，与 `archiveSession` 完全对称。工作区注册表新增 `unarchiveSession(id)`，从持久化的 `archivedSessionIds` 状态中过滤掉该 id，并对不在集合中的 id 直接完成而不写入（幂等跳过可避免丢失的取消归档重试产生破坏）。现有的 `host/archived-sessions-changed` 帧和 `workspace.list` 基线已经携带该集合，因此无需新增事件或重连路径。

工作区浏览器现在在分组树和单列表底部都渲染「已归档」区（`deriveArchived` 从会话列表中筛选出顶层已归档 id，按最近优先排序）。已归档行复用会话行菜单，但把「归档」替换为「取消归档」，并增加一个破坏性的「删除」；取消归档把该行恢复到记账槽，删除则在提交前先弹出确认对话框。由于归档从不触碰 workspace 记账，取消归档会恢复会话的分组视图席位。

持久删除是持久化层原语，而非工作区浏览器的取巧。`SessionPersistence.delete(id)`（一个默认拒绝的非抽象方法，因此测试替身可直接继承）委托给 `PersistenceCoordinator.delete(id)`：先等待进行中的退休排空，再在按 id 链上串行化、拒绝仍然活动（被拥有）的会话、失效任何保留的冷视图、调用新增的 `PersistenceBackend.deleteStored(id)` 钩子，并丢弃内存状态。JSONL 删除会话目录；SQLite 删除 `sessions` 行（`events` 级联删除）。`session.delete({ sessionId })` 只对真正运行中的 Agent 以 `session-busy` 拒绝，然后通过新增的 `AgentRegistry.dispose(id)` 处置任何空闲的已挂接会话——注册表保留了每个 `create`/`resume` 句柄的共享拆除能力，因此这也覆盖了在 `ensureSession` 之外挂接的会话（子代理与 fork）——随后调用 `workspaceRegistry.removeSession(id)`（归档集加上每个记账槽），最后才删除持久日志。处置会先冲刷并释放协调器的活动所有权，因此部分失败时该会话会退化为 Ungrouped，而不是留下悬空记账。session-query 读模型通过其现有的列表对账收敛。

## Alternatives considered

**只做取消归档、不做「已归档」视图** —— 否决。单独恢复 RPC 只能让取消归档通过未来某个界面才能实现；没有可见的「已归档」区，用户仍然找不到自己归档了什么，而这正是最初的诉求。视图才是修复本身。

**用筛选/开关代替固定尾部区块** —— 否决。筛选会让现有的分组/单列表/搜索派生更复杂，并把已归档会话藏在额外控件之后；固定区块让它们始终出现在其落点。

**只清归档集、不做持久化原语** —— 否决。只清记账会让持久日志在磁盘上成为孤儿，并在下一次列表对账时复活；删除必须移除存储，因此它落在持久化层。

**强删运行中会话的日志** —— 否决。在写入中途处置运行中 Agent 的持久日志是另一个更危险的生命周期问题，因此 `session-busy` 仍强制先显式停止；删除前只会处置空闲的已挂接会话。

## Consequences

用户可以重新看到已归档会话，把会话恢复到所在工作区（或 Ungrouped），并从「已归档」区经一次确认动作永久删除。取消归档的线协议和重连基线保持不变，因为它们本就携带归档集合。删除拓宽了持久化 Service Definition 和两个第一方后端各一个存储钩子，并新增了 `AgentRegistry.dispose(id)` 作为任意活动 Agent 的注册表级拆除能力；协调器的活动会话拒绝仍是防止只追加契约被中途写入删除破坏的安全边界。
