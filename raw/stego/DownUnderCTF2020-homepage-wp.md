# DownUnderCTF 2020 - homepage

## 题目简述

题目把 flag 藏在比赛首页 SVG logo 的 405 个圆点中。页面脚本 `/scripts/splash.js` 含一段二进制数据，但其顺序与 DOM 中打乱的圆点位置绑定；直接按源码顺序分组会得到乱码。logo 的颜色渐变提示应按空间位置重新排列，并把两种 fill 颜色解释为二进制位。

## 解题过程

在浏览器开发者工具中检查 `#logo circle`，可以看到每个点有 `cx`、`cy` 坐标；鼠标经过后，页面脚本会把对应 bit 反映到 `style.fill`。先对全部圆点触发一次 `mouseover`，再按 `cx` 为主键、`cy` 为次键排序，也就是列从左到右、同列从上到下读取。

把下面脚本粘贴到首页 Console：

```javascript
const mouseover = new Event("mouseover");

document.querySelectorAll("#logo circle")
  .forEach(circle => circle.dispatchEvent(mouseover));

const dots = Array.from(document.querySelectorAll("#logo circle"));

dots.sort((a, b) => {
  const ax = a.cx.baseVal.value;
  const bx = b.cx.baseVal.value;
  const ay = a.cy.baseVal.value;
  const by = b.cy.baseVal.value;
  return ax === bx ? ay - by : ax - bx;
});

const zeroFill = dots[0].style.fill;
const bits = dots
  .map(dot => dot.style.fill === zeroFill ? "0" : "1")
  .join("");

let decoded = "";
for (let i = 0; i + 8 <= bits.length; i += 8) {
  decoded += String.fromCharCode(parseInt(bits.slice(i, i + 8), 2));
}

console.log(decoded);
```

按 8 bit 转 ASCII 后输出：

```text
DUCTF{i_turn3d_mysE1f_inT0_s0m3_f1ag}
```

本题的视觉关系已经完整表示为 SVG 坐标排序规则；仓库未提供必须保留的页面截图，且截图本身无法替代可复现的 DOM 读取逻辑，因此没有额外创建图片目录。

## 方法总结

- 核心技巧：把 SVG 元素的空间位置还原成稳定序列，再将两种颜色映射为 bit 并按字节解码。
- 识别信号：大量规则排列的 SVG 元素、两类颜色状态、题面强调 homepage/logo，以及源码中出现无法直接解码的 bit 串，都提示存在空间置换。
- 复用要点：DOM 顺序不等于视觉顺序；排序时要明确主次坐标，并先触发页面需要的事件让运行时状态落到元素属性上。末尾不足 8 bit 的填充不能强行解成字符。
