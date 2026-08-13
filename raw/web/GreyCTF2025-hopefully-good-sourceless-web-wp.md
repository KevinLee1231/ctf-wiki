# GreyCTF 2025 Hopefully Good Sourceless Web Writeup

## 题目简述

赛时题面不提供源码。页面有一个 flag 校验表单和一个任意文件读取表单。指纹与网络数据表明前端使用 Next.js 15 Server Actions；校验动作闭包捕获了后端返回的 flag，而 Next.js 为了把这个闭包动作交给客户端，会将捕获变量加密后随 React Server Components 数据发送。

目标不是猜 flag，而是利用文件读取取得 Next.js 的 Server Action 加密密钥，再解开浏览器已经收到的闭包参数。

## 解题过程

先通过响应头、`/_next/` 静态资源和表单提交格式识别 Next.js。用页面提供的文件读取动作读取：

```text
.next/server/server-reference-manifest.json
```

独立部署的构建产物中，该 manifest 保存 Server Actions 使用的加密密钥。与此同时，在初始 RSC 响应或提交校验表单时的网络请求中，可以找到校验动作对应的 bound argument：一段 Base64 字符串。源码所对应的闭包结构为：

```typescript
const flag = await getLatestFlag();

async function checkFlag(formData: FormData) {
  "use server";
  if (flag !== formData.get('flag')) throw new Error("Wrong flag!");
}
```

也就是说，这段密文中就包含被捕获的 `flag`。官方解法使用 manifest 中的 Base64 密钥做 AES-GCM 解密。Next.js 的封装格式是：先对整个 bound argument 做 Base64 解码，前 16 字节作为 IV，剩余部分作为带认证标签的密文：

```javascript
const originalPayload = atob(boundArgument);
const iv = originalPayload.slice(0, 16);
const ciphertext = originalPayload.slice(16);

const key = await crypto.subtle.importKey(
  'raw', bytes(atob(encryptionKey)), 'AES-GCM', true, ['decrypt']
);
const plain = await crypto.subtle.decrypt(
  { name: 'AES-GCM', iv: bytes(iv) }, key, bytes(ciphertext)
);
```

解密并按 UTF-8 输出闭包序列化内容，即可读到：

```text
grey{more_nextjs_server_actions_shenanigans}
```

## 方法总结

Server Action 的闭包变量不是公开明文，但其机密性完全依赖构建密钥。一旦应用同时暴露任意文件读取，manifest 中的密钥与客户端可见的 bound argument 就能拼成完整解密链。排查此类问题时应同时关注 RSC 网络载荷、`.next/server` 构建产物和 Action 捕获变量；修复的根本仍是移除任意文件读取，并避免让高价值秘密进入需要序列化到客户端的闭包。
