# SecOps Agent — Architecture & Build Guide

> 基于 AgentScope v2。本文档描述架构设计思路、技术选型理由和构建方向，不是确定的实施计划。
> 阅读顺序：先看 README.md 了解项目总览，再看本文档理解架构决策和构建脉络。

---

## 1. 架构分层

核心原则：**分层解耦**。Agent 推理不感知底层工具实现，不感知沙箱容器细节，不感知前端渲染方式。

```
前端 (React/Vue, SSE) ←→ AgentScope (FastAPI + 事件系统)
                            ├── Agent 层 (ReAct loop; 主 Agent → 子 Agent via Agent-as-Tool)
                            ├── 权限层 (plan / default / allow)
                            ├── MCP 工具层 (tools as MCP servers; Toolkit 自动发现)
                            └── Workspace 层 (local / Docker / cloud)
```

每层职责：

| 层 | 职责 | 依赖方向 |
|----|------|---------|
| Agent 推理层 | ReAct 循环、任务分解、子 Agent 调度、聚合结论 | 只依赖工具抽象接口 |
| 权限控制层 | 工具调用/文件读写/命令执行的三级审批 | 独立，不参与推理 |
| 工具抽象层 | 定义能力接口 + 适配具体工具 | 不感知 Agent 逻辑 |
| Workspace 层 | 提供隔离执行环境，管理容器生命周期 | 不感知上层业务 |

---

## 2. Agent 层设计

### 2.1 为什么多 Agent

单个 Agent 承担所有职责会导致：系统提示词过长、推理负担过重、工具调用混乱（扫描工具和处置工具混在一起）。

拆分思路：按**安全运营职能域**划分 Agent，每个只拥有完成其职能所需的工具子集。

### 2.2 Agent 分工

| Agent | 职能 | 持有的工具域 |
|-------|------|------------|
| Orchestrator | 任务分解、路由、聚合、审批 | 子 Agent 调度、人工审批、SOP 匹配 |
| Scan | 网络探测、资产发现 | 端口扫描、服务探测、漏洞扫描 |
| Analysis | 告警研判、关联分析 | 日志查询、威胁情报、异常检测 |
| Response | 处置执行 | 封禁/解封、主机隔离、通知推送 |

分工粒度可根据实际复杂度调整（多几个少几个都行）。

### 2.3 Agent 间通信

子 Agent 以 **Tool 的形式**注册到主控 Agent。主控在推理循环中"调用子 Agent"实际上是在执行一个 Tool——该 Tool 内部启动完整的 ReAct 子会话，结束后将结论返回主控。这走的是 AgentScope 标准的 Tool Calling 机制，不需要额外消息总线或 RPC 层。

### 2.4 系统提示词

每个 Agent 的系统提示词从独立模板文件加载，不硬编码在 Agent 构造函数中。原因：
- 提示词需要独立迭代（调 prompt 不应改代码）
- SOP 路径和自主路径可能需要不同风格的提示词
- 未来支持 A/B 测试

---

## 3. 工具抽象层设计

### 3.1 核心思路

Agent 面向**能力接口**编程，不面向具体工具编程。如果 Agent 直接调 `nmap`，换 `masscan` 就要改推理逻辑。

```
Agent 推理层（只知道"有一个 PortScanner 能力"）
        │
        ▼
抽象接口层（定义 PortScanner 的输入输出 Schema）
        │
        ▼
Adapter 层（NmapAdapter 实现 PortScanner，MasscanAdapter 也实现）
        │
        ▼
Sandbox 执行层（Adapter 在沙箱内调具体工具二进制）
```

### 3.2 能力域参考

以下列出常见安全运营能力域，具体实现时按需增减：

| 能力域 | 接口职责 | 可适配工具举例 |
|-------|---------|-------------|
| 端口扫描 | 给定目标，返回开放端口和服务映射 | nmap / masscan / rustscan |
| 漏洞扫描 | 给定目标，返回漏洞列表及严重程度 | nuclei / OpenVAS |
| 威胁情报 | 给定 IOC，返回信誉和来源 | VirusTotal / AlienVault OTX / Shodan |
| 日志查询 | 给定查询条件，返回匹配日志条目 | ELK / Splunk / Loki |
| 访问控制 | 给定动作和目标，执行 ACL 变更 | iptables / nftables / Palo Alto API |
| 主机操作 | 给定主机和动作，执行主机级操作 | SSH / WinRM / EDR API |
| 通知推送 | 给定渠道和内容，发送通知 | 钉钉 / 飞书 / 邮件 / Slack |

替换底层工具：新增 Adapter → 改注册表绑定 → 完成。Agent 推理逻辑和 SOP Playbook 不受影响。

### 3.3 MCP 协议

AgentScope 原生支持 MCP 协议。外部工具作为 MCP Server 运行，Agent 通过 Toolkit 发现并调用。工具可用任意语言实现、独立部署。

---

## 4. 执行环境

### 4.1 Workspace 定位

AgentScope 的 Workspace 抽象将本地文件系统、Docker 容器、云沙箱统一为同一套接口。Agent 逻辑不感知底层是哪种环境。

本项目在此基础上：
- 自定义 Workspace 镜像，预装安全工具
- 利用 AgentScope 原生权限系统控制安全边界
- 按用户会话绑定执行环境，同一 session 复用

### 4.2 连接外部设备

安全工具需要触达外部设备（防火墙、交换机、EDR 端点）。Docker Workspace 容器天然通过 NAT 访问外网。额外需要处理：隔离内网路由、凭证安全注入、网络出口白名单。这些属于运维配置层。

### 4.3 Workspace 类型选用

| Workspace 类型 | 适用场景 |
|---------------|---------|
| Docker Workspace | 运行 CLI 安全工具 — 默认选择 |
| Browser Workspace | 查询 Web 端威胁情报平台 |
| 本地 Workspace | 开发调试，无需容器隔离 |

---

## 5. 事件驱动与双模工作

### 5.1 两条路径

外部安全事件进入系统后分为两条路径，互补而非替代：

- **SOP 路径**（人工预设）：事件匹配到已有 Playbook → 按预设步骤执行。Agent 在此路径下是执行者和解释者。
- **自主推理路径**（ReAct）：事件未匹配到任何 Playbook → Agent 在工具空间中自行决定每一步查什么。

自主推理中验证有效的调查模式可沉淀为新的 SOP Playbook。

### 5.2 行动权限分级

无论哪条路径，最终操作都经过统一风险分级，风险等级在工具注册时标注：

- **低风险**（查情报、生成报告）：自动执行
- **中风险**（扩大监控、通知人员）：自动执行，事后通知
- **高风险**（封禁 IP、隔离主机、修改 ACL）：生成建议 → 推送审批 → 确认后执行

### 5.3 人工介入保留点

以下环节始终保留人工入口：Playbook 编写与维护、自主推理结果审核、高风险操作审批、紧急中断 Agent 并手动注入指令。

---

## 6. 前端通信

前后端使用 SSE（Server-Sent Events）。AgentScope 内建事件系统流式输出模型调用、工具执行、用户确认等事件，前端 SSE 直接消费。

前端需要处理的 SSE 事件：`text`（流式文本）、`tool_call`（折叠卡片）、`tool_result`（展开结果）、`approval_request`（弹窗审批）、`error`（错误提示）、`done`（本轮结束）。

页面划分：AI 对话、告警中心、仪表盘、审批面板（具体根据产品需求增减）。

---

## 7. 构建顺序

下面用四阶段推进，每阶段产生可运行的增量：

### Phase 1 — 最小闭环

目标：跑通"一个 Agent + 一个工具 + 前端对话"。

- AgentScope 服务能启动
- 注册 1-2 个简单工具（如执行 Shell 命令、查询威胁情报 API）
- 前端能发送消息并看到 SSE 流式回复和工具调用过程
- Docker Compose 能一键启动 backend + redis + frontend

### Phase 2 — 工具抽象层

目标：按能力域定义接口，实现 3-4 个核心适配器。

- 定义 PortScanner、ThreatIntel 等接口
- 实现 NmapAdapter、VTAdapter 等
- 注册表绑定 + 环境自动发现
- 自定义 Sandbox 镜像预装安全工具

### Phase 3 — 多 Agent

目标：Orchestrator + 专职 Agent 协作。

- Orchestrator 能拆解任务并路由到子 Agent
- 子 Agent 以 Tool 形式注册
- 实现人工审批 Tool（SSE 推送 → 前端弹窗 → 回调）

### Phase 4 — 扩展

目标：事件驱动、SOP 引擎、前端完善。

- 接入外部设备事件 (EventBus)
- SOP Playbook 引擎（YAML 定义 → 执行）
- 前端告警中心、仪表盘、审批面板
- 测试覆盖

---

## 8. 可参考的 MCP 安全工具

以下开源 MCP 服务器可直接作为工具层实现参考或直接集成：

### 安全工具类（参考 FuzzingLabs MCP Security Hub）

- **nmap-mcp-server** — 端口扫描和服务探测
- **nuclei-mcp-server** — 漏洞扫描
- **sqlmap-mcp-server** — SQL 注入检测
- **ghidra-mcp-server** — 二进制逆向分析
- **yara-mcp-server** — 恶意软件规则匹配
- **prowler-mcp-server** — 云安全审计
- **gitleaks-mcp-server** — 密钥泄露扫描

GitHub: [FuzzingLabs/mcp-security-hub](https://github.com/FuzzingLabs/mcp-security-hub)

### 威胁情报类

- **cve-mcp-server** — CVE 查询、EPSS 评分、CISA KEV、MITRE ATT&CK、Shodan、VirusTotal、GreyNoise 等 27 个安全情报工具
- GitHub: [mukul975/cve-mcp-server](https://github.com/mukul975/cve-mcp-server)

### MCP 安全防护（运行时保护）

- **mcpwall** — 确定性安全代理，"iptables for MCP"，阻断危险工具调用、密钥泄露检测
- **Aperion Shield** — 本地护栏，45+ 自适应安全规则，覆盖 SQL/Git/文件系统/密钥/供应链

---

