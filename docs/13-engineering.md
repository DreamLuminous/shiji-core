# 13 工程落地与软件开发

[返回总目录](../README.md)

返回目录  ·  阅读导航

本章命令和目录是开发示例。Alpha 0.1 采用 Vue 3、TypeScript、Vite 与 Dexie，SQLite 示例用于桌面或移动存储设计，不直接在 Dexie 中执行。

本章把产品规划转为工程步骤：先完成可运行原型，再经过内测和数据可靠性验证形成稳定软件。AI、硬件和专业合作作为后续独立模块推进。

## 13.1 总体工程策略

### 13.1.1 开发原则

工程采用核心闭环优先：完成记录、保存、回顾、检索、导入导出与删除，再扩展其他模块。每次新增功能应有明确需求、依赖和验收证据，避免范围失控。

### 13.1.2 第一性原理拆解

第一版验证用户是否愿意留下并回看记录。AI、硬件和健康协作暂不进入最小闭环，防止额外能力掩盖基础体验问题。

## 13.2 项目仓库设计

采用：GitHub作为主仓库。原因：开源生态；版本管理；Issue管理；社区协作。

### 13.2.1 总仓库结构

**目录与文件路径**

```text
Shiji/README.md
Shiji/LICENSE
Shiji/docs/
Shiji/docs/vision.md
Shiji/docs/architecture.md
Shiji/docs/privacy.md
Shiji/docs/roadmap.md
Shiji/web/
Shiji/web/src/
Shiji/web/public/
Shiji/web/package.json
Shiji/android/
Shiji/android/app/
Shiji/android/build.gradle
Shiji/desktop/
Shiji/desktop/src/
Shiji/ai/
Shiji/ai/models/
Shiji/ai/rag/
Shiji/ai/prompts/
Shiji/hardware/
Shiji/hardware/firmware/
Shiji/hardware/pcb/
Shiji/hardware/protocol/
Shiji/tests/
Shiji/tests/functional/
Shiji/tests/security/
Shiji/tests/performance/
```

## 13.3 第一版软件开发方案

Shiji Alpha 0.1 以三个月为计划窗口，交付创建、查看、编辑、删除、关键词搜索、本地保存、JSON 导入导出和隐私状态。AI、云同步和硬件后置。真实用户内测须通过数据可靠性与已声明保护能力的发布检查。

### 13.3.1 技术选型

Web 原型选用 Vue 3、TypeScript 和 Vite，分别承担界面、类型约束和构建。工程还需固定依赖版本、提交锁文件，并在 README 记录已验证的运行环境。

### 13.3.2 项目初始化

在计划使用的开发环境中创建项目。示例命令采用交互式脚手架，按提示选择 Vue 与 TypeScript；随后进入生成的项目目录再安装依赖。

```bash
npm create vite@latest shiji-web
```

选择：Vue，TypeScript

进入项目目录：cd shiji-web。命令在本地开发终端执行；依赖安装成功后提交 package-lock.json 等实际使用的锁文件，后续环境按锁文件复现。

进入 shiji-web 目录后安装项目依赖。

```bash
npm install
```

第三步：安装状态管理

```bash
npm install pinia
```

第四步：安装数据库工具

```bash
npm install dexie
```

Dexie 用于封装 IndexedDB 访问。后续把数据库操作收敛到 Repository，页面不直接操作表。安装完依赖后运行 npm run dev，验证首页可访问，再建立数据库版本和基础索引。

首轮实现顺序：建立数据库与 Repository，完成创建和查询，再接时间轴、编辑、删除、关键词搜索及迁移包。每个步骤保存演示和测试记录，完整闭环通过后再处理多媒体。

## 13.4 第一版页面设计

目录：

**目录与文件路径**

```text
src/pages/
src/pages/Home.vue
src/pages/Timeline.vue
src/pages/Search.vue
src/pages/Settings.vue
src/components/
src/components/MomentCard.vue
src/components/Editor.vue
src/components/Tag.vue
src/store/
src/store/moment.ts
src/database/
src/database/db.ts
```

## 13.5 核心数据系统实现

### 13.5.1 数据库初始化

文件：database/db.ts，设计：

```text
MomentDatabase
tables:
moments
tags
attachments
settings
```

### 13.5.2 Moment对象

以下为 Moment 核心字段示例。createdAt 和 eventAt 统一为 UTC Unix 毫秒，修订号和隐私等级遵循附录 AA.2；数据库列名使用 snake_case 时，存储适配器提供明确映射。

```json
{
  "id": "m001",
  "content": "今天完成第一个版本",
  "type": "text",
  "createdAt": 1792022400000,
  "eventAt": 1792022400000,
  "privacyLevel": "highest",
  "status": "active",
  "revision": 1
}
```

## 13.6 记录功能实现流程

保存流程依次校验非空内容、生成记录 ID、设置创建时间和默认状态，再通过 Repository 执行事务写入。提交成功后更新列表并清空编辑器；失败时保留输入，显示可重试提示。保存期间禁用重复提交，重试使用同一操作 ID。

编辑流程：详情页进入编辑后保留原对象 ID，校验输入并提交新 revision。提交失败不覆盖本地编辑草稿；与其他设备形成并发修改时走冲突流程。保存成功后，依赖旧正文的搜索索引和 AI 缓存需更新或失效。

## 13.7 功能验收

REC-001：输入正文并保存。预期生成唯一记录，正文与时间字段正确；核查数据库后登记实际结果。

REC-002：保存完成后关闭浏览器，再次打开。预期已提交记录可读取；另测保存中被强制关闭时的事务状态。

REC-003：在线完成应用壳缓存后断网重启。预期基础记录、读取、检索和导出可使用，外部功能显示离线状态。

## 13.8 搜索功能实现

关键词搜索先通过存储适配器实现，输入“硬件”后返回匹配记录。SQLite 可用下列 SQL 表达查询含义，实际实现采用参数化查询；Web Dexie 端使用对应查询逻辑并测量性能。

```sql
SELECT *
FROM moments
WHERE content LIKE '%硬件%';
```

## 13.9 导出系统实现

JSON 迁移包包含格式版本、导出时间、记录、标签及关系、附件清单和校验信息。下面只展示包头结构；导入校验及完整备份要求见附录 AA.5。

```json
{
"version":"1.0",
"moments":[],
"tags":[]
}
```

未来：支持：Markdown。PDF。

## 13.10 安全第一版设计

虽然没有服务器。但是仍然需要保护。

### 13.10.1 数据加密

AES-GCM 用于加密敏感正文和附件；密码派生 KEK 包装随机生成的 DEK。每次加密使用唯一 IV，并保存格式与认证信息。字段加密、数据库整体加密和搜索索引保护须分别设计和测试。

### 13.10.2 密码处理

禁止：直接保存密码。保存：密码派生参数、盐与经过 KEK 包装的 DEK；不保存原始密码。

## 13.11 第一版测试计划

功能测试连续执行 1,000 次记录操作，比较已提交对象与读取结果，目标无已确认记录丢失；同时注入写入失败和强制关闭，验证失败提示与恢复状态。

准备 10,000 条记录测试时间轴首屏，工程优化目标为 2 秒。与 6.5 的 3 秒 MVP 基线分别登记，均需注明设备、版本和冷启动条件。

安全测试检查持久化文件、附件、索引和日志是否出现声明为加密的明文内容。检查时说明设备锁定与密钥解锁状态；测试不能覆盖未声明的保护范围。

## 13.20 工程总结

工程重点是长期可靠的数据管理和可核查的 AI 辅助流程。架构、界面、授权和测试应共同支持记录完整、分析可追溯、迁移可验证和问题可恢复。
