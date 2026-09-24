# 模块协议治理（Protocol Governance）

> **协议版本**：v1（`_protocol_version = 1`）
> **规范**：`docs/MODULE_PROTOCOL_SPEC.md`
> **兼容性套件**：`tools/conformance.py`
> **校验器**：`tools/modcheck.py`（R1–R18）

---

## 1. 三个版本号的区别（**易混淆，先厘清**）

| 版本号 | 位置 | 含义 | 变更影响 |
|--------|------|------|---------|
| `_format_version` | 注册表顶层 | **JSON 结构**版本 | 读写解析器 |
| `_protocol_version` | 注册表顶层 | **模块契约语义**版本 | 所有模块的 `abi_version` |
| `abi_version` | 每个模块 | 该模块遵循的协议版本 | 须等于 `_protocol_version`（R10）|

> **校验**：`python tools/modcheck.py` 的 R10 会强制 `abi_version == _protocol_version`。

---

## 2. 什么进核心，什么留扩展（W6.3）

### 2.1 核心（**不可移除、不可替换**）

| 项 | 理由 |
|----|------|
| 注册表结构（`modules` / `primitives`）| 一切编排的基础 |
| 13 个必填字段 | 缺任一即无法参与编排 |
| 校验规则 R1–R18 | 协议的可执行定义 |
| 兼容性套件 `conformance.py` | 第三方的唯一准入标准 |

> **核心的定义**：移除它会**破坏协议的完整性**（而非"少一个功能"）。

### 2.2 扩展（**可自由增删**）

| 项 | 说明 |
|----|------|
| W2 字段（`ordering` / `conflicts_with` / `consumes` / `produces`）| 可选声明；缺省有默认语义 |
| `maturity`（W6.4）| 可选标注；未声明则跳过校验（R18）|
| 未来新字段 | **协议允许扩展** —— `conformance` 对未知字段只**警告**不报错（C9）|

### 2.3 判定原则

```
问: 移除它会不会让"协议"不再是协议？
  会 -> 核心（不可移除）
  不会，只是少个能力 -> 扩展（可自由增删）
```

---

## 3. 模块成熟度分级（W6.4）

| 级别 | 含义 | 准入条件（**机器可校验**，R18）|
|------|------|------------------------------|
| `experimental` | 实验性，接口可能变 | 无要求（但必须标注）|
| `community` | 社区验证 | ≥1 条 drill 记录 |
| `verified` | 经攻防验证 | ≥3 条 drill 且**至少 1 条 BREACHED** |
| `core` | 核心，多轮实战 | ≥5 条 drill + **BREACHED** + 被 ≥2 个参考组合使用 |

### 3.1 为什么要求 `BREACHED`（**关键设计**）

> **只被"验证成功"的模块，说明它没被真正攻击过。**
>
> 诚实边界（P4）要求承认"被打穿过" —— 那才是成熟度的证据。
> 一个 15 条 drill、0 条 BREACHED 的模块，其强度是**未经证伪的声明**。

**本仓库实测结果**（如实分级）：

| 模块 | drill | BREACHED | 组合 | 级别 |
|------|-------|----------|------|------|
| `license` | 14 | **2** | 4 | **core** |
| `dbiguard` | 7 | **1** | 2 | **core** |
| `integrity` | **15** | 0 | 3 | community |
| `selfcrypt` | 14 | 0 | 3 | community |
| `antidebug` | 8 | 0 | 3 | community |
| 其余 8 个 | 1–6 | 0–1 | 1–4 | community |

**注意**：`integrity`（15 条 drill）**评不上 core** —— 因为从未被打穿。
这不是缺陷，而是分级**如实地**表达了"未被证伪 ≠ 强"。

---

## 4. 版本化与变更流程（W6.1）

### 4.1 变更分级

| 变更 | 是否递增 `_protocol_version` | 例子 |
|------|---------------------------|------|
| **增加可选字段** | ❌ 不递增 | 加 `maturity`（v1 内已做）|
| **新增校验规则**（不拒绝既有注册表）| ❌ 不递增 | 加 R17/R18 |
| **新增必填字段** | ✅ **递增** | 未来若 `conflicts_with` 变必填 |
| **改变既有字段语义** | ✅ **递增** | 如 `deps` 从"能力依赖"改为"顺序依赖" |
| **移除字段/规则** | ✅ **递增** | —— |

### 4.2 变更检查清单（**提交前必过**）

```
□ python tools/conformance.py modules.json     # 协议合规
□ python tools/modcheck.py --all-compositions  # 仓库一致 + R1-R18
□ python tools/modcheck.py --selftest          # 规则自证能抓错
□ python tools/conformance.py --selftest       # 套件自证能抓错
□ 若递增协议版本: 更新 _protocol_version + 全部模块 abi_version + 本文件
```

### 4.3 向后兼容承诺

| 承诺 | 内容 |
|------|------|
| **同版本内只增不改** | 不改变既有字段语义、不移除字段 |
| **新增字段可选** | 旧模块缺省时按文档默认值处理 |
| **未知字段容忍** | `conformance` 对未知字段**警告**而非报错（前向兼容）|

---

## 5. 第三方模块准入（W6.2）

### 5.1 自证兼容（**核心机制**）

第三方**无需阅读本仓库源码**，只需：

```bash
# 1. 写入自己的注册表片段（或完整注册表）
# 2. 跑兼容性套件
python tools/conformance.py my-registry.json

# 3. 通过即宣称"兼容协议 v1"
```

**套件覆盖**：结构 / 必填字段 / 值域 / W2 字段格式 / 资源名 /
ordering 无环 / conflicts 完整性 / known_limits 非空 / 协议版本一致。

**输出约定**：
- `[conformance] PASS` -> 兼容
- `[conformance] FAIL: N 项不合规` -> 不兼容（逐条给出原因）
- `[!]` 警告 -> 兼容但有提示（未知字段等）

### 5.2 准入流程

```
1. 实现模块（按 MODULE_PROTOCOL_SPEC.md）
2. 写注册表片段 + 13 必填字段
3. python tools/conformance.py <你的注册表>   -> PASS
4. 提交 PR（附 conformance 输出）
5. 维护者审阅: 重点看 known_limits（诚实边界）与 maturity 证据
```

### 5.3 质量分级与准入的关系

| 级别 | 准入要求 |
|------|---------|
| `experimental` | 通过 `conformance` 即可 |
| `community` | + ≥1 条 drill 记录 |
| `verified` | + ≥3 条 drill 且 ≥1 条 BREACHED |
| `core` | + ≥5 条 drill + BREACHED + 被 ≥2 个参考组合使用 |

> **`core` 不是"投票"或"维护者指定"，而是证据达标的自然结果**（R18 强制校验）。

---

## 6. 现有校验工具一览

| 工具 | 面向 | 自检 | 作用 |
|------|------|------|------|
| `modcheck.py` | 本仓库 | `--selftest` | R1–R18 + 注册表漂移 + 组合自洽 |
| **`conformance.py`** | **第三方** | `--selftest` | 协议合规（可独立实现）|
| `solve_profile.py` | 使用者 | `--selftest` | 策略 -> 模块集 |
| `identity.py` | 发行方 | `--selftest` | 用户种子派生 |
| `observability.py` | 审计者 | `selftest` | 可观测性度量 |
| `migration_gate.py` | 审计者 | `--selftest` | 迁移成本门禁 |
| `mutate_check.py` | 维护者 | — | 检查器变异测试 |

> **全部带 `--selftest` 的工具都能自证"会抓错"** —— 这是门禁失效事件
> （2026-09-23，7/13 项曾静默通过）后确立的**强制纪律**。

---

*协议 v1 · 2026-09-23 · W6 协议治理*
