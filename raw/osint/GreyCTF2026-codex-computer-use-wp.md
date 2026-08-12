# Codex Computer Use

## 题目简述

题面给出的是一个公开的 [Codex Trace](https://traces.com/s/jn7c59d3c3e847cwmdctga3z5d87h8mn)，提示是“看看我的 agent 用 computer use 做了什么”。这不是需要攻击 CTFd 的 Web 题；关键证据已经被公开记录在 agent 的会话日志中。需要从公开 trace 还原 agent 为 GreyCTF 2026 找到并在 Chrome 中打开的页面。

公开页的标题为“Opened GreyCTF 2026 registration page in Chrome”。会话消息显示，用户要求 agent 用 Computer Use 在 Chrome 打开 GreyCTF 2026 的注册页；agent 先从 CTFtime 找到当时的注册入口，随后明确回复“正在打开”，最后确认该页面已在 Chrome 中打开。这些内容足以说明公开 trace 泄露了原本只应在执行会话中出现的导航结果。

## 解题过程

### 从公开记录而非目标站点取证

打开题面所给的 Trace 页面，阅读消息时间线即可。页面既会渲染 agent 的自然语言回复，也把相同的消息放在页面的初始数据中，因此不必登录、注册或对目标站点进行枚举。若浏览器界面加载不稳定，读取公开页面的 HTML 同样能看到以下三项连续证据：

1. 用户要求 agent 使用 Computer Use 打开 GreyCTF 2026 registration page。
2. agent 说明自己从 CTFtime 找到了当前注册入口，并将这个入口作为内联 URL 写入回复。
3. agent 的最终消息确认 GreyCTF Qualifiers 2026 注册页已在 Chrome 中打开；trace 的自动摘要也记录了“找到 URL → 用插件打开 → 确认成功”这一序列。

因此题目考察的不是对注册系统的攻击，而是意识到公开 agent trace 会把检索结果、最终 URL 和已执行的浏览器动作一起暴露。赛时应以 trace 记录的入口和动作作为解题路径；归档仓库没有保存该动态注册页当时的完整响应，不能把今天页面的内容臆测为 trace 当时所见。

### 验证

两种阅读方式应得到相同结论：Trace 页面可见的对话与其页面初始数据中的 `messages` 记录包含同一条注册入口和“已在 Chrome 打开”的确认。不要把挑战误做成登录绕过，也不要在公开服务上尝试凭据或扫描；公开记录本身已经证明了泄露链。归档材料能复现这条证据链，但不保留赛时动态页面的完整响应，flag 以下述题目仓库给出的官方值为校验。

题目给出的校验结果为：

```text
grey{be_careful_when_sh4ring_agent_traces!1!}
```

## 方法总结

- 核心技巧：把公开的 agent trace 当作 OSINT 证据源，优先阅读用户消息、agent 回复、工具动作摘要和初始数据中的结构化消息。
- 识别信号：题面强调 agent、computer use、共享 trace 或“看它做了什么”时，应先检查是否泄露了 URL、搜索结果、文件路径、截图文字或已执行动作。
- 复用要点：公开运行记录的风险不止是最终文本。中间检索、导航和工具调用的上下文同样可能暴露可直接复用的入口；WP 中应保存关键结论，而不是只留下可能失效的 trace 链接。
