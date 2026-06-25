---
title: sql优化(index篇)
date: 2026-06-25 10:00:04
tags:
---

#### 常用命令级含义

| 命令 / 参数                                       | 用法示例                                                        | 含义                                                     | 使用场景                             | 注意事项                                         |
| ------------------------------------------------- | --------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------ | ------------------------------------------------ |
| `SHOW CREATE TABLE`                               | `SHOW CREATE TABLE process_node_record_assign_user;`            | 查看完整建表语句，包括字段、主键、索引、字符集、表引擎等 | 加索引前查看表结构                   | 输出内容较长，适合复制出来分析                   |
| `SHOW INDEX FROM`                                 | `SHOW INDEX FROM process_node_record_assign_user;`              | 查看某张表已有索引信息                                   | 判断索引是否已存在，避免重复添加     | 重点看 `Key_name`、`Column_name`、`Seq_in_index` |
| `DESC`                                            | `DESC process_node_record_assign_user;`                         | 简单查看表字段结构                                       | 快速看字段名、类型、是否可空、默认值 | 不如 `SHOW CREATE TABLE` 完整                    |
| `EXPLAIN`                                         | `EXPLAIN SELECT ...;`                                           | 查看 SQL 执行计划                                        | 判断 SQL 是否走索引、扫描多少行      | 重点看 `type`、`key`、`rows`、`Extra`            |
| `EXPLAIN ANALYZE`                                 | `EXPLAIN ANALYZE SELECT ...;`                                   | 实际执行 SQL 并分析真实耗时                              | MySQL 8.0+ 中更准确地分析性能        | 会真实执行 SQL，生产环境慎用                     |
| `ALTER TABLE ... ADD INDEX`                       | `ALTER TABLE 表名 ADD INDEX 索引名 (字段1, 字段2);`             | 给表添加普通索引                                         | 优化查询、关联、排序、过滤           | 大表添加索引可能产生锁等待                       |
| `ADD INDEX IF NOT EXISTS`                         | `ALTER TABLE 表名 ADD INDEX IF NOT EXISTS idx_xxx (字段1);`     | 如果索引不存在才添加                                     | 避免重复加索引报错                   | 部分 MySQL 版本可能不支持                        |
| `DROP INDEX`                                      | `DROP INDEX idx_xxx ON 表名;`                                   | 删除普通索引                                             | 索引加错、无效、影响写入性能时删除   | 不要随便删除主键索引                             |
| `ALTER TABLE ... DROP INDEX`                      | `ALTER TABLE 表名 DROP INDEX idx_xxx;`                          | 删除普通索引的另一种写法                                 | 和 `DROP INDEX ... ON ...` 效果类似  | 删除前先 `SHOW INDEX` 确认索引名                 |
| `ALTER TABLE ... DROP PRIMARY KEY`                | `ALTER TABLE 表名 DROP PRIMARY KEY;`                            | 删除主键索引                                             | 极少数表结构调整场景                 | 生产环境一般不要随便执行                         |
| `ALGORITHM=INPLACE`                               | `ALTER TABLE 表名 ADD INDEX idx_xxx (字段), ALGORITHM=INPLACE;` | 尽量原地修改表结构，不完整复制整张表                     | 线上加索引时降低影响                 | 如果当前操作不支持，会报错                       |
| `ALGORITHM=COPY`                                  | `ALTER TABLE 表名 ADD INDEX idx_xxx (字段), ALGORITHM=COPY;`    | 复制整张表，生成新表后替换旧表                           | 老版本或不支持在线 DDL 的场景        | 对线上影响较大，容易阻塞                         |
| `ALGORITHM=INSTANT`                               | `ALTER TABLE 表名 ADD COLUMN xxx INT, ALGORITHM=INSTANT;`       | 元数据级别快速变更，几乎瞬间完成                         | MySQL 8.0 某些字段变更场景           | 添加普通索引通常不是 `INSTANT`                   |
| `LOCK=NONE`                                       | `ALTER TABLE 表名 ADD INDEX idx_xxx (字段), LOCK=NONE;`         | DDL 执行期间尽量不阻塞读写                               | 生产环境在线加索引                   | 不代表完全无锁，仍可能有短暂 MDL 锁              |
| `LOCK=SHARED`                                     | `ALTER TABLE 表名 ADD INDEX idx_xxx (字段), LOCK=SHARED;`       | 允许读，不允许写                                         | 某些在线 DDL 不能完全无锁时使用      | 写操作会被阻塞                                   |
| `LOCK=EXCLUSIVE`                                  | `ALTER TABLE 表名 ADD INDEX idx_xxx (字段), LOCK=EXCLUSIVE;`    | 读写都阻塞                                               | 极少数必须独占表的 DDL 场景          | 生产环境慎用                                     |
| `USING BTREE`                                     | `ADD INDEX idx_xxx (字段1, 字段2) USING BTREE`                  | 指定使用 BTree 索引结构                                  | 普通查询、范围查询、排序、关联优化   | InnoDB 普通索引默认就是 BTree                    |
| `VISIBLE`                                         | `ALTER TABLE 表名 ALTER INDEX idx_xxx VISIBLE;`                 | 设置索引对优化器可见                                     | 让 MySQL 优化器可以使用该索引        | MySQL 8.0+ 支持                                  |
| `INVISIBLE`                                       | `ADD INDEX idx_xxx (字段1, 字段2) INVISIBLE`                    | 创建不可见索引，优化器默认不使用                         | 生产环境先建索引但不立即影响执行计划 | MySQL 8.0+ 支持                                  |
| `SET optimizer_switch='use_invisible_indexes=on'` | `SET optimizer_switch='use_invisible_indexes=on';`              | 当前会话允许使用不可见索引                               | 测试不可见索引是否有效               | 只影响当前会话                                   |
| `SET SESSION lock_wait_timeout`                   | `SET SESSION lock_wait_timeout = 10;`                           | 设置当前会话等待元数据锁的超时时间                       | 避免 ALTER TABLE 一直卡住            | 超时后会失败，不会无限等待                       |
| `SHOW FULL PROCESSLIST`                           | `SHOW FULL PROCESSLIST;`                                        | 查看当前数据库连接和正在执行的 SQL                       | 加索引前检查是否有慢 SQL、锁等待     | 重点看 `Time`、`State`、`Info`                   |
| `information_schema.innodb_trx`                   | `SELECT * FROM information_schema.innodb_trx;`                  | 查看当前 InnoDB 事务                                     | 排查长事务、未提交事务               | 长事务可能导致 DDL 等待 MDL 锁                   |
| `ANALYZE TABLE`                                   | `ANALYZE TABLE process_node_record_assign_user;`                | 更新表和索引统计信息                                     | 加索引后让优化器更准确选择索引       | 大表执行也可能有一定开销                         |
| `COUNT(*)`                                        | `SELECT COUNT(*) FROM 表名;`                                    | 统计表数据量                                             | 加索引前评估表大小和风险             | 大表 `COUNT(*)` 可能较慢                         |
| `SHOW VARIABLES LIKE 'wait_timeout'`              | `SHOW VARIABLES LIKE 'wait_timeout';`                           | 查看数据库连接空闲超时时间                               | 排查连接被 MySQL 主动断开问题        | 和连接池 `maxLifetime` 需要配合                  |
| `SHOW VARIABLES LIKE 'net_read_timeout'`          | `SHOW VARIABLES LIKE 'net_read_timeout';`                       | 查看 MySQL 网络读超时时间                                | 排查网络读超时导致连接异常           | 值过小可能导致长 SQL 报通信异常                  |
| `SHOW VARIABLES LIKE 'net_write_timeout'`         | `SHOW VARIABLES LIKE 'net_write_timeout';`                      | 查看 MySQL 网络写超时时间                                | 排查网络写超时导致连接异常           | 值过小也可能导致连接中断                         |

#### SQL 场景里，最常用的一套

| 步骤 | 命令                                                           | 目的                      |
| ---- | -------------------------------------------------------------- | ------------------------- |
| 1    | `SHOW CREATE TABLE 表名;`                                      | 查看表结构和已有索引      |
| 2    | `SHOW INDEX FROM 表名;`                                        | 确认索引是否已存在        |
| 3    | `EXPLAIN SELECT ...;`                                          | 查看当前 SQL 是否走索引   |
| 4    | `SHOW FULL PROCESSLIST;`                                       | 检查是否有长 SQL 或锁等待 |
| 5    | `SET SESSION lock_wait_timeout = 10;`                          | 避免加索引一直等待锁      |
| 6    | `ALTER TABLE ... ADD INDEX ..., ALGORITHM=INPLACE, LOCK=NONE;` | 尽量在线低锁添加索引      |
| 7    | `ANALYZE TABLE 表名;`                                          | 更新优化器统计信息        |
| 8    | `EXPLAIN SELECT ...;`                                          | 再次验证是否命中新索引    |

#### 当前场景的完整模板(耗时5s优化到3s)，还可走进一步优化(sql优化--第一个查询查询所以字段没走索引触发回表)

```sql
-- 1. 查看表结构
SHOW CREATE TABLE process_node_record_assign_user;
SHOW CREATE TABLE process_node_record;

-- 2. 查看已有索引
SHOW INDEX FROM process_node_record_assign_user;
SHOW INDEX FROM process_node_record;

-- 3. 查看是否有长 SQL 或锁等待
SHOW FULL PROCESSLIST;

-- 4. 设置当前会话锁等待超时时间，避免一直卡住
SET SESSION lock_wait_timeout = 10;

-- 5. 添加索引
ALTER TABLE process_node_record_assign_user
ADD INDEX idx_user_status_instance_node_execution (
  user_id,
  status,
  process_instance_id,
  node_id,
  execution_id
) USING BTREE,
ALGORITHM=INPLACE,
LOCK=NONE;

-- 6. 更新统计信息
ANALYZE TABLE process_node_record_assign_user;

-- 7. 验证 SQL 是否走索引
EXPLAIN
SELECT
  a.*,
  b.`status`
FROM process_node_record_assign_user a
INNER JOIN process_node_record b
  ON a.process_instance_id = b.process_instance_id
 AND a.node_id = b.node_id
 AND a.execution_id = b.execution_id
 AND b.`status` = 1
WHERE
  a.user_id = '1829737509048532993'
  AND a.`status` = 1;
```
