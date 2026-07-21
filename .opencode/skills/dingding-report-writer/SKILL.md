---
name: dingding-report-writer
description: >-
  当用户要写、生成、确认、发送或配置钉钉日报/周报时，必须使用本技能。
  触发语包括：写日报、写周报、发送日报、发送周报、确认发送日报、确认发送周报、
  配置日报收件人、配置周报收件人、daily report、weekly report、DingTalk worklog。
  本技能定义了从 OpenCode/XCode 工作区会话采集数据、生成清单式草稿、先配置收件人、
  发送前自然语言确认、读取配置文件以及调用 dingding_worklog_create_report 时固定
  toChat=false 的完整流程。
metadata:
  version: 0.1.0
---

# 钉钉日报/周报写作与发送

本技能用于根据 XCode 工作区会话数据生成钉钉日报和周报，并通过钉钉日志 MCP 发送。流程必须分成两步：先生成清单式草稿，再等待用户用自然语言确认发送。确认时不要使用 `question` 组件。

## 文件位置

- 配置文件：`/workspace/.xcode/dingding-report-config.yaml`

## 核心规则

- 先生成草稿，再发送。用户没有明确确认前，绝不调用 `dingding_worklog_create_report`。
- 不要使用 `question` 组件。用普通文本要求确认，例如：`如需发送，请回复：确认发送日报`。
- 调用钉钉日志发送时，`toChat` 必须固定为 `false`。
- 收件人、模板名称和来源标识都从 `/workspace/.xcode/dingding-report-config.yaml` 读取。
- 如果日报或周报缺少收件人配置，必须先配置收件人，再生成或发送对应报告。
- 如果配置缺少模板名称，用普通文本询问；能解析并保存时，写回配置文件。
- 钉钉日志的发送主体是当前授权的钉钉用户。配置文件保存的是接收人 `toUserIds`，不是可伪装的发件人。
- 日报和周报都必须是清单式总结，正文条目使用有序列表，避免长段落。
- 查询“今天”“本周”时，以 `Asia/Shanghai` 的自然日/自然周为准，并把计算出的时间范围按 RFC3339 传给 MCP。

## 配置结构

配置文件使用以下结构。配置文件可以不存在或为空；遇到不存在、空文件或缺少 `daily`/`weekly` 结构时，先初始化或补齐下面的默认结构，再继续后续流程。不要把用户可变配置写回技能目录。

```yaml
daily:
  templateName: 日报
  toUserIds: []
  ddFrom: opencode-daily-report

weekly:
  templateName: 周报
  toUserIds: []
  ddFrom: opencode-weekly-report
```

## 首次配置前置条件

当用户首次要求写日报、写周报、发送日报、发送周报或配置收件人时，先检查对应配置：

1. 如果 `/workspace/.xcode/dingding-report-config.yaml` 不存在、为空或不是有效配置，先创建/补齐默认结构。已有字段不要覆盖，只补缺失字段。
2. 如果 `toUserIds` 为空，先配置收件人。用户给出姓名或手机号时，使用钉钉联系人 MCP 解析 `userId`；不要猜测 `userId`。
3. 如果模板名称为空，先询问模板名称，或调用钉钉日志模板 MCP 展示可用模板供用户用自然语言选择。

## 日报流程

当用户要求写日报或生成日报时：

1. 读取 `/workspace/.xcode/dingding-report-config.yaml` 中的 `daily` 配置；如果配置文件不存在或为空，先按“首次配置前置条件”初始化默认结构。
2. 如果 `daily.toUserIds` 为空，先配置日报收件人，配置完成前不要继续生成或发送日报。
3. 按 `Asia/Shanghai` 计算今天的时间范围：00:00:00 到 23:59:59。
4. 将上海本地时间范围转换成 RFC3339 作为 MCP 查询参数。
5. 调用 `xcode_xcode_query_workspace_sessions` 查询当天归档的工作区会话：
   - `status: archived`
   - `archived_after`: 当天开始时间，RFC3339 格式
   - `archived_before`: 当天结束时间，RFC3339 格式
   - 分页查询直到取完相关结果。
6. 调用 `xcode_xcode_query_workspace_sessions` 查询当天创建的工作区会话：
   - `status: all`
   - `created_after`: 当天开始时间，RFC3339 格式
   - `created_before`: 当天结束时间，RFC3339 格式
   - 分页查询直到取完相关结果。
7. 对同时出现在“当天创建”和“当天归档”里的会话去重；必要时保留两个事实：今天创建、今天归档。
8. 生成使用有序列表的清单式日报草稿。
9. 展示草稿，并用普通文本要求确认：
   - `如需发送，请回复：确认发送日报`
   - `如需修改，请直接说明修改内容`
10. 到此停止。用户确认前不要发送。

日报草稿格式：

```markdown
今天工作：
1. ...

明天计划：
1. ...
```

## 周报流程

当用户要求写周报或生成周报时：

1. 读取 `/workspace/.xcode/dingding-report-config.yaml` 中的 `weekly` 配置；如果配置文件不存在或为空，先按“首次配置前置条件”初始化默认结构。
2. 如果 `weekly.toUserIds` 为空，先配置周报收件人，配置完成前不要继续生成或发送周报。
3. 按 `Asia/Shanghai` 计算本周范围：周一 00:00:00 到周日 23:59:59。
4. 调用 `dingding_worklog_get_send_report_list` 查询本周已发送的日报：
   - `report_template_name`: `daily.templateName`
   - `startTime`: 本周开始时间，毫秒时间戳
   - `endTime`: 本周结束时间，毫秒时间戳
   - `modifiedStartTime` 和 `modifiedEndTime` 使用同一个本周范围，除非任务需要更窄的修改时间过滤。
   - 分页查询直到取完本周日报。
5. 对每一篇日报调用 `dingding_worklog_get_report_entry_details` 读取详情内容。
6. 周报只从这些日报内容中汇总。除非用户明确要求兜底，否则不要直接查询 workspace sessions 来写周报。
7. 生成使用有序列表的清单式周报草稿。
8. 展示草稿，并用普通文本要求确认：
   - `如需发送，请回复：确认发送周报`
   - `如需修改，请直接说明修改内容`
9. 到此停止。用户确认前不要发送。

周报草稿格式：

```markdown
本周完成工作：
1. ...

下周计划：
1. ...
```

## 确认与发送

只有当用户当前消息明确确认发送已经展示过的草稿时，才可以发送，例如：

- `确认发送日报`
- `发送这份日报`
- `确认发送周报`
- `发送这份周报`

发送前必须：

1. 重新读取对应的配置段。
2. 解析钉钉日志模板：
   - 优先用配置的 `templateName` 调用 `dingding_worklog_get_template_details_by_name`。
   - 如果模板详情里没有可用的模板 ID，再调用 `dingding_worklog_get_available_report_templates`，按名称匹配。
3. 将草稿映射到模板的文本字段。文本字段使用 Markdown 内容。
4. 调用 `dingding_worklog_create_report`，参数必须包含：
   - `templateId`: 解析得到的模板 ID
   - `ddFrom`: 配置中的 `ddFrom`
   - `toUserIds`: 配置中的接收人
   - `toChat`: `false`
   - `contents`: 与模板兼容的字段列表；文本字段使用 `contentType: markdown`。

如果模板有多个文本字段，优先填入主要的总结/内容字段。如果模板要求多个必填字段，按字段名把报告章节拆分到对应字段。

## 配置更新

当用户要求修改收件人、模板或来源标识时：

- 只更新相关配置项。
- 用户提供姓名或手机号时，使用钉钉联系人 MCP 解析 `userId`。
- 不要猜测或编造 `userId`。

## 异常处理

- 如果日报没有查到 workspace sessions，仍生成一份简短日报，明确说明今天没有可用工作区记录，并保留可用的计划/风险项。
- 如果周报没有查到本周已发送日报，说明周报无法基于日报汇总，并询问是否允许改用 workspace sessions 作为兜底数据源。
- 如果缺少钉钉模板或收件人配置，发送前停止，并用普通文本询问缺失配置。
- 如果钉钉 MCP 调用需要授权，提示用户完成授权流程，不要编造数据。
