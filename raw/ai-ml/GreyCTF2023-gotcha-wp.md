# GreyCTF2023 Gotcha

## 题目简述

服务连续生成四位大写字母验证码，要求同一会话在 120 秒内累计答对 100 次。错误答案只显示提示，不会清零分数，因此可以使用 OCR 自动化完成；判题核心是图像文字识别，而不是绕过会话或伪造分数。

## 解题过程

保持一个 `requests.Session`，从页面的 `data:image/jpeg;base64,...` 提取验证码。图片本身背景简单，限制字符集为大写英文字母即可显著减少误识别：

```python
def read_captcha(encoded):
    img = Image.open(BytesIO(base64.b64decode(encoded)))
    return pytesseract.image_to_string(
        img,
        config="--psm 7 -c tessedit_char_whitelist=ABCDEFGHIJKLMNOPQRSTUVWXYZ",
    ).strip()[:4]
```

每次 POST 后会重定向并生成下一张图，循环读取、识别和提交：

```python
with requests.Session() as session:
    page = session.get(base_url).text
    while "grey{" not in page:
        encoded = re.search(r'base64,(.*?)"', page).group(1)
        answer = read_captcha(encoded)
        page = session.post(submit_url, data={"captcha": answer}).text
```

分数达到 100 后，运行时配置返回：

```text
grey{I_4m_hum4n_n0w_059e3995f03a783dae82580ec144ad16}
```

仓库 README 把前缀写成了 `greyctf`，但部署 Dockerfile 注入的实际值是上述 `grey{...}`。

## 方法总结

自动验证码题首先应分析生成字符集、长度、背景噪声和计分状态机。本题没有失败惩罚，所以不需要为少量 OCR 错误设计复杂纠错；会话复用、单行识别模式和字符白名单已经足够。归档时应以实际部署配置为准，并明确记录 README 与运行值不一致之处。
