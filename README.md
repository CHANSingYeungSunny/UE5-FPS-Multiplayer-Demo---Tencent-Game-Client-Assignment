# UE5 FPS Multiplayer Demo - Tencent Game Client Assignment

**Repository**: https://github.com/YOUR_USERNAME/UE5-FPS-TencentGame-Demo

## 🎮 Gameplay
Multiplayer game where players compete to kill AI enemies. First to reach **10 points** wins.

## 🚀 How to Run
1. Open `TencentGame.uproject` with UE5.x
2. **Play Settings**: Number of Players = **2**, Net Mode = **Play as Listen Server**
3. Test: Enemy AI, kill sync, score display, win condition

## 🛠️ Technical Implementation
- **Enemy AI**: NavMeshBoundsVolume + AIController + Behavior Tree (Chase → Attack → Server Damage)
- **Multiplayer Weapons**: Server_Fire RPC + Projectile LineTrace → Server ApplyDamage
- **Score Sync**: PlayerState Replicated Score variable + GameMode OnEnemyKilled event
- **Win Condition**: GameState Replicated MatchState + UMG Victory Screen

[Demo Video Link]
