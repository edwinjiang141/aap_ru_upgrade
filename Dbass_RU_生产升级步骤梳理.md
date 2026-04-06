# Dbass RU 生产升级步骤梳理（含命令版）

> 来源：`DbassRU实施报告.docx`。保留实施步骤与执行命令，去除日志与命令结果输出。

## 0. 变量约定（建议先统一）
```bash
# 通用变量
export PATCH_BASE=/u01/patch1928
export TOOL_DIR=/u01/patch1928/ru.20241125

# 节点备份目录
export BACKUP_DIR_NODE1=/u01/patch1928/rac1_backup1928
export BACKUP_DIR_NODE2=/u01/patch1928/rac2_backup1928

# Gold Image 路径（按现场实际文件名调整）
export GRID_IMAGE=/u01/patch1928/gimage/grid_home_2025-09-14_10-35-23AM.zip
export DB_IMAGE=/u01/patch1928/gimage/db_home_2025-09-14_10-46-27AM.zip

# 目标 Home（与报告保持一致）
export TARGET_GRID_HOME=/u01/app/19.0.0.0/grid2
export TARGET_DB_HOME=/u01/app/oracle/product/19.0.0.0/dbhome_2
```

## 1. 升级前准备
### 1.1 执行数据库预检查
```bash
# 示例：按报告中的检查脚本执行（遍历实例后 sqlplus 检查）
sh precheck_db.sh
```

### 1.2 各节点创建目录（Node1/Node2 均执行）
```bash
mkdir -p /u01/patch1928/rac1_backup1928   # Node1
mkdir -p /u01/patch1928/rac2_backup1928   # Node2
mkdir -p /u01/patch1928/gimage
cp -r /u01/patch1926/ru.20241125 /u01/patch1928/
```

### 1.3 清理上次升级遗留
```bash
rm -rf /u01/patch1926
rm -rf /u01/app/oracle/product/19.0.0.0/dbhome_1.backup.for.switch_back
rm -rf /u01/app/19.0.0.0/grid.backup.for.switch_back
```

## 2. 备份当前运行环境
### 2.1 执行 RU 工具预检查
```bash
cd ${TOOL_DIR}
perl upgrade_ru_with_opatch --step_00_precheck
```

### 2.2 执行节点备份
```bash
# Node1
cd ${TOOL_DIR}
perl upgrade_ru_with_gold_image --step_00_backup --backup_dir=${BACKUP_DIR_NODE1}

# Node2
cd ${TOOL_DIR}
perl upgrade_ru_with_gold_image --step_00_backup --backup_dir=${BACKUP_DIR_NODE2}
```

## 3. 准备并解压 Gold Image
### 3.1 分发 image 压缩包
```bash
# 示例：从制品机拷贝到生产节点
scp *.zip <target-host>:/u01/patch1928/gimage
```

### 3.2 root 用户执行解压（各节点）
```bash
cd ${TOOL_DIR}
perl upgrade_ru_with_gold_image \
  --step_00_unzip \
  --unzip_switch_backup_image \
  --target_grid_home=${TARGET_GRID_HOME} \
  --target_oracle_home=${TARGET_DB_HOME} \
  --grid_image=${GRID_IMAGE} \
  --db_image=${DB_IMAGE}
```

### 3.3 解压后基础校验
```bash
ls -ld /u01/app/19.0.0.0 /u01/app/oracle/product/19.0.0.0
ls -l ${TARGET_GRID_HOME} ${TARGET_DB_HOME}
```

## 4. 节点切换前状态采集
### 4.1 记录软链接（Node1/Node2）
```bash
cd /u01/app/19.0.0.0/grid
find ./ -type l -exec ls -l {} \; > /tmp/grid_symlink_before.log

cd /u01/app/oracle/product/19.0.0.0/dbhome_1
find ./ -type l -exec ls -l {} \; > /tmp/db_symlink_before.log
```

### 4.2 保存集群状态
```bash
su - grid -c 'crsctl stat res -t > ~/crsctl_stat_before.log'
```

### 4.3 检查 PDB tempfile
```bash
su - oracle
sh check_tempfile.sh
```

## 5. Rolling 升级执行（先 Node2，后 Node1）
### 5.1 升级 Node2
```bash
# 停实例
srvctl stop instance -node cnifcxd001dbadm02 -f

# 执行切换（root）
cd ${TOOL_DIR}
/usr/bin/perl upgrade_ru_with_gold_image \
  --step_52_switch_one_node_to_original_path_not_start_instance \
  --target_grid_home=${TARGET_GRID_HOME} \
  --target_oracle_home=${TARGET_DB_HOME}

# 启实例
srvctl start instance -node cnifcxd001dbadm02
```

### 5.2 升级 Node1
```bash
# 停实例
srvctl stop instance -node cnifcxd001dbadm01 -f

# 执行切换（root）
cd ${TOOL_DIR}
/usr/bin/perl upgrade_ru_with_gold_image \
  --step_52_switch_one_node_to_original_path_not_start_instance \
  --target_grid_home=${TARGET_GRID_HOME} \
  --target_oracle_home=${TARGET_DB_HOME}

# 启实例
srvctl start instance -node cnifcxd001dbadm01
```

## 6. 升级后执行 datapatch
### 6.1 降低作业并发（job_queue_processes=0）
```bash
# 可通过脚本批量执行
sh set_pars.sh   # 脚本内执行: alter system set job_queue_processes=0;
```

### 6.2 执行 datapatch
```bash
cd ${TOOL_DIR}
perl upgrade_ru_with_gold_image --step_08_datapatch
```

### 6.3 恢复作业并发（job_queue_processes=160）
```bash
# 可通过脚本批量执行
sh set_pars.sh   # 脚本内执行: alter system set job_queue_processes=160;
```

## 7. 升级后环境修复与核查
### 7.1 按记录恢复软链接
```bash
# Oracle Home（示例）
cd /u01/app/oracle/product/19.0.0.0/dbhome_1
mkdir -p ./suptools/oracle.ahf/bin/
mkdir -p ./suptools/oracle.ahf/common/venv/bin/
ln -s /opt/oracle.ahf/exachk/exachk ./suptools/oracle.ahf/bin/exachk
ln -s /opt/oracle.ahf/python/bin/python3 ./suptools/oracle.ahf/common/venv/bin/python3
ln -s /usr/openv/netbackup/bin/libobk.so64 ./libobk.so64

# Grid Home（所有节点）
cd /u01/app/19.0.0.0/grid
ln -s /u01/app/19.0.0.0/grid/crs/utl/cmdllroot.sh ./crs/install/cmdllroot.sh
ln -s /opt/oracle.ahf/ ./suptools/oracle.ahf
ln -s /opt/oracle.ahf/tfa/bin/tfactl ./bin/tfactl
```

### 7.2 数据库健康检查
```bash
cd /home/oracle/odas
sh check_version.sh
sh check_sqlpatch.sh
sh check_pdb.sh | grep -v SEED
```

### 7.3 清理中间目录
```bash
rm -rf /u01/app/oracle/product/19.0.0.0/dbhome_1.backup.for.switch_back
rm -rf /u01/app/19.0.0.0/grid.backup.for.switch_back
```

### 7.4 集群状态复核
```bash
su - grid -c 'crsctl stat res -t > ~/after_upgrade.log'
su - grid -c 'diff ~/after_upgrade.log ~/crsctl_stat_before.log'
```

## 8. 参数收口
```bash
# 可通过脚本批量执行
sh set_pars.sh
# 脚本内核心语句：
# alter system set "_adg_parselock_timeout"=550 sid='*';
```

## 9. AAP 自动化编排建议（带命令映射）
1. `precheck`：`sh precheck_db.sh`
2. `prepare_dirs_and_cleanup`：`mkdir/cp/rm` 初始化与清理
3. `backup_node`：`perl upgrade_ru_with_gold_image --step_00_backup`
4. `goldimage_distribute_and_unzip`：`scp` + `--step_00_unzip`
5. `collect_baseline`：`find -type l` + `crsctl stat res -t` + tempfile 检查
6. `rolling_switch_node2`：`srvctl stop/start` + `--step_52...`
7. `rolling_switch_node1`：`srvctl stop/start` + `--step_52...`
8. `datapatch`：`job_queue_processes=0` + `--step_08_datapatch` + 参数恢复
9. `restore_links`：按基线恢复软链接
10. `postcheck_and_compare`：版本、sqlpatch、pdb、cluster diff
11. `final_parameter_tuning`：`_adg_parselock_timeout` 收口

> 建议每个 stage 固化：前置校验、执行命令、成功判定、失败回滚。
