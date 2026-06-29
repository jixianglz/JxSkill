# HJATS 实盘引擎技能（OpenClaw / 小龙虾专用）

## 概述

HJATS 实盘引擎的 OpenClaw 端操作手册。Cline 负责代码，OpenClaw 负责跑。

## 目录

1. [项目结构](#项目结构)
2. [启动实盘](#启动实盘)
3. [监控检查](#监控检查)
4. [问题诊断](#问题诊断)
5. [CSV 日志解读](#csv-日志解读)
6. [协作流程](#协作流程)

---

## 项目结构

```
/home/ubuntu/HJATS/
├── run_live.py                  ← 实盘入口（OpenClaw 启动）
├── run_backtest.py              ← 回测入口（Cline 跑）
├── scripts/
│   └── hjats_agent.py           ← AI 协同接口
├── src/
│   ├── engine/
│   │   ├── driver.py            ← DP 线程
│   │   ├── strategy.py          ← SM 线程
│   │   ├── order_manager.py     ← OM 线程
│   │   ├── risk_manager.py      ← 风控
│   │   └── live_status.py       ← 状态文件管理
│   └── data/
│       └── live_logger.py       ← CSV 日志
├── strategies/
│   └── config.ini               ← 策略配置
├── .env                         ← API 密钥
├── reports/                     ← CSV 日志目录
│   └── live_YYYYMMDD_HHMMSS/    ← 每次实盘会话生成
└── .clinerules                  ← Cline 规则
```

## 启动实盘

### 方式一：AI Agent 接口（推荐）

```bash
cd /home/ubuntu/HJATS
python3 scripts/hjats_agent.py start-live
```

返回示例：
```json
{
  "action": "start_live",
  "pid": 12345,
  "log": "/tmp/live_engine.log",
  "status_cmd": "python3 scripts/hjats_agent.py status"
}
```

### 方式二：直接运行

```bash
cd /home/ubuntu/HJATS
tmux new -s hjats-live "python3 run_live.py"
# 或后台:
nohup python3 run_live.py > /tmp/live_engine.log 2>&1 &
```

### 停止

```bash
python3 scripts/hjats_agent.py stop-live
```

### 重启流程（更新代码后）

```bash
# 1. 停旧引擎
python3 scripts/hjats_agent.py stop-live

# 2. 拉最新代码
git pull

# 3. 启新引擎
python3 scripts/hjats_agent.py start-live

# 4. 等第一个监控tick确认状态
sleep 30
python3 scripts/hjats_agent.py status
```

## 监控检查

### 快速状态

```bash
python3 scripts/hjats_agent.py status
```

关键字段：

| 字段 | 说明 | 正常值 |
|:---|:---|:---|
| `running` | 引擎运行 | `true` |
| `paused` | 风控暂停 | `false` |
| `balance` | 账户余额 | > 0 |
| `signal.value` | 策略信号 | -1/0/1 |
| `strategy_last` | 策略上次执行时间 | 5分钟内 |
| `monitor_last` | 监控上次执行时间 | 30秒内 |
| `errors` | 错误列表 | `[]` |

### 原始状态文件

```bash
cat /tmp/live_status.json | python3 -m json.tool
```

完整字段结构：
```json
{
  "timestamp": "2026-06-29 22:00:00",
  "engine_running": true,
  "engine_paused": false,
  "account": {
    "balance": 20.0,          // 账户余额
    "equity": 20.0,           // 权益
    "daily_pnl": 0.0,         // 当日盈亏
    "daily_max_dd": 0.0,      // 当日最大回撤
    "peak_equity": 20.0       // 权益峰值
  },
  "position": {
    "side": null,             // LONG/SHORT/null
    "size": 0.0,              // 持仓量
    "entry_price": 0.0,       // 开仓价
    "current_price": 1569.11, // 当前市价
    "unrealized_pl": 0.0      // 浮动盈亏
  },
  "signal": {
    "value": 0,               // -1做空/0无/1做多
    "indicators": {
      "ind1": 1570.13,        // MA10
      "ind2": 1577.20         // MA30
    }
  },
  "engine": {
    "strategy_interval": 300, // 策略间隔(秒)
    "monitor_interval": 30,   // 监控间隔(秒)
    "strategy_last_run": null,
    "monitor_last_run": null,
    "strategy_ok": true,
    "monitor_ok": true,
    "errors": []
  },
  "daily": {
    "trade_count": 0,
    "win_count": 0,
    "loss_count": 0,
    "start_balance": 20.0,
    "start_date": "2026-06-29"
  },
  "last_orders": []
}
```

### CSV 日志

实盘会话自动生成在 `reports/live_YYYYMMDD_HHMMSS/` 下：

| 文件 | 内容 | 频率 |
|:---|:---|:---|
| `strategy_ticks.csv` | 策略 tick 记录 | 每5分钟 |
| `monitor_ticks.csv` | 监控 tick 记录 | 每30秒 |
| `trades.csv` | 成交记录 | 有交易时 |

查看：
```bash
cat reports/live_*/strategy_ticks.csv
cat reports/live_*/monitor_ticks.csv | tail -5
```

## 问题诊断

### 1. 状态文件卡住不更新
```
现象: status timestamp 超过 2 分钟没变
检查: 引擎是否还活着
```
```bash
pgrep -f run_live.py
tail -20 /tmp/live_engine.log
```

### 2. 错误定位
```
常见错误:
- "Invalid symbol." → Binance 交易对格式问题
- "NoneType object has no attribute 'loc'" → rawdata_show 未初始化
- SM/OM thread died → 线程异常退出，需配合 Cline 修
```

### 3. 信号异常
```
现象: signal 值异常或 indicators 为空
检查: strategy.py 中的信号算法或 run_live.py 中的传递逻辑
```

### 4. 引擎卡死
```
现象: 进程在但状态不更新
诊断: cat /proc/<PID>/wchan
      通常为 futex_wait_queue → 队列死锁
处理: 停引擎 → 告知用户 → 等 Cline 修复后重启
```

## CSV 日志解读

### strategy_ticks.csv

```
timestamp,signal,ma10,ma30,close,balance
2026-06-29 22:16:48,0,1570.13,1577.2,1562.6,20.0
2026-06-29 22:21:48,0,1568.93,1576.94,1562.53,20.0
```

- signal=0: 无信号，MA10 < MA30
- signal=1: 金叉，MA10 上穿 MA30 → 做多
- signal=-1: 死叉，MA10 下穿 MA30 → 做空
- 差价 = ma10 - ma30，正数表示金叉状态

### monitor_ticks.csv

```
timestamp,balance,price,risk_action,risk_reason
2026-06-29 22:12:18,20.0,1569.11,none,
```

- risk_action: `none`(正常) / `pause`(暂停) / `stop`(停止)
- risk_reason: 触发原因

## 协作流程

```
用户需求 → 分析 → 需要改代码？ → Cline 写 / OpenClaw 报告问题
                                        ↓
                                  Cline 修完 commit
                                        ↓
                                  OpenClaw 拉代码重启验证
                                        ↓
                                  监控运行状态
                                        ↓
                                  发现问题 → 报告用户 → 回到 Cline
```

**严禁：** OpenClaw 发现 bug 后直接改代码。修复 → Cline，运行 → OpenClaw。

---

*谁写的：OpenClaw 小龙虾，2026-06-29*
*最后更新：2026-06-29*
