---
name: "engramory-init"
description: "在项目中初始化 Engramory 记忆协议。当用户要求安装 Engramory、初始化记忆系统、或配置 Agent 记忆时触发。"
---

# Engramory 初始化

在当前项目中初始化 Engramory 记忆协议。采用全局 + 项目双层架构，全局规则和 Hook 已预装在 `~/.trae-cn/`，本 Skill 只需为项目创建记忆目录。

## 架构说明

Engramory 采用**全局 + 项目**双层架构：

| 层级 | 位置 | 内容 | 作用域 |
|------|------|------|--------|
| 全局 | `~/.trae-cn/user_rules/rule-engramory-memory.md` | 记忆规则（始终加载） | 所有项目自动生效 |
| 全局 | `~/.trae-cn/hooks.json` | PreToolUse Hook 配置 | 所有项目自动生效 |
| 全局 | `~/.trae-cn/hooks/engramory_index_guard.js` | 索引大小守卫脚本 | 所有项目共享 |
| 项目 | `.engramory-memory/MEMORY.md` | 记忆索引 + 详细笔记 | 仅当前项目 |

## 触发条件

用户要求安装 Engramory、初始化记忆系统、或在项目中配置 Agent 记忆时触发。

## 初始化步骤

按以下顺序执行，每一步必须成功后才执行下一步：

### 1. 确认环境

- 确认 `node --version` 可执行（Hook 脚本依赖 Node.js）
- 确认全局 Engramory 已安装：检查以下文件是否存在：
  - `~/.trae-cn/user_rules/rule-engramory-memory.md`
  - `~/.trae-cn/hooks.json`
  - `~/.trae-cn/hooks/engramory_index_guard.js`

如果全局文件缺失，先执行"全局安装"步骤（见下方），再继续。

### 2. 创建项目记忆目录

在项目根目录下创建 `.engramory-memory/` 目录，并在其中创建 `MEMORY.md` 索引文件：

```markdown
# Memory Index

> Pointers only — the actual content lives in the linked files, never here.
> Soft cap 150 lines / 20 KB (warn). Hard cap 200 lines / 25 KB (compact first).
> One line per memory: `- [<title>](<slug>.md) — <one-line hook>`
> Plain-text & unencrypted: never store secret *values* here (only where they
> live).

## user
<!-- who the user is: identity, role, durable preferences -->

## feedback
<!-- how the agent should behave (procedural). Each file carries Why: / How to apply: -->

## project
<!-- current work state not in code/git. Absolute dates. Why: / How to apply: -->

## reference
<!-- pointers to external resources: URLs, dashboards, tickets, log paths -->
```

### 3. 验证安装

逐一检查：

- `.engramory-memory/MEMORY.md` 存在且内容正确
- 全局规则文件存在
- 全局 Hook 脚本存在
- 全局 `hooks.json` 包含 `engramory_index_guard` 命令

验证 Hook 可执行：

```bash
echo '{"tool_name":"Write","tool_input":{"file_path":"<项目路径>/.engramory-memory/MEMORY.md","content":"test"}}' | node "~/.trae-cn/hooks/engramory_index_guard.js"
```

应静默退出（exit 0，无输出）。

### 4. 输出安装结果

向用户汇报安装完成的文件列表，并提示：

- 重启 Trae 后 Hook 才会生效
- 如项目中有其他名为 `MEMORY.md` 的文件，需设置环境变量 `ENGRAMORY_INDEX_PATH` 指向真正的记忆索引路径

---

## 全局安装（仅在缺失时执行）

如果全局 Engramory 文件缺失，按以下步骤补装：

### G1. 创建全局规则

创建 `~/.trae-cn/user_rules/rule-engramory-memory.md`，内容如下：

```markdown
# Memory (Engramory)

你拥有一个基于文件的记忆系统，位于 `.engramory-memory/` 目录（索引文件：`MEMORY.md`）。

## 启动任务时

读取 `MEMORY.md`（每行一条记忆），仅打开看起来相关的详细文件。将记忆视为可能过时的背景信息——在基于记忆行动之前，验证任何文件/标志/版本。

## 学习到值得保留的信息时

1. 确认该信息**不在**仓库 / git / 项目规则中已经存在（不重复事实来源）
2. 确认该信息**不是**密钥凭据值（密钥、令牌、密码等只记录存放位置，不记录值本身）
3. 搜索索引，**更新已有笔记**而非重复创建
4. 否则创建一个原子 markdown 文件（一个事实），包含 frontmatter：
   - `name` / `description`（精炼的一行摘要）/ `type`（`user | feedback | project | reference`）/ `created` + `updated`（`YYYY-MM-DD`）
   - `feedback` 或 `project` 类型笔记必须在正文中包含 **`Why:`** 和 **`How to apply:`** 行
5. 在 `MEMORY.md` 中添加一行指针
6. **删除**被证实错误的记忆

## 保持索引精简

- `MEMORY.md` 软上限 150 行 / 20 KB（警告），硬上限 200 行 / 25 KB（必须压缩）
- 超限时：压缩长行、合并重复、归档冷门笔记

## 完整协议

详见 `engramory/SKILL.md`
```

### G2. 创建全局 Hook 脚本

将 `engramory_index_guard.js` 复制到 `~/.trae-cn/hooks/` 目录。

如果源文件不可用，参考下方内容创建最小版本的 Node.js 守卫脚本。

```javascript
#!/usr/bin/env node
/**
 * Engramory 索引大小守卫 — Trae PreToolUse Hook
 *
 * 阻止记忆索引 (MEMORY.md) 超出宿主实际加载的窗口大小。
 * 通过环境变量配置（均可选）：
 *   ENGRAMORY_HARD          硬行上限，默认 200
 *   ENGRAMORY_WARN          软行警告，默认 150
 *   ENGRAMORY_HARD_BYTES    硬字节上限，默认 25600 (25 KB)
 *   ENGRAMORY_WARN_BYTES    软字节警告，默认 20480 (20 KB)
 *   ENGRAMORY_INDEX_NAME    索引文件名，默认 "MEMORY.md"
 *   ENGRAMORY_INDEX_PATH    索引的绝对路径（覆盖名称匹配）
 */
const fs = require("fs");
const path = require("path");

function envInt(name, defaultVal) {
  const raw = process.env[name];
  if (raw == null) return defaultVal;
  const val = parseInt(raw.trim(), 10);
  return Number.isFinite(val) && val > 0 ? val : defaultVal;
}

function lineCount(text) {
  if (!text) return 0;
  return text.endsWith("\n") ? text.split("\n").length - 1 : text.split("\n").length;
}

function byteSize(text) {
  return Buffer.byteLength(text, "utf-8");
}

function fmtKB(n) {
  return n < 1024 ? `${n} B` : `${(n / 1024).toFixed(1)} KB`;
}

function emit(decision, reason, context) {
  const hso = { hookEventName: "PreToolUse" };
  if (decision != null) hso.permissionDecision = decision;
  if (reason != null) hso.permissionDecisionReason = reason;
  if (context != null) hso.additionalContext = context;
  process.stdout.write(JSON.stringify({ hookSpecificOutput: hso }) + "\n");
  process.exit(0);
}

function applyEdits(current, edits) {
  let result = current;
  for (const e of edits) {
    const old = e.old_string || "";
    const replacement = e.new_string || "";
    if (!old) continue;
    if (e.replace_all) {
      result = result.split(old).join(replacement);
    } else if (result.indexOf(old) !== -1 && result.indexOf(old) === result.lastIndexOf(old)) {
      result = result.replace(old, replacement);
    }
  }
  return result;
}

function main() {
  let inputData = "";
  process.stdin.setEncoding("utf-8");
  process.stdin.on("data", (chunk) => { inputData += chunk; });
  process.stdin.on("end", () => {
    try {
      const data = JSON.parse(inputData);
      const tool = data.tool_name || "";
      const ti = data.tool_input || {};
      const filePath = ti.file_path || "";
      if (!filePath) { process.exit(0); return; }

      const indexName = process.env.ENGRAMORY_INDEX_NAME || "MEMORY.md";
      const indexPath = process.env.ENGRAMORY_INDEX_PATH || "";

      let isTarget = false;
      if (indexPath) {
        try {
          isTarget = fs.realpathSync(filePath).toLowerCase() === fs.realpathSync(indexPath).toLowerCase();
        } catch {
          isTarget = path.resolve(filePath).toLowerCase() === path.resolve(indexPath).toLowerCase();
        }
      } else {
        isTarget = path.basename(filePath).toLowerCase() === indexName.toLowerCase();
      }

      if (!isTarget) { process.exit(0); return; }

      const hard = envInt("ENGRAMORY_HARD", 200);
      const warn = envInt("ENGRAMORY_WARN", 150);
      const hardB = envInt("ENGRAMORY_HARD_BYTES", 25600);
      const warnB = envInt("ENGRAMORY_WARN_BYTES", 20480);

      let current = "";
      try { current = fs.readFileSync(filePath, "utf-8"); } catch { current = ""; }

      const curLines = lineCount(current);
      const curBytes = byteSize(current);

      let result = "";
      if (tool === "Write") {
        result = ti.content != null ? ti.content : (ti.file_text || "");
      } else if (tool === "Edit") {
        result = applyEdits(current, [ti]);
      } else if (tool === "MultiEdit") {
        result = applyEdits(current, ti.edits || []);
      } else {
        process.exit(0); return;
      }

      const pLines = lineCount(result);
      const pBytes = byteSize(result);

      const overHard = pLines > hard || pBytes > hardB;
      const overWarn = pLines > warn || pBytes > warnB;
      const worsens = (pLines > hard && pLines > curLines) || (pBytes > hardB && pBytes > curBytes);

      if (worsens) {
        emit("deny", `Engramory: 编辑会使索引增长至 ${pLines} 行 / ${fmtKB(pBytes)}，超出上限 (${hard} 行 / ${fmtKB(hardB)})。请先执行压缩流程：指针化长行、合并重复、归档冷门笔记。`);
      } else if (overHard) {
        emit(null, null, `Engramory: 索引仍超出上限，但本次编辑在缩小，允许继续。请持续压缩至 ${hard} 行 / ${fmtKB(hardB)} 以下。`);
      } else if (overWarn) {
        emit(null, null, `Engramory: 索引接近上限 (${pLines} 行 / ${fmtKB(pBytes)})，建议执行压缩流程。`);
      } else {
        process.exit(0);
      }
    } catch {
      process.exit(0);
    }
  });
}

main();
```

### G3. 创建全局 hooks.json

创建 `~/.trae-cn/hooks.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "node \"C:/Users/LKJ/.trae-cn/hooks/engramory_index_guard.js\""
          }
        ]
      }
    ]
  }
}
```

**关键**：将路径替换为实际绝对路径，使用正斜杠。

---

## 记忆写入条件

Engramory **不是每次对话都写记忆**，只在同时满足以下条件时才生成文件：

| 条件 | 说明 |
|------|------|
| **信息是 durable 的** | 未来会话也用得上，而非当前任务的临时上下文 |
| **不在已有来源中** | 仓库代码、git、项目规则、CLAUDE.md 中未记录 |
| **不是秘密值** | 密钥、令牌、密码只记录存放位置，不记录值本身 |
| **索引中无重复** | 搜索 `MEMORY.md` 确认没有已有笔记覆盖 |

## 记忆协议速查

读记忆：启动任务时读 `MEMORY.md` → 按相关性打开详细文件 → 视为可能过时的背景信息

写记忆：学习到 durable 信息 → 检查负面范围 → 去重 → 写原子文件 + 更新索引 → 错误时删除

压缩：超 150 行/20KB 警告 → 超 200 行/25KB 必须压缩 → 指针化 → 合并重复 → 归档冷门
