# no-support 原理说明（`Sandboxie-skip-auto-build-master`）

## 结论

`Sandboxie-skip-auto-build-master` 的主机制是**底层签名函数短路**：

- 在驱动层 `KphVerifySignature(...)` 直接 `return STATUS_SUCCESS;`

这是典型“单点核心校验绕过”实现，覆盖面广、改动点集中。

---

## 核心代码路径

### 1) 终点校验函数直接恒成功

文件：`Sandboxie/core/drv/verify.c`

- `KphVerifySignature(...)` 被改为：
  - 不再执行公钥导入/哈希签名验签
  - 直接返回 `STATUS_SUCCESS`

由于其他函数（如文件/缓冲区验证）都依赖这个函数，结果是签名验证链路整体被短路。

---

### 2) 上层逻辑大体保留，靠底层“永真”通过

同文件内 `KphValidateCertificate()` 仍有证书字段解析（`TYPE/UPDATEKEY/SIGNATURE` 等），但最终签名判定依赖 `KphVerifySignature()`，因此会被统一放行。

---

### 3) 业务限制点仍在代码里，但输入状态不再触发拒绝

例如：

- `MountManager.cpp` 里 `UseFileImage/UseRamDisk` 的证书等级检查分支仍存在
- `process.c` 的 `CERT_IS_LEVEL(...)` 限制条件仍存在

但在证书链被底层放行后，这些分支通常不再进入“拒绝路径”。

---

## 与多点改法的差异

该版本不依赖 `NOSUPPORT_PATCH`，也不需要在 UI/向导/更新器里做大量条件编译；它用“改一个验证终点函数”影响整条链路，属于高集中度方案。

---

## 维护特征

- 优点：修改点少，定位清晰
- 代价：对底层验证实现耦合高，一旦上游改验证架构，失效概率更高
