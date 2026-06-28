# VS Code Server + Cline AI 插件使用指南

## 概述

通过 code-server（VS Code Server）在浏览器中运行 VS Code，配合 Cline AI 插件进行代码开发、脚本执行、数据分析等工作。

## 环境信息

- **code-server 版本**: 4.125.0
- **Cline 后端模型**: DeepSeek
- **访问方式**: 浏览器访问 code-server Web 界面
- **操作系统**: Linux (Ubuntu)

## 已知问题与解决方案

### 1. Cline 执行命令时卡住 / 页面无响应

**症状**: Cline 执行 shell 命令（如 `pip install`）时，界面卡住不动，需要刷新页面并点击"恢复任务"才能继续。

**根因**: 
- Cline 通过 **WebSocket** 连接到 code-server 的终端
- 浏览器与 code-server 之间的 WebSocket 连接不稳定（切后台、网络抖动都会断）
- WebSocket 断开后，命令实际仍在服务器上运行，但 Cline 收不到输出，表现为"卡住"
- 刷新页面 = 重连 WebSocket，Cline 重新获取终端状态

**注意**: Cline 的 `timeout` 设置（`cline.timeout`）解决的是 **AI 生成超时** 问题，不是 WebSocket 断连问题。

### 2. pip install 卡住

**症状**: Cline 执行 `pip install xxx` 长时间无响应。

**原因**: pip install 下载+编译耗时较长（尤其是 matplotlib、numpy、pandas 等），超过 WebSocket 保持时间。

### 3. 长时间命令（脚本执行、模型训练等）卡住

与 pip install 类似，任何耗时较长的 shell 命令都可能触发 WebSocket 断连。

## 关键配置

### code-server 配置文件

路径: `~/.config/code-server/config.yaml`

```yaml
bind-addr: 127.0.0.1:8080
auth: password
password: <your-password>
cert: false
```

### Cline 设置 (VS Code settings.json)

路径: `~/.local/share/code-server/User/settings.json`

```json
{
  "cline.apiProvider": "deepseek",
  "cline.deepseekApiKey": "sk-xxx",
  "cline.timeout": 600,
  "comments.openView": "never",
  "keyboard.layout": "de"
}
```

| 设置项 | 说明 | 建议值 |
|:---|:---|:---:|
| `cline.timeout` | AI 生成/工具调用的超时时间（秒） | 600（10分钟） |
| `cline.apiProvider` | AI 后端 | deepseek |
| `cline.deepseekApiKey` | API 密钥 | 按需填写 |

## 最佳实践：让 Cline 可靠执行命令

### 核心原则

Cline 执行命令时，WebSocket 仅需在**发送命令的瞬间**保持连接。命令本身应在**后台运行**，Cline 通过**读取日志文件**来检查结果。

### 命令执行模式（推荐）

所有耗时命令应使用以下模式：

```bash
# ❌ 不要这样（会占住 WebSocket，容易断）
pip install matplotlib numpy pandas

# ✅ 应该这样（瞬间返回，后台跑，写日志） 
nohup pip install matplotlib numpy pandas > /tmp/pip_install.log 2>&1 &
```

然后 AI 轮询读取日志文件：

```python
import time
# 检测安装是否完成
with open('/tmp/pip_install.log', 'r') as f:
    content = f.read()
    if 'Successfully installed' in content:
        print('安装完成！')
```

### 给 Cline 的任务提示模板

在每次任务的**最前面**加上这段指令：

```
重要规则：
1. 所有 shell 命令必须使用以下方式执行：
   nohup <command> > /tmp/xxx.log 2>&1 &
2. 不要阻塞等待命令完成
3. 通过读日志文件（/tmp/xxx.log）检查命令进度和结果
4. 写文件、读文件等操作可以直接做，不需要特殊处理
```

### 需要预先安装的常用包

手动在终端装一次，之后 Cline 直接 import，不用再 pip install：

```bash
pip install matplotlib numpy pandas plotly
```

## 开发工作流

### 推荐流程

1. **手动预装依赖**（终端操作，一次搞定）
2. **给 Cline 的任务**包含后台命令执行指令
3. **Cline 执行期间**不要切浏览器页面
4. **如果卡了**：刷新页面 → 恢复任务（命令实际还在跑）
5. **完成后**：手动审核生成的代码和文件

### 文件组织建议

```
项目根目录/
├── backtest_reports/       # 回测报告（独立目录，方便管理）
│   ├── 2026-06-25/
│   │   ├── report.csv
│   │   ├── chart.png
│   │   └── report.html
│   └── ...
├── scripts/                # 脚本文件
└── data/                   # 数据文件
```

## 项目代码操作规范（重要）

### OpenClaw 操作项目代码的规则

> 本规则适用于任何 AI 工具（OpenClaw、Cline 等）操作项目代码时的行为约束。

1. **修改文件前必须先确认** — 涉及项目代码的增、删、改，必须先向用户说明变更内容，获得确认后才能执行
2. **添加文件不能删除原有代码** — 更新文件时应在原有内容基础上添加，不可覆盖/替换整个文件
   - 如果确实需要大范围修改，应先 diff 展示变更内容，获得确认
3. **项目入口/上下文文件（如 `.context/CONTEXT_CURSOR.md`）** 作为 AI 快速理解项目的入口，更新时应保持原有结构和内容，仅新增或更新对应章节
4. **Git 跟踪的文件** 修改前应对比 diff，确保不会丢失 git 历史中的原有内容
5. **不确定时多问** — 对文件操作范围不确定时，宁可先问再改，不要先改后说

### 操作前检查清单

```markdown
- [ ] 是否已向用户说明要改什么？
- [ ] 修改是「添加」还是「覆盖」？
- [ ] 查看过原始文件内容？
- [ ] 有 git diff 确认变更范围？
- [ ] 用户已明确确认？
```

## 故障排查速查

| 现象 | 原因 | 解决 |
|:---|:---|:---|
| Cline 卡住不动 | WebSocket 断连 | 刷新页面 → 恢复任务 |
| AI 回答到一半停了 | `cline.timeout` 太短 | 改大 timeout（建议 600） |
| pip install 没反应 | 同上 | 用 nohup 后台装 + 读日志 |
| 每次执行命令都卡 | 每个命令都占 WebSocket | 后台执行 + 日志轮询 |
| 页面白屏/崩溃 | code-server 进程挂了 | 重启 code-server 服务 |

## 相关资源

- [code-server GitHub](https://github.com/coder/code-server)
- [Cline GitHub](https://github.com/cline/cline)
- [JxSkill 仓库](https://github.com/jixianglz/JxSkill)（本 skill 存放处）
- [HJATS 仓库](https://github.com/jixianglz/HJATS)（回测系统）
