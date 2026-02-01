# PRD: Runtime Hardware-Bound License Verification

> **Parent Document**: [Product Requirements Document (CrewAgent)](prd.md)

## 1. 概述

本 PRD 详细描述了 Runtime 环境的 License 许可机制需求。
目标是实现一套基于**硬件绑定**和**网络自助激活**的许可体系，防止盗版和未授权共享，同时支持离线环境运行。
商业模式演进分为两个阶段：Phase 1 (Builder 工具发卡) 和 Phase 2 (SaaS 平台发卡)。

---

## 2. 用户故事 (User Stories)

*   **作为软件供应商**，我希望 Runtime 在试用期后锁定核心功能，并要求输入与机器硬件绑定的激活码，以便实现软件收费并防止 License 被随意复制分发。
*   **作为终端用户**，我希望拥有 15 天的免费试用期，以便在购买前评估软件功能。
*   **作为系统管理员**，我希望能通过离线方式（输入激活码）激活软件，以便在内网或保密环境中使用。
*   **作为管理员/代理商**，我希望能使用 Builder 工具为客户生成专属激活码，以便灵活进行销售。

---

## 3. 问题描述 (Problem Definition)

当前 Runtime 缺乏有效的权限管控机制：
1.  **无试用期限制**：软件可无限期免费使用，无法转化为商业收入。
2.  **缺乏防盗版机制**：如果仅使用简单的通用激活码，容易被一人购买后全网共享（Key Sharing）。
3.  **离线需求**：部分目标客户位于离线环境，纯在线验证（Online Activation）不可行。

需要一种平衡方案：既能强力防止 Key 共享（硬件绑定），又能支持离线激活（非对称加密签名）。

---

## 4. 验收标准A (Acceptance Criteria)

### AC-1: 全局试用期控制
- **初次运行检测**：软件首次启动时，记录不可篡改的“首次运行时间戳”。
- **倒计时提示**：在试用期（15天）内，UI 显著位置显示“试用期剩余 X 天”。
- **强制锁定**：试用期结束后，拦截所有核心功能（Agent 运行、Workflow 执行），仅保留“激活界面”和基础设置/导出功能。

### AC-2: 硬件指纹生成 (Fingerprinting)
- **采集**：在激活页面，自动采集本机硬件特征（CPU Serial, Mainboard UUID, MAC Address 等）。
- **生成**：通过 Hash 算法生成唯一的 `Machine ID`。
- **展示**：在界面显示 `Machine ID` 并支持一键复制，提示用户“请将此码发送给管理员获取激活码”。

### AC-3: 离线激活验证
- **输入**：用户输入 License Key。
- **验签**：系统使用内置公钥 (Public Key) 验证 License Key 的数字签名。
- **硬件比对**：解密 Payload，校验 `Payload.MachineID` 是否与本机一致。
- **防时间篡改**：校验 `Payload.ExpiresAt` 是否过期，并检查系统时间是否遭回拨（如 LastRunTime > CurrentTime）。
- **解锁**：验证全部通过后，解除功能锁定，持久化保存激活状态。

### AC-4: 管理端发卡 (Phase 1)
- **工具入口**：在 CrewAgent Builder 中提供“License 生成器”工具。
- **输入**：管理员输入客户提供的 `Machine ID`。
- **输出**：生成加密签名的 `License Key` 字符串。

---

## 5. 技术设计 (Technical Design)

### 5.1 数据结构

```typescript
interface LicensePayload {
  machineId: string;      // 硬件指纹 (必填，防共享)
  issuedAt: number;       // 签发时间
  expiresAt: number;      // 过期时间 (-1 为永久)
  type: 'trial' | 'commercial'; 
  customerName?: string;  // 客户标识
}
```

### 5.2 核心算法
*   **指纹库**: 使用 `node-machine-id` 获取跨平台的唯一标识。
*   **加密套件**: 
    *   **Phase 1**: 使用现成的 RSA 或 Ed25519 签名算法库。
    *   **Private Key**: 存储在 Builder (混淆/加密保护)。
    *   **Public Key**: 嵌入在 Runtime 源码中。

### 5.3 安全对抗
*   **防重置**: 将安装时间写入系统级位置（如 Windows Registry, macOS Keychain, 或用户根目录下的隐蔽文件 `.config/.init`）。
*   **防倒流**: 每次运行更新“最后活跃时间”。启动时主要检查：`CurrentTime < LastActiveTime` 则判定为时间被篡改。

---

## 6. 演进路线 (Roadmap)

| 阶段 | 范围 | 核心目标 |
| :--- | :--- | :--- |
| **Phase 1** | Runtime 硬件锁 + Builder 离线发卡 | 低成本快速上线，满足人工售卖需求。 |
| **Phase 2** | Web SaaS 平台发卡 | 自动化售卖，代理商配额管理，Builder 联网扣费发卡。 |
