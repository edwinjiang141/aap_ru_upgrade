# AAP 编排 Oracle DB RU 升级设计方案（V2，聚焦实施）

## 1. 范围说明（按你的要求收敛）

本方案 **只描述如何利用已有 AAP 能力实现 Oracle RU 升级编排与执行**，不包含以下内容：

- AAP 平台安装、容量、备份、升级、日常运维；
- AAP 平台账号体系/凭据平台建设；
- 客户内部变更平台（ITSM）流程设计。

默认前提（由客户提供）：

1. AAP 平台可用，且已创建可执行 Workflow/Job Template 的项目空间；
2. 变更平台可提供：变更号、变更描述、主机清单、窗口时间、审批状态；
3. 升级所需账户信息与凭据由客户变更平台/凭据系统托管，并可被 AAP 在运行时安全获取。

---

## 2. 目标（与 KPI 对齐）

1. **减少登录主机获取密码次数**：
   - 升级执行由 AAP 发起，DBA 不再逐台登录主机手工执行。
   - 人工侧“获取密码/输入密码”降为 0（执行日）。
2. **将手工 RU 升级改为 AAP 自动化执行**：
   - 用 Workflow 固化步骤、顺序、审批断点、失败处理。
   - 每次升级形成统一执行与审计记录。

---

## 3. 输入与触发模型（典型场景）

你给出的典型场景可直接映射为：

```text
客户变更平台发起变更
  -> 变更信息（变更号、内容、主机、步骤、窗口、审批状态）
  -> 触发 AAP 既有 Workflow
  -> Workflow 调用已编排的 RU 执行链路
  -> 回写执行结果（成功/失败、日志、关键检查项）
```

### 3.1 建议的最小输入字段

- `change_id`：变更号
- `change_title`：变更标题/内容摘要
- `target_hosts`：主机列表（RAC 节点）
- `ru_version`：目标 RU（如 19.28）
- `include_ojvm`：是否包含 OJVM
- `maintenance_window_start/end`：窗口时间
- `run_mode`：`precheck_only | full_upgrade | resume_from_step`
- `resume_step`：断点续跑步骤（可选）

---

## 4. 复用仓库现有能力（不改工具主逻辑）

### 4.1 可直接复用的 RU 工具

- `src/upgrade_ru_with_gold_image`
- `src/upgrade_ru_with_opatch`
- `src/ru_patch_number.ini`
- `Dbass_RU_生产升级步骤梳理.md`

### 4.2 策略

- Phase 1：AAP 负责编排与审计，Perl 工具继续做执行引擎；
- Phase 2：再逐步把通用动作 Ansible 原生化（非本轮范围）。

---

## 5. Workflow 设计（实施核心）

## 5.1 流程拓扑

```text
WF_oracle_ru_upgrade
  -> JT00_precheck
  -> JT01_prepare_dirs_cleanup
  -> JT02_backup_node2
  -> JT03_backup_node1
  -> JT04_unzip_gold_image
  -> JT05_collect_baseline
  -> Approval_A (是否进入切换)
  -> JT06_rolling_switch_node2
  -> JT07_rolling_switch_node1
  -> JT08_datapatch
  -> Approval_B (是否进入收尾)
  -> JT09_restore_links
  -> JT10_postcheck_compare
  -> JT11_finalize_and_export
  -> Notification
```

> 说明：仍按 RAC rolling 思路，先 node2 后 node1，避免全量中断。

### 5.2 每个 JT 的执行要点

- **JT00_precheck**
  - `sh precheck_db.sh`
  - `perl upgrade_ru_with_opatch --step_00_precheck`

- **JT01_prepare_dirs_cleanup**
  - 目录创建、历史残留清理（幂等处理）

- **JT02/JT03_backup_node**
  - `perl upgrade_ru_with_gold_image --step_00_backup --backup_dir=...`

- **JT04_unzip_gold_image**
  - `--step_00_unzip --unzip_switch_backup_image ...`

- **JT05_collect_baseline**
  - 软链接、`crsctl stat res -t`、tempfile 等基线采集

- **JT06/JT07_rolling_switch_node**
  - `srvctl stop instance`
  - `--step_52_switch_one_node_to_original_path_not_start_instance`
  - `srvctl start instance`

- **JT08_datapatch**
  - `job_queue_processes=0`
  - `--step_08_datapatch`
  - 参数恢复

- **JT09_restore_links**
  - 按基线/标准模板恢复软链接

- **JT10_postcheck_compare**
  - `check_version.sh/check_sqlpatch.sh/check_pdb.sh`
  - `crsctl` 前后对比

- **JT11_finalize_and_export**
  - 参数收口
  - 导出本次变更执行摘要（JSON/Markdown）

---

## 6. 与变更平台对接（重点）

> 这里不设计客户变更平台本身，只定义“如何消费变更信息并执行”。

### 6.1 对接方式

- **方式 A：Webhook 触发（推荐）**
  - 变更平台在“审批通过”时调用 AAP Workflow Launch API。
- **方式 B：人工在 AAP 发起**
  - 操作员把变更平台字段复制到 Survey 后启动。

### 6.2 参数映射（示例）

- `change_id` -> AAP extra_vars `change_id`
- `target_hosts` -> 动态 inventory limit/group
- `ru_version` -> 选择 `ru_patch_number.ini` 对应版本校验
- `run_mode/resume_step` -> 控制流程全量或断点续跑

### 6.3 凭据处理边界

- 凭据保存在客户变更平台/凭据系统；
- AAP 仅在作业运行时读取临时凭据并注入执行；
- 本方案不涉及凭据系统建设，仅约定“可被 AAP 安全获取”。

---

## 7. 失败处理与回滚建议

### 7.1 失败分支

任一关键节点失败后：

1. 自动采集失败上下文（stdout/stderr、主机状态、最后成功步骤）；
2. 输出“建议动作”：重试当前步骤 / 从断点续跑 / 进入回滚子流程；
3. 通知变更平台与值班组。

### 7.2 断点续跑

通过 `run_mode=resume_from_step` + `resume_step` 实现，避免整链路重跑。

---

## 8. 最小可落地版本（MVP）

建议先实现这 6 个 JT：

1. JT00_precheck
2. JT02_backup_node2
3. JT03_backup_node1
4. JT06_rolling_switch_node2
5. JT07_rolling_switch_node1
6. JT08_datapatch

先跑通“最短闭环”，再补充 unzip/baseline/收尾/报表。

---

## 9. 实施计划（3 个迭代）

### Iteration 1（1~2 周）
- 打通变更信息入参 + JT00/JT02/JT03；
- 跑通 precheck + backup 自动化。

### Iteration 2（1~2 周）
- 上线 JT06/JT07/JT08（rolling + datapatch）；
- 增加 Approval_A 与失败分支。

### Iteration 3（1 周）
- 补齐 JT04/JT05/JT09/JT10/JT11；
- 回写变更平台执行摘要，形成标准升级报告。

---

## 10. 下一轮建议先确认 4 件事

1. 变更平台到 AAP 的触发方式：Webhook 还是人工 Survey？
2. 主机清单格式：主机名列表还是预定义主机组？
3. 凭据获取方式：每次变更动态下发，还是按系统预关联凭据引用？
4. 首批上线范围：先 precheck+backup，还是直接 rolling 全流程？

---

## 11. 参考（用于实施对齐）

- Red Hat AAP 2.4 Automation Controller 用户文档（Workflow、Survey、Approval、Notification、API）。
- Oracle OPatch/Datapatch 相关官方文档（用于步骤约束与执行顺序校验）。


---

## 12. 开发计划与工作量评估（仅实施方，不含客户侧）

> 前提：
> - 继续复用现有仓库 RU 工具（`upgrade_ru_with_opatch`、`upgrade_ru_with_gold_image`），不做 Ansible 原生化改造；
> - 评估单位为“人天（PD）”；
> - 客户侧工作量不计入总 PD，仅标注“客户协助点”。

### 12.1 WBS（实施方）

| WBS | 工作项 | 主要产出 | 预估工作量（PD） |
|---|---|---|---:|
| 1 | 需求澄清与流程冻结 | 字段清单、流程图、边界清单 | 1.5 |
| 2 | Inventory/变量模型设计 | 主机分组、节点角色、变量规范 | 1.0 |
| 3 | Workflow 模板开发（JT00~JT11） | Workflow + Job Template 初版 | 3.0 |
| 4 | 脚本封装 Playbook 开发 | 调用 Perl 工具的任务封装、参数校验、日志采集 | 4.0 |
| 5 | 变更平台入参映射 | Webhook/API/Survey 入参映射与校验 | 2.0 |
| 6 | 失败分支与断点续跑 | fail 分支、resume_from_step、错误码规范 | 2.0 |
| 7 | 报告与回写 | 执行摘要（JSON/Markdown）、结果回传接口适配 | 1.5 |
| 8 | SIT/UAT 联调支持 | 联调问题修复、脚本参数调优 | 3.0 |
| 9 | 上线演练与切换支持 | 灰度演练、首批变更护航 | 2.0 |
| 10 | 文档与交付 | 运维手册、Runbook、交接材料 | 1.5 |

**实施方合计：21.5 PD**

建议按角色拆分（可并行）：

- 自动化工程师（主）：13.0 PD
- Oracle DBA（脚本参数/校验支持）：5.0 PD
- DevOps/接口工程师（Webhook/API 对接）：2.0 PD
- 项目管理与发布协同：1.5 PD

### 12.2 里程碑排期（建议 4 周）

- **第 1 周（设计冻结）**：WBS 1~2，完成流程冻结与变量模型；
- **第 2 周（开发）**：WBS 3~5，完成 Workflow/JT 和变更入参映射；
- **第 3 周（增强）**：WBS 6~7，完成失败分支、断点续跑、执行摘要；
- **第 4 周（联调上线）**：WBS 8~10，完成 UAT、演练、上线与文档交付。

### 12.3 不确定性与缓冲

建议设置 **15% 风险缓冲（约 3.2 PD）**，用于：

- 生产环境差异导致的脚本参数适配；
- 变更平台字段不一致/回写接口变更；
- 首次生产窗口中的临时策略调整。

**含缓冲总量：约 24.7 PD（可按 25 PD 管控）。**

---

## 13. 客户协助点（仅标注，不计入实施方工作量）

1. 提供可用的 AAP 项目空间、可执行入口与网络连通性；
2. 提供变更平台到 AAP 的触发能力（Webhook/API）与字段定义；
3. 提供凭据系统到 AAP 的安全取数通道（运行时注入）；
4. 提供目标主机清单、维护窗口和审批策略；
5. 组织 UAT/生产演练窗口，安排业务侧观察与验收。

