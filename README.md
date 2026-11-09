# StanSkillHub 1.0（SSKH）· 类 DBMS 的 Skill 管理平台

> **一句话**：在 EasySkillHub 的资源站之上，把 Skill 库当成**一个数据库**来管 ——
> 逻辑库 / 索引 / 约束 / 事件 / 事务台账 / 查询台，全部有；并且能把 Skill **真的挂到一个智能体上跑一次**。

| 项 | 值 |
| --- | --- |
| 产品全名 | **StanSkillHub** |
| 版本 | **1.0** |
| 缩写 | **SSKH** |
| 端口 | **30801** |
| 定位 | 管理平台模式（把 Skill 库类比 DBMS） |
| 软件开发方法 | 原型开发 + 敏捷开发 |
| 目标客户 | 中大型软件企业、知识挖掘与大数据型企业 |
| 与简易版的关系 | **包含** ESKH 的全部 11 个页面与 26 个端点，行为一致 |
| 数据库 | H2 2.2.x（默认）/ MySQL 8（`SKHUB_DB=mysql`），**23 张表** |

## 30 秒跑起来

```cmd
cd NS-StanSkillHub
start.cmd
```

首次启动：建 23 张表 → 导入 375 个 Skill → 写入库语义种子（默认库 / 暂存库 / 4 个索引 / 5 条约束 / 3 个事件）
→ 写入 9 条评估指标 → 登记 2 个 Agent 与 2 个 MCP。打开 <http://127.0.0.1:30801>。

## 这个版本比简易版多了什么（四块）

### 1. Skill-MS 库引擎（核心）

| 对象 | 是什么 | 在哪 |
| --- | --- | --- |
| **逻辑库 Skillset** | Skill 的"分类抽屉"，类比 database | 「库管理」 |
| **索引** | 给属性建"目录页"，带状态机 `BUILDING→READY→STALE→DROPPED` 与命中统计 | 「索引」 |
| **约束** | 7 类：`NOT_NULL / UNIQUE / RANGE / ENUM / REGEX / CHECK_EXPR / REFERENCE`，分 `ENFORCED` 与 `WARN` | 「约束」 |
| **事件** | `INSERT / UPDATE / STATUS_CHANGE / RISK_HIGH` → `TAG / AUDIT / STATUS / NOTIFY / EVALUATE`，带重试与死信 | 「事件」 |
| **事务台账** | 每次写操作"被哪条约束放行、动了哪些索引、触发了哪些事件" | 「事件」页下方 |
| **查询台** | 类 SQL 只读查询 + **查询计划**（用了哪个索引 / 为什么全表扫） | 「查询台」 |

**铁律：所有对 Skill 的写操作都经过引擎**（编辑 / 归档 / 恢复 / 贡献发布）。
违反 `ENFORCED` 约束 → HTTP 422 + 约束 code + 修复建议；违反 `WARN` → 放行并记告警。

### 2. 在线实测（单点）

选 Skill + Agent（DeepSeek-flash）+ MCP + 一组超参 → 输入问题 →
得到 **输出 + 推理过程 + 指标分**，并与期望输出逐项对比。

### 3. 简单质量评估

只用**统计学与最优化**（加权评分、z-score、TOPSIS）：
任务效能 5 项（正确性 / 期望覆盖 / 内容完整 / 任务完成 / 一致性）+
通信性能 4 项（首字效率 / 时延效率 / 吞吐 / 成功率）。**不做**知识图谱与假设检验（那属专业版）。

### 4. 接入管理

Agent / MCP 注册、单键探活、一键体检；出网策略（代理 + 公司根证书）在页面自述。

## 这个版本**不**做什么

- ❌ 知识图谱 / 知识库 / 实体关系抽取 —— 属专业版 PSKH
- ❌ 实验矩阵（一个 Skill × 多 Agent × 多超参）—— 属专业版 PSKH
- ❌ 统计检验（t 检验）、聚类、回归 —— 属专业版 PSKH
- ❌ 登录鉴权（同简易版：`X-SKH-Actor` / `X-SKH-Role`）
- ❌ Docker / 镜像 / 部署交付

## 相对简易版新增的代码

```
backend/src/main/java/org/skillhub/
├─ business/  MsEngine（写门禁+索引+事件+台账）· MsAdminService · MsQueryService
│             EvalService（实测+评分+TOPSIS）· AccessService（注册+探活）· LocalSkillLookupProvider
├─ mapper/    MsMapper · EvalMapper
├─ model/     MsEntities（12 个嵌套实体）
├─ port/      AgentPort · McpPort
├─ adapter/   agent/{OpenAiCompatibleAgentAdapter, OfflineAgentAdapter, AgentRouter}
│             mcp/{BuiltinMcpAdapter, HttpMcpAdapter, McpRouter}
├─ config/    NetEgress（代理 + 动态根证书）· CertLocator · MsSeeder
├─ support/   ConstraintEvaluator · MsQueryParser · Metrics
└─ action/v1/ MsAction · EvalAction

frontend/src/
├─ api/       ms.js · eval.js
└─ views/     MsOverview · MsIndex · MsConstraint · MsEvent · MsQuery · LiveEval · Quality · Access
```

## 验收

```powershell
powershell -ExecutionPolicy Bypass -File scripts\verify.ps1   # 接口 40 项 + 页面 12 路由
node scripts\api-test.mjs
cd <母本仓库> ; node tools\verify-all.mjs          # 三系统一致性（需要三个工程同时在仓库里）
```

当前状态：接口 **40/40 通过**，页面 **12/12 路由通过**，三系统一致性 **30/30 通过**。

## 已知限制（诚实清单）

1. **不做登录**：`X-SKH-Role` 可任意伪造，不能当权限控制用。
2. **索引是"逻辑索引"**：真实数据存在 `ms_index_entry`，用于查询计划与命中统计；
   它不替代 MySQL 自己的 B-tree（数据库层仍有自己的索引）。
3. **查询台只读且只认白名单**：不支持 JOIN / 子查询 / 聚合分组。
4. **事件动作是进程内**：`NOTIFY` 只落日志，没有真正发通知。
5. **实测默认回退离线桩**：断网仍能跑通链路，但结果是桩数据（会标 `offline`）。
6. **质量评估只用统计与最优化**：这是刻意划定的边界，不是能力缺失。
7. **贡献仍是四步同一人完成**：没有独立审核人角色。
