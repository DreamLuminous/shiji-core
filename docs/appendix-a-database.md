# 附录 A 数据库设计

[返回总目录](../README.md)

返回目录  ·  阅读导航

## A.1 数据库设计目标

本附录的 SQL 用于说明早期逻辑对象。整数类型和状态映射为历史示意，正式实现统一使用附录 AA 的字符串枚举与字段契约。数据库列名与应用字段建立明确映射；迁移脚本补充外键、唯一约束、索引、版本升级和加密落盘方案。

时己数据库不是简单保存文本。它需要支持：多年积累；多媒体；AI分析；多设备同步；权限管理；数据迁移。因此设计目标：目标1：数据长期稳定。预计：10年。百万级记录。

目标2：事实与分析分离。用户记录：不能被AI覆盖。

目标3：支持模块扩展，新增能力时不需要推倒重做。

## A.2 数据库总体结构

采用：关系数据库 + 向量数据库。关系数据库：保存：结构化信息。例如：记录时间。项目。标签。向量数据库：保存：语义信息。例如：相似经历。整体：

结构要素：SQLite/PostgreSQL；用户经验数据 / AI向量数据；Moment / Embedding；Project / Vector；Tag / Similarity；Attachment。

## A.3 用户表 User

作用：用户身份。SQL：

```sql
CREATE TABLE user
(
id TEXT PRIMARY KEY,
created_time INTEGER NOT NULL,
nickname TEXT,
avatar TEXT,
status INTEGER DEFAULT 1
);
```

字段说明：

| 字段 | 说明 |
| --- | --- |
| id | 用户唯一ID |
| created_time | 创建时间 |
| nickname | 显示名称 |
| avatar | 头像 |
| status | 状态 |

## A.4 经验记录表 Moment

核心表。

```sql
CREATE TABLE moment
(
id TEXT PRIMARY KEY,
user_id TEXT,
content TEXT,
type INTEGER,
created_time INTEGER,
event_time INTEGER,
location TEXT,
privacy_level INTEGER,
encrypted INTEGER DEFAULT 1
);
```

字段解释：content：保存：用户原始表达。例如：今天完成了示例图像项目0.3版本。禁止：AI直接修改。

type：类型：

```text
0 text
1 image
2 audio
3 video
4 hardware
5 system
```

event_time：事件发生时间。区别：created_time：记录时间。event_time：事情时间。

## A.5 附件表 Attachment

```sql
CREATE TABLE attachment
(
id TEXT PRIMARY KEY,
moment_id TEXT,
file_path TEXT,
file_type TEXT,
file_size INTEGER,
hash TEXT
);
```

用途：保存：图片。音频。文件。

## A.6 标签系统

Tag：

```sql
CREATE TABLE tag
(
id TEXT PRIMARY KEY,
name TEXT,
created_time INTEGER
);
```

关联：

```sql
CREATE TABLE moment_tag
(
moment_id TEXT,
tag_id TEXT
);
```

## A.7 项目系统

时己特别适合：学生。开发者。创作者。

Project：

```sql
CREATE TABLE project
(
id TEXT PRIMARY KEY,
name TEXT,
description TEXT,
start_time INTEGER,
end_time INTEGER,
status INTEGER
);
```

状态：

```text
0 计划
1 开始
2 进行
3 完成
4 归档
```

Project Moment关联：

```sql
CREATE TABLE project_moment
(
project_id TEXT,
moment_id TEXT
);
```

例如：记录：“示例图像项目增加批处理功能”，关联：项目：示例图像项目。

## A.8 里程碑系统

Milestone：

```sql
CREATE TABLE milestone
(
id TEXT PRIMARY KEY,
project_id TEXT,
title TEXT,
description TEXT,
time INTEGER
);
```

例如：项目：示例图像项目。里程碑：2026-09，首次公开测试。

## A.9 AI洞察表

非常关键。

```sql
CREATE TABLE insight
(
id TEXT PRIMARY KEY,
content TEXT,
create_time INTEGER,
confidence FLOAT,
status INTEGER
);
```

状态：

```text
0 未确认
1 用户认可
2 用户修改
3 用户拒绝
```

## A.10 AI来源关系表

为什么需要单独表？因为：一个洞察可能来自：多个记录。

```sql
CREATE TABLE insight_source
(
insight_id TEXT,
moment_id TEXT
);
```

例如：AI：“你长期关注隐私保护。”，来源：记录A。记录B。记录C。

## A.11 用户反馈表

```sql
CREATE TABLE feedback
(
id TEXT PRIMARY KEY,
insight_id TEXT,
content TEXT,
time INTEGER
);
```

作用：形成：人与AI共同修正。

## A.12 设备表 Device

未来硬件需要。

```sql
CREATE TABLE device
(
id TEXT PRIMARY KEY,
user_id TEXT,
device_type TEXT,
last_sync INTEGER,
status INTEGER
);
```

设备：手机。电脑。Moment硬件。

## A.13 同步记录表 Sync

```sql
CREATE TABLE sync_record
(
id TEXT PRIMARY KEY,
device_id TEXT,
moment_id TEXT,
sync_time INTEGER,
status INTEGER
);
```

用途：避免重复同步。

## A.15 数据库关系总结

最终ER关系：

结构要素：User；Moment ---------------- Attachment；Tag；Moment；Project；Milestone；Moment；Insight；Feedback；User；Device；Sync。
