# 开发交接 / HANDOFF（换机器 or 新会话续接用）

> 给「在新电脑 / 新 Claude 会话里继续开发四武地牢」的人看。先读本文 + `DESIGN.md` + `git log` 即可恢复上下文。

## 一句话现状
单文件 HTML5 Canvas 像素 Roguelite，**已发布上线**：<https://cyys312.github.io/siwu-dungeon/>
当前 **v1.6.1**：4 武器 × 3 流派 = **200 天赋（全机制型）**、10 Boss、宝库/挑战/商店/祭坛房、元进度魂树。

## 怎么跑起来
纯静态、零依赖。任选其一：
- 直接双击 `index.html` 用浏览器打开；
- 或本地起服务器（推荐，避免个别浏览器限制）：`python3 -m http.server 8642`，浏览器开 `http://localhost:8642`。

## 怎么测（重要）
游戏内置 `DBG` 调试接口（浏览器控制台）：
- `DBG.start('hammer'|'sword'|'mage'|'ranger')` 直接开局
- `DBG.learn('W13', 2)` 灌天赋、`DBG.god()` 无敌、`DBG.warpRoom('boss'|'vault'|'shop')` 传送、`DBG.killAll()` 清场
- `run`, `player`, `enemies` 都是全局，`dmgEnemy/tryAttack/killEnemy/update` 等函数也是全局，可在控制台直接调
- **确定性单元测试套路**：新开注入实例 → `run.shopDmg=0`（去掉元进度污染）→ `freeze=0` → 构造极简 enemy 对象 → 调 `dmgEnemy`/`tryAttack` 读结果。全部在一个同步 eval 里跑保证原子性。

### 测试踩过的坑
1. `META.up.dmg` 会加进 `run.shopDmg` 污染基础伤害 → 测前置 `run.shopDmg=0`。
2. `freeze>0` 会让 `update()` 提前 return → 测前置 `freeze=0`。
3. 「命中即消耗」型子实体（火星四溅 P10 / 法术回响 U12 / 爆裂箭）会生成在敌人身上同帧被吃掉，测「存活数」会=0，要测「总伤害」。
4. 乘法+四舍五入的加成在低基础会被吞（曾坑 B11）→ 低基础加成优先用 flat 加法。

## 代码结构（index.html 单文件 ~2600 行）
- `TALENTS[]`（约 285 行起）：天赋定义。字段 `id/w/s/name/max/desc`，可选 `req/key/awaken/awaken2/tag:'primary'`。
- `tc('id')`：读天赋等级。`genOffer()` 抽卡（按 w/s 过滤 + 权重 + 键石保底）。`applyTalent()` 应用。
- 战斗钩子：`dmgEnemy`(伤害结算/暴击/易伤/DoT)、`killEnemy`(掉落/on-kill)、`damagePlayer`(减伤/护盾/格挡)、`tryAttack`(各武器攻击)、`releaseBow`、`updateSwords`(飞剑)、`updateFireOrbs`(环火)、`explodeMine`、`zapChain`、`enemyUpdate`、`update`(主循环)。
- **加新天赋的高效套路**：def 放「觉醒段前一个块」，再把 `tc('id')` 接进对应武器的伤害/攻速公式行 or 相应钩子。链式/爆炸类务必加终止守卫（`dot:true` / 标记位 / `!e.dead` / arm 检查）防无限递归。
- 通用暴击通道：`dmgEnemy` 的 `o.critBonus`；通用易伤：`e.markT>0` 受伤 ×1.2。
- 究极觉醒开关：`run.aw2h/aw2s/aw2m/aw2r` 驱动公式（在 `applyTalent` 里置位）。

## 平衡待办（实机试玩后记录，未改）
1. 奥术流「穿透+分裂」单体爆发偏高（法师约 3s 磨掉 Boss）。
2. 远程/AoE 风筝流几乎不挨打 → 可给深层怪加突进/远程施压。
3. Boss 血量已随玩家等级缩放（v1.6.1 `powScale`），如仍觉得太快/太肉在 `spawnBoss` 调系数。

## 部署
GitHub Pages（`main` 分支根目录，legacy 源）。push 到 main 自动 build+deploy。
> ⚠️ 偶发 `deploy Failed: "Deployment failed, try again later"` 是 GitHub Pages 后端临时故障，**不是代码问题**。去 Actions 里 Re-run failed jobs 即可；即便某次没部署，线上仍是上一个成功版本。
