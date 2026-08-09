# The Gang 3

## 题目简述

上一题的帖子回复中留下 AES-GCM 挑战。解密后得到 Discord 邀请；聊天内容描述了 Bengaluru 机场附近一座 110 英尺高的雕像，需要提交其坐标。

## 解题过程

按照回复中给出的密钥、nonce、密文和认证标签执行 AES-GCM 解密。认证通过后的明文是 Discord 邀请，说明参数和消息完整无误。

服务器聊天给出三条关键信息：会面点在 Bengaluru 机场 Terminal 2 附近、雕像高约 110 英尺、机场以雕像人物命名。它们共同指向 Kempegowda International Airport 的 Nadaprabhu Kempegowda 雕像。

在地图中定位雕像并读取坐标，按题面保留三位小数：

```text
n00bz{13.199,77.682}
```

## 方法总结

AES-GCM 解密不仅要得到明文，还要验证认证标签。地理定位部分则需把高度、航站楼和人物命名三条线索交叉使用；单独搜索机场中心点可能得到错误坐标。
