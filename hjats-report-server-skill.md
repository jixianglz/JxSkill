# HJATS Report Server Skill

## 概述

HJATS 回测系统生成的报告，通过一个手机自适应的网页服务器对外展示。AI（Cline）只需按规范生成报告 HTML 文件，服务器自动扫描展示。

## 项目结构

```
~/HJATS/
├── src/                        # 框架核心（引擎、数据、broker等）
├── scripts/
│   ├── fetch_data.py           # Binance 数据下载
│   └── report_generator.py     # 生成网页回测报告
├── run.py                      # 框架入口
├── run_backtest.py             # 独立回测运行器
├── data/                       # K 线数据 CSV
├── reports/                    # ← 回测报告目录（重点）
│   ├── report_server.py        # 动态首页服务器
│   ├── bt_20260625_161139_.../ # 每次回测一个独立目录
│   │   ├── report.html         # 手机自适应网页报告
│   │   ├── chart_kline.html    # K线图+买卖点
│   │   └── chart_equity.html   # 收益曲线
│   └── ...更多回测目录...
└── MCP servers/                # AI 可调用工具
    ├── hjats-data              # fetch_klines, list_data_files
    └── hjats-backtest          # run_backtest, list_reports
```

## 报告服务器架构

```
浏览器(手机) 📱
     │
     ▼
Cloudflare Tunnel (公网入口)
https://xxxx.trycloudflare.com ← 每次重启会变
     │
     ▼
Python HTTP Server (本机 :8081)
~/HJATS/reports/report_server.py
     │
     ▼
扫描 reports/ 下 bt_* 目录 → 生成动态首页
                         → 静态文件服务（HTML/PNG 等）
```

### 进程管理（tmux）

两个服务跑在 tmux 会话 `claw` 里：

| 窗口名 | 内容 | 端口 |
|:---|:---|:---:|
| `report-server` | Python 报告服务器 | 8081 |
| `cf-tunnel` | Cloudflare 隧道 | → 8081 |

查看/管理方法：
```bash
tmux attach -t claw          # 进入 tmux 会话
tmux list-windows -t claw    # 列出窗口
tmux kill-window -t claw:cf-tunnel  # 杀隧道进程
```

## Cline 生成报告的规范

### 报告目录结构

每次回测生成一个新目录，命名格式：
```
bt_{YYYYMMDD}_{HHMMSS}_{开始日期}_{结束日期}_{品种}_{周期}
```

例：`bt_20260625_161139_2026-06-24_2026-06-25_ETHUSDT_5m`

目录下放三个文件（全部独立 HTML，不依赖外部资源）：

| 文件 | 用途 | 说明 |
|:---|:---|:---|
| `report.html` | 完整报告 | 包含所有图表和数据的综合页面 |
| `chart_kline.html` | K线+买卖点 | 价格走势图，标注买入卖出位置 |
| `chart_equity.png` | 收益曲线 | 账户净值曲线 |

### 生成报告时的注意事项

1. **自包含 HTML** — 所有 JS/CSS 内联或引用 CDN，不依赖本地文件
2. **手机自适应** — 增加 `<meta name="viewport">` 标签
3. **目录路径** — 所有文件放到 `reports/bt_xxx/` 下
4. **不要修改 report_server.py** — 它只做目录扫描和文件服务

### Python 首先生成图保存，如果已经报存好，生成html只需要读取png，加上echarts

当你生成报告 HTML 时，如果图表已经是 PNG 图片文件，HTML 中直接引用即可：

```python
# 1. 先用 matplotlib 生成图
import matplotlib.pyplot as plt
plt.plot(...)
plt.savefig(f'{report_dir}/chart_kline.png', dpi=150, bbox_inches='tight')

# 2. HTML 中引用图片
html = f'''
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0">
  <title>回测报告</title>
  <style>
    body {{ 
      font-family: -apple-system, sans-serif; 
      background: #0f0f1a; 
      color: #e0e0e0; 
      padding: 16px; 
      max-width: 800px; 
      margin: 0 auto; 
    }}
    img {{ max-width: 100%; height: auto; border-radius: 8px; }}
  </style>
</head>
<body>
  <h1>ETHUSDT 5m 回测报告</h1>
  <img src="chart_kline.png" alt="K线图">
  <img src="chart_equity.png" alt="收益曲线">
</body>
</html>
'''
with open(f'{report_dir}/report.html', 'w') as f:
    f.write(html)
```

## 已知问题 & 解决方案

### Cloudflare Tunnel URL 每次重启都会变

当前是临时隧道（`trycloudflare.com`），重启后会生成新的随机域名。

**解决：要么后续配置固定域名，要么每次重启后问我拿新 URL。**

### 隧道断了怎么办

查看 tmux 窗口状态：
```bash
tmux capture-pane -t claw:cf-tunnel -p | tail -5
```
如果断了，删掉旧窗口重建。

### 手机访问一定要用 Cloudflare 隧道吗

目前是的，因为服务器在云上，没有公网 IP。隧道提供 HTTPS 公网入口。

## 给 Cline 的提示词模板

每次让 Cline 跑回测并生成报告时，在任务最前面加上：

```
重要：
1. 报告目录统一放在 ~/HJATS/reports/ 下
2. 目录命名格式：bt_{YYYYMMDD}_{HHMMSS}_{品种}_{周期}
3. 生成 3 个文件：report.html（完整报告）、chart_kline.html 或者 .png（K线+买卖点）、chart_equity.png（收益曲线）
4. HTML 必须是自包含的，所有资源内联
5. 添加 meta viewport 标签支持手机浏览
6. 图片（如果有）用 matplotlib 保存为 PNG，HTML 中用 <img> 引用
7. 不要修改 ~/HJATS/reports/report_server.py
```

## 相关技能

- [vscodeserver-skill.md](https://github.com/jixianglz/JxSkill/blob/main/vscodeserver-skill.md)
