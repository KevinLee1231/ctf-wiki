# DownUnderCTF 2020 - Leggos

## 题目简述

页面用 JavaScript 禁用右键菜单，并拦截 `Ctrl+C`、`Ctrl+V`、`Ctrl+U` 和 F6，试图阻止查看源码。真正的 flag 直接写在页面引用的外部 JavaScript 文件中。这些客户端限制只能影响交互，无法阻止浏览器已经下载的资源被查看。

## 解题过程

页面头部引用了：

```html
<script src="disableMouseRightClick.js"></script>
```

同时安装键盘和右键事件处理器：

```javascript
document.addEventListener('contextmenu', function(e) {
    e.preventDefault();
    alert('not allowed');
});
```

可以在地址栏使用 `view-source:`，也可以直接请求资源、在开发者工具 Network/Sources 中打开，或用命令行下载：

```bash
curl -s http://target/disableMouseRightClick.js
```

文件末尾有注释：

```javascript
// the source reveals my favourite secret sauce
// DUCTF{n0_k37chup_ju57_54uc3_r4w_54uc3_9873984579843}
```

因此 flag 为：

```text
DUCTF{n0_k37chup_ju57_54uc3_r4w_54uc3_9873984579843}
```

题目中的意大利面酱图片只是页面装饰，不承载解题信息，因此不提取为 WP 图片。

## 方法总结

浏览器必须先把 HTML、JavaScript、CSS 和图片下载到客户端才能渲染，前端事件监听器无法成为访问控制。遇到“禁右键”“禁快捷键”时，直接枚举页面引用资源、查看网络请求或用独立 HTTP 客户端读取即可；敏感信息绝不能靠隐藏 UI 或阻止查看源码来保护。
