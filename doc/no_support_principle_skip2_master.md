# no-support 原理说明（`Sandboxie_skip2_-master`）

## 结论

`Sandboxie_skip2_-master` 的实现不是单点“签名函数恒成功”，而是**多处放宽校验**，核心是：

- 取消服务端关键签名拦截（`STATUS_INVALID_SIGNATURE` -> `STATUS_SUCCESS`）
- 去掉 `UseFileImage/UseRamDisk` 的证书等级门槛
- UI 侧遇到签名错误时继续运行

整体属于“多点绕过 + 保持主流程不报错”的风格。

---

## 核心代码路径

### 1) 服务进程签名校验被放行

文件：`Sandboxie/core/drv/api.c`

- 在 `Api_SetServicePort(...)` 里，原始逻辑是：
  - `MyIsCallerSigned()` 失败 -> `STATUS_INVALID_SIGNATURE`
- 该版本改成：
  - `MyIsCallerSigned()` 失败 -> `STATUS_SUCCESS`

这会让未通过签名校验的调用方继续完成服务端口设置流程。

---

### 2) 镜像沙盒/内存盘的证书门槛被移除

文件：`Sandboxie/core/svc/MountManager.cpp`

- `AcquireBoxRoot(...)` 中原本对 `UseFileImage/UseRamDisk` 有证书等级检查
- 该版本把这段前置拦截分支去掉（不再先置 `errlvl=0x66` 并打 6008/6009 拒绝日志）

效果是：相关挂载路径不再被许可证等级拦住。

---

### 3) GUI 侧签名错误提示被弱化为“继续”

文件：`SandboxiePlus/SandMan/SandMan.cpp`

- `ConnectSbieImpl()` 中遇到 `STATUS_INVALID_SIGNATURE` 时
  - 原本会弹出致命错误提示
  - 该版本注释掉弹窗并直接 `Status = SB_OK`

表现上是“连接继续，不中断主流程”。

---

## 与“单点算法绕过”差异

该版本没有采用 `NOSUPPORT_PATCH` 这种统一编译开关，也不是只改一个 `KphVerifySignature()` 出口，而是针对几个关键业务节点分散修改，目标是让流程“尽量不失败”。

---

## 维护特征

- 优点：对用户可见路径（连接、挂载）见效直接
- 代价：修改点分散，后续跟上游同步时更容易产生冲突
