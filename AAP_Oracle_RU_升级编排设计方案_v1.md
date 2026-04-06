# AAP 编排 Oracle DB RU 升级设计方案（V1，讨论稿）

## 1. 背景与目标

当前 Oracle RU 升级流程以人工登录主机+手工执行为主，存在以下问题：

- 登录次数多，重复获取/输入密码，影响运维 KPI（登录与凭据使用频次）。
- 跨节点、跨用户（root/grid/oracle）流程长，人工串行执行容易漏步。
- 审批、审计、回滚证据分散，缺少统一执行轨迹。

本方案以 **Red Hat Ansible Automation Platform (AAP) 2.4 Automation Controller** 作为统一编排层，复用仓库已有 RU 工具（`upgrade_ru_with_opatch`、`upgrade_ru_with_gold_image`）实现“可审计、可回放、可灰度”的自动化升级。

---

## 2. 设计原则

1. **最小登录面**：
   - 人员不再 SSH 到 DB 主机执行升级。
   - 通过 AAP Job/Workflow 发起，凭据由平台托管与注入。
2. **复用现有工具链**：
   - 保留当前 Perl 工具与步骤语义，先“编排化”，后“重构化”。
3. **可中断可恢复**：
   - 关键阶段拆分为可重跑节点（idempotent/可跳过步骤）。
4. **生产安全优先**：
   - 变更前后都做基线采集；引入 Approval Gate 和超时策略。
5. **合规可审计**：
   - AAP Activity Stream + Job stdout + 结构化工单变量，满足追踪。

---

## 3. 参考输入（来自仓库现状）

### 3.1 已有升级步骤可直接映射为编排节点

仓库文档已经把生产 RU 流程拆为 11 个阶段，可直接映射为 Workflow Node：

1. precheck
2. prepare_dirs_and_cleanup
3. backup_node
4. goldimage_distribute_and_unzip
5. collect_baseline
6. rolling_switch_node2
7. rolling_switch_node1
8. datapatch
9. restore_links
10. postcheck_and_compare
11. final_parameter_tuning

这为 AAP Workflow 拆分提供了天然边界。

### 3.2 现有脚本能力可作为“执行引擎”

- `upgrade_ru_with_gold_image`：已覆盖备份、unzip、switch、datapatch 等关键动作。
- `upgrade_ru_with_opatch`：保留 opatch 路径下 precheck/补丁逻辑。
- `ru_patch_number.ini`：可作为版本参数库（RU/OJVM patch id、opatch 最低版本、sha256）。

结论：**短期不改脚本主体，先做 AAP 封装与流程治理。**

---

## 4. AAP 目标架构（逻辑）

```text
ITSM/人工触发
   -> AAP Workflow Template: oracle_ru_upgrade_workflow
      -> JT00 precheck
      -> JT01 prepare
      -> JT02 backup (node2,node1)
      -> JT03 unzip_gold_image
      -> JT04 collect_baseline
      -> Approval Gate A（进入切换）
      -> JT05 rolling_switch_node2
      -> JT06 rolling_switch_node1
      -> JT07 datapatch
      -> Approval Gate B（进入收尾）
      -> JT08 restore_links
      -> JT09 postcheck_compare
      -> JT10 finalize
      -> Notification (success/fail)
```

补充：
- 按生产窗口可配置 schedule（UTC 注意事项）。
- 节点失败自动转入“失败分支”：采集日志 + 发通知 + 触发回滚建议任务（可选）。

---

## 5. 凭据与“减少登录次数”专项设计

这是本需求的核心 KPI，建议分 3 层落地：

### 5.1 人员层：禁用人工直登升级

- 生产 RU 执行入口统一为 AAP Workflow Launch。
- 操作员仅有 `execute` 权限（可跑流程，不可查看明文凭据）。
- root/grid/oracle 密码不再在升级时由人手工输入。

### 5.2 平台层：统一凭据编排

- 使用 AAP Credential 管理：
  - Machine Credential（SSH key 优先，次选密码）
  - Become 凭据（必要时）
  - Vault/外部密钥系统凭据
- 对接 AAP External Credential Plugin：
  - HashiCorp Vault Secret Lookup
  - HashiCorp Vault Signed SSH（推荐，短时证书，降低密钥长期暴露）
- 运行时动态注入，不在 playbook 明文落盘。

### 5.3 流程层：减少重复鉴权/切换

- Job Template 内通过 `become_user` 切换 root/grid/oracle，避免人工反复 `su -`。
- 单节点内步骤尽量合并为一次连接会话的任务块（减少连接建立与 PAM 认证）。
- 对 node2/node1 采用顺序控制（serial=1）保证滚动，避免并发竞争。

**KPI 建议量化口径**：
- 手工登录次数：从“每次升级每节点多次登录”降到“0 次（执行日）”。
- 人员获取密码次数：从“每阶段一次或多次”降到“0 次（执行日）”。
- 凭据访问次数：以 AAP 审计日志统计“机器调用次数”，替代“人工查看次数”。

---

## 6. Workflow 详细编排建议（首版）

### 6.1 Workflow 输入参数（Survey）

建议在 Workflow Survey 统一收集：

- 变更单号 / 批次号
- RU 目标版本（如 19.28）
- 是否含 OJVM
- backup_dir（node1/node2）
- gold image 路径（grid/db）
- 目标 HOME 路径（grid/db）
- 运行模式（`precheck_only` / `full_upgrade` / `resume_from_step`）
- 是否启用回滚预案任务

### 6.2 Job Template 与脚本步骤映射

- **JT00_precheck**
  - 执行：`precheck_db.sh` + `upgrade_ru_with_opatch --step_00_precheck`
  - 输出：环境检查报告（版本、补丁、连通性）

- **JT01_prepare_dirs_cleanup**
  - 执行：目录准备 + 残留清理（幂等化）

- **JT02_backup_node2 / JT02_backup_node1**
  - 执行：`upgrade_ru_with_gold_image --step_00_backup --backup_dir=...`

- **JT03_unzip_gold_image**
  - 执行：`--step_00_unzip --unzip_switch_backup_image ...`

- **JT04_collect_baseline**
  - 执行：软链接基线、`crsctl stat res -t`、tempfile 检查
  - 输出：统一归档到作业 artifact 路径

- **Approval Gate A**
  - 人工确认“基线通过 + 维护窗口开始”

- **JT05_rolling_switch_node2**
  - 执行：stop instance -> `--step_52_switch...` -> start instance

- **JT06_rolling_switch_node1**
  - 同上

- **JT07_datapatch**
  - 执行：`job_queue_processes` 降低 -> `--step_08_datapatch` -> 恢复

- **Approval Gate B**
  - 人工确认是否进入最终收尾（适合高风险窗口）

- **JT08_restore_links**
  - 按基线或标准模板恢复软链接

- **JT09_postcheck_compare**
  - `check_version.sh/check_sqlpatch.sh/check_pdb.sh`
  - `diff before/after crsctl`

- **JT10_finalize**
  - 参数收口（如 `_adg_parselock_timeout`）
  - 输出“升级完成报告（JSON/Markdown）”

### 6.3 失败分支与恢复策略

- 任一关键节点失败：
  1. 自动保存失败上下文（stdout、stderr、主机状态）。
  2. 通知 DBA 值班组（Email/Slack/Webhook）。
  3. 可选进入“回滚工作流子流程”（以审批节点控制）。

---

## 7. AAP 平台规范落地建议

### 7.1 RBAC 分层

- **Platform Admin**：维护凭据、执行环境、全局设置
- **DBA Automation Admin**：维护模板/工作流
- **Operator**：仅执行权限（Launch + 查看结果）
- **Auditor**：只读审计（Activity Stream + Job 历史）

### 7.2 Execution Environment（EE）标准化

为避免“控制节点环境漂移”，建议自定义 EE：

- 基础：AAP 支持的最小 EE 镜像
- 增加：
  - `oracle` 相关命令依赖（按实际）
  - `community.general` 等所需 collection
  - 企业 CA 证书
- 通过 private automation hub 管理版本（`oracle-ru-ee:v1/v2`）

### 7.3 审批与通知

- 在切换前和收尾前设置 Approval Node。
- 配置通知模板：
  - start/success/fail 触发到 Email/Slack/Webhook。
- 通知中附带：变更号、节点、失败步骤、作业 URL。

### 7.4 审计与报表

- 利用 Activity Stream 记录“谁在何时执行了什么”。
- 产出每次升级审计包：
  - 输入参数快照
  - 每节点执行日志
  - 前后对比报告
  - 失败重试/审批记录

---

## 8. 与 Oracle RU 工具的衔接建议

### 8.1 第一阶段（低风险）：命令封装

- AAP Playbook 只负责：
  - 参数校验
  - 远程命令执行
  - 结果收集
- Perl 脚本保持原样，降低改造风险。

### 8.2 第二阶段（增强）：模块化改造

- 将常用动作（stop/start、基线采集、datapatch 控制）逐步 Ansible 化。
- 把 `ru_patch_number.ini` 转为结构化变量（YAML/CMDB）以便动态校验。

### 8.3 第三阶段（平台化）：事件驱动与全自动

- 对接 ITSM（变更单审批通过自动触发 Workflow）。
- EDA（Event-Driven Ansible）接入维护窗口事件（可选）。

---

## 9. 实施路径（建议 4 周）

### 第 1 周：梳理与 PoC
- 固化 Inventory 分组（rac_node1/rac_node2）。
- 完成 JT00~JT04（precheck 到 baseline）。
- 接通 Vault 凭据读取。

### 第 2 周：核心滚动升级
- 完成 JT05~JT07（双节点切换 + datapatch）。
- 加入 Approval Gate、失败分支。

### 第 3 周：收尾与审计
- 完成 JT08~JT10。
- 完成通知模板、审计报表脚本。

### 第 4 周：演练与上线
- UAT 演练（至少 2 轮：正常/失败回滚场景）。
- 编写 Runbook 与值班手册。
- 生产灰度上线（先一套低风险集群）。

---

## 10. 风险与控制

- **风险 1：脚本非幂等导致重试副作用**
  - 控制：步骤级状态文件 + `resume_from_step` 参数 + 人工审批断点。

- **风险 2：多用户切换权限差异（root/grid/oracle）**
  - 控制：统一 sudoers 基线 + AAP credential 测试模板。

- **风险 3：补丁介质/版本不一致**
  - 控制：预检查校验 sha256 与 `ru_patch_number.ini`。

- **风险 4：生产窗口超时**
  - 控制：节点级超时、并行前置检查、关键动作秒级通知。

---

## 11. 我建议的“渐进讨论顺序”

为了快速进入可执行状态，建议我们下一轮先定这 5 个问题：

1. 你们当前 AAP 是 2.4 还是更高版本？控制面是 VM 安装还是 OpenShift 安装？
2. 凭据源准备选哪种：AAP 内置凭据、HashiCorp Vault、还是企业 PAM（如 CyberArk）？
3. 生产 RAC 升级是固定两节点滚动，还是存在 3+ 节点/Data Guard 联动？
4. 你们是否接受在“切换前/切换后”增加 Approval Gate（人工确认点）？
5. 首批上线范围：先从 `precheck+backup` 半自动开始，还是直接上全链路？

---

## 12. 外部规范与实践参考（用于方案对齐）

- AAP Automation Controller Workflow、Survey、Approval Node、Schedules、Notifications、RBAC、Activity Stream、Credential Plugin（官方文档）。
- AAP External Credential 对接 HashiCorp Vault（Secret Lookup / Signed SSH）用于动态凭据。
- Oracle OPatchAuto 官方实践：RAC rolling/non-rolling、前置检查、备份、datapatch 后处理。

> 注：以上参考用于“控制平面规范”和“数据库补丁执行约束”双线对齐，确保方案既符合 AAP 平台治理，也不偏离 Oracle RU 官方维护逻辑。

