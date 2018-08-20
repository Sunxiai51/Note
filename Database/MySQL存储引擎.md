# MySQL�洢����

���ĵ�����MySQL 5.7�еĴ洢���档

> ���벢������MySQL 5.7�ٷ��ĵ���

##14. The InnoDB Storage Engine

InnoDB��һ����ƽ��߿ɿ��Ժ͸����ܵ�ͨ�ô洢���棬��MySQL 5.7��Ĭ�ϵĴ洢���棬�����������ԣ�

- InnoDB��DML������ѭACIDģ�ͣ�֧��������ύ���ع����쳣�ָ�

- ֧���м�����Oracle-style consistent reads�������˶��û������Ժ�����

- InnoDB���Ż��˸��������Ĳ�ѯ��ÿ��InnoDB������һ����Ϊ `clustered index` ��������������ʹ����������������ʱռ�����ٵ�IO��Դ

- ֧�����Լ�������롢���¡�ɾ���������ֶ�ʱ�����м����ȷ��һ����

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

���ô洢����ķ�ʽ��

```sql
-- ENGINE=INNODB not needed unless you have set a different

-- default storage engine.
CREATE TABLE t1 (i INT) ENGINE = INNODB;

-- Simple table definitions can be switched from one to another.
CREATE TABLE t2 (i INT) ENGINE = CSV;
CREATE TABLE t3 (i INT) ENGINE = MEMORY;
```

Ҳ�����ֶ�����MySQL��ǰ�Ự��Ĭ�����棺

```sql
SET default_storage_engine=NDBCLUSTER;
```

�޸ı����棺

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

MyISAM���������ļ�����ʽ�洢�ڴ����У���Щ�ļ��������Ա�����ͷ����������ָʾ�ļ����͵���չ���� `.frm` �ļ��洢���ʽ�� `.MYD` (MYData) �ļ��洢�����ݣ� `.MYI` (MYIndex) �ļ��洢��������

### 15.3 The MEMORY Storage Engine

�ڴ�洢���洴�����д洢���ڴ��е�������;�ı������������ܱ�����Ӳ��������Դ�жϵ�Ӱ�죬������Щ��������ʱ����������Ϊ���������ݵ�ֻ�����档

#### When to Use MEMORY or NDB Cluster

��������Ա׼��ʹ��MEMORY����洢��Ҫ�ġ��߿��õĻ�Ƶ�����µ�����ʱ��Ӧ����NDBȺ���Ƿ��Ǹ��õ�ѡ��һ������MEMORY��������������������ԣ�

- �漰˲̬���ǹؼ����� (��Ự����򻺴�) �Ĳ�������MySQL������ͣ������������MEMORY���е����ݽ���ʧ
- �ڴ�洢�Խ��п��ٴ�ȡ������ʱ�ӡ����ݾ������ȫ�������ڴ��У������赼�²���ϵͳ���������ڴ�ҳ
- ֻ�����Զ�Ϊ�������ݷ���ģʽ (��������)

NDBȺ����MEMORY����ͬ���߱������ܣ������ṩMEMORY�����в����õĶ������ԣ�

- �м������Ͷ��̲߳��������ڿͻ���֮��ĵ�����
- ��������
- ��ѡ����֧�ֵ����ݳ־���
- �����ڵ�����ϵ��޹�����ϵ�ṹ�Ͷ�����������99.999%����
- ��ڵ��Զ����ݷַ���������Ա�����Զ����Ƭ������������
- ֧�ֱ䳤�������� (���� `BLOB` �� `TEXT` )

### 15.4 The CSV Storage Engine



### 15.5 The ARCHIVE Storage Engine


### 15.6 The BLACKHOLE Storage Engine



### 15.7 The MERGE Storage Engine



### 15.8 The FEDERATED Storage Engine



### 15.9 The EXAMPLE Storage Engine



// TODO ...