# Speak Softly Love

## 题目简述

OSINT 题围绕 DOSMid、Mateusz Viste、gopher 站点和比特币捐赠信息。解法从音乐视频、个人主页、音频文件、SVN 记录和 gopher 页面逐步定位答案。

## 解题过程

### 关键观察

OSINT 题围绕 DOSMid、Mateusz Viste、gopher 站点和比特币捐赠信息。

### 求解步骤

Q1：8ssDGBTssUI
DOSMid: The Godfather theme played on an 8086 computer
Q2：r178
Q3：从个人网站：Mateusz Viste - 主页 --- Mateusz Viste - homepage 中找到音频 https://mate
usz.viste.fr/mateusz.ogg
Q4：个人网站上有gopher链接：gopher://gopher.viste.fr/
找到bitcoin donate
Q4：16TofYbGd86C7S6JuAuhGkX4fbmC9QtzwT
RCTF{wh3n_8086_s4ng_s0f7ly_0f_l0v3}

### 参考链接补充

外链在这题里分别承担不同证据角色：

- YouTube 视频 ID `8ssDGBTssUI` 对应 DOSMid 在 8086 上演奏《The Godfather》主题曲，给出题目名 “Speak Softly Love” 与 DOSMid 作者线索。
- Mateusz Viste 主页说明其长期维护 retro-computing/open-source 项目，并显式提供 gopherspace 入口；这把搜索方向从公开视频转到作者个人站点和 gopher/SVN 服务。
- `mateusz.ogg` 是主页暴露的音频文件线索，用于确认站点归属和作者信息，不是最终 flag 本身。
- PDF 中补回的 `svn log svn://svn.mateusz.fr/dosmid` 结果给出 `r178`；gopher 站点中的 donate 信息给出比特币地址 `16TofYbGd86C7S6JuAuhGkX4fbmC9QtzwT`。

这些证据合在一起形成答案链：视频定位 DOSMid/作者，个人站定位 gopher/SVN，SVN 与 gopher 分别回答 revision 和 donation address。

### PDF 外链

- <https://www.youtube.com/watch?v=8ssDGBTssUI>
- <https://mateusz.viste.fr/>
- <https://mateusz.viste.fr/mateusz.ogg>

### 跨页补回：SVN 与 gopher 证据

svn log svn://svn.mateusz.fr/dosmid
...
r178 | mv_fox | 2016-05-09 01:21:38 +0800 (Mon, 09 May 2016) | 1 line

if too many 'soft' errors occur in a row, dosmid aborts (protects against
'soft errors loops', typically with playlist filled with non-existing files)
lynx gopher://gopher.viste.fr/

## 方法总结

- 核心技巧：音乐/人物 OSINT。
- 识别信号：题面线索指向具体老软件、作者主页、音频文件和 gopher 服务。
- 复用要点：网页、SVN、gopher、捐赠地址都可能是同一人物线索链的一部分。
