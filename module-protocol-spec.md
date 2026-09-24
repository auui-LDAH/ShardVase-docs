# ShardVase 模块协议规范 v1

> **这是什么**：一份**独立于实现**的开放规范。第三方可据此实现模块，
> 无需阅读本项目源码。
>
> **与其它文档的区别**：
> - `docs/MODULAR_ARCH.md` —— **本项目的**模块化架构（含实现细节）
> - `docs/CONTRACT.md` —— **本项目的**实现契约
> - **本文档** —— **协议本身**（版本化、可独立实现、可独立测试）
>
> **版本**：协议 v1（`_protocol_version = 1`）
> **机器可读契约**：`modules.json`（本文档是其**规范说明**）
> **校验工具**：`tools/modcheck.py`（含 `--selftest` 负向测试）

---

## 1. 设计目标

| 目标 | 说明 |
|------|------|
| **可独立实现** | 第三方按本文档写模块，无需读本项目源码 |
| **可机器校验** | 每条规则都有对应检查（`modcheck` 的 R1–R16）|
| **向后兼容** | 新字段**只增不改**；旧模块缺新字段时按默认值处理 |
| **效果而非实现** | 协议声明模块**做什么**，不规定**怎么做** |

---

## 2. 概念模型

```
┌─────────────────────────────────────────────────────────┐
│  宿主程序 (Host)                                         │
│  · 提供 main()、VM、许可证判定等「宿主下限」               │
└───────────────┬─────────────────────────────────────────┘
                │ 组合 (Composition) = 模块集合 + 顺序
                ▼
┌─────────────────────────────────────────────────────────┐
│  模块 (Module)  × N                                     │
│  · 每个模块 = 源码 + 头文件 + 契约声明 (modules.json)     │
│  · 通过 phases 参与生命周期，通过 caps 声明能力           │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  原语 (Primitive)  × M                                  │
│  · 无独立契约的公共设施（如 crc32、vm_rekey）             │
│  · 参与链接顺序与数据流，但不作为防护能力计数             │
└─────────────────────────────────────────────────────────┘
```

**关键区分**：
- **模块**：有完整契约，是防护能力单元
- **原语**：公共设施，仅需 `source` + `reason`（说明为何不独立成模块）

---

## 3. 契约字段规范

### 3.1 必填字段（13 项）

| 字段 | 类型 | 语义 | 校验规则 |
|------|------|------|---------|
| `source` | string | 源文件路径（相对仓库根）| R1（存在性）|
| `header` | string | 头文件路径 | R1 |
| `exports` | string[] | 导出的符号名 | R2（与头文件声明一致）|
| `deps` | string[] | 依赖的**其它模块** | R3（与源码 include 一致）|
| `phases` | string[] | 参与的阶段 | R4（值域）|
| `caps` | string[] | 能力标签 | R4（值域）|
| `modifies_text` | bool | 是否改写 `.text` | R5 |
| `verifies_text` | bool | 是否校验 `.text` | R5 |
| `page_align` | bool | 是否需独占页 | R6 |
| `needs_writable_dir` | bool | 是否需可写目录 | — |
| `known_limits` | string[] | **诚实边界**（不得为空）| R7 |
| `evidence` | string | 演练记录路径 | R8（drill 覆盖）|
| `abi_version` | int | 协议版本（须等于顶层 `_protocol_version`）| **R10** |

### 3.2 值域定义

**`phases`**（生命周期阶段，按执行顺序）：
```
init    初始化     —— 一次性设置
scan    扫描       —— 采集信息，不改状态
decide  判定       —— 基于报告做决策
enforce 执行       —— 施加策略（可能改状态）
watch   监视       —— 持续/周期检查
```

**`caps`**（能力标签）：
```
detect   检测（只读观察）
prevent  阻止（主动干预）
verify   校验（完整性）
conceal  隐藏（提高分析成本）
respond  响应（对已发现威胁采取行动）
```

### 3.3 W2 新增字段（协议 v1 引入）

#### `conflicts_with` —— 显式冲突声明

```json
"conflicts_with": [
  {
    "with": "integrity",
    "reason": "selfcrypt 原地改写 .text，而 integrity 校验 .text",
    "resolution": "必须启用 selfcrypt 掩码豁免"
  }
]
```

| 键 | 必填 | 语义 |
|----|------|------|
| `with` | ✅ | 冲突的模块名（须存在，**R13**）|
| `reason` | 建议 | 为何冲突 |
| `resolution` | ✅ | **如何共存**（**R14**：不能只说"冲突"不说怎么办）|

> **设计原则**：冲突**不等于禁止组合**。多数冲突有已知的共存条件
> （如掩码豁免）。协议要求把条件**写出来**，使编排器能自动满足。

#### `ordering` —— 模块级先后约束

```json
"ordering": {
  "after":  ["vm", "crc32"],
  "before": ["selfcrypt"],
  "priority": 80
}
```

| 键 | 语义 | 校验 |
|----|------|------|
| `after` | 必须在这些模块**之后** | **R11**（引用须存在）|
| `before` | 必须在这些模块**之前** | **R11** |
| `priority` | 无约束时的确定性排序依据（小=前）| — |

> **校验**：`ordering` 必须**无环**（**R12**，可拓扑排序）。
>
> **为何需要**：此前顺序是 `SOURCE_ORDER` 的**全局硬编码列表**，
> 第三方模块无法声明自己的先后约束。
>
> **注**：`after`/`before` 可引用**原语**（如 `crc32`），不只是模块。

#### `consumes` / `produces` —— 数据流声明

```json
"consumes": ["text.bytes", "crc.value"],
"produces": ["integrity.state"]
```

**资源名格式**（**R15**）：小写点分，形如 `<域>.<项>`
- 合法：`text.bytes`、`vm.blob`、`license.state`
- 非法：`BAD_NAME`、`text`、`Text.Bytes`

**数据流自洽**（**R16**，warning）：若模块 `consumes` 资源 X，
则应有某模块（或原语）`produces` X，或 X 属**外部输入白名单**：
```
text.bytes / rdata.bytes / pdata.bytes   由编译器产出
strings.table                            由构建系统产出
vm.blob                                  由宿主提供
```

> **为何是 warning 而非 error**：新模块的数据来源可能尚未实现完，
> 阻塞构建过严。但**必须可见**（不静默）。

---

## 4. 校验规则清单

| 规则 | 内容 | 级别 |
|------|------|------|
| R1 | `source`/`header` 文件存在 | error |
| R2 | `exports` 与头文件声明一致 | error |
| R3 | `deps` 与源码 `#include` 一致 | error |
| R4 | `phases`/`caps` 值域合法 | error |
| R5 | `modifies_text` × `verifies_text` 不同时成立 | error |
| R6 | `page_align` 模块页资源不冲突 | error |
| R7 | `known_limits` 非空 | error |
| R8 | 每模块 ≥1 条 drill 记录 | warning |
| **R10** | `abi_version` == 顶层 `_protocol_version` | error |
| **R11** | `ordering` 引用存在 | error |
| **R12** | `ordering` 无环 | error |
| **R13** | `conflicts_with.with` 引用存在 | error |
| **R14** | `conflicts_with` 必须含 `resolution` | error |
| **R15** | 资源名格式合法 | error |
| **R16** | `consumes` 有来源或属外部白名单 | warning |

**全部规则均可自证能抓错**：`python tools/modcheck.py --selftest`
（用已知违规输入验证每条规则会报错 —— 见 §6）。

---

## 5. 实现一个模块（第三方最小步骤）

```
1. 写 src/my_module.c     —— 实现 prot_my_module_* 函数
2. 写 include/protect/my_module.h —— 声明导出
3. 在 modules.json 的 modules 下加一项，声明 13 个必填字段 + 可选 W2 字段
4. 加 drills/*.json 一条演练记录（R8）
5. 跑 tools/modcheck.py 校验
6. 写 compositions/my_comp.json 声明组合，跑 tools/compose.py
```

**最小契约示例**（虚构模块 `myguard`）：
```json
"myguard": {
  "source": "src/myguard.c",
  "header": "include/protect/myguard.h",
  "exports": ["prot_myguard_scan"],
  "deps": ["syscall"],
  "phases": ["scan"],
  "caps": ["detect"],
  "modifies_text": false,
  "verifies_text": false,
  "page_align": false,
  "needs_writable_dir": false,
  "known_limits": ["示例模块: 仅演示契约, 无实际防护能力"],
  "evidence": "docs/ATTACK_DRILL_EXAMPLE.md",
  "abi_version": 1,
  "consumes": [],
  "produces": ["myguard.report"],
  "ordering": { "priority": 50 }
}
```

---

## 6. 兼容性与版本化

| 项 | 约定 |
|----|------|
| **协议版本** | `_protocol_version`（顶层）。**语义版本**，非 JSON 格式版本 |
| **JSON 格式版本** | `_format_version`（顶层）。仅指文件结构 |
| **兼容性承诺** | 同 `_protocol_version` 内：**只增字段，不改语义** |
| **破坏性变更** | 递增 `_protocol_version`；旧模块的 `abi_version` 不匹配 -> 校验失败 |
| **新增字段** | 视为**可选**；旧模块缺省时按文档默认值处理 |

**实现者自测清单**：
```
□ python tools/modcheck.py --selftest     # 校验器自身健康
□ python tools/modcheck.py --compose <你的组合>
□ 13 个必填字段齐全
□ known_limits 非空（诚实边界是协议要求）
□ 有 drill 记录
```

---

## 7. 设计原则（为什么这样设计）

| 原则 | 体现 |
|------|------|
| **P4 诚实边界是资产** | `known_limits` 为**必填**且不得为空 |
| **效果而非实现** | `caps`/`phases` 声明能力，不规定实现方式 |
| **冲突有解而非禁止** | `conflicts_with.resolution` 必填 |
| **声明即可校验** | 每条规则都有机器检查（R1–R16）|
| **检查器自证能抓错** | `--selftest` 内置负向测试 |
| **向后兼容** | 新字段只增不改；缺省有默认语义 |

---

## 8. 与实现的关系

| 文档/工具 | 角色 |
|-----------|------|
| **本文档** | 协议规范（**版本化、独立**）|
| `modules.json` | 机器可读契约（本文档的实例）|
| `tools/modcheck.py` | 校验器（实现 R1–R16）|
| `docs/MODULAR_ARCH.md` | 本项目架构（含实现细节）|
| `docs/CONTRACT.md` | 本项目实现契约 |

> **规范先于实现**：若实现与本文档冲突，**以本文档为准**，实现需修正。
> （这是"开放规范"与"内部文档"的根本区别。）

---

*协议 v1 · 2026-09-23 · 引入 W2 形式化字段（`abi_version`/`ordering`/`conflicts_with`/`consumes`/`produces`）*
