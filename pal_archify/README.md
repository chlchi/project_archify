# PAL 幻兽联机游戏项目架构全景解析

> 本项目通过 **[Archify](https://github.com/tt-a1i/archify)** 架构可视化体系，对 **Unity 客户端 (`D:\Coding\Game\pal`)** 与 **.NET 8 独立服务端 (`D:\Coding\Game\pal\PalGameServer`)** 进行了全方位架构逆向分析与建模。
> 本目录包含全套已通过 Archify Showcase 级标准校验（0 错误、0 警告、9/9 项检查全部通过）的交互式技术图表与可运行成品。

---

## 快速导航与成果清单

你可以直接在浏览器中打开 **[`index.html`](./index.html)** 专属架构中枢门户，或返回 **[根全景中枢](../index.html)**，亦可独立查看以下 5 大 Archify 架构图表（支持深浅色切换、节点聚焦检索、路径高亮、动态轨迹回放、卡片分享与矢量导出）：

| 图表类型 | 交付 HTML 成品 | 规范源文件 (Typed JSON IR) | 核心讲解主题 |
|---|---|---|---|
| **01. Architecture (系统架构)** | **[`pal_system.architecture.html`](./pal_system.architecture.html)** | [`pal_system.architecture.json`](./pal_system.architecture.json) | 客户端-服务端分层架构、网络边界与模块拓扑 |
| **02. Workflow (业务流程)** | **[`pal_gameplay.workflow.html`](./pal_gameplay.workflow.html)** | [`pal_gameplay.workflow.json`](./pal_gameplay.workflow.json) | 帕鲁球瞄准/抛物线投掷/3次摇动捕获/出战/联机判定 |
| **03. Sequence (交互时序)** | **[`pal_network.sequence.html`](./pal_network.sequence.html)** | [`pal_network.sequence.json`](./pal_network.sequence.json) | TCP 握手、20Hz 状态中继、权威受击裁决与死亡时序 |
| **04. Dataflow (数据流图)** | **[`pal_sync.dataflow.html`](./pal_sync.dataflow.html)** | [`pal_sync.dataflow.json`](./pal_sync.dataflow.json) | 输入物理采样、20Hz定率死区过滤、权威裁决与远端插值管道 |
| **05. Lifecycle (生命周期)** | **[`pal_lifecycle.lifecycle.html`](./pal_lifecycle.lifecycle.html)** | [`pal_lifecycle.lifecycle.json`](./pal_lifecycle.lifecycle.json) | 联机漫游主轨状态机、受击防作弊、收服伴宠与终态收敛 |

---

## 一、项目整体架构与技术栈概览

本项目是一款融合了 **第三人称 3D 动作冒险**、**幻兽（帕鲁/宝可梦）瞄准投掷捕获与养成**、**多人实时联机对抗** 的游戏系统。

### 1. 技术栈构成
- **Unity 游戏客户端** (`D:\Coding\Game\pal`):
  - **引擎版本**：Unity 3D
  - **物理系统**：基于 PhysX 的 `Rigidbody` 刚体、抛物线速度解算与射线检测 (`Physics.Raycast`)
  - **寻路与 AI**：Unity 内置 `NavMeshAgent` 驱动野生怪物与伙伴的巡逻/追踪/攻击
  - **网络通信**：自研轻量 TCP 客户端 (`NetworkManager.cs`)，采用单例模式 + 独立接收后台线程 + 主线程安全队列
  - **实体管理**：远端玩家代理管理器 (`OtherPlayerManager.cs`)，负责多客户端镜像生成与平滑插值 (`SmoothMove`)
- **PalGameServer 服务端** (`D:\Coding\Game\pal\PalGameServer`):
  - **运行时**：.NET 8.0 控制台服务应用
  - **网络接入**：`System.Net.Sockets.TcpListener` 监听 `8888` 端口，多线程并发接入与保持
  - **通信协议**：自定义文本管道定界符协议（UTF-8 编码，`CMD|arg1|arg2...\n`）
  - **状态模型**：全服集中式内存字典 `Dictionary<string, ClientInfo>`，搭配 `clientsLock` 线程安全锁
  - **权威裁决**：服务端独占血量扣减、死亡判定与 1.0 秒攻击去重防作弊系统

---

## 二、Archify 五大架构图深度解析

### 1. 系统总体架构图 (`pal_system.architecture.html`)
- **核心视图定位**：梳理客户端与服务端之间的物理边界、信任边界以及各大子系统模块职责。
- **架构模块划分**：
  - **客户端信任域 (Local Client Host)**：
    - `Player Controller`：第三人称摄像机朝向解算，WASD 移动，跳跃与刚体物理速度赋予。
    - `Pal Catch System`：鼠标右键地面射线瞄准，按仰角赋予刚体初速发射精灵球，触发碰撞与捕获。
    - `Companion & Enemy AI`：NavMesh 寻路驱动野生怪物状态机（`Idle` / `Attack` / `Die`），捕获后成为 `Member` 伴宠随行协同战斗。
    - `NetworkManager`：网络 IO 与游戏渲染解耦，后台接收线程将网络包推入主线程队列。
    - `OtherPlayerManager`：远端网络玩家镜像容器，根据服务器下发的坐标进行帧间插值平滑，挂载头顶 Billboard 血条。
  - **服务端信任域 (Server Host Process)**：
    - `TcpListener :8888`：主监听接入，通过 `AcceptClients` 线程为每个客户端派生独立工作线程。
    - `ClientInfo Registry`：记录客户端 `Id`、`PlayerId`、`LastPosition`、`Health`（初始 100）、`IsAlive`。
    - `Message Router`：解析管道字符串分发指令。
    - `Combat Adjudicator`：权威处理 `PLAYER_HIT`，进行 1 秒去重防刷与血量生命周期裁决。
    - `Broadcast Engine`：封装 `BroadcastToOthers` 与 `BroadcastToAll`，执行事件广播。

---

### 2. 核心业务流程图 (`pal_gameplay.workflow.html`)
- **核心流程**：按照 Archify Schema v2 泳道模型，分为 3 大阶段：
  1. **瞄准与投掷阶段 (Aim & Throw)**：
     - 本地玩家按住鼠标右键 (`Fire2`)，激活瞄准准星，通过摄像机向地面发射射线实时调整精灵球发射父物体的水平朝向。
     - 松开右键，实例化 `Pokeball` 刚体，按预设仰角（如 20°）和初速度向量发射。
  2. **捕获与召唤阶段 (Capture & Summon)**：
     - 飞行中的球体通过 `OnCollisionEnter` 碰撞到野生怪物 (`Enemy`)。
     - 怪物立即进入被捕获受击逻辑，球体将自身刚体设为 Kinematic 停止物理运动。
     - 启动 `CoDoCatching` 协程：球体进行 3 轮左右摇摆旋转动画（每次倾斜 5° 并回摆），计算捕获概率。
     - 捕获成功后，球体飞向玩家并自毁，将怪物名称添加到 `GameMode.Instance.pets` 仓库。
     - 玩家按下 `R` 键，调用 `GameMode.Instance.ReleasePal()`，在场景中实例化伴宠 `Member` 实体，协助攻击敌人。
  3. **联机战斗与伤害结算 (Combat & Adjudication)**：
     - 玩家左键发射子弹 (`Bullet`)，子弹触发碰撞远端玩家或怪物。
     - 本地 `Bullet.cs` 触发 `NetworkManager.SendMessage("PLAYER_HIT|...")`。
     - 服务端进行防作弊去重校验，通过后扣除目标血量，全服广播 `PLAYER_DAMAGE_RECEIVED`，血量归零广播 `PLAYER_DIED`。

---

### 3. 网络交互时序图 (`pal_network.sequence.html`)
- **通信时序全流程**：
  1. **连接握手**：
     - 客户端调用 `ConnectToServer()` 与服务端 8888 端口建立 TCP 连接。
     - 服务端分配 8 字符十六进制 ClientId，立即回复 `WELCOME|clientId|服务器连接成功`。
     - 客户端收到后发送 `PLAYER_JOIN|playerId|x,y,z`，服务端保存玩家初始位置，并将新玩家广播给在场所有远端客户端；同时将已在线的全部玩家及血量列表回传给新玩家。
  2. **高频位姿同步**：
     - 本地 `NetworkPlayerController` 定时器触发，当距离位移大于 `0.1m` 或旋转角大于 `5°` 时，构造并发送 `PLAYER_UPDATE|playerId|pos|rot|action`。
     - 服务端中继调用 `BroadcastToOthers`，远端客户端在主线程队列中取出消息，驱动对应镜像执行 `SmoothMove` 平滑插值。
  3. **战斗裁决与广播**：
     - 本地判定子弹命中远端玩家 B，发送 `PLAYER_HIT|A|B|damage|hitPosition`。
     - 服务端提取攻击者和目标 ID，查询 `LastHitTimes` 字典。若 1.0 秒内同一位置已有攻击，则判定为高频外挂或重复包予以忽略；若校验通过，更新攻击时间并执行 `target.Health -= damage`。
     - 服务端调用 `BroadcastToAll` 广播 `PLAYER_DAMAGE_RECEIVED|B|damage|newHealth|A`。
     - 玩家 B 及其他观察者更新头顶血条，若血量归零，触发全服 `PLAYER_DIED` 死亡广播。

---

### 4. 状态同步数据流图 (`pal_sync.dataflow.html`)
- **数据流拓扑**（分为五大流式阶段）：
  - **阶段 0：输入物理采样**：
    - `input_motion`：Update 采集键盘 WASD 与鼠标输入，FixedUpdate 物理重力与刚体速度推导。
    - `input_combat`：PhysX 碰撞触发 `OnCollisionEnter` 生成受击事件。
  - **阶段 1：差异过滤定率**：
    - `rate_limiter`：协程 `SendPositionUpdates` 锁定 `sendRate = 20Hz`（周期 50ms），切断逐帧发包。
    - `threshold_filter`：阈值计算 `Vector3.Distance > 0.1` 与 `Quaternion.Angle > 5.0`，静止状态直接丢弃。
    - `hit_packager`：将击中目标、伤害数值、碰撞坐标打成高优先级事件包。
  - **阶段 2：协议传输通道**：
    - `motion_packet`：格式化为 `PLAYER_UPDATE|...` 文本行。
    - `combat_packet`：格式化为 `PLAYER_HIT|...` 文本行。
    - 通过 TCP NetworkStream 顺序压入底层 Socket 发送缓冲区。
  - **阶段 3：服务端状态裁决**：
    - `server_router`：StreamReader 读取文本行，分发到对应处理函数。
    - `anticheat_state`：提取复合键校验时间戳，扣除内存状态中的 `ClientInfo.Health`。
  - **阶段 4：远端镜像呈现**：
    - `remote_smooth`：目标位姿与四元数作为插值目标，在 Update 中以 `smoothMoveSpeed` 线性插值，杜绝网络丢包导致的瞬移卡顿。
    - `ui_render`：头顶 Billboard 画布更新血条百分比，播放受击音效与击退动画。

---

### 5. 实体生命周期状态机 (`pal_lifecycle.lifecycle.html`)
- **状态流转规则**：
  - **主轨 5 阶段**：`离线未连接 (Offline)` → `TCP握手连接 (Connecting)` → `入场漫游探索 (In-World)` → `交战与投球捕获 (In-Combat)` → `持久存活在线 (Victory-Alive)`。
  - **战斗分支与受击状态**：漫游中受到攻击进入 `受击去重裁决 (Under-Hit)`，血量大于 0 恢复正常行动；血量归零收敛至终态 `生命归零阵亡 (Player-Died)`。
  - **幻兽捕获分支**：交战中投掷精灵球进入 `精灵球3次晃动 (Catching-Shake)`，判定成功收敛至终态 `收服伴宠加入背包 (Pet-Mastered)`，并在场景中召唤随行伙伴。
  - **会话注销分支**：主动退出或断线触发 `主动断开连接 (Voluntary-Quit)`，服务端注销 ClientInfo 资源，收敛至终态 `连接彻底关闭 (Session-Closed)`。

---

## 三、关键核心代码实现剖析

### 1. 客户端网络 IO 与主线程解耦机制 (`NetworkManager.cs`)
Unity 不允许非主线程访问其大部分 API（如 `Transform`、`GameObject` 等）。`NetworkManager` 采用了经典的 **双缓冲/队列解耦设计**：
```csharp
// 后台守护线程持续拉取流数据
private void ReceiveMessages() {
    StreamReader reader = new StreamReader(networkStream, Encoding.UTF8);
    while (isConnected && tcpClient.Connected) {
        string message = reader.ReadLine();
        if (message != null) {
            lock (lockObj) {
                messageQueue.Enqueue(message); // 压入线程安全队列
            }
        }
    }
}

// 主线程 Update 每帧出队并派发事件
private void Update() {
    lock (lockObj) {
        while (messageQueue.Count > 0) {
            string message = messageQueue.Dequeue();
            OnMessageReceived?.Invoke(message); // 在主线程中安全执行渲染与对象创建
        }
    }
}
```

### 2. 客户端 20Hz 阈值差异发包压缩 (`NetworkPlayerController.cs`)
为了防止高帧率下（如 144Hz）疯狂向网络套接字写入导致网络缓冲区溢出和严重的带宽浪费，客户端实现了时间定率与空间死区双重保护：
```csharp
private IEnumerator SendPositionUpdates() {
    while (true) {
        yield return new WaitForSeconds(1f / sendRate); // 20Hz (50ms)
        if (NetworkManager.Instance != null && ShouldSendUpdate()) {
            SendPlayerState();
        }
    }
}

private bool ShouldSendUpdate() {
    Vector3 currentPos = transform.position;
    Quaternion currentRot = transform.rotation;
    // 双重阈值过滤：仅当移动超0.1米或旋转超5度时才发送网络包
    bool posChanged = Vector3.Distance(currentPos, lastSentPosition) > positionThreshold;
    bool rotChanged = Quaternion.Angle(currentRot, lastSentRotation) > rotationThreshold;
    return posChanged || rotChanged;
}
```

### 3. 服务端防作弊与权威战斗裁决 (`Program.cs`)
在网络对战游戏中，客户端上传的击中事件必须由服务端进行完整性与合法性校验。`PalGameServer` 通过复合时间戳机制阻止了连击脚本外挂：
```csharp
// 构造复合键：攻击者_目标_击中部位坐标
string hitKey = $"{attackerId}_{targetId}_{hitPosition}";
DateTime now = DateTime.Now;

if (attackerClient.LastHitTimes.ContainsKey(hitKey)) {
    TimeSpan timeSinceLastHit = now - attackerClient.LastHitTimes[hitKey];
    if (timeSinceLastHit.TotalSeconds < 1.0) { // 1秒内同一位置重复攻击判定为非法外挂
        Console.WriteLine($"忽略重复击中: {attackerId} -> {targetId} 位置: {hitPosition}");
        return; // 静默丢弃，阻止多次扣血
    }
}

// 校验通过，记录本次攻击时间并结算血量
attackerClient.LastHitTimes[hitKey] = now;
targetClient.Health = Math.Max(0, targetClient.Health - damage);
BroadcastToAll($"PLAYER_DAMAGE_RECEIVED|{targetId}|{damage}|{targetClient.Health}|{attackerId}");
```

---

## 四、项目工程优势与演进优化建议

### 1. 当前实现的工程优势
1. **轻量精简，无重度第三方依赖**：服务端纯原生 .NET 8 实现，体积极小，零框架学习成本，启动速度极快。
2. **多线程架构边界清晰**：网络接收线程、处理线程与控制台监控分离，锁粒度适中。
3. **玩法机制闭环完整**：从输入控制、瞄准抛物线、物理碰撞、状态机到伙伴协同与多人联机，具备极高的可玩性雏形。
4. **防作弊意识明确**：服务端对血量、存活与攻击频率具备权威裁决机制。

### 2. 下一步商业化/生产化演进建议
| 优化维度 | 当前现状 | 潜在瓶颈 | 建议重构方案 |
|---|---|---|---|
| **传输协议** | 基于 TCP 字节流 | 遇到弱网或丢包时存在 **TCP 队头阻塞**，导致移动严重迟滞 | 引入基于 UDP 的可靠传输层（如 **KCP**、**LiteNetLib** 或 Unity **Netcode for GameObjects**） |
| **序列化格式** | 纯文本管道定界符 (`CMD|...`) | 字符串解析产生大量 GC 垃圾，带宽开销较大 | 替换为轻量高性能二进制序列化（如 **Protocol Buffers** 或 **MemoryPack**） |
| **客户端同步** | 纯服务器转发 + 远端插值 | 客户端高延迟时玩家操作反馈有延迟感 | 实现**客户端本地预测 (Client-Side Prediction)** 与服务端延迟补偿 (Lag Compensation) |
| **网络广播范围** | 全服无脑广播 (`BroadcastToOthers`) | 玩家数增长时，服务器网络带宽呈 $O(N^2)$ 爆炸 | 引入 **AOI 空间九宫格视野裁剪算法**，仅向视距内玩家广播同步数据 |
| **数据持久化** | 内存字典 (`ClientInfo`) | 服务端重启导致所有玩家数据与收服伴宠丢失 | 引入 Redis 缓存会话 + MySQL/SQLite 持久化存档 |

---

*报告生成时间：2026年9月*  
*生成工具：Google DeepMind Antigravity Agentic Assistant & Archify Diagram Compiler*
