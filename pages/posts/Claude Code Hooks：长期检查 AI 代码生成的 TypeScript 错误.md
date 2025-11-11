---
title: Claude Code Hooks：长期检查 AI 代码生成的 TypeScript 错误
date: 2025-11-11T10:55:00.000+00:00
lang: zh
duration: 12min
---

[[toc]]

### 问题背景

日常用 Claude Code"写/改文件"很爽，但落地到 TS 项目常见两个坑：
1）生成阶段没红线，过一会儿才冒出类型错误；2）不少错误不触发就没人管。
本文给出一个**基于 Hooks 的自动化 TSC 校验方案**：每次 `Write/Edit/MultiEdit` 后立即跑 `tsc --noEmit`，有错就**当场拦截**、可见可修，显著提升可用性与稳定性。

---

## 一、核心思路（一句话）

把“**约束**”写进 Hook，而不是写进提示词。**PostToolUse** 事件在 Claude Code 执行工具（写文件、编辑等）之后触发，我们在这里挂接脚本做 TSC 体检，有错直接失败返回。
官方 Hook 机制、事件点与配置文件位置见文档。([code.claude.com](https://code.claude.com/docs/en/hooks?utm_source=chatgpt.com 'Hooks reference - Claude Code Docs'))

---

## 二、完整实现步骤

#### 1. 放置 Hook 脚本（建议项目内可版本化）

- 路径：`<your-project>/.claude/hooks/tsc-check.sh`

- 记得加执行权限：`chmod +x .claude/hooks/tsc-check.sh`

**示例脚本（可直接用）：**

```bash
#!/usr/bin/env bash
set -euo pipefail

# 读取 Hook 输入（JSON）—— Claude Code 以 stdin 传入
HOOK_INPUT="$(cat)"

# 解析出工具名与目标文件（可选）
TOOL_NAME="$(echo "$HOOK_INPUT" | jq -r '.tool_name // .event // ""')"
# MultiEdit/Write/Edit 的输入结构不同，这里做兜底提取所有 .ts/.tsx
TARGETS="$(echo "$HOOK_INPUT" | jq -r '..|.file_path? // empty' | grep -E '\.tsx?$' || true)"

# 从任意一个目标文件向上寻找 tsconfig.json
find_project_root() {
  local start="$1"
  local dir; dir="$(dirname "$start")"
  while [[ "$dir" != "/" && -n "$dir" ]]; do
    if [[ -f "$dir/tsconfig.json" ]]; then
      echo "$dir"; return 0
    fi
    dir="$(dirname "$dir")"
  done
  return 1
}

# 选择最合适的 tsc 命令（支持 monorepo / references）
choose_tsc_cmd() {
  local root="$1"
  if [[ -f "$root/tsconfig.app.json" ]]; then
    echo "npx tsc --project tsconfig.app.json --noEmit"
  elif [[ -f "$root/tsconfig.build.json" ]]; then
    echo "npx tsc --project tsconfig.build.json --noEmit"
  elif grep -q '"references"' "$root/tsconfig.json" 2>/dev/null; then
    if [[ -f "$root/tsconfig.src.json" ]]; then
      echo "npx tsc --project tsconfig.src.json --noEmit"
    else
      echo "npx tsc --build --noEmit"
    fi
  else
    echo "npx tsc --noEmit"
  fi
}

echo "⚡ Running TypeScript checks..." >&2

# 收集需要检查的项目根（去重）
mapfile -t roots < <(for f in $TARGETS; do find_project_root "$f" || true; done | sort -u)

err_count=0
summary=""
for root in "${roots[@]}"; do
  [[ -z "$root" ]] && continue
  pushd "$root" >/dev/null
  cmd="$(choose_tsc_cmd "$root")"
  echo "  ▶ $(basename "$root"): $cmd" >&2
  if ! out="$($cmd 2>&1)"; then
    echo "  ❌ Errors in $(basename "$root")" >&2
    summary+=$'\n\n'"=== $(basename "$root") ==="$'\n'"$out"
    ((err_count++))
  else
    # 再保险：扫一下输出里是否出现 error TS
    if echo "$out" | grep -q "error TS"; then
      echo "  ❌ Errors in $(basename "$root")" >&2
      summary+=$'\n\n'"=== $(basename "$root") ==="$'\n'"$out"
      ((err_count++))
    else
      echo "  ✅ OK" >&2
    fi
  fi
  popd >/dev/null
done

if (( err_count > 0 )); then
  echo "🚨 TypeScript errors detected ($err_count project(s))" >&2
  # 非零退出：让 Claude Code 把这次操作标为失败，从而显式暴露问题
  echo "$summary" >&2
  exit 1
fi

exit 0
```

> ✅ 优点：落地即用；Monorepo/References 友好；错误即刻可见
> ❌ 缺点：每次编辑都跑 TSC，超大仓库需注意耗时（可通过 matcher 过滤、目录白名单优化）

> 说明：Hook 输入通过 **stdin JSON** 传递，Claude Code 官方明确了 Hook 的输入输出、并在 PostToolUse 事件下支持基于匹配器（`matcher`）筛选工具名（支持正则/通配），非常适合拦截 `Write|Edit` 场景。([code.claude.com](https://code.claude.com/docs/en/hooks 'Hooks reference - Claude Code Docs'))

---

#### 2. 在 settings.json 注册 Hook（PostToolUse）

放**项目级**（推荐团队共用）或**用户级**（全局生效），文件位置与优先级见官方说明。([code.claude.com](https://code.claude.com/docs/en/settings 'Claude Code settings - Claude Code Docs'))

**项目级 `.claude/settings.json` 示例：**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/tsc-check.sh",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

要点：

- **事件名**：`PostToolUse`（执行工具之后）。
- **matcher**：`Write|Edit|MultiEdit` 正则同时覆盖常见写入/编辑工具。Hook 匹配规则、正则与 `*`/空匹配等用法详见文档。

- **$CLAUDE_PROJECT_DIR**：官方提供的项目根环境变量，方便在任何目录下调用项目内脚本。([code.claude.com](https://code.claude.com/docs/en/hooks 'Hooks reference - Claude Code Docs'))

---

#### 3. 验证与调试

- 运行 `/hooks` 查看是否已注册；脚本需可执行。

- 开启调试：用 `claude --debug` 观察 Hook 执行细节与输出。

- JSON 结构错、权限不够、超时等都是高频问题，官方给了排查清单。([code.claude.com](https://code.claude.com/docs/en/hooks 'Hooks reference - Claude Code Docs'))

---

## 三、为什么选 PostToolUse？

- **目标明确**：只在“代码已改动”之后跑类型检查，避免噪声。所以在编写 skills 的时候也需要注意， claude code 有哪些 hooks 可以被直接使用。

- **失败可见**：非零退出码会把这次操作标记为失败，问题当场暴露。

- **按照官方配置来**：事件、输入/输出协议、超时/并行/去重策略都有明确规范。([code.claude.com](https://code.claude.com/docs/en/hooks 'Hooks reference - Claude Code Docs'))

> 在当前场景下也有部分可以被替代的内容， 可选替代：
>
> - **PreToolUse**（执行前拦截）：适合做权限/命令白名单、审批流，不适合 TSC。
> - **UserPromptSubmit**（用户提交前）：适合提示校验/注入上下文，不做类型检查。([code.claude.com](https://code.claude.com/docs/en/hooks 'Hooks reference - Claude Code Docs'))

---

## 四、进阶增强（按需选择）

#### 1. 只检查受影响包（Monorepo 优化）

- 结合 `git diff --name-only HEAD` + 路径到包的映射，筛选最小集合再跑 TSC。

- 也可在 Hook 中读 `pnpm -w -F`/`nx affected` 来跑增量。

#### 2. 与 Lint/Format 联动

- 在同一个 `PostToolUse` 下追加 Prettier/ESLint 命令。官方支持并行执行，互不阻塞。([code.claude.com](https://code.claude.com/docs/en/hooks 'Hooks reference - Claude Code Docs'))

---

## 五、常见问题

- **超时/卡顿？**
  给 `timeout` 合理值；大型仓库做“受影响包”筛选；必要时把 TSC 调成 `--build --noEmit` 并按需分项目并行。

- **Web 环境与本地差异？**
  可根据 `CLAUDE_CODE_REMOTE` 环境变量做分支逻辑。([code.claude.com](https://code.claude.com/docs/en/hooks 'Hooks reference - Claude Code Docs'))

---

## 六、方案对比

| 方案                        | 适用场景           | 优点                                     | 缺点             |
| --------------------------- | ------------------ | ---------------------------------------- | ---------------- |
| PostToolUse + TSC（本文）   | 文件实际改动后兜底 | 错误当场可见、失败即中断、最贴近真实落地 | 大仓库需控时     |
| PreToolUse + 规则校验       | 事前网关           | 可做审批/白名单，阻断危险命令            | 不检查类型本身   |
| UserPromptSubmit + 提示校验 | 输入前治理         | 统一风格、注入上下文                     | 距离代码落地较远 |

---

## 七、实践建议

- **把 Hook 放项目级**：团队共享、可评审、可追溯。

- **先“窄后宽”**：从 `Write|Edit` 小范围试点，再放开到 `MultiEdit`。

- **增量优化**：大仓库用“受影响包”策略；固定 TSC 工具链版本；必要时启用缓存。

- **权限先行**：`permissions.deny` 先写好，避免读到敏感文件。([code.claude.com](https://code.claude.com/docs/en/settings 'Claude Code settings - Claude Code Docs'))

---

## 参考与延伸

- **Hooks 入门指南 / 参考文档（官方）**：事件点、输入输出、匹配器、并行/超时、调试与安全建议。([code.claude.com](https://code.claude.com/docs/en/hooks-guide?utm_source=chatgpt.com 'Get started with Claude Code hooks - Claude Code Docs'))

- **Settings 配置（官方）**：多级 `settings.json`、权限系统、插件与 Hook 配置入口。([code.claude.com](https://code.claude.com/docs/en/settings 'Claude Code settings - Claude Code Docs'))
