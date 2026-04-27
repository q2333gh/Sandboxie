# no-support 原理说明（本工作区最新提交 `eb2890ba`）

## 结论

本工作区最新提交（`eb2890ba`）的机制是**编译期开关驱动的多模块改造**：

- 新增 `nosupport.h` 并定义 `NOSUPPORT_PATCH`
- 驱动层、管理器、向导、设置页、更新请求等多个模块按 `#ifdef NOSUPPORT_PATCH` 改行为

它不是单点验签短路，而是“统一开关 + 多路径显式放行”。

---

## 核心代码路径

### 1) 驱动层证书状态直接构造为可用

文件：`Sandboxie/core/drv/verify.c`

- `KphValidateCertificate()` 在 `NOSUPPORT_PATCH` 分支里直接填充：
  - `active=1`
  - `opt_sec/opt_enc/opt_net/opt_desk=1`
  - `level=eCertMaxLevel`
- 然后直接 `return STATUS_SUCCESS`

这是“状态注入”式放行，不依赖真实证书内容。

---

### 2) 进程特性门控直接条件编译绕过

文件：`Sandboxie/core/drv/process.c`

- 对 `UseSecurityMode / ConfidentialBox` 等受证书限制的判断块，使用 `#ifndef NOSUPPORT_PATCH` 包裹
- 在 `NOSUPPORT_PATCH` 打开时，这些拒绝或延迟终止分支不执行

---

### 3) SandMan 侧同步改为“可运行状态”

关键文件：

- `SandboxiePlus/SandMan/SandMan.cpp`
- `SandboxiePlus/SandMan/main.cpp`
- `SandboxiePlus/SandMan/Windows/SettingsWindow.cpp`
- `SandboxiePlus/SandMan/Wizards/SetupWizard.cpp`
- `SandboxiePlus/SandMan/OnlineUpdater.cpp`

主要行为：

- `ReloadCert()` 中将 `g_CertInfo` 显式置为可用高等级
- 对 `STATUS_INVALID_SIGNATURE` 不再走致命阻断路径
- 设置页/向导改成“证书检查已禁用”模式
- 更新请求参数中按开关处理 `update_key`

---

## 与另外两种方案的关系

- 相比 `skip-auto-build`：不是“改一个签名函数”，而是多模块一致化改造
- 相比 `skip2`：不只是放宽若干业务节点，而是引入统一宏开关，便于整套开启/关闭

---

## 维护特征

- 优点：行为可控、意图清晰（开关化）
- 代价：改动面广，后续合并上游时冲突概率高于单点方案
