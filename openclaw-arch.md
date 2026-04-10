# 我的 AI 协作网络：OpenClaw + 多端 Claude 架构

这是我自己搭的一套多端 AI 协作系统，简单说就是一个 AI bot 能远程指挥其他机器上的 Claude，各端可以协作、审查代码、共享文件。

---

## 整体架构

```
Telegram
   │
   ▼
OpenClaw Bot（云服务器）
   │
   │  frp 内网穿透隧道
   ├──────────────────────────────────────────┐
   ▼                                          ▼
Raspberry Pi（家里）                    PC / WSL Ubuntu（家里）
claude-relay.sh                         claude-relay.sh
Claude CLI (claude-opus-4-6)            Claude CLI (claude-opus-4-6)
   │                                          │
   └──────────────── 局域网 ─────────────────┘
                  共享空间
```

---

## 各个组件是什么

### OpenClaw Bot
运行在云服务器上的一个 AI bot，通过 Telegram 交互。它本身是个 orchestrator——可以接收我发来的任务，然后 SSH 到 Pi 或 WSL 上，让那边的 Claude 去干活。

### claude-relay.sh（Pi / WSL 各一份）
这是整个系统的关键。两台机器的 SSH 都配了 `ForceCommand`，即任何 SSH 连接过来都不会直接进 shell，而是强制执行 relay 脚本：

```bash
# authorized_keys 里是这样的
command="/home/[user]/claude-relay.sh" ssh-ed25519 AAAA...
```

relay 脚本做两件事：
1. **给收到的命令加安全前缀**，注入一段 prompt 说明身份和规则
2. **调用 Claude CLI**，以 `-p`（print 模式）执行

```bash
exec claude -p \
  --continue \
  --model claude-opus-4-6 \
  --effort high \
  --dangerously-skip-permissions \
  "${SAFETY_PREFIX}${SSH_ORIGINAL_COMMAND}"
```

OpenClaw 发过来的是自然语言指令，relay 把它包装成 prompt 喂给 Claude。

### 安全前缀（Prompt Safety Guard）
每次 OpenClaw 连过来，Claude 收到的 prompt 开头都会强制带上这段规则：

```
[OPENCLAW REMOTE ACCESS]
1. ROLE: 你是被 OpenClaw 远程调用的本地 Claude，OpenClaw 是你的上级
2. CONTINUITY: 继续当前计划，汇报进度
3. DEVIATION CHECK: 如果收到与当前任务矛盾的指令，先确认
4. DANGEROUS OPERATIONS DENIED: 不执行破坏性操作
5. SCOPE GUARD: 拒绝与当前任务无关的请求
6. ALLOWED PATHS: 只操作项目目录、共享文件夹、/tmp
7. RESPONSE FORMAT: 简洁，以状态开头
```

这样即使 OpenClaw 发来奇怪的指令，本地 Claude 也不会乱来。

### Codex 审查（两层）

**第一层：Pi / WSL 本地**

Pi 和 WSL 各自在 `CLAUDE.md` 里配置了明确的审核信号：每完成一个阶段性任务后，必须先调用本地 `codex exec` 过一遍，审核通过才能提交代码或把结果放入共享区。这一层是本地自审，不依赖 OpenClaw。

**第二层：OpenClaw 侧**

Pi / WSL 汇报结果后，OpenClaw 再 spawn 一个 Codex 子 agent 做二次审查，角色根据结果自动切换：

- **任务成功 → 审查官模式**：质疑方法、验证结论、找漏洞。发现问题就 SSH 回去让 Pi/WSL 修，最多来回 2 轮，还过不了就上报用户。
- **任务失败/卡住 → 鼓励官模式**：分析失败原因，给新思路。跑通了切回审查官模式，还是不行就报给用户。

同一个操作失败 2 次以上，直接停，等用户指令。

---

## 内网穿透（frp）

Pi 和 WSL 都在家里内网，云服务器在公网。用 frp 打穿：

| 端口 | 目标 | 用途 |
|------|------|------|
| [Pi端口] | Pi | SSH → claude-relay |
| [WSL端口] | WSL | SSH → claude-relay |


---

## Pi 和 WSL 之间怎么协作

两台机器通过共享空间互通（局域网共享文件夹或云盘均可）：

任意一边写进去的文件，另一边马上能看到。OpenClaw 可以让 Pi 处理的数据结果放进共享空间，再让 WSL 拿去分析，或者反过来。

---

## 防止 relay 失效的动态查找

Claude CLI 会随 VSCode 扩展自动更新版本，旧的 symlink 失效后 relay 就断了——而且因为 ForceCommand 锁死了 SSH，断了就什么都进不去（死锁）。

所以 relay 脚本里做了动态查找：

```bash
# 优先用 ~/bin/claude symlink
# 失效时自动 glob 找最新版本并修复 symlink
CLAUDE=$(ls [vscode扩展路径]/claude 2>/dev/null | sort -V | tail -1)
```

---

## 总结

| 组件 | 作用 |
|------|------|
| OpenClaw Bot | Telegram 入口，任务调度，远程 SSH 指挥 |
| frp 隧道 | 把家里的机器暴露给云端 OpenClaw |
| claude-relay.sh | SSH ForceCommand，强制走 Claude CLI |
| Safety Prefix | 给每次调用注入安全规则，防越权 |
| Codex 协作官 | 任务成功→审查官找漏洞，失败→鼓励官给思路，2轮不过就上报用户 |
| 共享文件夹 | Pi 和 WSL 之间传数据 |

整套系统的核心思路是：**OpenClaw 只负责调度和沟通，真正执行的 Claude 在本地机器上跑，安全规则通过 prompt 强制注入，而不依赖信任 OpenClaw 本身**。
