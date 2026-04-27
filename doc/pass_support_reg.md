# 一、目标与背景

* 分析对象：Sandboxie Plus（开源沙盒软件）
* 核心问题：软件许可证（激活码）验证机制
* 方法：基于源码进行逆向分析

---

# 二、整体验证流程

典型调用链：

```
输入许可证
→ 格式解析
→ 多层验证函数
→ 底层算法校验（核心）
```

功能调用路径：

```
功能触发
→ 验证链
→ 算法函数
```

---

# 三、关键调用链（精简）

```
ApplyCertificate
→ ReloadCert
→ ReloadConf
→ NtDeviceIoControlFile（进入驱动层）
→ Conf_Api_Reload
→ MyValidateCertificate
→ KphValidateCertificate
→ KphVerifySignature（核心）
```

---

# 四、许可证结构

字段要求：

```
NAME        必须
SIGNATURE   必须（长度 ≥ 6）
TYPE        必须（决定权限）
DATE        可选
UPDATEKEY   可选
SOFTWARE    固定
```

---

# 五、验证逻辑拆解

## 1. 格式校验

* NAME 非空
* SIGNATURE 非空

---

## 2. 业务校验

* 检查许可证类型（TYPE）
* 检查有效期（DATE）

---

## 3. 哈希与签名验证

相关函数：

* `MyInitHash` → 初始化哈希结构
* `MyFinishHash` → 完成哈希计算
* `KphVerifySignature` → **签名校验核心**

---

# 六、关键点（核心机制）

## 1. 验证核心

* `KphVerifySignature` 是最终判定函数
* 特点：

  * 纯算法函数
  * 无副作用（不修改外部状态）

---

## 2. 副作用问题

* 部分函数会修改全局变量，例如：

  ```
  Verify_CertInfo.valid = 1
  ```
* 仅修改返回值可能导致验证失败（因为状态不一致）

---

# 七、关键突破思路

## 优先策略

修改**最底层算法函数**

原因：

* 所有验证路径最终都会调用该函数
* 可统一控制验证结果

---

## 不推荐策略

* 逐层修改验证函数
* 仅修改返回值（忽略副作用）

---

# 八、特殊字段机制

## TYPE 字段

可选值：

* `CONTRIBUTOR`
* `PERSONAL-HUGE`

特性：

* 可绕过有效期检查

---

# 九、构造规则总结

有效许可证满足：

* 基本字段完整
* TYPE 使用高权限类型
* SIGNATURE 长度合法
* 格式符合解析规则

---

# 十、驱动层验证机制

* 验证逻辑位于驱动层
* 通过：

  ```
  NtDeviceIoControlFile
  ```

  与内核通信

---

# 十一、系统限制

## 驱动签名要求

* Windows 要求驱动必须签名

### 测试模式方案：

```
bcdedit /set testsigning on
```

副作用：

* 安全性降低
* 系统显示测试模式
* 影响部分软件（如反作弊）

---

# 十二、通用逆向方法论

## 标准路径

```
提示信息
→ 定位验证入口
→ 跟踪调用链
→ 找到算法函数
→ 修改或控制算法结果
```

---

## 核心原则

* 优先控制“结果产生点”（算法函数）
* 注意隐藏状态（全局变量）
* 验证路径可能多条，但底层算法通常唯一

---

# 十三、补充观察

* SIGNATURE 长度限制与哈希计算相关
* 存在类似验证函数：

  * `KphVerifyFile`
  * `KphVerifyCurrentProcess`

---

# 总结

核心在于：
**定位并控制最终签名验证算法函数，同时确保相关状态变量一致，从而影响整个验证体系。**
