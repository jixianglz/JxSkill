# AI 协作规则（OpenClaw + Cline + HJATS）

## 项目

**HJATS** — 自动化交易系统（CTA 实盘引擎）
- OpenClaw: 实盘运行、监控、状态查询
- Cline: 代码开发、修复、重构
- 分工明确，不越界

## 核心规则

### 1. 不能擅自修改项目代码

涉及 HJATS 项目代码的**增、删、改**，必须先向用户说明变更内容，获得确认后才能执行。

**正确做法：**
- 发现问题 → 报告给用户 → 让 Cline 修
- 需要改进 → 提方案给用户确认 → 让 Cline 实现

**错误做法：**
- 发现 bug 直接改代码
- 看到可以优化的地方直接写

### 2. 实盘协作分工

```
OpenClaw（监控层）              Cline（开发层）
─────────────────              ──────────────
✓ 启动/停止实盘引擎             ✓ 开发新功能
✓ 周期性检查引擎状态            ✓ 修复 bug
✓ 报告异常                     ✓ 重构代码
✓ 查询报告/数据                ✓ 跑回测分析
✓ 只读操作（cat/log/status）    ✓ 写文件操作
```

### 3. 实盘运行流程

```
1. Cline 完成代码修改并 commit
2. OpenClaw 启动引擎: python3 run_live.py
3. OpenClaw 监控状态（周期性检查）
4. 发现异常 → 报告用户 → Cline 修
5. 修复后 OpenClaw 重启引擎
```

### 4. 操作前检查清单

```markdown
- [ ] 要向用户说明改什么？
- [ ] 是「读」还是「写」操作？
- [ ] 如果是写操作——获得用户确认了吗？
- [ ] 当前引擎是否在运行？（影响操作时机）
```

## 实盘常用命令

```bash
# OpenClaw 执行：
python3 scripts/hjats_agent.py status       # 查看状态
python3 scripts/hjats_agent.py start-live   # 后台启动
python3 scripts/hjats_agent.py stop-live    # 停止
python3 scripts/hjats_agent.py list-data    # 查看数据
python3 scripts/hjats_agent.py list-reports # 查看报告
cat /tmp/live_status.json | python3 -m json.tool  # 原始状态
```

## 原则

1. **读自由，写确认** — 任何写操作（创建/修改/删除文件）必须先报告
2. **有问题先报告，不擅自修** — OpenClaw 的职责是发现和报告，Cline 的职责是修复
3. **不确定就多问一句** — 宁可多问一次，不要改错一行
4. **规则持续更新** — 实际操作中发现的边界情况，应当补充进本文件

---

*本规则与 `vscodeserver-skill.md` 中的「项目代码操作规范」同源，独立成文以方便查阅。*
