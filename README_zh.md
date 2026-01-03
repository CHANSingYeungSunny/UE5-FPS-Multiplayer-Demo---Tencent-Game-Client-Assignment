# UE5 FPS 多人联机 Demo - 开局一课客户端大作业

**仓库**: https://github.com/你的用户名/UE5-FPS-TencentGame-Demo

## 🎮 玩法说明
多人联机击杀会移动/攻击的敌人，**击杀 1 个敌人 +1 分，先到 10 分获胜**。

## 🚀 运行步骤
1. 用 UE5.x 打开 `TencentGame.uproject`
2. **Play 设置**: Number of Players = **2**，Net Mode = **Play as Listen Server**
3. 测试要点：敌人追击、击杀同步、分数显示、胜利界面

## 🛠️ 技术实现
- **敌人 AI**: NavMeshBoundsVolume + AIController + Behavior Tree (发现→追击→攻击→服务器伤害)
- **多人武器**: Server_Fire RPC + 发射物 LineTrace → 服务器 ApplyDamage
- **得分同步**: PlayerState 复制 Score 变量 + GameMode OnEnemyKilled 事件
- **胜利判定**: GameState 复制比赛状态 + UMG 胜利界面

[演示视频链接]
