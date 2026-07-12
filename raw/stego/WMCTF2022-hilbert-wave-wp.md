# hilbert_wave

## 题目简述

题目给出 104 个 `.wav` 音频，题面提示“只会玩 9 宫格数独”和“希尔伯特曲线是一条可以充满空间的曲线”。每个音频实际编码了一张 $128 \times 128$ 的 RGB 图片：三声道分别对应 RGB 三个颜色通道，采样值不超过 255；像素顺序不是普通逐行扫描，而是按 7 阶 Hilbert 曲线展开。解题目标是把每个 wav 逆 Hilbert 映射还原成数字图片，识别每张图缺失的数字，把 104 个数字拼成大整数，再 `long_to_bytes` 得到 flag。

## 解题过程

首先是一堆音频，au查看后可以看见有间隙的波纹点

直接用wave读一下数据可以发现其值都不大于255（ps：原始数据是49152的一维信息，但是通过声道可以知道是RGB三个颜色分别分到了三个音轨上面），易得其本来为图片，且可以发现 $49152=128\times128\times3$。

再根据题目名称hilbert_wave可以知道其通过了希尔伯特的处理，逆处理一波可以得到图像（ps：下面脚本为更好的可以图片ocr，进行了二值化处理）

~~~python
import wave
import matplotlib.pyplot as plt
import numpy as np
from PIL import Image
import time
from tqdm import tqdm

def wav_to_pic(wav,pic):
    plt.rcParams['font.sans-serif'] = ['SimHei']
    plt.rcParams['axes.unicode_minus'] = False

    f = wave.open(wav, "rb")
    params = f.getparams()
    nchannels, sampwidth, framerate, nframes = params[:4]
    str_data = f.readframes(nframes)
    # print(nchannels, sampwidth, framerate, nframes)
    f.close()

    wave_data = np.fromstring(str_data, dtype=np.short).reshape((16384, 3))
    def _hilbert(direction, rotation, order):
        if order == 0:
            return

        direction += rotation
        _hilbert(direction, -rotation, order - 1)
        step1(direction)

        direction -= rotation
        _hilbert(direction, rotation, order - 1)
        step1(direction)
        _hilbert(direction, rotation, order - 1)

        direction -= rotation
        step1(direction)
        _hilbert(direction, -rotation, order - 1)

    def step1(direction):
        next = {0: (1, 0), 1: (0, 1), 2: (-1, 0), 3: (0, -1)}[direction & 0x3]

        global x, y
        x.append(x[-1] + next[0])
        y.append(y[-1] + next[1])

    def hilbert(order):
        global x, y
        x = [0,]
        y = [0,]
        _hilbert(0, 1, order)
        return (x, y)

    x, y = hilbert(7)
    inx = []
    for i in range(len(x)):
        inx.append((x[i],y[i]))
    inx = np.array(inx)
    new_p = Image.new('RGB', (128,128))
    for i in range(len(inx)):
        if tuple(wave_data[i]) != (255,255,255):
            new_p.putpixel(inx[i], (0,0,0))
        else:
            new_p.putpixel(inx[i], (255,255,255))
    
    new_p.save(pic)

for i in tqdm(range(104)):
    wav_to_pic('wavs/'+str(i)+'.wav','res/'+str(i)+'.png')
~~~

然后可以发现上面有部分是缺省了一个数字而有的没有缺省，把没有缺省的代入0，缺省的代入缺省的数字。这里用了百度的ocr（图片不多，也可以人工去查看），把所有数字凑起来以后long_to_bytes即可得到flag。

~~~python
res2 = []
import requests,base64,json
from urllib.parse import quote_from_bytes
import time
from tqdm import tqdm

requests.packages.urllib3.util.ssl_.DEFAULT_CIPHERS = 'ALL'
url='https://aip.baidubce.com'
path = '/rest/2.0/ocr/v1/accurate_basic'
headers = {}
headers['Content-Type'] = 'application/x-www-form-urlencoded;charset=UTF-8'
headers['Host'] = 'aip.baidubce.com'
params = {}
params['access_token'] = '***********************************************************'

for ii in tqdm(range(104)):
    time.sleep(1)
    body = 'image='+quote_from_bytes(base64.b64encode(open('res/'+str(ii)+'.png','rb').read()))
    r = requests.Session()
    rr = r.post(url+path,data=body,headers=headers,params=params,verify=False)
    res = json.loads(rr.text)

    f_res = ""
    print(res)
    for i in range(len(res["words_result"])):
        f_res += res["words_result"][i]["words"]
    print(f_res)
    res2.append(str(f_res))
    

print(res2)
res3 = ''
for i in res2:
    for j in [1,2,3,4,5,6,7,8,9]:
        flag = 1
        if str(j) not in i:
            res3+=str(j)
            flag = 0
            break
    if flag:
        res3+='0'

print(res3)
from Crypto.Util import number
print(number.long_to_bytes(int(res3)))
~~~

## 方法总结

本题核心是识别“一维音频数据其实是二维图像数据”，再根据题名使用 Hilbert 曲线逆映射恢复空间顺序。$49152=128\times128\times3$、三声道、采样值不超过 255 是判断 RGB 图片的关键证据。

识别信号是：大量同尺寸 wav、采样点可作为 0 到 255 的像素值、题名包含 `hilbert`、题面提到九宫格数字。7 阶 Hilbert 曲线刚好覆盖 $128\times128$ 个点。

复现时先把 104 张图按 Hilbert 顺序还原并二值化，再识别每张图中 1 到 9 缺失的数字；若 1 到 9 都不缺，则记为 0。最终把 104 位数字串转成整数并执行 `long_to_bytes`。
