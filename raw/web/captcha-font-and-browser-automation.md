# CTF Web - CAPTCHA Font Analysis and Browser Automation

## 阅读定位

- 本卷处理 Web 挑战中由动态字体、短生命周期表达式和浏览器交互组成的 CAPTCHA 自动化。
- OCR、字体解析和 Selenium 只是求解手段；决定性约束来自页面生成与提交协议，因此物理归档在 Web，而不是 OSINT 或 AI/ML。

## Table of Contents

- [Dynamic-Font CAPTCHA: OCR and Glyph-Contour Routes (Square CTF 2018)](#dynamic-font-captcha-ocr-and-glyph-contour-routes-square-ctf-2018)
  - [路线一：Selenium + Tesseract](#路线一selenium--tesseract)
  - [路线二：TTF Glyph Contour Diffing](#路线二ttf-glyph-contour-diffing)

---

## Dynamic-Font CAPTCHA: OCR and Glyph-Contour Routes (Square CTF 2018)

题目在网页中生成算术表达式，并频繁改变字体字符与真实数字/运算符之间的映射。人工抄写赶不上刷新周期，但浏览器截图和响应中的字体文件都提供了可自动化入口。

### 路线一：Selenium + Tesseract

```python
from io import BytesIO
from PIL import Image
from selenium import webdriver
import pytesseract

driver = webdriver.Chrome()
driver.get(TARGET)
driver.execute_script("document.body.style.zoom='450%'")

image = Image.open(BytesIO(driver.get_screenshot_as_png()))
expression = pytesseract.image_to_string(image)
expression = expression.replace("x", "*").replace("{", "(").replace("}", ")")

# 仅在表达式已经过字符白名单校验时求值。
answer = eval(expression, {"__builtins__": {}}, {})
driver.execute_script(
    "document.getElementsByName('answer')[0].value=arguments[0]",
    answer,
)
driver.find_element("tag name", "form").submit()
```

OCR 路线适合字体轮廓仍接近普通字符、页面渲染稳定的情况。先放大并裁剪表达式区域，再为常见误识别字符做受控归一化；不要对未经白名单验证的任意字符串直接 `eval`。

### 路线二：TTF Glyph Contour Diffing

如果 OCR 因动态 cmap 映射而持续失败，应从 HTML/CSS 中提取内嵌 TTF。字符编码映射可能变化，但字形轮廓通常保持不变，可以把 `glyf` 表中的 contour 与参考字体逐一匹配。

```bash
ttx -t cmap -t glyf -d glyph-out challenge.ttf
# glyph-out 中保留 cmap 与每个 glyph 的轮廓 XML
diff glyph-out/challenge.glyf.xml reference/reference.glyf.xml
```

**关键结论：** 两条路线共享同一个提交自动化框架：获取当轮页面、恢复表达式、求值并在过期前提交。先尝试截图 OCR；若字符映射频繁变化或 OCR 不稳定，再利用字体轮廓与 cmap 的分离关系构造确定性映射。

**参考：** Square CTF 2018 — C8, writeups 12160, 12161, 12178
