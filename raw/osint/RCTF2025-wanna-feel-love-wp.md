# Wanna Feel Love

## 题目简述

多阶段 OSINT/音频题，线索围绕 “I Feel Fantastic”。需要依次处理 spammimic 隐写、XM sample 的二进制节拍、Ghostarchive 视频元数据、AndroidWorld/PayPal 购买信息和 Find a Grave 页面。

## 解题过程

### 关键观察

多阶段 OSINT/音频题，线索围绕 “I Feel Fantastic”。

### 求解步骤

Challenge 1
将邮件的文字部分用 spammimic (https://www.spammimic.com/decode.cgi) 解码得到：
Challenge 2
challenge.xm 的 samples 有个 5: feel，是二进制数据，每 0.05s 是一个 bit
Don't just listen to the sound; this file is hiding an 'old relic.' Try
looking for the 'comments' that the player isn't supposed to see.
import soundfile as sf
import numpy as np

def solve_ask_clustering(filename):
    print(f"正在分析文件: {filename}")
    data, samplerate = sf.read(filename)

    if len(data.shape) > 1:
        data = data[:, 0]

    # 1. 计算每个 0.05s 切片的能量 (RMS)
    bit_duration = 0.05
    samples_per_bit = int(samplerate * bit_duration)
    total_bits = len(data) // samples_per_bit

    rms_values = []

    for i in range(total_bits):
        start = i * samples_per_bit
        end = start + samples_per_bit
        chunk = data[start:end]

        # 计算均方根值 (RMS) 作为能量指标，比单纯看最大值更准确
        if len(chunk) > 0:
            rms = np.sqrt(np.mean(chunk**2))
            rms_values.append(rms)
        else:
            rms_values.append(0)

    # 2. 寻找动态阈值

    sorted_rms = sorted(rms_values)
    # 找相邻差值最大的位置，通常就是 0 和 1 的分界线
    diffs = np.diff(sorted_rms)
    split_index = np.argmax(diffs)

    # 阈值设为分界线两边的中间值
    threshold = (sorted_rms[split_index] + sorted_rms[split_index + 1]) / 2
    # 3. 重新生成二进制串
    binary_bits = []
    for rms in rms_values:
        if rms > threshold:
            binary_bits.append('1')
        else:
            binary_bits.append('0')

    binary_str = "".join(binary_bits)

Challenge 3
打开 https://ghostarchive.org/varchive/rLy-AwdCOmI ，可看到：
<video>  源指向 .../rLy-AwdCOmI.mp4 ，即视频 ID。
同一段落显示 Uploader: Creepyblog  与 Original upload date: Thu, 16 Apr 20
09 。发现交上去不对。
后续找到Mallory Marlowe 的 Substack 文章（https://mallorymarlowe.substack.com/p/
4-i-feel-fantastic-hey-hey-hey ）说明 Creepyblog 在 2009-04-15 上传。
答案
Video Id：rLy-AwdCOmI
Upload Date：2009-04-16
Uploader：Creepyblog
Challenge 4
1. 打开
（Android Music Videos 订购页）：
    # 检查是否匹配
    if binary_str.startswith("0100100100"):
        print(">>> 开头匹配成功！正在解码...")
    else:
        print(">>> 开头有差异，请检查输出的阈值是否合理。")
    print("-" * 30)
    # 5. 转 ASCII
    result = ""
    for i in range(0, len(binary_str), 8):
        byte = binary_str[i:i+8]
        if len(byte) == 8:
            try:
                result += chr(int(byte, 2))
            except:
                result += "?"
    print("解码结果:")
    print(result)
    print("-" * 30)
if __name__ == "__main__":
    solve_ask_clustering('Feel.flac')
I Feel Fantastic heyheyhey
https://androidworld.com/prod68.htm
页面底部提供 PayPal 表单，action="<https://www.paypal.com/cgi-bin/webscr
>" ，即唯一购买入口。
表单中的 business  字段为 crwillis@androidworld.com （Chris Willis），即售卖
者/收款邮箱。
2. 创作年份：
Substack 文章写明 Tara 的音乐视频诞生于 2003-2004。该页面列出的 5 首曲目即在此期间
制作。
尝试提交邮箱发现不正确，找到一篇晒单的blog，可以找到Sender是Chris Wills： https://yitzilit
t.medium.com/the-story-behind-i-feel-fantastic-tara-the-singing-android-and-john-berger
on-fc83de9e8f36
答案
Purchase Link：https://androidworld.com/prod68.htm
Sender（付款接收邮箱）：Chris Willis
Creation Year：2004 （音乐首次成形于 2003-2004，DVD 版在 2004 年最终成形）
Challenge 5
1. 通过 Bing RSS 与 Reddit 贴讨论，提到“John Louis Bergeron” 葬于 Resurrection Park
Cemetery。
2. 进一步访问 Find a Grave，检索 John Louis Bergeron，即可确认其纪念页面（题目要求去掉末
尾 /）
https://www.findagrave.com/memorial/63520325/john_louis-bergeron
答案
Developer’s Digital Grave：https://www.findagrave.com/memorial/63520325/john_
louis-bergeron

### 参考链接补充

本题的外链是分阶段证据链，不应只保留 URL：

- `spammimic` 的 `decode.cgi` 是 Challenge 1 的文本隐写解码器；邮件正文看似垃圾广告文本，实际应丢进 decoder 得到下一阶段提示。
- Ghostarchive 页面用于验证视频 ID、上传者和原始上传时间：`rLy-AwdCOmI` 指向归档视频，页面元数据给出 `Uploader: Creepyblog` 与原始上传日期线索；最终提交时以题目接受的 `2009-04-16` 为准。
- Mallory Marlowe/Substack 与 Medium 文章用于补背景：`I Feel Fantastic`/Tara 相关音乐视频形成于 2003-2004，解释为什么 Creation Year 应取 `2004`，并辅助区分 uploader、creator、seller/sender 这几个容易混淆的字段。
- AndroidWorld 的 `prod68.htm` 是购买入口证据：页面提供 PayPal 购买表单，正文中的 `business`/收款字段用于定位销售者邮箱或姓名；若题目不接受邮箱，应继续用晒单/文章交叉确认显示名。
- Find a Grave 页面是最后一阶段的确认页：检索 `John Louis Bergeron` 并确认 memorial URL，提交时按题目要求去掉末尾 `/`。

### PDF 外链

- <https://www.spammimic.com/decode.cgi>
- <https://ghostarchive.org/varchive/rLy-AwdCOmI>
- <https://mallorymarlowe.substack.com/p/4-i-feel-fantastic-hey-hey-hey>
- <https://androidworld.com/prod68.htm>
- <https://yitzilitt.medium.com/the-story-behind-i-feel-fantastic-tara-the-singing-android-and-john-bergeron-fc83de9e8f36>
- <https://www.findagrave.com/memorial/63520325/john_louis-bergeron>

## 方法总结

- 核心技巧：文本隐写、音频比特恢复和互联网档案 OSINT。
- 识别信号：邮件文字像垃圾文本、XM sample 可疑、视频元数据和购买页面可交叉验证。
- 复用要点：音频题不要只听声音，优先检查 sample、comment、metadata 和外部存档。
