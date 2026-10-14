# 14 数据库 接口与安全架构

[返回总目录](../README.md)

返回目录  ·  阅读导航

逻辑模型与加密存储分开设计；encrypted 字段或算法名称本身不代表数据已经加密。规范化字段、授权、导出和迁移规则见附录 AA。

本章设计长期存储和数据处理的基础规则，包括对象关系、接口、加密、备份恢复、同步和生命周期。可靠性要求通过约束、迁移脚本及测试实现，不能仅凭数据模型完整认定。

## 14.1 数据架构总体设计

用户经验模型（Personal Experience Model）以记录为基础，以主题和项目组织关联，附件提供资料，洞察和反馈保留分析过程。

结构要素：User；Experience；Moment / Project；Attachment；Insight；Feedback。

## 14.2 核心数据对象设计

### 14.2.1 User用户对象

User 表示本地资料所属的使用者。单机版可使用本地身份，不要求注册账号；后续多设备、同步或分享再扩展认证与授权。

```json
{
"id":
"u001",
"createdAt":
"2026-10-15",
"nickname":
"example-user",
"settings":
{}
}
```

字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | UUID | 用户编号 |
| createdAt | 时间 | 创建时间 |
| nickname | 字符串 | 昵称 |
| settings | JSON | 配置 |

### 14.2.2 Moment瞬间记录

这是整个系统最核心对象。定义：用户某个时间点主动保存的一次经验。例如：“今天完成示例图像项目移动端优化。”，数据库：

```sql
CREATE TABLE moment
(
id TEXT PRIMARY KEY,
user_id TEXT,
content TEXT,
type TEXT,
created_time INTEGER,
event_time INTEGER,
privacy_level TEXT,
status TEXT
);
```

字段解释：id：唯一编号。例如：UUID。

content：正文。保存：用户原话。注意：不能被AI修改。

type：类型：

```text
text     文字
image    图片
audio    音频
video    视频扩展
hardware 硬件事件
```

created_time：创建时间。例如：用户什么时候记录。

event_time：事件发生时间。区别：记录时间 ≠ 事情发生时间。例如：晚上写：“今天上午发生了一件事。”

privacy_level：隐私等级。设计：

```text
public   公开策略
shared   指定分享策略
private  私密
highest  高度私密

策略标签不自动授予访问权。
```

默认采用 highest 等级；该字段只表达策略，实际访问仍由用途、处理位置和授权范围共同判断。

### 14.2.3 Attachment附件对象

作用：保存：图片。音频。文件。数据库：

```sql
CREATE TABLE attachment
(
id TEXT PRIMARY KEY,
moment_id TEXT,
file_type TEXT,
path TEXT,
size INTEGER,
hash TEXT
);
```

字段：hash：文件指纹。作用：检测：文件是否被修改。

### 14.2.4 Tag标签对象

作用：帮助用户组织经验。表：

```sql
CREATE TABLE tag
(
id TEXT PRIMARY KEY,
name TEXT
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

### 14.2.5 Project项目对象

时己特别适合：记录长期项目。例如：示例图像项目。结构：

```sql
CREATE TABLE project
(
id TEXT PRIMARY KEY,
name TEXT,
description TEXT,
start_time INTEGER,
status TEXT
);
```

状态：

```text
planned   计划
active    进行中
paused    暂停
completed 完成
archived  归档
```

### 14.2.6 Milestone里程碑

记录：项目关键节点。例如：2026-09：发布0.2版本。结构：

```sql
CREATE TABLE milestone
(
id TEXT PRIMARY KEY,
project_id TEXT,
title TEXT,
time INTEGER
);
```

### 14.2.7 Insight AI洞察对象

非常重要。因为：AI结果不能混入用户事实。必须独立。结构：

```sql
CREATE TABLE insight
(
id TEXT PRIMARY KEY,
content TEXT,
source_ids TEXT,
confidence FLOAT,
status INTEGER
);
```

字段：source_ids：表示：这个AI结论来自哪些记录。例如：

```text
moment001
moment023
moment089
```

confidence 如保留，必须说明生成方法和适用范围。模型自报数值不是正确率，未经校准不得作为准确性保证或访问控制依据。

status：

```text
0 待确认
1 用户认可
2 用户修改
3 用户拒绝
```

### 14.2.8 Feedback用户反馈对象

记录：用户如何修正AI。结构：

```sql
CREATE TABLE feedback
(
id TEXT,
insight_id TEXT,
content TEXT,
time INTEGER
);
```

例如：AI：“你越来越关注隐私。”，用户：“准确，但主要来自示例图像项目项目。”，保存。

## 14.3 数据生命周期设计

生命周期规则覆盖创建、读取、修改、分析、导出和删除，需同时处理关联对象与派生数据。

### 14.3.1 创建阶段

用户产生记录。数据：进入本地数据库。状态：Local。

### 14.3.2 使用阶段

用户：查看。修改。关联。

### 14.3.3 分析阶段

AI读取：必须经过权限。流程：

流程：Moment；Permission Check；AI Analysis。

### 14.3.4 删除阶段

删除时同步清理正文、附件引用、关系、索引、向量和缓存；失败项登记并可重试，未完成全部预定范围前不显示永久删除完成。

## 14.4 API设计

0.1 先定义本地 Repository 调用契约。下列路径表示未来服务接口的示意，不要求首版启动 HTTP 服务；正式网络接口另补认证、授权、幂等和错误响应。

### 14.4.1 创建记录接口

接口：POST

/moment/create

请求：

```json
{
"content":
"今天完成项目",
"type":
"text"
}
```

返回：

```json
{
"id":
"123",
"status":
"success"
}
```

### 14.4.2 查询记录

接口：

/moment/list

参数：

```json
{
"page":1,
"size":20
}
```

返回：

```json
{
"data":[]
}
```

### 14.4.3 搜索接口

/search

请求：

```json
{
"keyword":
"示例图像项目"
}
```

### 14.4.4 导出接口

/export

参数：

```json
{
"type":
"json"
}
```

### 14.4.5 AI分析接口

未来：

/ai/insight

请求：

```json
{
"range":
"30days",
"permission":
true
}
```

## 14.5 加密安全架构

这是整个项目核心。

### 14.5.1 安全目标

静态加密用于降低设备丢失或存储文件泄露时的资料暴露风险，前提是攻击者未取得可用解密密钥。

保护范围必须说明设备是否锁定、密钥是否已解锁。静态加密无法保证抵御已控制运行中终端的攻击者。

验收应核查密文、密钥存储、认证失败处理和日志残留，按实际验证范围描述保护能力。

### 14.5.2 加密流程

流程：用户密码；密钥派生；主密钥；数据加密密钥；AES-GCM；数据库。

### 14.5.3 密钥设计

采用：双层密钥。

第一层：用户密码。用户记忆。

第二层：随机生成密钥。真正加密数据。结构：

流程：Password；Key Encryption Key；Data Encryption Key；User Data。

使用随机 DEK 加密数据，使用密码派生的 KEK 包装 DEK。修改密码时重新派生 KEK 并重包 DEK；先验证新包装可解密，再替换旧包装，避免中断造成无法恢复。

## 14.6 备份恢复设计

备份恢复需让用户明确保存了什么、能否恢复，以及丢失密码和恢复材料的后果。

### 14.6.1 备份文件

可使用 .shiji 作为完整加密备份容器的扩展名。容器应包含格式版本、数据库对象、附件、清单和校验信息；扩展名本身不保证完整或加密。

### 14.6.2 恢复流程

导入前检查格式和容量，输入有效解密材料后验证认证信息与清单，在临时区解析和校验，再事务提交。失败时不改写当前数据；成功后对比对象和附件，并显示恢复结果。

恢复验收：使用独立测试库验证正确密码、错误密码、损坏容器、缺失附件、不支持版本和中断导入。成功恢复后核查数量、关系与文件校验；失败时确认旧库仍完整可用。

### 14.6.3 防止用户忘记密码

设计：恢复密钥。但是必须：告诉用户：“密码遗忘且没有有效恢复路径时，无法解密全部数据。”

## 14.7 同步安全设计

未来：多端。错误方式：服务器保存明文。禁止。正确：端到端加密。流程：设备A：加密 → 服务器：密文 → 设备B：解密。服务器不知道：用户记录内容。

## 14.8 威胁模型分析

数据库文件泄露：使用经验证的加密持久层，并检查密钥、索引及日志是否暴露内容。

服务端被攻击：同步仅保存密文和必要元数据，同时限定元数据、认证和密钥分发的保护范围。

威胁3：AI误判。解决：事实与洞察分离。

威胁4：第三方窥探。解决：权限控制。

设备丢失：撤销后续同步访问，并按策略轮换密钥；不承诺清除已下载到离线设备的资料。

## 14.9 安全验收标准

SEC-001：取得应用持久化文件，预期未持有有效密钥时无法还原受保护内容；另查附件、索引和日志残留。

SEC-002：错误密码。要求：无法解密。

SEC-003：关闭网络。要求：核心功能正常。

SEC-004：删除数据。要求：在声明的删除范围内无法通过应用恢复；外部备份另行管理。

## 14.10 本阶段完成标准

完成后：时己拥有：数据能力：□完整经验模型，□长期存储，□项目管理，□AI隔离，安全能力：□本地加密，□备份恢复，□权限控制，扩展能力：□支持多端，□支持硬件，□支持AI
