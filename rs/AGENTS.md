# FaceFusion Rust 移植 — 开发规范

---

## 一、项目概述

将 Python 版 [FaceFusion](https://github.com/facefusion/facefusion)（v3.6.1）完整移植到 Rust，实现 **零配置、跨平台、单入口** 的人脸操作工具。

完整架构设计、模块树、阶段路由、技术决策详见 **[roadmap.md](./roadmap.md)**。

---

## 二、TDD 开发工作流

所有功能开发必须遵循 TDD 模式：**先写测试，后写实现**。

由于 Rust 的编译时检查特性，TDD 各阶段含义如下：

- **RED**：编写测试代码 + 被调函数的最小桩（签名 + `todo!()`）。测试必须编译通过但执行失败（`todo!()` panic 或断言不成立），以此确认测试有效。
- **GREEN**：用真实实现替换 `todo!()`，使测试通过。只写让当前测试通过的最少代码。
- **REFACTOR**：优化代码结构、补充注释、消除重复。任何重构后必须重新运行测试确保仍通过。

```
明确需求与设计
  ↓
[RED] 编写测试代码 + 函数桩（签名 + todo!()）
  ↓ 测试编译通过但执行失败 ← 确认测试有效
[GREEN] 编写最小实现代码（通过测试）
  ↓ 测试全部通过
[REFACTOR] 优化代码结构
  ↓
  ├→ 重构后重新运行测试，确保仍通过
  ↓
提交
```

### 工作流步骤

```
[RED] 编写测试代码 + 调用的函数桩
  ↓
cargo check --workspace                 # 步骤  0: 快速编译检查
  ↓
cargo fmt --all -- --check              # 步骤  1: 格式检查
  ↓
cargo clippy --workspace -- -D warnings       # 步骤  2: 全量静态分析
  ↓ 有告警 → 修复代码 → 回到步骤 0
  ↓ 无告警
cargo test --workspace                  # 步骤  3: 确认测试编译通过但执行失败
  ↓ ┌─ 测试失败（预期）→ 测试有效，进入步骤 4
  │ └─ 测试通过 → 测试无意义，重写测试
  ↓
[GREEN] 编写功能实现代码
  ↓
cargo fmt --all                         # 步骤  4: 格式化新代码
  ↓
cargo clippy --workspace -- -D warnings       # 步骤  5: 静态分析新代码
  ↓ 有告警 → 修复代码 → 回到步骤 4
  ↓ 无告警
cargo test --workspace                  # 步骤  6: 测试应全部通过
  ↓ 有失败 → 修复代码 → 回到步骤 4
  ↓ 全部通过
[REFACTOR] 优化代码 + 补全注释
  ↓
同步更新注释和文档                      # 步骤  7: 检查所有注释与代码一致
  ↓
MDPR 分层审查（见第三节）               # 步骤 7.5: M→D→P→R 逐层自检
  ↓ 有问题 → 修复。代码微调回到步骤 4；行为变更须先补测试（RED）
  ↓ 无问题
cargo doc --workspace --document-private-items  # 步骤 7.6: 文档检查（含 broken link / 缺失注释）
  ↓ 有警告 → 修复注释 → 回到步骤 4
  ↓ 无警告
cargo test --workspace                  # 步骤  8: 重构后确认测试通过
  ↓
cargo build --release                   # 步骤  9: 构建 release 二进制
  ↓
完成 ✓
```

### 各步骤详解

| 步骤 | 命令 | 说明 |
|------|------|------|
| 0. 编译检查 | `cargo check --workspace` | 仅类型检查，几秒完成 |
| 1. 格式检查 | `cargo fmt --all -- --check` | 检查代码风格是否合标 |
| 2. 静态分析 | `cargo clippy --workspace -- -D warnings` | 所有 warning 升级为 error |
| 3. 验证测试 | `cargo test --workspace` | RED 阶段：新增测试须失败（预期），其余测试须通过（`cargo test` 运行全部测试并报告所有失败，多个失败时分批修复即可） |
| 4. 格式化 | `cargo fmt --all` | 自动修复格式问题 |
| 5. 静态分析 | `cargo clippy --workspace -- -D warnings` | 检查新代码的静态质量 |
| 6. 功能测试 | `cargo test --workspace` | GREEN 阶段：全部测试应通过 |
| 7. 同步文档 | 人工审查 | 确认所有注释与代码一致（见第六节） |
| 7.5. 分层审查 | MDPR | M→D→P→R 四层逐步自检（见第三节） |
| 7.6. 文档检查 | `cargo doc --workspace --document-private-items` | 无 broken link / 缺失注释（需设 `RUSTDOCFLAGS="-D warnings"`） |
| 8. 重构验证 | `cargo test --workspace` | REFACTOR 阶段：重构后测试仍通过 |
| 9. 构建 | `cargo build --release` | 确保最终二进制可成功生成 |

### Clippy 静态检查

`cargo clippy` 配置（`clippy.toml` 与 crate 级 `#![warn(...)]` 属性）启用严格模式，覆盖正确性、安全、复杂度、性能、风格等全部 lint 维度。所有 warning 视为 error（`-D warnings`），确保零警告通过。

注：`cargo fmt --all` 的 `--all` 是 Cargo 级旗标，与 `--workspace` 语义等价但为社区惯例写法，故保留。其他 cargo 子命令（`check`、`clippy`、`test`）统一使用 `--workspace`。

clippy 报告的每个 warning 必须修复。极少数无法修复的例外需 `#[allow(clippy::lint_name)]` + 注释说明原因，不得整模块禁用。

### TDD 原则

| 原则 | 说明 |
|------|------|
| 先测试后代码 | 不写测试就写功能代码 = 不合规 |
| 测试即文档 | 测试描述功能行为，替代额外的规格说明 |
| 最小实现 | 只写让当前测试通过的最少代码 |
| 红→绿→重构 | 严格遵守 RED → GREEN → REFACTOR 循环 |
| 参数化测试 | 使用 `rstest` 或 `#[test_case]` 宏避免重复测试代码 |
| 提交前门禁 | 必须通过 `cargo fmt --all && cargo clippy --workspace -- -D warnings && cargo test --workspace` 三重门禁（该命令链在 GREEN 阶段执行一次，提交前作为最终门禁再执行一次） |

### TDD 测试编写要求

1. **参数化测试** — 使用 `rstest` crate（`#[rstest]` + `#[case]`）或 `test-case` crate 覆盖多组输入
2. **测试名使用中文 snake_case** — 描述测试意图（如 `test_检测人脸_空图返回空`；注意非 ASCII 标识符在部分终端/CI 可能影响 `cargo test <filter>` 匹配，CI 脚本建议使用 `-p <crate> <test_fn>` 精确指定）
3. **覆盖三个维度** — 正常路径（happy path）、边界条件（零值/空值/极值）、异常输入（错误魔数、截断数据）
4. **每个公开函数** — 至少有一个测试用例
5. **新增功能** — 测试必须先于实现代码提交

---

## 三、MDPR 四层次审查

单次审查不可能发现所有问题。MDPR 将审查拆为 **4 层独立视角**，每层只关注一类问题，彻底查完再进入下一层。TDD 保证 **功能正确**，MDPR 保证 **代码健壮**。

### 四层定义

```
M — 模块与安全 (Module & Safety)
  → 会 panic 吗？有 unsafe 吗？Send/Sync 正确吗？会死锁吗？索引越界吗？
  → 检查点: 所有 unwrap()/expect()、unsafe{} 块、Mutex/RwLock 持有路径、slice/[index]

D — 数据与格式 (Data & Format)
  → 序列化/反序列化对称吗？二进制读写与 Python 版逐字节一致吗？Endian 正确吗？
  → 检查点: 魔数、字段顺序与类型、字节序与对齐、序列化往返一致性

P — 边界与异常 (Process & Boundary)
  → 零值、空值、类型极值、截断数据每种情况的处理是否正确？Result 正确传播吗？
  → 检查点: 对每个输入参数穷举其类型的所有边界类别

R — 语义与一致性 (Result & Semantics)
  → 行为与 Python 原版一致吗？错误类型语义准确吗？默认值/降级值一致吗？
  → 检查点: 对照同名函数逐条件/逐算术表达式比对
```

### 审查检查清单

代码提交前必须逐项自检：

```
□ M1 [panic 风险] 是否不存在 unwrap() 和 panic!()（§5 禁止）？所有 expect(msg) 是否仅在已证明不可能失败的位置使用并附注释？
□ M2 [unsafe] 每个 unsafe 块是否有 // SAFETY: 注释说明不变量保证？
□ M3 [Send/Sync] 跨线程类型是否正确实现了 Send + Sync（尤其是 *mut, Cell, RefCell）？
□ M4 [死锁] 所有 Mutex/RwLock 的持有路径是否可能形成循环等待？
□ M5 [索引] 所有 slice[idx] / slice[start..end] 是否已验证边界？
□ M6 [内存] 是否存在 Arc 循环引用导致泄漏？是否有无上限增长的集合？

□ D1 [序列化对称] 所有 Serialize/Deserialize、Write/Read 路径是否往返一致？
□ D2 [字节格式] 二进制读写字段顺序与类型是否与 Python 原版对应？
□ D3 [魔数] 读写两端魔数是否一致？
□ D4 [字节序] 是否显式指定字节序（避免平台依赖）？
□ D5 [对齐] #[repr(C)] 结构体的字段布局是否与 Python struct 格式对应？

□ P1 [零值] 输入为 0 / "" / &[] / None / 0x0 尺寸时是否返回错误而非 panic？
□ P2 [极值] 类型上界/下界、NaN/INFINITY 是否被检测并正确处理？
□ P3 [下溢/溢出] 有符号→无符号转换、运算溢出是否使用 checked_/saturating_/wrapping_ 方法？
□ P4 [截断] 输入数据短于预期时是否返回明确的错误而非 panic？
□ P5 [Error 传播] 所有 Result 是否被正确处理（用 `?` 传播或用 `log` 记录），无吞没？

□ R1 [行为一致] 同名函数与 Python 版的条件判断、算术表达式是否一致？
□ R2 [错误类型] 是否使用 thiserror 自定义语义化错误类型（非裸 String）？
□ R3 [默认值] 默认参数/降级逻辑与 Python 版行为一致？
□ R4 [日志级别] trace/debug/info/warn/error 的使用是否恰当？
```

### 审查流程

MDPR 作为工作流步骤 **7.5**，位于同步文档之后、重构验证之前：

```
同步更新注释和文档      # 步骤 7
  ↓
MDPR 分层审查           # 步骤 7.5: M→D→P→R 逐层自检
  ↓ 有问题 → 修复。代码微调回到步骤 4；行为变更须先补测试（RED）
  ↓ 无问题
cargo doc ...           # 步骤 7.6: 文档检查
  ↓
cargo test --workspace  # 步骤 8
  ↓
cargo build --release   # 步骤 9
```

---

## 四、测试策略

所有测试通过 `cargo test --workspace` 一键运行。

### 测试分层

| 层级 | 位置 | 说明 |
|------|------|------|
| **单元测试** | `crates/*/src/` 内 `#[cfg(test)] mod tests` | 不依赖外部文件，所有数据在内存中构造 |
| **集成测试** | `crates/*/tests/` 目录 | 跨模块组合验证，可读取 testdata/ |
| **端到端测试** | 根 `tests/` 目录 | 全管线验证：输入 → 处理 → 输出 |

### 测试覆盖原则

每个公开 API 面必须有测试，覆盖三个维度：

- **正常路径**：典型输入的正常处理流程
- **边界条件**：输入参数的零值、空值、类型极限值
- **异常输入**：格式错误、数据损坏、不完整结构

核心管线模块（推理、人脸分析、处理器）的测试密度应高于基础设施模块。具体覆盖目标由实现者按模块复杂度自行判断，应在代码审查中论证覆盖充分性。

### 边界测试类别

受测函数的每个输入参数必须按以下类别系统覆盖，漏测视为不合规：

| 类别 | 示例 |
|------|---------|
| 空/零值 | 0 字节、空切片、空字符串、0x0 尺寸、None |
| 类型极值 | 类型下界/上界、NaN、INFINITY |
| 格式/完整性 | 错误魔数、截断数据、不完整结构 |
| 并发/资源 | 多线程竞争、资源耗尽、超时 |

### testdata/

`testdata/` 按数据类型分类组织（合法输入、损坏输入、边界构造样例）。所有测试数据通过脚本生成或手工构造，控制在 100KB 以下，不作为 Git LFS 管理。

---

## 五、编码规范

### 错误处理

- **库代码** (`facefusion-core`)：使用 `thiserror` 自定义语义化错误类型。凡可能失败的操作必须返回 `Result`（纯函数如 getter/构造器/纯计算除外）。**禁止 `unwrap()` 和 `panic!`**（仅限提交代码；RED 阶段 `todo!()` 桩为临时例外，须在 GREEN 阶段全部替换），所有错误必须通过 `Result` 传播
- **`expect(msg)`**：CLI 和库代码均仅限已证明不可能失败的位置使用，须注释说明证明理由
- **CLI 入口**（workspace 根二进制）：使用 `anyhow::Result` 做顶层错误包装
- **错误信息** 使用英文（便于日志检索），格式为 `"failed to {action}: {reason}"`

```rust
// ✓ 正确
#[derive(Debug, Error)]
pub enum FaceFusionError {
    #[error("failed to load model at {path}: {source}")]
    ModelLoad { path: String, #[source] source: ort::Error },
    #[error("no faces detected in input image")]
    NoFacesDetected,
}

// ✗ 错误
fn process() -> Image { todo!().unwrap() }  // 库代码禁止 unwrap
fn process() -> Image { panic!("oops"); }   // 禁止 panic
```

### 命名规范

遵循 Rust 标准命名惯例（RFC 430）。Crate 名 `kebab-case`，模块和文件名 `snake_case`，类型名 `PascalCase`，函数和变量 `snake_case`，常量和静态量 `SCREAMING_SNAKE_CASE`。Trait 名通常使用动词、-er 后缀或形容词（如 `-able`）。Rust 标准库既有 `Iterator`、`Read` 也有 `Clone`、`Display`、`Serialize`，以社区惯例为准。

### unsafe 使用

- 所有 `unsafe` 代码集中封装在专门模块，对外提供 safe 接口
- 每个 `unsafe {}` 块前必须有 `// SAFETY: <invariant justification>` 注释
- 禁止裸指针跨函数边界传递
- 禁止在 safe 函数中依赖调用者保证安全前提

### 日志级别

`tracing` 宏选用标准：

| 级别 | 用途 | 示例 |
|------|------|------|
| `error!` | 操作失败、不可恢复错误 | 模型加载失败、文件写入失败 |
| `warn!` | 降级行为、资源接近上限、可恢复异常 | 模型未下载→跳过处理器、内存使用 >80% |
| `info!` | 关键阶段节点 | 模型加载完成、处理开始/结束、帧进度里程碑 |
| `debug!` | 详细执行信息，默认不输出 | 帧号、参数值、推理耗时 |
| `trace!` | 极度详细，仅开发调试 | 张量形状、像素值、中间结果 |

### 并发模型

| 场景 | 运行时 | 说明 |
|------|--------|------|
| ONNX 推理 | tokio::spawn_blocking | `ort::Session::run()` 是阻塞调用 |
| 逐帧预处理 | rayon::par_iter | 图片 resize/normalize 等独立操作 |
| FFmpeg 子进程 | tokio::process | 非阻塞子进程管理 |
| Web Server | axum + tokio | 多线程异步运行时 |
| 全局状态 | tokio::sync::RwLock + Arc | 共享可变状态的安全访问 |

各组件的线程池严格分离，避免 `rayon` 与 `tokio::spawn_blocking` 的线程池竞争。

### 禁止模式

项目的禁止模式由 `clippy.toml` 的 lint 级别统一定义，凡归类为 warn 或 deny 级别的 pattern 均为强制禁止。

对于 clippy 无法静态检测的模式（如裸指针跨 API 边界、整模块 lint 禁用 `#[allow(clippy::*)]`、全局可变静态变量 `static mut`），由 MDPR 审查 M 层逐例检查。

---

## 六、注释与文档规范

### 注释层级

```
层级 1: 模块头注释（//! 文档注释）
   位置: 每个 .rs 文件的 mod 声明或 lib.rs 顶部
   内容: 描述模块整体功能和职责边界

层级 2: 类型注释（/// 文档注释）
   位置: 每个公开 struct/enum/trait 定义之前
   内容: 用途、设计意图

层级 3: 函数/方法注释（/// 文档注释）
   位置: 每个公开函数或方法定义之前
   内容: 功能描述、参数含义、返回值、# Panics / # Errors / # Examples

层级 4: 常量/枚举变体注释（/// 或 //）
   位置: 每个公开常量或枚举变体之前或同行
   内容: 用途和取值含义

层级 5: 内部逻辑注释（//）
   位置: 复杂算法、非直观逻辑、边界条件处理处
   内容: 解释"为什么这样写"，而非"做了什么"；
         若逻辑移植自 Python 原版，注明源文件位置
```

### 注释语言

- 所有注释使用 **简体中文**
- 技术术语保留英文（如 NMS、affine matrix、embedding）
- 代码引用使用反引号（如 `warp_affine()`）
- 错误消息使用英文
- 移植自 Python 原版的逻辑，注释中注明 Python 源文件及行号。Python 源码位于项目根目录 `facefusion/` 下，按同名模块对应查找

### 注释同步检查清单

开发者在步骤 7（同步文档）时必须逐项确认：

```
□ 新增/修改的公开符号是否都有文档注释？
□ 修改行为的函数，其注释是否已更新？
□ 删除的代码，其注释是否已同步移除？
□ 注释中的字段名、参数名是否与代码实际签名一致？
□ 模块头注释是否准确描述当前文件的功能范围？
□ # Panics / # Errors 文档是否与实际实现一致？
□ 内部算法注释是否反映最新实现逻辑（含 Python 源行列号引用）？
```

### 文档

| 文件 | 用途 |
|------|------|
| `AGENTS.md` | 开发规范（给人 + AI） |
| `CLAUDE.md` | 指向 AGENTS.md |
| `roadmap.md` | 完整架构、阶段路由、技术决策 |
| `clippy.toml` | Clippy 静态检查配置 |
| `rustfmt.toml` | 代码格式化配置 |

`cargo doc --workspace --document-private-items` 必须通过（需设 `RUSTDOCFLAGS="-D warnings"`，无 broken doc link 等警告）。

---

## 七、快速参考

```bash
# 质量门禁（一键：fmt + clippy + test + build）
cargo fmt --all && cargo clippy --workspace -- -D warnings && cargo test --workspace && cargo build --release

# 文档检查（需设 RUSTDOCFLAGS 将 warning 升级为 error）
# Unix:  RUSTDOCFLAGS="-D warnings" cargo doc --workspace --document-private-items
# Win:   $env:RUSTDOCFLAGS="-D warnings"; cargo doc --workspace --document-private-items

# 仅格式检查 / 格式化 / 静态分析 / 测试
cargo fmt --all -- --check
cargo fmt --all
cargo clippy --workspace -- -D warnings
cargo clippy --workspace --fix -- -D warnings   # 自动修复（谨慎使用，建议人工审查后提交）
cargo test --workspace

# 运行指定 crate 的单个测试
cargo test -p facefusion-core test_检测人脸_空图返回空

# 运行指定 crate 的全部测试
cargo test -p facefusion-core

# 构建 release
cargo build --release

# 检查依赖重复
cargo tree -e no-dev --duplicates
```
