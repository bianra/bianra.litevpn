# 自研轻量私有隧道 VPN — 最小接口与协议冻结（阶段一 v1.0）

日期：2026-05-03  
目的：冻结“安全模块 ↔ 网络模块”的最小接口；冻结“网络模块 ↔ 客户端”的最小控制消息；冻结错误码与日志事件字段。

> 注意：本文件冻结的是“字段与含义”，具体承载（JSON/二进制、是否加密、是否与数据面复用同一 TCP 连接）可在阶段二实现细化，但字段含义不应随意变更。

---

## 1. 安全模块 -> 网络模块：AuthResult（必选接口）

### 1.1 AuthResult 字段（冻结）
- allow: boolean  
  - true：认证通过，允许进入 IP 分配流程  
  - false：认证失败，必须拒绝连接
- device_id: string  
  - **allow=true 时必填**  
  - 定义：客户端证书 SHA-256 指纹（hex 字符串）
- reason: string  
  - **allow=false 时必填**（例如 `AUTH_FAILED` / `CERT_REVOKED` / `UNTRUSTED_DEVICE`）  
  - allow=true 时可填 `OK`
- binding_hint: string（可选）  
  - 取值：`STATIC` / `DYNAMIC` / `ANY`  
  - 含义：安全侧对“是否允许静态绑定/是否必须动态分配”的建议；最终由网络侧 IPAM 执行规则决定。

### 1.2 行为约束（冻结）
- allow=false：  
  - 网络侧必须：DENY + close
  - 不分配 VIP，不启用数据面
- allow=true：  
  - 网络侧进入 `allocate_ip(device_id)` 逻辑
  - 成功分配后发送 ASSIGNED

---

## 2. 网络模块 -> 客户端：控制消息（最小集合）

### 2.1 ASSIGNED（认证通过 + 分配 IP 后返回）
字段（冻结）：
- type: `ASSIGNED`
- vip: string（10.8.0.x）
- gw: string（固定 10.8.0.1）
- mask: string（固定 /24）
- ttl_seconds: number（建议 120）
- lease_id: string（建议包含，用于调试/审计）

语义约束：
- 客户端收到 ASSIGNED 后才能配置 TUN 地址并开始发送数据面帧（若协议复用）。

### 2.2 DENY（拒绝连接）
字段（冻结）：
- type: `DENY`
- reason: string（见错误码表）

语义约束：
- 客户端收到 DENY 后必须停止快速重试（建议指数退避）。
- 服务端发送 DENY 后应关闭连接。

---

## 3. 错误码（reason）建议表（冻结枚举）

认证相关：
- AUTH_FAILED：认证失败（泛化）
- CERT_REVOKED：证书已吊销
- UNTRUSTED_DEVICE：设备不在白名单/不可信
- PROTOCOL_ERROR：协议错误/字段缺失/版本不兼容

IP 分配相关：
- NO_AVAILABLE_IP：动态池耗尽
- STATIC_IP_CONFLICT：静态绑定 VIP 被占用且策略为拒绝新连接

---

## 4. IPAM 与数据面门控的冻结规则（摘要）
- 认证通过（allow=true）后才能调用 IPAM 分配；
- **先 ASSIGNED 再启用数据面**；
- 心跳：5s 一次；超时：60s 回收与断开；
- 静态冲突：拒绝新连接（不踢旧连接）并记录告警。

---

## 5. 日志（JSON Lines）schema（冻结）

### 5.1 通用字段（必填）
- ts: string（ISO8601）
- level: string（INFO/WARN/ERROR）
- event: string（事件名）
- connection_id: string
- reason: string（可空，但失败/释放必须填）

### 5.2 认证后建议必填
- device_id: string（证书指纹）
- vip: string
- lease_id: string

### 5.3 必须产出的事件名（冻结）
- TCP_CONNECTED
- TCP_DISCONNECTED
- AUTH_START
- AUTH_RESULT
- IP_ASSIGNED
- IP_RELEASED
- HEARTBEAT_SEEN
- HEARTBEAT_TIMEOUT
- STATIC_IP_CONFLICT
- NO_AVAILABLE_IP
- DATAPLANE_ENABLED
- DATAPLANE_DISABLED

### 5.4 事件字段建议（可选但推荐）
- pool: `STATIC` | `DYNAMIC`（用于 IP_ASSIGNED）
- ttl_seconds（用于 IP_ASSIGNED）
- old_connection_id/new_connection_id（用于 STATIC_IP_CONFLICT）