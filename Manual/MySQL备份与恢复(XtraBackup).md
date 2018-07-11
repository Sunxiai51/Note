# MySQL备份与恢复(XtraBackup)

## 1. 版本

- SUSE Linux Enterprise Server 11 (x86_64)
- Percona XtraBackup 2.4.1
- MySQL 5.7.22

## 2. 策略

XtraBackup对MySQL的备份提供支持，通过编写脚本对其具体的命令进行封装，方便于按固定周期调用以实现定时备份。

编写了以下4个脚本：

1. **全量备份脚本 (full_backup.sh)**：进行一次全量备份，并删除之前所有的备份
2. **增量备份脚本 (incremental_backup.sh)**：基于最近一次的备份进行一次增量备份
3. **Prepare脚本 (backup_prepare.sh)**：用于数据恢复前的prepare
4. **恢复脚本 (backup_restore.sh)**：用于从备份恢复数据

脚本调用有以下要求：

- 执行增量备份脚本时，必须至少存在一个未prepare的全量备份
- 执行Prepare脚本时，必须至少存在一个全量备份
- Prepare脚本只应该在执行恢复脚本之前执行，执行Prepare脚本前，请确保所有的备份行为已经停止，执行Prepare脚本后，一般会立刻执行恢复脚本
- 恢复脚本必须紧接着Prepare脚本执行，执行恢复脚本后，建议立刻执行一次全量备份脚本以清除之前prepare过的备份

执行脚本前，请首先阅读脚本开始部分的说明，并检查参数配置。

## 3. 备份与恢复脚本

### 3.1 全量备份脚本

```shell
#!/bin/bash
set -o errexit

# 全量备份脚本
#
# 通过xtrabackup对ewallet相关数据库进行全量备份，包括：
#  ewallet_mycat_0
#  ewallet_mycat_1
#  ewallet_mycat_2
#  ewallet_mycat_3
#  ewallet_mycat_default
#
# 执行前请检查以下参数配置
# ------------------ 参数配置 start ------------------

readonly backup_path="/home/ewallet/backups/mysql/" # 备份目录，请务必以“/”结尾
readonly xtrabackup_path="/usr/local/xtrabackup/bin/xtrabackup" # 本机xtrabackup路径

readonly mysql_datadir="/home/ewallet/soft/mysql/data/" # MySQL datadir真实目录(不能为链接)，请务必以“/”结尾
readonly mysql_user="root"
readonly mysql_password="123456"

# ------------------- 参数配置 end -------------------
#
# 该脚本将在${backup_path}下创建全量备份目录与本次备份日志文件
# 全量备份目录名为前缀“full_backup_”+备份时间戳，备份日志文件名与备份目录名相同，后缀为“.log”，例如，2018-7-6 10:46:17创建的全量备份目录为“full_backup_20180706104617”，日志文件为“full_backup_20180706104617.log”
#
# 全量备份完成后，将删除${backup_path}下之前的所有备份目录（通过对比时间戳的方式）

time_label="$(date '+%Y%m%d%H%M%S')"
dir_name="full_backup_${time_label}" # 备份文件夹
logfile_name="full_backup_${time_label}.log" # 日志文件名

# 判断是否存在备份目录
if [ ! -d ${backup_path} ]; then
  echo "Error: directory is not exists: ${backup_path}"
  exit 1
fi

# 创建备份文件夹
echo -e "[Note] Create backup directory ${backup_path}${dir_name}" >> ${backup_path}${logfile_name}
mkdir "${backup_path}${dir_name}"

# 执行备份
echo -e "[Note] Start to execute backup..." >> ${backup_path}${logfile_name}
echo -e "[Note] execute ${xtrabackup_path}\n  --datadir=${mysql_datadir}\n  --user=${mysql_user}\n  --password=${mysql_password}\n  --host=127.0.0.1\n  --backup\n  --target-dir=${backup_path}${dir_name}\n  --tables=\"^ewallet_mycat_([0-3]|default)[.].*\"" >> ${backup_path}${logfile_name}
${xtrabackup_path} --datadir=${mysql_datadir} --user=${mysql_user} --password=${mysql_password} --host=127.0.0.1 --backup --target-dir=${backup_path}${dir_name} --tables="^ewallet_mycat_([0-3]|default)[.].*"

# 备份完成，打印输出
echo -e "[Note] full backup success:\n  target-dir=${backup_path}${dir_name}\n  logfile=${backup_path}${logfile_name}" >> ${backup_path}${logfile_name}

# 全量备份成功后删除之前的备份
echo -e "[Note] ready to delete backup before ${time_label}" >> ${backup_path}${logfile_name}
for pre_backup_dir in `ls ${backup_path} | tr ' ' '\n' | grep -E "^(full|incr)_backup_[0-9]+$"`
do
	if [[ ${pre_backup_dir##*_} < ${time_label} ]];then
		echo "  delete ${backup_path}${pre_backup_dir}" >> ${backup_path}${logfile_name}
		rm -rf "${backup_path}${pre_backup_dir}"
		echo "  delete ${backup_path}${pre_backup_dir}.log" >> ${backup_path}${logfile_name}
		rm -rf "${backup_path}${pre_backup_dir}.log"
	fi
done

exit $?
```

### 3.2 增量备份脚本

```shell
#!/bin/bash
set -o errexit

# 增量备份脚本
#
# 通过xtrabackup对ewallet相关数据库进行增量备份，包括：
#  ewallet_mycat_0
#  ewallet_mycat_1
#  ewallet_mycat_2
#  ewallet_mycat_3
#  ewallet_mycat_default
# 基于最近一次的备份为基准进行增量备份，可以是基于全量备份，也可以是基于上一次的增量备份
#
# 执行前请检查以下参数配置
# ------------------ 参数配置 start ------------------

readonly backup_path="/home/ewallet/backups/mysql/" # 备份目录，请务必以“/”结尾
readonly xtrabackup_path="/usr/local/xtrabackup/bin/xtrabackup" # 本机xtrabackup路径

readonly mysql_datadir="/home/ewallet/soft/mysql/data/" # MySQL datadir真实目录(不能为链接)，请务必以“/”结尾
readonly mysql_user="root"
readonly mysql_password="123456"

# ------------------- 参数配置 end -------------------
#
# 该脚本将在${backup_path}下创建增量备份目录与本次备份日志文件
# 增量备份目录名为前缀“incr_backup_”+备份时间戳，备份日志文件名与备份目录名相同，后缀为“.log”，例如，2018-7-6 10:46:17创建的全量备份目录为“incr_backup_20180706104617”，日志文件为“incr_backup_20180706104617.log”
#
# xtrabackup需要基于某一次备份作为basedir来创建增量备份，该脚本将选择${backup_path}下备份目录名根据字符串排序最大的一个备份作为本次增量备份的basedir

time_label="$(date '+%Y%m%d%H%M%S')"
dir_name="incr_backup_${time_label}" # 备份文件夹
logfile_name="incr_backup_${time_label}.log" # 日志文件名

# 判断是否存在备份目录
if [ ! -d ${backup_path} ]; then
  echo "Error: directory is not exists: ${backup_path}"
  exit 1
fi

# 选择上次备份的目录作为增量备份base目录
last_dir=`ls ${backup_path} | tr ' ' '\n' | grep -E "^(full|incr)_backup_[0-9]+$" | sort | tail -n1`
if [ ! -f "${backup_path}${last_dir}/xtrabackup_checkpoints" ];then
  echo "Error: file 'xtrabackup_checkpoints' is not found in the last backup directory ${backup_path}${last_dir}"
  exit 2
fi

# 创建备份文件夹
echo -e "[Note] Create backup directory ${backup_path}${dir_name}" >> ${backup_path}${logfile_name}
mkdir "${backup_path}${dir_name}"

# 执行备份
echo -e "[Note] Start to execute backup..." >> ${backup_path}${logfile_name}
echo -e "[Note] execute ${xtrabackup_path}\n  --datadir=${mysql_datadir}\n  --user=${mysql_user}\n  --password=${mysql_password}\n  --host=127.0.0.1\n  --backup\n  --target-dir=${backup_path}${dir_name}\n  --tables=\"^ewallet_mycat_([0-3]|default)[.].*\"\n  --incremental-basedir=${backup_path}${last_dir}" >> ${backup_path}${logfile_name}
${xtrabackup_path} --datadir=${mysql_datadir} --user=${mysql_user} --password=${mysql_password} --host=127.0.0.1 --backup --target-dir=${backup_path}${dir_name} --tables="^ewallet_mycat_([0-3]|default)[.].*" --incremental-basedir=${backup_path}${last_dir}

# 备份完成，打印输出
echo -e "[Note] incremental backup success:\n  incremental-basedir=${backup_path}${last_dir}\n  target-dir=${backup_path}${dir_name}\n  logfile=${backup_path}${logfile_name}" >> ${backup_path}${logfile_name}

exit $?
```

### 3.3 Prepare脚本

```shell
#!/bin/bash
set -o errexit

# 执行xtrabackup prepare
#
# 通过xtrabackup对最近一次备份进行恢复前的准备操作
# 注意：该脚本执行后可能导致后续的增量备份丢失数据，故该脚本只应该在恢复数据前执行，执行后请立即恢复数据，并在恢复数据后的下一次有效备份只能为全量备份
#
# 执行前请检查以下参数配置
# ------------------ 参数配置 start ------------------

readonly backup_path="/home/ewallet/backups/mysql/" # 备份目录，请务必以“/”结尾
readonly xtrabackup_path="/usr/local/xtrabackup/bin/xtrabackup" # 本机xtrabackup路径

# ------------------- 参数配置 end -------------------
#
# 该脚本将根据上次备份类型选择prepare策略
# 如果上次备份为全量备份，直接prepare该全量备份两次
# 如果上次备份为增量备份，依次以--apply-log-only模式prepare离上次全量备份较近的增量备份，最后prepare上次全量备份两次
# 例如，按照时间顺序有以下备份(F表示全量，I表示增量)：F1--I2--I3--I4，执行prepare顺序为：I2--I3--I4--F1

# 上次全量备份的目录
last_full_backup_dir=`ls ${backup_path} | tr ' ' '\n' | grep -E "^full_backup_[0-9]+$" | sort | tail -n1`
if [ ! -f "${backup_path}${last_full_backup_dir}/xtrabackup_checkpoints" ];then
  echo "File 'xtrabackup_checkpoints' is not found in the last backup directory ${last_full_backup_dir}"
  exit 2
fi

# 上次备份的目录
last_dir=`ls ${backup_path} | tr ' ' '\n' | grep -E "^(full|incr)_backup_[0-9]+$" | sort | tail -n1`
if [ ! -f "${backup_path}${last_dir}/xtrabackup_checkpoints" ];then
  echo "File 'xtrabackup_checkpoints' is not found in the last backup directory ${backup_path}${last_dir}"
  exit 2
fi

# 根据上次备份类型选择prepare策略，如果上次备份为增量备份，需要依次prepare
if [ "${last_full_backup_dir}" != "${last_dir}" ];then
  # 上次备份为增量备份，依次prepare
  ${xtrabackup_path} --prepare --apply-log-only --target-dir=${backup_path}${last_full_backup_dir}
  for pre_backup_dir in `ls ${backup_path} | tr ' ' '\n' | grep -E "^incr_backup_[0-9]+$" | sort `
  do
    if [[ ${pre_backup_dir##*_} > ${last_full_backup_dir##*_} ]];then
      ${xtrabackup_path} --prepare --apply-log-only --target-dir=${backup_path}${last_full_backup_dir} --incremental-dir=${backup_path}${pre_backup_dir}
    fi
  done
fi

# 执行两次prepare
${xtrabackup_path} --prepare --target-dir=${backup_path}${last_full_backup_dir}
${xtrabackup_path} --prepare --target-dir=${backup_path}${last_full_backup_dir}

# 完成，打印输出
echo -e "backup prepare success:\n  target-dir=${backup_path}${last_full_backup_dir}\n"

exit $?
```

### 3.4 恢复脚本

```shell
#!/bin/bash
set -o errexit

# 恢复MySQL数据
#
# 对xtrabackup最近一次备份进行恢复，恢复前需要首先执行backup_prepare.sh
# 恢复后建议立刻进行一次全量备份
#
# 执行前请检查以下参数配置
# ------------------ 参数配置 start ------------------

readonly backup_path="/home/ewallet/backups/mysql/" # 备份目录，请务必以“/”结尾
readonly mysql_datadir="/home/ewallet/soft/mysql/data/" # MySQL datadir真实目录(不能为链接)，请务必以“/”结尾

# ------------------- 参数配置 end -------------------
#
# 通过rsync将上次全量备份目录中的数据同步至${mysql_datadir}

# 上次全量备份的目录
last_full_backup_dir=`ls ${backup_path} | tr ' ' '\n' | grep -E "^full_backup_[0-9]+$" | sort | tail -n1`
if [ ! -f "${backup_path}${last_full_backup_dir}/xtrabackup_checkpoints" ];then
  echo "File 'xtrabackup_checkpoints' is not found in the last backup directory ${last_full_backup_dir}"
  exit 2
fi

# 同步数据
rsync -rvt --exclude 'xtrabackup_checkpoints' --exclude 'xtrabackup_logfile' ${backup_path}${last_full_backup_dir}/ ${mysql_datadir}
chown -R mysql:mysql ${mysql_datadir}

# 完成，打印输出
echo -e "backup restore success\n"

exit $?
```

