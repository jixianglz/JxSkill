# JxSkill — AI 技能库

> 记录 AI（OpenClaw / Cline / 小龙虾）在 HJATS 自动化交易系统中的操作技能和协作规范。
> 重装后直接 `git clone`，按手册干活。

## 目录

| 文件 | 说明 |
|:---|:---|
| `ai-cooperation-rules.md` | OpenClaw + Cline 协作规范与分工约定 |
| `hjats-live-engine-skill.md` | OpenClaw 实盘操作手册（启动/监控/问题处理） |
| `hjats-report-server-skill.md` | HJATS 报告服务器 & 移动端查看指南 |
| `vscodeserver-skill.md` | VS Code Server + Cline 插件使用指南 |
| `cline-history/` | 历史对话记录（按日期归档） |

## 快速上手（小龙虾重装后）

```bash
# 1. 克隆技能库
git clone git@github.com:jixianglz/JxSkill.git ~/JxSkill

# 2. 读完这些文件就知道怎么干了
cat ~/JxSkill/hjats-live-engine-skill.md      # OpenClaw 实盘操作
cat ~/JxSkill/ai-cooperation-rules.md          # 协作规则
cat ~/JxSkill/hjats-report-server-skill.md     # 报告服务
```

## 项目背景

**HJATS** — 自动化 CTA 交易系统，回测+实盘一体。
- **Cline**: 写代码、修 bug、跑回测分析
- **OpenClaw（小龙虾）**: 跑实盘引擎、监控状态、报告异常
- 分工明确，不越界

## 仓库结构

```
JxSkill/
├── README.md                           ← 本文件
├── ai-cooperation-rules.md             ← AI协作规则（核心）
├── hjats-live-engine-skill.md          ← OpenClaw 实盘操作手册
├── hjats-report-server-skill.md        ← 报告服务器指南
├── vscodeserver-skill.md               ← VS Code Server + Cline 指南
└── cline-history/                      ← 历史对话记录
    ├── 2026-06-27_04-43-55_试下响应.md
    ├── 2026-06-27_09-24-46_把文件先提交一下.md
    ├── 2026-06-27_09-37-07_hjas自动化开发先看下当前文档进展中间对话被op.md
    ├── 2026-06-27_10-03-08_我正在进行一个自动化交易系统的开发.md
    └── 2026-06-28_17-07-00_配置外部化_风控增强.md
    └── 2026-06-29_OpenClaw-实盘协作记录.md        ← 本次OpenClaw协作记录
```
