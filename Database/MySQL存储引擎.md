# MySQL存储引擎

本文档介绍MySQL 5.7中的存储引擎。

> 翻译并整理自MySQL 5.7官方文档。

##14. The InnoDB Storage Engine

InnoDB是一种能平衡高可靠性和高性能的通用存储引擎，是MySQL 5.7中默认的存储引擎，具有以下特性：

- InnoDB的DML操作遵循ACID模型，支持事务的提交、回滚和异常恢复

- 支持行级锁和Oracle-style consistent reads，增加了多用户并发性和性能

- InnoDB表优化了根据主键的查询，每个InnoDB表都具有一个称为 `clustered index` 的主键索引，它使根据主键查找数据时占用最少的IO资源

- 支持外键约束，插入、更新、删除相关外键字段时将进行检查以确保一致性

| Feature                                                      | Support                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| B-tree indexes                                               | Yes                                                          |
| Backup/point-in-time recovery (Implemented in the server, rather than in the storage engine.) | Yes                                                          |
| Cluster database support                                     | No                                                           |
| Clustered indexes                                            | Yes                                                          |
| Compressed data                                              | Yes                                                          |
| Data caches                                                  | Yes                                                          |
| Encrypted data (Implemented in the server via encryption functions. Data-at-rest tablespace encryption is available in MySQL 5.7 and later.) | Yes                                                          |
| Foreign key support                                          | Yes                                                          |
| Full-text search indexes                                     | Yes (InnoDB support for FULLTEXT indexes is available in MySQL 5.6 and later.) |
| Geospatial data type support                                 | Yes                                                          |
| Geospatial indexing support                                  | Yes (InnoDB support for geospatial indexing is available in MySQL 5.7 and later.) |
| Hash indexes                                                 | No (InnoDB utilizes hash indexes internally for its Adaptive Hash Index feature.) |
| Index caches                                                 | Yes                                                          |
| Locking granularity                                          | Row                                                          |
| MVCC                                                         | Yes                                                          |
| Replication support (Implemented in the server, rather than in the storage engine.) | Yes                                                          |
| Storage limits                                               | 64TB                                                         |
| T-tree indexes                                               | No                                                           |
| Transactions Yes                                             | Yes                                                          |
| Update statistics for data dictionary                        | Yes                                                          |

## 15. Alternative Storage Engines

### 15.1 Setting the Storage Engine

设置存储引擎的方式：

```sql
-- ENGINE=INNODB not needed unless you have set a different

-- default storage engine.
CREATE TABLE t1 (i INT) ENGINE = INNODB;

-- Simple table definitions can be switched from one to another.
CREATE TABLE t2 (i INT) ENGINE = CSV;
CREATE TABLE t3 (i INT) ENGINE = MEMORY;
```

也可以手动设置MySQL当前会话的默认引擎：

```sql
SET default_storage_engine=NDBCLUSTER;
```

修改表引擎：

```sql
ALTER TABLE t ENGINE = InnoDB;
```

### 15.2 The MyISAM Storage Engine

| Feature                                                      | Support                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| B-tree indexes                                               | Yes                                                          |
| Backup/point-in-time recovery (Implemented in the server, rather than in the storage engine.) | Yes                                                          |
| Cluster database support                                     | No                                                           |
| Clustered indexes                                            | No                                                           |
| Compressed data                                              | Yes (Compressed MyISAM tables are supported only when using the compressed row format. Tables using
the compressed row format with MyISAM are read only.) |
| Data caches                                                  | No                                                           |
| Encrypted data (Implemented in the server via encryption functions. Data-at-rest tablespace encryption is available in MySQL 5.7 and later.) | Yes                                                          |
| Foreign key support                                          | No                                                           |
| Full-text search indexes                                     | Yes                                                          |
| Geospatial data type support                                 | Yes                                                          |
| Geospatial indexing support                                  | Yes                                                          |
| Hash indexes                                                 | No                                                           |
| Index caches                                                 | Yes                                                          |
| Locking granularity                                          | Table                                                        |
| MVCC                                                         | No                                                           |
| Replication support (Implemented in the server, rather than in the storage engine.) | Yes                                                          |
| Storage limits                                               | 256TB                                                        |
| T-tree indexes                                               | No                                                           |
| Transactions                                                 | No                                                           |
| Update statistics for data dictionary                        | Yes                                                          |

MyISAM表以三个文件的形式存储在磁盘中，这些文件的名称以表名开头并具有用于指示文件类型的扩展名。 `.frm` 文件存储表格式， `.MYD` (MYData) 文件存储表数据， `.MYI` (MYIndex) 文件存储表索引。

### 15.3 The MEMORY Storage Engine

内存存储引擎创建具有存储在内存中的特殊用途的表。由于数据易受崩溃、硬件问题或电源中断的影响，仅将这些表用作临时工作区或作为其他表数据的只读缓存。

#### When to Use MEMORY or NDB Cluster

当开发人员准备使用MEMORY引擎存储重要的、高可用的或频繁更新的数据时，应考虑NDB群集是否是更好的选择。一个典型MEMORY引擎的用例包括以下特性：

- 涉及瞬态、非关键数据 (如会话管理或缓存) 的操作。当MySQL服务暂停或重新启动，MEMORY表中的数据将丢失
- 内存存储以进行快速存取，减少时延。数据卷可以完全容纳在内存中，而无需导致操作系统交换虚拟内存页
- 只读或以读为主的数据访问模式 (更新受限)

NDB群集与MEMORY引擎同样具备高性能，并且提供MEMORY引擎中不可用的额外特性：

- 行级锁定和多线程操作，用于客户端之间的低争用
- 可伸缩性
- 可选磁盘支持的数据持久性
- 不存在单点故障的无共享体系结构和多主机操作，99.999%可用
- 跨节点自动数据分发，开发人员无需自定义分片或分区解决方案
- 支持变长数据类型 (包括 `BLOB` 和 `TEXT` )

### 15.4 The CSV Storage Engine



### 15.5 The ARCHIVE Storage Engine


### 15.6 The BLACKHOLE Storage Engine



### 15.7 The MERGE Storage Engine



### 15.8 The FEDERATED Storage Engine



### 15.9 The EXAMPLE Storage Engine



// TODO ...