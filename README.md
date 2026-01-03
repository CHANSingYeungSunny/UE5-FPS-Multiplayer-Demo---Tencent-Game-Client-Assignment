# UE5 FPS Multiplayer Demo - 开局一课客户端大作业

**Repository**: https://github.com/CHANSingYeungSunny/UE5-FPS-Multiplayer-Demo---Tencent-Game-Client-Assignment?tab=readme-ov-file

## 🌐 Language / 语言

| [🇺🇸 English](#english-version-) | [🇨🇳 中文](#chinese-version-) |
|--------------------------------|--------------------------------|

---

## English Version 🇺🇸

### 🎮 Gameplay
Multiplayer game where players compete to kill AI enemies. **Kill 1 enemy = +1 point. First to 10 points wins.**

### 🚀 How to Run
1. Open `TencentGame.uproject` with UE5.x
2. **Play Settings**: Number of Players = **2**, Net Mode = **Play as Listen Server**
3. Test: Enemy AI, kill sync, score display, win screen

### 🛠️ Technical Implementation
- **Enemy AI**: NavMeshBoundsVolume + AIController + Behavior Tree (Detect→Chase→Attack→Server Damage)
- **Multiplayer Weapons**: Server_Fire RPC + Projectile LineTrace → Server ApplyDamage
- **Score Sync**: PlayerState Replicated Score + GameMode OnEnemyKilled
- **Win Condition**: GameState Replicated MatchState + UMG Victory UI

[Demo Video - if available]

---

## Chinese Version 🇨🇳

### 🎮 玩法说明
多人联机击杀会移动/攻击的敌人，**击杀 1 个敌人 +1 分，先到 10 分获胜**。

### 🚀 运行步骤
1. 用 UE5.x 打开 `TencentGame.uproject`
2. **Play 设置**: Number of Players = **2**，Net Mode = **Play as Listen Server**
3. 测试要点：敌人追击、击杀同步、分数显示、胜利界面

### 🛠️ 技术实现
- **敌人 AI**: NavMeshBoundsVolume + AIController + Behavior Tree (发现→追击→攻击→服务器伤害)
- **多人武器**: Server_Fire RPC + 发射物 LineTrace → 服务器 ApplyDamage
- **得分同步**: PlayerState 复制 Score 变量 + GameMode OnEnemyKilled 事件
- **胜利判定**: GameState 复制比赛状态 + UMG 胜利界面

[演示视频链接 - 如有]
