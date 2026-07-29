# I Love This World

## 题目简述

附件 `ilovethisworld.svp` 是 Synthesizer V 工程文件。题面提示“让电脑唱出 flag”，并明确说明日文歌词中的 flag 是假的。直接查看文件可知它不是封闭二进制，而是一份 JSON，包含轨道、音符、歌词、音高和发音参数。

真正的隐藏通道是每个音符的 `phonemes` 字段。项目使用的音素集在轨道配置中写得很清楚：

```json
{
  "database": {
    "name": "Eleanor Forte (Lite)",
    "language": "english",
    "phoneset": "arpabet"
  }
}
```

也就是说，表面日文歌词只是伪装，英文 ARPABET 发音才是按音符顺序嵌入的消息。

## 解题过程

### 提取音符发音

工程的目标音符位于 `library[0].notes`。依照数组原有顺序取出 `phonemes`：

```powershell
$project = Get-Content -Raw "ilovethisworld.svp" | ConvertFrom-Json
$project.library[0].notes | ForEach-Object { $_.phonemes }
```

得到 49 组音素：

```text
eh f, eh l, ey, jh iy, k ow l ax n,
eh s, iy, k ey, ey, ay,
ow p ax n k er l iy b r ae k ih t,
eh s, ow, eh m, iy, w ah n, z iy, eh f, ey, aa r, ey,
d ah b ax l y uw, ey, w ay, t iy, eh m, aa r, w ah n, f ay v,
eh s, iy, k y uw, y uw, iy, eh l, t iy, ow, ow, y uw, aa r,
d iy, aa r, iy, ey, eh m, t iy, d iy, w ay,
k l ow s k er l iy b r ae k ih t
```

### 将 ARPABET 读成字符

这里不是把整串做语音识别，而是逐组读出它们代表的英文字母、数字和标点名称。例如：

| ARPABET | 读音 | 字符 |
|---|---|---|
| `eh f` | F | `F` |
| `eh l` | L | `L` |
| `k ow l ax n` | colon | `:` |
| `ow p ax n k er l iy b r ae k ih t` | open curly bracket | `{` |
| `w ah n` | one | `1` |
| `f ay v` | five | `5` |
| `k l ow s k er l iy b r ae k ih t` | close curly bracket | `}` |

按顺序转写后得到完整语句：

```text
FLAG:SEKAI{SOME1ZFARAWAYTMR15SEQUELTOOURDREAMTDY}
```

去掉前面的 `FLAG:`，最终 flag 为：

```text
SEKAI{SOME1ZFARAWAYTMR15SEQUELTOOURDREAMTDY}
```

其中 `1Z` 和 `15` 分别来自 “one z” 与 “one five”，不能凭自然语言观感擅自改成 `IS`。题面给出的正则 `SEKAI\{[A-Z0-9]+\}` 也能用于校验转写结果。

## 方法总结

- 核心技巧：把 `.svp` 当作结构化 JSON 检查，比较可见歌词与音符发音字段，定位真正的隐藏通道。
- 识别信号：歌词语言和 `phoneset: arpabet` 不一致、题面强调“唱出”且给出严格 flag 正则，都提示应读取发音而不是翻译日文。
- 复用要点：工程文件、字幕工程和音序器项目常把信息藏在事件级参数中。提取时必须保留原始事件顺序，并用格式约束区分字母与同音数字，避免主观纠错。
