# OS Project - Pi Side (Raspberry Pi)

## Architecture

Three-node system coordinated by OpenClaw:

- **OpenClaw** (remote orchestrator): Coordinates tasks, connects to both Pi and WSL via frp tunnels
- **This Pi**: EdgeTPU data collection, usbmon measurement, inference benchmarking
- **PC** (WSL Ubuntu): Data analysis, modeling, visualization
- All nodes can be used directly by the user

```
         OpenClaw (orchestrator)
              │
      frp server [RELAY]:7000
          ┌───┴───┐
     :Pi  │       │:WSL
          │       │
   This Pi       PC WSL
          │       │
          └──LAN──┘  (shared folder)
```

## Shared Folder

Pi、WSL、Windows 三端挂载同一个共享目录，数据双向流通。

## Key Paths

- Project workspace: `~/Desktop/[PROJECT]/`
- Shared data folder: `[SHARED_MOUNT]/`

## Pi 系统信息

- sudo 免密或用 `echo '[SUDO_PASS]' | sudo -S` 模式

## Codex 协作

- 完成一个任务/步骤/动作后，或遇到瓶颈需要头脑风暴时，必须调用 Codex CLI 商量/审核
- 调用方式: `codex exec --full-auto -p "你的问题"`（不指定 -m，使用 config 默认模型）
  - `--full-auto`: 给予完整权限（可执行命令、读写文件）
  - config（~/.codex/config.toml）设置了默认模型和 `model_reasoning_effort = "high"`,以及用--continue保持会话连续性
  - **始终使用最新可用模型**：定期检查 `codex` 支持的最新模型并更新 config
- Codex 的角色：**只做检查/思考/质疑/鼓励/提供新思路**，不让它实现任何东西
- 把当前上下文和问题简要描述给 Codex，让它从旁观者角度审视

## Git 规范

- 完成一个阶段性工作后主动 git commit
- commit message 用简洁的中文或英文描述改动
- 不要积攒大量改动再一次性 commit
- 优先推送到云端远程仓库; 如果远程不可用则先本地 commit
- commit 不要带 Co-Authored-By 署名

## 实验结果共享

- 实验产出（csv、统计数据、分析结论等）都放到共享文件夹下，方便 PC 端同步查看
- 在共享目录 `results/` 下按实验名分目录，结果文件命名清晰
- 临时中间文件可以放本地，最终结果必须同步到共享目录

## 记忆与进度

- 工作中学到的经验、发现、进度等及时写入 memory 目录下的专题 md 文件
- 不要占用 MEMORY.md 的空间，新建或追加到对应主题的 md 文件中
- MEMORY.md 只放索引和最核心的总结

## 任务完成确认
- 完成任务后，主动做一次自检：
  - 代码能正常运行吗？
  - 结果符合预期吗？
  - 有没有遗漏的步骤？
- 自检通过后再回复完成
- 每个阶段性工作完成后主动 git commit
- commit message 用简洁的中英文描述改动
- 不要积攒大量改动再一次性 commit


# CLAUDE.md — 行为规则

## 文件操作
- **修改文件必须用 Edit 工具**，不许用 bash sed/awk/python 写文件
- **新建文件用 Write 工具**，不许用 bash echo/cat heredoc
- bash 只用于系统命令和无法用内置工具完成的操作
