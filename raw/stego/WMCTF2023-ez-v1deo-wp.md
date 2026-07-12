# EZ_v1deo

## 题目简述


原始方向为 `Steg`；归档时按主要障碍归入 `forensics`，因为解法主体是视频像素 LSB 隐写提取与载体恢复。

题目是视频 LSB 隐写。flag 信息藏在视频每一帧像素的最低有效位中，直接观看原视频不可见；需要逐帧提取 LSB 并重新生成高对比度视频。

## 解题过程

Video LSB 隐写，需要按帧提取后才能拿到 flag。

~~~python
import cv2
import numpy as np

def extract_lsb(frame):
    return frame & 1

def main(input_video, output_video):
    cap = cv2.VideoCapture(input_video)
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
    fps = int(cap.get(cv2.CAP_PROP_FPS))
    fourcc = cv2.VideoWriter_fourcc(*'XVID')

    out = cv2.VideoWriter(output_video, fourcc, fps, (width, height), isColor=True)

    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break

        lsb_frame = extract_lsb(frame) * (255, 255, 255)
        out.write(lsb_frame.astype(np.uint8))

    cap.release()
    out.release()
    cv2.destroyAllWindows()

if __name__ == '__main__':
    input_video = 'flag.avi'
    output_video = 'out.avi'
    main(input_video, output_video)
~~~

脚本对每一帧执行 `frame & 1`，再乘以 255，把最低位信息变成黑白图像写入新视频 `out.avi`。播放输出视频即可看到隐藏内容。

## 方法总结

- 核心技巧：逐帧提取视频像素 LSB 并重建可视化视频。
- 识别信号：题目给出视频且画面无明显异常时，应检查各通道最低位、帧间差分和 alpha/颜色通道。
- 复用要点：视频隐写要保留帧率、分辨率和编码参数，否则重建视频可能错帧或无法播放。
