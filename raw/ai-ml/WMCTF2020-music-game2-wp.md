# Music_game2

## 题目简述

本题延续上一题的语音识别服务，但目标变成构造能欺骗后台神经网络的音频。非预期做法可以随机向音频中掺噪声碰撞识别结果；预期做法是基于官方 notebook 中的模型构造对抗样本：对 `example.wav` 的 MFCC 特征做反向传播调整，使模型预测偏向指定类别，再把调整后的 MFCC 近似还原成 wav 并上传。

## 解题过程

经过前面两个开胃菜的预热，前期准备工作应该完成了。

这道题说白了就是在音频里掺入些噪音，让后台的神经网络作出错误的判断。玩过上一题的师傅都知道，这个识别有点拉。

既然是掺杂，我比赛前突发奇想了一种“佛系解题”法

```python
while True:
    sig1, mod = sf.read('example.wav')
    for i in range(1):
        num = random.randint(0, len(sig1))
        sig1[num] = sig1[num] + random.random() - 0.5
        sig1[num] = sig1[num] + random.random() - 0.5
        sig1[num] = sig1[num] + random.random() - 0.5
    sf.write('/tmp/test.wav', sig1, mod)
    wavd = diff('/tmp/test.wav')
    mfccd = checkdifferent('/tmp/test.wav')
    (num, lab) = detect('/tmp/test.wav')
    print("wavd:{}  mfccd:{}  type:{}  mun:{}".format(wavd, mfccd, lab, num))
    if lab != 0 and num > 0.9:
        break
```

看到random了吧，就是随机往里写点随机的数据。这样居然真的可以，只是因为题目条件比较苛刻，向左很难成功欺骗。（比赛时有两个师傅提到向左不能伪造，听了他们伪造的音频，估计是用了相似的手段。）

下面是预期解。原 WP 引用的外部资料，本质上只依赖两个概念：一是 adversarial example，即在输入上加入人类不明显但模型敏感的微小扰动，使模型输出被推向攻击者指定类别；二是 MFCC 反变换到 wav 会有损耗，因此需要在“特征空间优化”和“音频重建”之间反复校正。后来才知道 librosa 已经有 `mfcc_to_audio` 方法了。

这次拿到的官方 notebook 中，脚本直接加载题目模型 `model.h5`，把 `example.wav` 转成 MFCC，并统一成 `30 * 20` 的输入格式：帧数超过 30 时从头尾裁掉，少于 30 时复制最后一帧补齐。`object_type_to_fake = 3` 用来指定要伪造成的目标类别。

预期解的关键不是随机改 wav 字节，而是让 Keras 对目标类别输出对输入 MFCC 求梯度，然后沿梯度方向更新 MFCC。由于 MFCC 还原成音频后再提取 MFCC 会产生误差，脚本每轮都会先把当前 `hacked_mfcc` 还原成 wav，再重新提取 MFCC 做预测和梯度计算，并用与原始 MFCC 的平均差异限制扰动幅度。

核心流程可以概括为：

```python
model = models.load_model('model.h5')
model_input_layer = model.layers[0].input
model_output_layer = model.layers[-1].output

object_type_to_fake = 3
original_mfcc = get_wav_mfcc('example.wav')
hacked_mfcc = np.copy(original_mfcc).reshape(1, 30, 20)
learning_rate = 1000

cost_function = model_output_layer[0, object_type_to_fake]
gradient_function = K.gradients(cost_function, model_input_layer)[0]
grab_cost_and_gradients_from_model = K.function(
    [model_input_layer, K.learning_phase()],
    [cost_function, gradient_function],
)

while True:
    test_mfcc = get_mfcc(get_mfcc_wav(hacked_mfcc), 16000).reshape(1, 30, 20)
    cost, gradients = grab_cost_and_gradients_from_model(test_mfcc)
    diff = checkdifferent(test_mfcc.reshape(30, 20))
    if cost > 0.95 and diff < 4:
        break

    newhacked_mfcc = hacked_mfcc + gradients * learning_rate
    if checkdifferent(get_mfcc(get_mfcc_wav(newhacked_mfcc), 16000)) >= 4:
        learning_rate /= 2
    else:
        hacked_mfcc = newhacked_mfcc
```

最后再把稳定后的 MFCC 反变换成音频，写出 `yours.wav`：

```python
while True:
    recon = get_mfcc_wav(hacked_mfcc)
    librosa.output.write_wav('yours.wav', recon, 16000)
    if checkdifferent(get_wav_mfcc('yours.wav')) < 4:
        break
```

本题解法不唯一，这可能不是最优解，其他解法可以围绕同一模型和同一输入特征继续改进。

最后上传的问题不知道大家是不是做上一题时就有解决，前端自己加一个上传按钮就行了，或者用requests做...方法很多 外放估计行不通了。

## 方法总结

语音模型对抗题的识别信号是：给出模型、样例音频和目标分类，且普通录音/TTS 已不足以稳定通过。可复用流程是先把音频转为模型实际使用的特征表示，例如 MFCC；在特征空间按目标类别反向传播；再把特征近似还原成 wav 并重新送入模型验证。随机加噪可作为低成本碰撞尝试，但稳定解应保留 loss、目标类别、梯度更新和上传验证流程。
