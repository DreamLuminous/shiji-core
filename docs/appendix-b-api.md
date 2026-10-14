# 附录 B 接口设计规范

[返回总目录](../README.md)

返回目录  ·  阅读导航

下列 HTTP 端点为服务化草案。GET 搜索以 URL 查询参数传递，不依赖请求体；创建、修改、删除、授权及分析统一采用附录 AA 的契约。

## B.1 API设计原则

第一版通过本地 Repository 调用这些能力，无需建立网络服务器。本附录为服务化接口草案；统一端点、鉴权与错误处理以附录 AA 的接口契约为基线。

虽然第一版本地运行：仍然按照服务化设计。原因：未来：多端；云同步；团队协作。

## B.2 REST接口设计

基础：HTTPS。格式：JSON。

用户接口：创建用户：POST

/user/create

请求：

```json
{
"name":"example-user"
}
```

返回：

```json
{
"id":"u001"
}
```

记录接口：创建记录：POST

/moment

请求：

```json
{
"content":
"今天完成实验",
"type":
"text"
}
```

返回：

```json
{
"id":"example-record-001",
"status":"ok"
}
```

查询记录：GET

/moment/list

参数：

```text
page=1
size=20
```

返回：

```json
{
"total":100,
"data":[]
}
```

搜索接口：GET

/search

请求：

```json
{
"keyword":"项目"
}
```

返回：相关记录。

项目接口：创建项目：POST

/project

添加记录：POST

```text
/project/{id}/moment
```

AI接口：生成洞察：POST

/ai/insight

请求：

```json
{
"range":"30days",
"type":"summary"
}
```

返回：

```json
{
"insight":
"近期项目记录增加"
}
```

硬件接口：同步设备：POST

/device/sync

数据：

```json
{
"device":"moment001",
"events":[]
}
```
