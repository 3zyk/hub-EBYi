> 作业2:阅读06-stock-bi-agent代码，回答如下问题:
>
> 1. 什么是前后端分离?
> 2. 历史对话如何存储，以及如何将历史对话作为大模型的下一次输入;

# 问题1：前后端分离

官方定义：前端项目、后端项目完全独立开发、独立部署、独立运行，只通过接口（API）进行数据通信。

流程：

1. 用户打开浏览器 / APP → **前端页面先加载好**
2. 前端通过 **接口 API** 向后端请求数据
3. 后端只查询数据库、处理逻辑，**只返回纯数据（JSON）**，不返回页面
4. 前端拿到数据，自己渲染到页面上展示

在06-stock-bi-agent项目中，体现在：纯前端（Streamlit） + 纯后端（FastAPI） + 只靠接口通信。

1. 后端提供接口：`/user/login`、`/stock/info`、`/chat/send`

2. 前端**只调用这些接口**，拿到数据后自己渲染页面

## 前端：

```
./demo/       
  - chat/
  - stock/
  - user/
  - mcp/
  - data/
./user/
  - streamlit_demo.py
```

## 后端

`main_server.py`：后端入口 

`/api/`：外部数据接口 

`/routers/`：API路由 

`/services/`：业务逻辑 

`/models/`：数据库

 `/agent/`：智能代理

# 问题2：历史对话如何存储，以及如何将历史对话作为大模型的下一次输入

- 存储：把用户提问 + AI 回答一对一对地存到后端数据库（你的 `conversations.db`）

- 传递：每次新提问时，从数据库取出最近 N 条历史，拼进请求里一起发给大模型

1. 前端传来 `session_id`：由 `routers/chat.py` 接收

2. 转发给 `services/chat.py`：真正的历史对话逻辑全部在这里：



## 存储对话

### 1. 前端传入对话 ID（路由层：routers/chat.py）

前端通过 **`session_id`** 标识一段连续对话，路由接口接收并传递给服务层：

![image-20260419211703440](D:\Code\STUDY\TASK\week12\1)

> 前端发送聊天消息，就是请求这个接口

### 存储 1：业务聊天记录（给前端展示）

在 **`services/chat.py`** 中，使用 **`append_message2db`** 函数将**用户消息**和**AI 回答**存入数据库 `server.db` 的 `ChatMessageTable` 表：

![image-20260419212415715](D:\Code\STUDY\TASK\week12\2)

![image-20260419212352393](D:\Code\STUDY\TASK\week12\3)

### 存储 2：大模型上下文记忆（给 AI 续聊）

在 **`services/chat.py`** 中，使用 **`AdvancedSQLiteSession`** 自动管理历史对话，存入框架数据库 `conversations.db`：

![image-20260419213021181](D:\Code\STUDY\TASK\week12\4)

该会话会**自动保存每一轮问答**，无需手动写 SQL。

## 输入历史对话

在 **`services/chat.py`** 调用大模型时，**传入携带历史的 `session` 对象**：

![image-20260419213143781](D:\Code\STUDY\TASK\week12\5)

根据 `session_id` 自动从 `conversations.db` 读取**该对话所有历史**，自动拼接为：系统提示词 + 历史对话 + 新问题，一起发送给大模型。

大模型因此实现**连续对话、上下文记忆**。