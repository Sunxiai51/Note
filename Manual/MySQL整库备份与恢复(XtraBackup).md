# MySQL整库备份与恢复(XtraBackup)

## 1. 版本

- SUSE Linux Enterprise Server 11 (x86_64)
- Percona XtraBackup 2.4.1
- MySQL 5.7.22

## 2. 策略

XtraBackup对MySQL的备份提供支持，通过编写脚本对其具体的命令进行封装，方便于按固定周期调用以实现定时备份。

> [Percona XtraBackup官方文档](https://www.percona.com/doc/percona-xtrabackup/LATEST/index.html)

编写了以下4个脚本（后附脚本源码）：

1. **全量备份脚本 (full_backup.sh)**：进行一次全量备份，并删除之前所有的备份
2. **增量备份脚本 (incremental_backup.sh)**：基于最近一次的备份进行一次增量备份
3. **Prepare脚本 (backup_prepare.sh)**：用于数据恢复前的prepare
4. **恢复脚本 (backup_restore.sh)**：用于从备份恢复数据

脚本调用有以下要求：

- 执行增量备份脚本时，必须至少存在一个未prepare的全量备份
- 执行Prepare脚本时，必须至少存在一个全量备份
- Prepare脚本只应该在执行恢复脚本之前执行，执行Prepare脚本前，请确保所有的备份行为已经停止，执行Prepare脚本后，一般会立刻执行恢复脚本
- 恢复脚本必须紧接着Prepare脚本执行，执行恢复脚本后，建议立刻执行一次全量备份脚本以清除之前prepare过的备份

执行任一脚本前，请首先阅读脚本开始部分的说明，并检查参数配置。

## 3. 脚本源码

### 3.1 全量备份脚本

```shell
#!/bin/bash
set -o errexit

# 全量备份脚本
#
# 通过xtrabackup对本地数据库进行全量备份
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
echo -e "[Note] execute ${xtrabackup_path}\n  --datadir=${mysql_datadir}\n  --user=${mysql_user}\n  --password=${mysql_password}\n  --host=127.0.0.1\n  --backup\n  --target-dir=${backup_path}${dir_name}" >> ${backup_path}${logfile_name}
${xtrabackup_path} --datadir=${mysql_datadir} --user=${mysql_user} --password=${mysql_password} --host=127.0.0.1 --backup --target-dir=${backup_path}${dir_name}

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
# 通过xtrabackup对本地数据库进行增量备份
#
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
echo -e "[Note] execute ${xtrabackup_path}\n  --datadir=${mysql_datadir}\n  --user=${mysql_user}\n  --password=${mysql_password}\n  --host=127.0.0.1\n  --backup\n  --target-dir=${backup_path}${dir_name}\n  --incremental-basedir=${backup_path}${last_dir}" >> ${backup_path}${logfile_name}
${xtrabackup_path} --datadir=${mysql_datadir} --user=${mysql_user} --password=${mysql_password} --host=127.0.0.1 --backup --target-dir=${backup_path}${dir_name} --incremental-basedir=${backup_path}${last_dir}

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
#
# 注意：
# 1.该脚本会清空之前存在的数据
# 2.恢复后建议立刻进行一次全量备份
#
# 执行前请检查以下参数配置
# ------------------ 参数配置 start ------------------

readonly backup_path="/home/ewallet/backups/mysql/" # 备份目录，请务必以“/”结尾
readonly mysql_datadir="/home/ewallet/soft/mysql/data/" # MySQL datadir真实目录(不能为链接)，请务必以“/”结尾

# ------------------- 参数配置 end -------------------
#
# 通过cp将上次全量备份目录中的数据copy至${mysql_datadir}

# 上次全量备份的目录
last_full_backup_dir=`ls ${backup_path} | tr ' ' '\n' | grep -E "^full_backup_[0-9]+$" | sort | tail -n1`
if [ ! -f "${backup_path}${last_full_backup_dir}/xtrabackup_checkpoints" ];then
  echo -e "File 'xtrabackup_checkpoints' is not found in the last backup directory ${last_full_backup_dir}"
  exit 1
fi

# 清空数据
if [ ! -f "${mysql_datadir}ibdata1" ] || [ ! -d "${mysql_datadir}mysql" ];then
  echo -e "Please confirm the 'mysql_datadir:${mysql_datadir}'."
  exit 2
fi
if [ "/" != "${mysql_datadir}" ];then
  rm -rf ${mysql_datadir}*
fi

# 恢复数据
cp -rf ${backup_path}${last_full_backup_dir}/* ${mysql_datadir}
rm -f ${mysql_datadir}xtrabackup_*
rm -f ${mysql_datadir}backup-my.cnf
chown -R mysql:mysql ${mysql_datadir}

# 完成，打印输出
echo -e "backup restore success\n"

exit $?
```

## 4. Tips & FAQ

### 4.1 如何备份数据

1. 确保全量备份脚本与增量备份脚本中的参数配置正确（请注意脚本注释中声明的要求，这很重要）
2. 按照前文所述策略执行脚本

可以通过cron等计划任务工具来实现数据库的定期备份，例如：

```shell
# 备份不可重叠，请根据一次备份的最大时长来设置时间间隔
0 0 /1 * * ? * # 每整点一次全量备份
0 10-50/10 * * * ? * # 每10分钟一次增量备份（整点除外）
```

Xtrabackup只支持本地备份，即必须在需要备份的数据库服务器上安装Xtrabackup，但备份文件可以独立存在，将备份文件copy到任何服务器上都可使用，这意味着你可以将备份文件存储到其它服务器上，你可以在备份完成后自行打包传输，也可以使用Xtrabackup的 `--stream` 选项。

### 4.2 备份了哪些数据？

备份了MySQL数据目录中所有数据。

Xtrabackup会对指定数据库的目录（主要包括指定表的 `.frm` 文件与 `.ibd` 文件）和MySQL数据字典文件（主要为 `ibdata1` 文件）进行拷贝。其中 `.frm` 为表结构文件，存储了相应表结构； `.ibd` 为表空间文件，存储了表中数据、索引等内容。

### 4.3 如何恢复数据？

Xtrabackup不提供数据恢复的功能，建议用户使用 `sync` / `tar` / `cp` 等方式恢复。

>  对于开启了 `innodb_file_per_table` 选项的数据库表备份，是可以仅通过其 `.frm` 文件与 `.ibd` 文件还原该表数据的，甚至在知晓其表结构的前提下仅通过 `.ibd` 文件即可还原数据，还原方法的相关文章已经有很多，例如：
>
> [MySQL innodb引擎下根据.frm和.ibd文件恢复表结构和数据 - CSDN博客](https://blog.csdn.net/hi__study/article/details/53489672)

恢复脚本的目的在于一次性恢复Xtrabackup所备份的数据，恢复操作步骤为：

1. 准备需要恢复数据的MySQL服务器

2. 将prepare好的备份文件copy至服务器备份目录（非MySQL数据目录）

3. 将恢复脚本copy至服务器，确认参数配置

4. 执行恢复脚本

5. 重启MySQL服务器

此时即可在该服务器上查询到备份的数据，恢复完成。

恢复前需要执行Xtrabackup prepare，该操作被封装在Prepare脚本中，恢复InnoDB的表时，prepare全量备份时的 `--apply-log-only` 选项是可选的，脚本中并未开启该选项，具体区别请自行参阅[Percona XtraBackup官方文档](https://www.percona.com/doc/percona-xtrabackup/LATEST/index.html)。

### 4.4 可以在非空的数据库中恢复数据吗？

恢复脚本将会清空目标数据库中所有原有数据，建议采用新安装的MySQL服务器进行恢复，或者在恢复前做好备份。
