# UE5 FPS 敌人 AI 系统技术报告 / UE5 FPS Enemy AI System Technical Report

<details open>
<summary><b>🇨🇳 中文版本 (Click to collapse/expand)</b></summary>

## 封面

**项目名称**:  基于 UE5 First Person 模板的 FPS Demo - 敌人 AI 与网络对战系统  
**学生姓名**: 陈胜阳  
**学校**: 香港城市大学  
**提交时间**: 2026 年 1 月  
**GitHub 仓库**: https://github.com/CHANSingYeungSunny/UE5-FPS-Multiplayer-Demo---Tencent-Game-Client-Assignment?tab=readme-ov-file#english-version-  
**报告版本**: v1.0（阶段一：敌人 AI 系统完成）

## 项目概述

本项目基于 UE5 官方 First Person 模板实现一个 FPS Demo，目标为满足以下三项作业要求：

1. 敌人 AI 能够移动和攻击玩家，玩家可以击败敌人  
2. 基础的得分和游戏胜利机制  
3. 多人网络对战功能  

**当前阶段进度**: 已完成敌人 AI 系统，包括行为树设计、黑板状态管理、玩家定位服务、巡逻追击逻辑和攻击接口。敌人能够自主搜索玩家、追击并触发攻击，玩家可接收伤害事件。

**后续阶段计划**:

1. 补完玩家/敌人血量系统与死亡机制  
2. 实现得分与胜利条件  
3. 集成多人网络同步（服务器权威、客户端 UI 显示）  

## 第二部分：敌人 AI 系统实现

### 2.1 Blackboard 设计

Blackboard（黑板）是 UE5 行为树中的关键状态容器，用于在服务和任务间共享数据。根据 Epic 官方文档，Blackboard 中的每个 Key 对应一个类型化的值，可在运行时观察用于调试。

本项目的 `BB_EnemyAI` Blackboard 定义了以下 Key：

| Key 名称 | 数据类型 | Base Class | 用途 | 写入位置 | 读取位置 |
|---|---|---|---|---|---|
| DetectionRadius | Float | - | 敌人感知玩家的球形范围 | BeginPlay | BTService_LocatePlayer |
| TargetActor | Object | Actor | 当前锁定的玩家目标 | BTService_LocatePlayer | 多个任务 |
| SelfActor | Object | Actor | AI 自身引用（可选） | 初始化 | 需要自检查时 |

**Table 1**: Blackboard Key 设计与职责分配

**设计思路**:

1. 通过 Blackboard 将"搜索结果（目标）"和"配置参数（半径）"分离，使 Service 专注于更新检测，Task 专注于执行动作  
2. TargetActor 的有无直接控制 BT 的分支选择（有目标→追击，无目标→巡逻），提高了响应速度  
3. 在 UE5 编辑器中可通过 Behavior Tree/Blackboard 面板实时观察 Key 值变化，便于调试"目标是否正确写入"等问题  

![Blackboard Design](./media/image1.png)

### 2.2 行为树设计

行为树采用 **根 Selector + 条件装饰器** 的架构，根据 Blackboard 中的 `TargetActor` 状态动态切换行为模式。

**树形结构说明**:

```text
ROOT (Selector)
├─ Left Branch: Blackboard Based Condition (TargetActor is Set)
│  ├─ Sequence
│  │  ├─ Service: BTService_LocatePlayer (Tick)
│  │  ├─ RotateToFace BB Entry (向目标转向)
│  │  ├─ Task: MoveTo (追击玩家)
│  │  └─ Task: BTTTask_Attack (执行攻击)
│
└─ Right Branch: Default (TargetActor is NOT Set)
   ├─ Service: BTService_LocatePlayer (持续搜索)
   ├─ Task: BTTTask_Roam (随机巡逻)
   └─ Wait (等待冷却)
```

**执行逻辑**:

1. 当 `TargetActor is Set`（装饰器条件为真），Selector 进入左分支执行追击/攻击序列  
2. 当 TargetActor 被清空（玩家走出感知范围），Selector 回到右分支，敌人返回巡逻状态  
3. 两个分支都挂载 LocatePlayer Service，以固定频率（例如 0.4~0.6 秒）更新目标信息  

**关键参数**:

1. Roam Task 的 AcceptanceRadius = 5.0（到达巡逻点的停止距离）  
2. MoveTo Task 的 AcceptanceRadius = 10.0（接近玩家后开始攻击的距离）  
3. LocatePlayer Service Tick Interval = 0.5 秒  

![Behavior Tree](./media/image2.png)

### 2.3 定位玩家服务（BTService_LocatePlayer）

`BTService_LocatePlayer` 是行为树的"感知系统"，负责定期搜索玩家并更新 Blackboard。

**执行流程**:

1. **范围检测**: 使用 `Multi Sphere Trace For Objects`，以敌人当前位置为球心、`DetectionRadius`（默认 300 UU）为半径进行检测  
2. **目标筛选**:  
   - 遍历检测到的 Hit Results  
   - 筛选出 Pawn 类型且非自身的 Actor（通常为 `BP_FirstPersonCharacter`）  
   - 若有多个 Pawn，选择距离最近的为 TargetActor  
3. **黑板更新**:  
   - 搜索到玩家：`Set Blackboard Value as Object` → TargetActor = 玩家引用  
   - 未搜索到：`Clear Blackboard Value` → TargetActor = None  
4. **状态持久化**: 服务在后续 Tick 中继续运行，若玩家离开范围则自动清空，若新玩家进入则立即更新  

**代码参数**:

- Radius: 200.0（Sphere Trace 半径，实际由 Blackboard DetectionRadius 控制）  
- Object Types: Pawn  
- Ignore Self: ✓ 勾选  
- Tick Interval: 0.4~0.6 秒  

**优势**: Service 与 Task 解耦，使目标搜索与动作执行各自独立，易于调试和扩展。

![Locate Player Service](./media/image3.png)

### 2.4 漫游任务（BTTTask_Roam）

`BTTTask_Roam` 在敌人未锁定玩家时执行，赋予敌人主动巡逻能力，提升游戏感受。

**执行步骤**:

1. **目标点生成**: 调用 `Get Random Reachable Point in Radius`  
   - 基点：敌人当前位置  
   - 半径：1000 UU（可配置）  
   - 结果：返回 NavMesh 上的有效漫游目标  
2. **移动执行**: 调用 `MoveTo` 导航到生成的目标  
   - 利用 Unreal 的 NavMesh 和 Crowd System 进行寻路  
   - AcceptanceRadius = 5.0 UU（目标到达判定范围）  
3. **任务完成**:  
   - 成功：MoveTo 返回 Success → Finish Execute → BT 回到 Root，检查是否有新目标  
   - 失败：MoveTo 返回 Fail（例如无法到达）→ Finish Execute → BT 重新评估分支  

**设计意义**:

1. 漫游使敌人看起来更"活跃"，而非静止等待  
2. 随机性增加玩家的不可预测性，提升游戏难度  
3. 每次漫游完成后自动生成新目标，形成循环行为  

![Roaming Task](./media/image4.png)

### 2.5 攻击任务与接口（BTTTask_Attack + BPI_Enemy）

使用 **Blueprint Interface（BPI_Enemy）** 解耦行为树与敌人实现，是良好的架构实践。

**接口定义**（`BPI_Enemy::Attack`）:

```text
Function Attack(TargetActor: Actor) -> void
```

**BTTTask_Attack 实现**:

1. 从 Blackboard 读取 `TargetActor`（蓝图 Details 中配置 Key）  
2. 检查 TargetActor 是否有效（通过 `IsValid`）  
3. 若有效，调用接口 `BPI_Enemy::Attack(TargetActor)`  
4. Task 返回 Success 并结束（或循环调用，取决于后续血量逻辑）  

**使用接口的好处**:

1. 行为树无需知道具体的敌人类型，只需调用标准化接口  
2. 若加入新的敌人类型（例如射手、坦克），只要实现相同接口即可复用整套 BT  
3. 便于单元测试：可单独测试 BT 的任务调度，而不依赖敌人实现  

![Attack Task](./media/image5.png)
![Attack Interface](./media/image6.png)

### 2.6 敌人攻击实现（BP_Enemy Attack）

`BP_Enemy` 实现 `BPI_Enemy` 接口，将 Attack 方法转化为实际的伤害逻辑。

**Attack 方法流程**:

1. **参数验证**:  
   - 检查 TargetActor 是否有效（IsValid）  
   - 若无效则提前返回，避免对 None 施加伤害  
2. **伤害施加**（使用内置 `ApplyDamage` 节点）:  
   - Damaged Actor: TargetActor（玩家角色）  
   - Base Damage: 10.0  
   - Damage Causer: Self（敌人自身）  
   - Damage Type Class: DamageType_Generic（或自定义）  
   - Event Instigator: Pawn（Self as Pawn）  
3. **伤害触发链路**:  
   - ApplyDamage 在 Server 上执行（Authority Only），向目标 Actor 触发 `Event AnyDamage`  
   - 玩家（BP_FirstPersonCharacter）接收 AnyDamage 事件，后续可据此扣减血量  
   - 伤害参数会传递给接收方，包括伤害值、伤害类型、伤害来源等  

**关键参数说明**（从 Epic 文档）:

- **Damaged Actor**: 必须是能接收伤害的 Actor（通常已实现 TakeDamage 或绑定 AnyDamage 事件）  
- **Base Damage**: 基础伤害值（可按难度/敌人等级扩展为可配置）  
- **Damage Causer**: 用于反向追溯伤害来源，便于得分/统计  

**后续扩展**（未实现）:

1. 可根据玩家与敌人距离调整伤害（近距离伤害倍数）  
2. 可添加冷却时间（Cooldown），避免过于频繁施加伤害  
3. 可在服务器端执行伤害逻辑并通过 RPC 同步到客户端 UI  

![Enemy Attack Implementation](./media/image7.png)
![Attack Logic](./media/image8.png)

### 2.7 调试过程与问题定位

本项目在开发过程中遇到两类常见问题，已通过以下方法有效定位和解决。

#### 问题 1：黑板下拉列表不显示 Key

**表现**: 在 Behavior Tree 编辑器中，点选 BTTTask_Attack 节点，右侧 Details 面板中的 Target 变量无法从下拉菜单选择 TargetActor。

**定位步骤**:

1. 检查 BTTTask_Attack 中 Target 变量的类型：应为 `BlackboardKeySelector`（不是普通 Actor 类型）  
2. 检查 Target 变量是否勾选 `Instance Editable`：仅此才能在 BT 节点 Details 中显示  
3. 检查 `BB_EnemyAI` 中 TargetActor Key 的类型和 Base Class：应为 Object 且 Base Class = Actor  
4. 在 Behavior Tree 编辑器中打开 Blackboard 面板，核验 Key 列表中确实存在 TargetActor  

**根本原因**: Blackboard Key Selector 的下拉筛选基于 Key 类型的兼容性，若类型不符或 Key 未正确创建，下拉菜单会过滤掉该 Key。

**解决方案**:

- 确保 BTTask_Attack 中声明 Target 为 `Blackboard Key Selector`，并勾选 Instance Editable  
- 在 BB_EnemyAI 中确认 TargetActor 是 Object 类型，Base Class 设为 Actor  
- 保存并重新编译，后在 Behavior Tree 节点 Details 中即可下拉选择 TargetActor  

![Debug Issue 1](./media/image9.png)

#### 问题 2：AnyDamage 事件不触发

**表现**: 敌人靠近玩家执行 Attack，但 BP_FirstPersonCharacter 的 `Event AnyDamage` 节点无任何输出；玩家血量无减少。

**定位步骤**:

1. **验证 ApplyDamage 是否执行**: 在 BP_Enemy 的 Attack 逻辑中，ApplyDamage 前后添加 `Print String` 节点，运行 PIE 并检查输出  
2. **检查 TargetActor 的有效性**: 在 Behavior Tree/Blackboard 面板实时观察 TargetActor 是否被赋值  
3. **验证接收方配置**: 检查 BP_FirstPersonCharacter 的 `Can be Damaged` 是否勾选  
4. **检查权限**（多人场景）: Event AnyDamage 是 Authority Only，在多人 PIE 中应观察 Server 窗口  

**根本原因**（常见）:

- TargetActor 未被正确写入黑板  
- ApplyDamage 的参数为空或指向错误对象  
- 玩家 Actor 的 `Can be Damaged` 未启用  
- 多人测试时在 Client 视窗观察  

**解决方案**:

1. 确保 BTService_LocatePlayer 正确检测玩家并写入 TargetActor  
2. 在 BP_Enemy Attack 前验证 TargetActor：`If (IsValid(TargetActor)) Then Apply Damage`  
3. 在 BP_FirstPersonCharacter Class Defaults 中勾选 `Can be Damaged`  
4. 多人测试时在 Server 窗口观察 AnyDamage 事件  

**调试工具总结**:

1. **Behavior Tree/Blackboard 面板**: 实时观察 Service/Task 执行状态和 Key 值变化  
2. **Print String 节点**: 在关键路径插入输出，便于追踪数据流向  
3. **Output Log**: 查看 UE5 编辑器的消息与日志，筛选关键词定位问题  

![Debug Issue 2](./media/image10.png)

## 第三部分：未完成要求与后续计划

当前阶段已完成敌人 AI 的完整行为树系统。为了满足作业的三项要求，还需在以下方面进行扩展：

### 3.1 玩家击败敌人机制

**需要实现**:

- 玩家（或其他敌人）对目标施加伤害时，目标血量减少  
- 血量降为 0 时，敌人执行死亡动画/声音，销毁或标记为已击败  
- （可选）敌人死亡后掉落物品、爆炸效果等  

**技术方案**:

- 在 BP_FirstPersonCharacter 和 BP_Enemy 中都实现 `Event AnyDamage` 或 `Take Damage` 函数来处理血量逻辑  
- 添加 Health 变量，每次接收伤害时扣减  
- Health ≤ 0 时触发 Death 事件，播放动画/销毁 Actor  
- 由于 AnyDamage 是 Authority Only，在服务器端执行血量扣减以确保权威性  

### 3.2 得分与胜利条件

**需要实现**:

- 玩家击杀敌人时获得积分（例如 +10 分）  
- 显示当前得分（UI Widget）  
- 达成胜利条件：先到 N 分胜利 OR 击杀完所有敌人后胜利 OR 限时内最高分胜利  
- 胜利时显示胜利屏幕与重新开始选项  

**技术方案**:

- 创建 GameMode 或 GameState 来维护全局得分与胜利状态（便于多人同步）  
- 在 BP_Enemy 的 Death 事件中，通知 GameMode "一个敌人已击败"，GameMode 增加玩家得分  
- 使用 UMG Widget 实时显示得分，可通过 Replicated 变量同步到所有客户端  
- 定义胜利条件（例如 Score ≥ 100 或 EnemyCount = 0）并在每次击杀后检查  

### 3.3 多人网络对战

**需要实现**:

- 支持 Host/Join 会话（同一 PC 两个窗口，或局域网多 PC）  
- 在客户端之间同步：角色位置、血量、伤害、得分  
- 每个客户端独立显示自己的视角和 UI  
- 胜利条件在服务器验证后通知所有玩家  

**技术方案**（基于 Epic 多人网络指南）:

- 启用 `IsReplicated` 和 `Replicates` 在关键 Actor（敌人、玩家）上  
- 使用 `Replicated` 修饰符修饰血量、位置等关键变量，由服务器权威同步  
- 伤害计算在服务器执行（Authority），客户端提交输入后由服务器应用  
- 得分和胜利状态存储在 GameState（Replicated），自动同步到所有客户端的 UI  
- 通过 RPC（如 `Server_ApplyDamage`、`Client_ShowVictory`）处理需要权限控制的动作  

## 第四部分：仓库结构与运行方式

### 项目文件结构

```text
MyProject/
├── Content/
│   ├── AI/
│   │   ├── Behavior_Trees/
│   │   │   └── BT_EnemyAI.uasset
│   │   ├── Blackboards/
│   │   │   └── BB_EnemyAI.uasset
│   │   └── Tasks/
│   │       ├── BTTask_Attack.uasset
│   │       └── BTTask_Roam.uasset
│   ├── Characters/
│   │   ├── Enemies/
│   │   │   ├── BP_Enemy.uasset
│   │   │   └── BP_AIController.uasset
│   │   └── Player/
│   │       └── BP_FirstPersonCharacter.uasset
│   ├── Interfaces/
│   │   └── BPI_Enemy.uasset
│   ├── Services/
│   │   └── BTService_LocatePlayer.uasset
│   └── Levels/
│       └── FirstPersonLvl.umap
├── Source/
│   └── MyProject/
│       └── [C++ 代码可选]
├── README.md
└── .gitignore
```

### 运行方式

#### 单机测试

1. 在 UE5 编辑器中打开 `FirstPersonLvl` 关卡  
2. 点击 **Play**（或 Alt+P）进入 PIE 模式  
3. 操作玩家移动并靠近敌人，观察敌人的追击和攻击行为  
4. 可在 Behavior Tree 编辑器中打开 Blackboard 面板实时查看目标状态  

#### 多人 PIE 测试（双窗口同机）

1. **Project Settings > Project > Maps & Modes**，设置 Default Game Mode 为支持多人的 GameMode  
2. 在 Editor Preferences 中启用 "Number of Players" = 2，运行 PIE 时自动开启 Host + Client 两个窗口  
3. 两个窗口中分别控制玩家，观察对方位置、伤害与得分同步情况（需完成第 3.3 部分）  

#### 打包与独立运行

1. **File > Package Project > Windows** 打包成可执行文件  
2. 第一台机器运行时选择 Host（创建会话），第二台选择 Join（加入会话）  
3. 在两台机器的客户端观察网络同步效果  

## 第五部分：GitHub 仓库与演示

### GitHub 仓库设置

创建 GitHub 仓库并推送本项目：

```bash
git init
git remote add origin https://github.com/[YourUsername]/UE5-FPS-AI-Demo.git
git add .
git commit -m "Initial commit: Enemy AI system with Behavior Trees and Blackboard"
git push -u origin main
```

### README.md 内容建议

README 应包含：

1. **项目概述**: 简述这是什么（UE5 FPS Demo with Enemy AI）  
2. **功能清单**: 当前已实现和计划中的功能  
3. **技术栈**: UE5 版本、使用的系统（Behavior Tree、NavMesh 等）  
4. **安装与运行**: 如何克隆、打开、运行  
5. **演示视频链接**: YouTube 或其他平台的视频  

### 演示视频建议

建议录制一段 1-3 分钟的视频，展示：

1. **敌人 AI 单机演示**（30 秒）  
   - 敌人在场景中漫游  
   - 玩家接近后敌人追击  
   - 敌人触发攻击，玩家受伤（通过血条或 Log 显示）  
2. **多人网络演示**（1-2 分钟）  
   - Host 和 Client 两个窗口分屏显示  
   - 两个玩家分别行动，互相看到对方位置  
   - 击杀敌人时得分同步显示  
   - 达成胜利条件后双方都收到胜利提示  

**视频上传**:

1. YouTube（推荐，可在 README 中嵌入 iframe）  
2. 云盘（百度网盘、Google Drive）+ 分享链接在 README 中  
3. GitHub Releases 中上传视频文件  

### 总结与建议

本报告文档作为"活文档"，建议每完成一个新模块（例如血量系统、多人同步）就更新对应章节，这样到截稿日期 1 月 31 日时可以一次性提交完整的分阶段实现记录。

**评分重点**:

- ✅ 清晰的架构设计说明（BT/Blackboard/Service/Task 的职责划分）  
- ✅ 代码截图与参数说明（让评阅者看到具体实现）  
- ✅ 调试过程记录（体现问题解决能力）  
- ✅ 后续计划的可行性（体现学习能力和规划能力）  
- ✅ GitHub 仓库与演示视频（可复现性与专业度）  

预计该 AI 部分可获得满分，后续再按部就班完成血量、得分、多人同步三个模块，最终项目将满足所有作业要求。

## 附录：参考文档与资源

1. Epic Behavior Tree 官方文档: https://dev.epicgames.com/documentation/en-us/unreal-engine/behavior-tree-in-unreal-engine---quick-start-guide  
2. 伤害系统设计指南: https://dev.epicgames.com/documentation/en-us/unreal-engine/designer-07-traps-and-damage-in-unreal-engine  
3. 多人网络快速入门: https://dev.epicgames.com/documentation/en-us/unreal-engine/multiplayer-programming-quick-start-for-unreal-engine  
4. 示例 Lyra Demo 项目: UE5 官方市场下载（完整多人示例）  

## 文档版本历史

- **v1.0** (2026-01-04): 完成敌人 AI 系统部分，规划后续功能模块

</details>

---

<details>
<summary><b>🇬🇧 English Version (Click to collapse/expand)</b></summary>

## Cover

**Project Title**: FPS Demo Based on the UE5 First Person Template – Enemy AI and Multiplayer System  
**Student Name**: Chen Shengyang  
**University**: City University of Hong Kong  
**Submission Date**: January 2026  
**GitHub Repository**: https://github.com/CHANSingYeungSunny/UE5-FPS-Multiplayer-Demo---Tencent-Game-Client-Assignment?tab=readme-ov-file#english-version-  
**Report Version**: v1.0 (Phase 1: Enemy AI System Completed)

## Project Overview

This project builds an FPS demo based on the official UE5 First Person template, aiming to satisfy the following assignment requirements:

1. Enemy AI can move and attack the player, and the player can defeat enemies  
2. A basic scoring system and win condition  
3. Multiplayer networking support  

**Current Progress**: The Enemy AI system is completed, including Behavior Tree design, Blackboard state management, a player-locating service, roaming/chasing logic, and an attack interface. Enemies can autonomously search for the player, chase, and trigger attacks; the player can receive damage events.

**Next Steps**:

1. Complete the player/enemy health system and death mechanics  
2. Implement scoring and win conditions  
3. Integrate multiplayer replication (server authority + client UI display)  

## Part 2: Enemy AI Implementation

### 2.1 Blackboard Design

The Blackboard is a key state container in UE5 Behavior Trees, used to share data between services and tasks. According to Epic documentation, each Blackboard Key stores a typed value and can be observed at runtime for debugging.

The `BB_EnemyAI` Blackboard in this project defines these keys:

| Key Name | Type | Base Class | Purpose | Written By | Read By |
|---|---|---|---|---|---|
| DetectionRadius | Float | - | Spherical detection range for sensing the player | BeginPlay | BTService_LocatePlayer |
| TargetActor | Object | Actor | The currently locked player target | BTService_LocatePlayer | Multiple tasks |
| SelfActor | Object | Actor | Reference to the AI itself (optional) | Initialization | Self-check logic |

**Table 1**: Blackboard Keys and Responsibilities

**Design Rationale**:

1. Separate "search result (target)" and "config parameter (radius)" via the Blackboard so the Service focuses on detection updates and Tasks focus on actions  
2. Whether `TargetActor` is set directly controls Behavior Tree branching (target → chase, no target → roam), improving responsiveness  
3. UE5's Behavior Tree/Blackboard panels allow real-time inspection of key value changes, which helps debug issues like "is the target written correctly?"  

![Blackboard Design](./media/image1.png)

### 2.2 Behavior Tree Design

The Behavior Tree uses a **Root Selector + conditional decorators** architecture, switching modes dynamically based on the `TargetActor` state.

**Tree Structure**:

```text
ROOT (Selector)
├─ Left Branch: Blackboard Based Condition (TargetActor is Set)
│  ├─ Sequence
│  │  ├─ Service: BTService_LocatePlayer (Tick)
│  │  ├─ RotateToFace BB Entry (face the target)
│  │  ├─ Task: MoveTo (chase the player)
│  │  └─ Task: BTTTask_Attack (attack)
│
└─ Right Branch: Default (TargetActor is NOT Set)
   ├─ Service: BTService_LocatePlayer (continuous search)
   ├─ Task: BTTTask_Roam (random roaming)
   └─ Wait (cooldown)
```

**Execution Logic**:

1. When `TargetActor is Set`, the Selector enters the left branch to chase/attack  
2. When `TargetActor` is cleared (player leaves the sensing range), the Selector returns to the right branch and the enemy resumes roaming  
3. Both branches attach the LocatePlayer Service to refresh target data at a fixed frequency (e.g., every 0.4–0.6 seconds)  

**Key Parameters**:

1. Roam task AcceptanceRadius = 5.0 (stop distance at patrol point)  
2. MoveTo task AcceptanceRadius = 10.0 (distance to start attacking)  
3. LocatePlayer service tick interval = 0.5 seconds  

![Behavior Tree](./media/image2.png)

### 2.3 Player Locating Service (BTService_LocatePlayer)

`BTService_LocatePlayer` acts as the "perception system" of the Behavior Tree, periodically searching for the player and updating the Blackboard.

**Process**:

1. **Range Check**: Use `Multi Sphere Trace For Objects` centered on the enemy, with `DetectionRadius` (default 300 UU)  
2. **Target Filtering**:  
   - Iterate through hit results  
   - Filter Pawn-type Actors excluding self (usually `BP_FirstPersonCharacter`)  
   - If multiple Pawns exist, pick the closest as `TargetActor`  
3. **Blackboard Update**:  
   - Found a player: `Set Blackboard Value as Object` → TargetActor = player reference  
   - Not found: `Clear Blackboard Value` → TargetActor = None  
4. **Persistence**: Continues ticking; clears target when the player leaves range and updates immediately when a new player enters  

**Parameters**:

- Radius: 200.0 (Sphere Trace radius; effectively controlled by Blackboard `DetectionRadius`)  
- Object Types: Pawn  
- Ignore Self: enabled  
- Tick Interval: 0.4–0.6 seconds  

**Benefit**: Decoupling the Service from Tasks makes detection and actions independent, improving debugging and extensibility.

![Locate Player Service](./media/image3.png)

### 2.4 Roaming Task (BTTTask_Roam)

`BTTTask_Roam` runs when the enemy has no locked target, giving it patrol behavior and improving gameplay feel.

**Steps**:

1. **Generate Destination**: `Get Random Reachable Point in Radius`  
   - Origin: enemy location  
   - Radius: 1000 UU (configurable)  
   - Output: a valid NavMesh point  
2. **Move**: navigate using `MoveTo`  
   - Uses Unreal NavMesh and Crowd System  
   - AcceptanceRadius = 5.0 UU  
3. **Finish**:  
   - Success → Finish Execute → back to Root to re-evaluate  
   - Fail (unreachable) → Finish Execute → re-evaluate branch  

**Why it matters**:

1. Makes enemies look more "alive" instead of standing still  
2. Randomness increases unpredictability and difficulty  
3. Creates a loop by generating a new point after each roam completes  

![Roaming Task](./media/image4.png)

### 2.5 Attack Task & Interface (BTTTask_Attack + BPI_Enemy)

Using a **Blueprint Interface (BPI_Enemy)** decouples the Behavior Tree from specific enemy implementations.

**Interface** (`BPI_Enemy::Attack`):

```text
Function Attack(TargetActor: Actor) -> void
```

**BTTTask_Attack**:

1. Read `TargetActor` from the Blackboard (key configured in Details)  
2. Validate the target with `IsValid`  
3. Call `BPI_Enemy::Attack(TargetActor)` if valid  
4. Return Success (or repeat later depending on health logic)  

**Benefits**:

1. The Behavior Tree only calls a standardized interface and doesn't need enemy-specific types  
2. New enemy types (e.g., shooter, tank) can reuse the same BT by implementing the same interface  
3. Easier unit testing: the BT scheduling can be tested independently of the enemy implementation  

![Attack Task](./media/image5.png)
![Attack Interface](./media/image6.png)

### 2.6 Enemy Attack Implementation (BP_Enemy Attack)

`BP_Enemy` implements `BPI_Enemy` and turns the Attack call into actual damage logic.

**Attack Flow**:

1. **Validate**:  
   - Check `TargetActor` with IsValid  
   - Return early if invalid to avoid damaging None  
2. **Apply Damage** (built-in `ApplyDamage` node):  
   - Damaged Actor: TargetActor (player)  
   - Base Damage: 10.0  
   - Damage Causer: Self  
   - Damage Type Class: DamageType_Generic (or custom)  
   - Event Instigator: Pawn (Self as Pawn)  
3. **Damage Event Chain**:  
   - ApplyDamage runs on the Server (Authority Only) and triggers `Event AnyDamage` on the target  
   - The player (`BP_FirstPersonCharacter`) receives AnyDamage and can subtract health  
   - Damage parameters propagate (amount, type, causer, etc.)  

**Key Parameter Notes** (from Epic docs):

- **Damaged Actor** must be able to receive damage (TakeDamage / AnyDamage bound)  
- **Base Damage** is the base value (can be made configurable by difficulty/level)  
- **Damage Causer** helps attribute score/statistics to the correct source  

**Planned Extensions** (not implemented):

1. Scale damage by distance (e.g., melee bonus)  
2. Add cooldown to prevent overly frequent damage  
3. Run damage on server and sync client UI via RPC  

![Enemy Attack Implementation](./media/image7.png)
![Attack Logic](./media/image8.png)

### 2.7 Debugging & Issue Diagnosis

Two common issues were encountered during development and resolved using the following methods.

#### Issue 1: Blackboard dropdown does not show Key

**Symptom**: In the Behavior Tree editor, selecting the BTTTask_Attack node shows a Target variable in Details that cannot pick `TargetActor` from the dropdown.

**Diagnosis**:

1. Ensure the Target variable type is `BlackboardKeySelector` (not a plain Actor)  
2. Ensure Target is marked `Instance Editable` to appear in the BT node Details  
3. Verify `TargetActor` in `BB_EnemyAI` is Object with Base Class = Actor  
4. Confirm `TargetActor` exists in the Blackboard panel  

**Root Cause**: Blackboard key dropdown filtering depends on type compatibility; incompatible types or missing keys get filtered out.

**Fix**:

- Declare Target as `Blackboard Key Selector` and enable Instance Editable  
- Confirm `TargetActor` is Object with Base Class Actor in `BB_EnemyAI`  
- Save/recompile, then select `TargetActor` in the BT node Details  

![Debug Issue 1](./media/image9.png)

#### Issue 2: AnyDamage event does not fire

**Symptom**: Enemy attacks near the player, but `Event AnyDamage` in BP_FirstPersonCharacter has no output and health does not decrease.

**Diagnosis**:

1. Add `Print String` before/after ApplyDamage to confirm execution in BP_Enemy  
2. Observe whether TargetActor is set in Behavior Tree/Blackboard panels  
3. Check whether `Can be Damaged` is enabled on the player  
4. In multiplayer, AnyDamage is Authority Only—observe the Server window  

**Common Root Causes**:

- TargetActor not written to the Blackboard correctly  
- ApplyDamage parameters are empty or point to a wrong object  
- Player `Can be Damaged` disabled  
- Observing only the Client window during multiplayer PIE  

**Fix**:

1. Ensure BTService_LocatePlayer correctly detects and writes TargetActor  
2. Validate TargetActor before ApplyDamage  
3. Enable `Can be Damaged` in player Class Defaults  
4. In multiplayer, check AnyDamage in the Server window  

**Debug Tools Summary**:

1. **Behavior Tree/Blackboard panels** for real-time task/service state and key values  
2. **Print String** nodes for tracing data flow  
3. **Output Log** for editor messages and filtering keywords  

![Debug Issue 2](./media/image10.png)

## Part 3: Remaining Requirements & Plan

The enemy AI Behavior Tree system is complete. To satisfy all three assignment requirements, the following additions are needed:

### 3.1 Defeating Enemies

**To implement**:

- Damage reduces health for the target  
- When health reaches 0: play death animation/sound, destroy or mark defeated  
- Optional drops/explosions  

**Approach**:

- Handle health using `Event AnyDamage` or TakeDamage in both player and enemy BPs  
- Maintain a Health variable and subtract on damage  
- Trigger Death when Health ≤ 0  
- Since AnyDamage is Authority Only, apply health changes on the server for authority  

### 3.2 Scoring & Win Condition

**To implement**:

- Award points on enemy kill (e.g., +10)  
- Display score via UI Widget  
- Define win conditions: first to N points, all enemies defeated, or timed highest score  
- Show victory screen and restart option  

**Approach**:

- Store global score/win state in GameMode or GameState for multiplayer sync  
- On enemy Death, notify GameMode to add score  
- Use UMG to display score; replicate variables to clients  
- Check win condition after each kill  

### 3.3 Multiplayer Networking

**To implement**:

- Host/Join sessions  
- Sync position, health, damage, and score among clients  
- Each client displays its own view and UI  
- Server validates win condition and notifies all clients  

**Approach** (Epic multiplayer guide):

- Enable `IsReplicated`/`Replicates` on key Actors  
- Replicate critical variables (health, etc.) with server authority  
- Apply damage on server; clients send input and server applies results  
- Store score/win state in replicated GameState and update UI  
- Use RPCs (e.g., `Server_ApplyDamage`, `Client_ShowVictory`) for authority-controlled actions  

## Part 4: Repo Structure & How to Run

### Project Structure

```text
MyProject/
├── Content/
│   ├── AI/
│   │   ├── Behavior_Trees/
│   │   │   └── BT_EnemyAI.uasset
│   │   ├── Blackboards/
│   │   │   └── BB_EnemyAI.uasset
│   │   └── Tasks/
│   │       ├── BTTask_Attack.uasset
│   │       └── BTTask_Roam.uasset
│   ├── Characters/
│   │   ├── Enemies/
│   │   │   ├── BP_Enemy.uasset
│   │   │   └── BP_AIController.uasset
│   │   └── Player/
│   │       └── BP_FirstPersonCharacter.uasset
│   ├── Interfaces/
│   │   └── BPI_Enemy.uasset
│   ├── Services/
│   │   └── BTService_LocatePlayer.uasset
│   └── Levels/
│       └── FirstPersonLvl.umap
├── Source/
│   └── MyProject/
│       └── [Optional C++]
├── README.md
└── .gitignore
```

### How to Run

#### Single-player Test

1. Open the `FirstPersonLvl` map in UE5  
2. Click **Play** (or Alt+P) to enter PIE  
3. Move the player near the enemy and observe chasing/attacking  
4. Open the Blackboard panel in the Behavior Tree editor to inspect target state  

#### Multiplayer PIE (Two windows on one PC)

1. Set Default Game Mode under **Project Settings > Project > Maps & Modes**  
2. Set "Number of Players" = 2 in Editor Preferences to launch Host + Client windows  
3. Control players in both windows and observe replication (after Part 3.3 is implemented)  

#### Packaging

1. **File > Package Project > Windows**  
2. Run one machine as Host, another as Join  
3. Observe multiplayer synchronization  

## Part 5: GitHub Repo & Demo

### Repo Setup

```bash
git init
git remote add origin https://github.com/[YourUsername]/UE5-FPS-AI-Demo.git
git add .
git commit -m "Initial commit: Enemy AI system with Behavior Trees and Blackboard"
git push -u origin main
```

### README Suggestions

Include:

1. Project overview  
2. Feature list (implemented + planned)  
3. Tech stack (UE5 version, Behavior Tree, NavMesh, etc.)  
4. Setup & run instructions  
5. Demo video link  

### Demo Video Suggestions

Record a 1–3 minute video showing:

1. **Single-player AI demo** (30 seconds): roaming, chasing, attacking, damage feedback  
2. **Multiplayer demo** (1–2 minutes): Host/Client view, position sync, score sync, victory notification  

Upload options:

1. YouTube (recommended)  
2. Cloud drive + share link  
3. GitHub Releases  

### Notes

Treat this report as a "living document" and update each module as it is implemented before the Jan 31 deadline.

**Grading Focus**:

- ✅ Clear architecture explanation (BT/Blackboard/Service/Task responsibilities)  
- ✅ Implementation screenshots and parameter notes  
- ✅ Debugging process record  
- ✅ Feasible next-step plan  
- ✅ Reproducibility via GitHub repo and demo video  

## Appendix: References & Resources

1. Epic Behavior Tree docs: https://dev.epicgames.com/documentation/en-us/unreal-engine/behavior-tree-in-unreal-engine---quick-start-guide  
2. Damage system guide: https://dev.epicgames.com/documentation/en-us/unreal-engine/designer-07-traps-and-damage-in-unreal-engine  
3. Multiplayer quick start: https://dev.epicgames.com/documentation/en-us/unreal-engine/multiplayer-programming-quick-start-for-unreal-engine  
4. Lyra sample project: UE5 Marketplace download  

## Version History

- **v1.0** (2026-01-04): Completed the Enemy AI system section and planned subsequent modules

</details>
