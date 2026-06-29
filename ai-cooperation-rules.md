# AI 协作规则（OpenClaw + Cline + HJATS）

## 项目

**HJATS** — 自动化交易系统（CTA 实盘引擎）
- **OpenClaw（小龙虾）**: 实盘运行、监控、状态查询、问题报告
- **Cline**: 代码开发、修复、重构、回测分析
- 分工明确，不越界

## 核心规则

### 规则一：不能擅自修改项目代码

涉及 HJATS 项目代码的**增、删、改**，必须先向用户说明变更内容，获得确认后才能执行。

✅ 正确做法：
- 发现问题 → 报告给用户 → 让 Cline 修
- 需要改进 → 提方案给用户确认 → 让 Cline 实现

❌ 错误做法：
- 发现 bug 直接改代码
- 看到可以优化的地方直接写
- 在 `run_live.py` 中顺手加功能

### 规则二：读自由，写确认

- **读操作**（cat/log/status/json/CSV）：随时可以执行
- **写操作**（创建/修改/删除项目文件）：必须先问

### 规则三：实盘协作分工

```
OpenClaw（监控层）              Cline（开发层）
─────────────────              ──────────────
✓ 启动/停止实盘引擎             ✓ 开发新功能
✓ 周期性检查引擎状态            ✓ 修复 bug
✓ 报告异常（错误/卡死/丢失）    ✓ 重构代码
✓ 查询状态/CSV/日志             ✓ 跑回测分析
✓ 拉取最新代码重启              ✓ 写文件操作
✓ 只读操作（cat/log/status）    ✓ 修改 .clinerules
```

### 规则四：实盘运行流程

```
1. Cline 完成代码修改并 commit
2. OpenClaw 停旧引擎 → 拉代码 → 启动新引擎
3. OpenClaw 周期性监控状态
4. 发现异常（卡死/崩溃/数据异常）→ 报告用户 → 等 Cline 修
5. Cline 修完后重复步骤 1-4
```

### 规则五：异常处理流程

```
发现异常（状态不更新/进程卡死/错误日志）
        │
        ▼
报告用户，描述具体现象
  - 什么时间发现
  - 错误日志内容
  - 进程状态
  - 可能原因分析
        │
        ▼
等用户确认 → 停引擎 → 等 Cline 修 → 重启验证
```

## 实盘常用命令

```bash
# OpenClaw 执行：
python3 scripts/hjats_agent.py status        # 查看状态
python3 scripts/hjats_agent.py start-live    # 后台启动
python3 scripts/hjats_agent.py stop-live     # 停止
python3 scripts/hjats_agent.py list-data     # 查看数据
python3 scripts/hjats_agent.py list-reports  # 查看报告

cat /tmp/live_status.json | python3 -m json.tool   # 原始状态
tail -20 /tmp/live_engine.log                       # 引擎日志
cat reports/live_*/strategy_ticks.csv               # 指标历史
cat reports/live_*/monitor_ticks.csv | tail -5      # 监控最新
```

## 状态文件字段速查

| 字段 | 说明 |
|:---|:---|
| `timestamp` | 最后更新时间 |
| `engine_running` | 引擎是否运行 |
| `account.balance` | 账户余额 |
| `account.daily_pnl` | 当日盈亏 |
| `position.side` | 持仓方向 (LONG/SHORT/null) |
| `signal.value` | 策略信号 (-1做空/0无/1做多) |
| `signal.indicators.ind1` | MA10 值 |
| `signal.indicators.ind2` | MA30 值 |
| `engine.strategy_last_run` | 上次策略执行时间 |
| `engine.monitor_last_run` | 上次监控执行时间 |
| `engine.errors` | 错误列表 |
| `daily.trade_count` | 今日交易次数 |
| `daily.win_count` | 盈利次数 |
| `daily.loss_count` | 亏损次数 |

## 操作前检查清单

```markdown
- [ ] 要向用户说明改什么？
- [ ] 是「读」还是「写」操作？
- [ ] 如果是写操作——获得用户确认了吗？
- [ ] 确认过是 Cline 的职责还是 OpenClaw 的职责？
- [ ] 当前引擎是否在运行？（要不要先停）
```

## 实战案例参考

### 案例：OM 线程崩溃（2026-06-29）

**现象**: engine 运行 ~2h 后状态文件卡住 2.5h 不更新

**诊断**:
```bash
# 1. 查进程 - 还在
pgrep -f run_live.py  →  3457031

# 2. 查日志
cat /tmp/live_engine.log  →  
ERROR:src.engine.order_manager:[OM] Error: 'NoneType' object has no attribute 'loc'
# rawdata_show 未初始化

# 3. 查线程
cat /proc/3457031/wchan  →  futex_wait_queue (死等)

# 4. 根因分析
# rawdata_show 只在回测模式初始化，实盘为 None
# OM 线程 crash → break 退出 → dp_queue.join() 死锁
```

**处理**: 报告用户 → Cline 修 → 拉代码重启验证 ✅

### 案例：indicators 空值时（本次实盘同一日）

**现象**: 状态文件 `indicators: {}` 为空

**诊断**:
```python
# run_live.py 中传递硬编码空字典
live_status.update_signal(dm.signal[-1], {})  # ← bug
```

**处理**: Cline 改为传入真实指标数据 ✅

## 原则

1. **读自由，写确认** — 任何写操作必须先报告
2. **有问题先报告，不擅自修** — OpenClaw 的职责是发现和报告
3. **不确定就多问一句** — 宁可多问一次，不要改错一行
4. **记录实战案例** — 每次修复都记录到本文件，方便下次快速定位
5. **规则持续更新** — 实际操作中发现的边界情况，应当补充进本文件

## 相关资源

- [HJATS 仓库](https://github.com/jixianglz/HJATS)
- [本技能库](https://github.com/jixianglz/JxSkill)
- [实盘操作手册](./hjats-live-engine-skill.md)
- [报告服务器指南](./hjats-report-server-skill.md)
- [VS Code Server 指南](./vscodeserver-skill.md)

---

*最后更新：2026-06-29 | 维护方：OpenClaw 小龙虾*
