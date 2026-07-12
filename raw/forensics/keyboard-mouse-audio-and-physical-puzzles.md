# CTF Misc - Keyboard, Mouse, Audio and Physical Puzzles

## 阅读定位

键盘/鼠标轨迹、音频谜题、3D 打印、视频与物理媒介类杂项技巧。


## USB Mouse PCAP Reconstruction

**模式（Hunt and Peck）：** USB HID mouse traffic captures on-screen keyboard typing. Use USB-Mouse-Pcap-Visualizer, extract click coordinates (falling edges), cumsum relative deltas for absolute positions, overlay on OSK image.


---

## Audio Challenges

```bash
sox audio.wav -n spectrogram  # Visual data
qsstv                          # SSTV decoder
```


---

## 3D Printer Video Nozzle Tracking (LACTF 2026)

**模式（flag-irl）：** Video of 3D printer fabricating nameplate. Flag is the printed text.

**Technique:** Track nozzle X/Y positions from video frames, filter for print moves (top/text layer only), plot 2D histogram to reveal letter shapes:
```python
# 1. Identify text layer frames (e.g., frames 26100-28350)
# 2. Track print head X position (physical X-axis)
# 3. Track bed X position (physical Y-axis from camera angle)
# 4. Filter for moves with extrusion (head moving while printing)
# 5. Plot as 2D scatter/histogram -> letters appear
```
