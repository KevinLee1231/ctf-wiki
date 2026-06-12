# D3MUG

## 题目简述

这道题目作为 D^3CTF 逆向分区打头阵的第一道题目，其实是做好了当签到题的打算的。对本题纯静态分析的难度可能不太容易，需要花上一些时间。但是发现题目把加密模块 libd3mug.so  和游戏逻辑给拆开之后，就可以简单的理解一下游戏逻辑，然后通过 frida hook 等框架拦截修改数据或者直接调用函数，从而直接跑出 flag。

题目是 Unity3D + IL2CPP Android 游戏。游戏逻辑负责读取谱面和记录 note 击打时间，加密模块 `libd3mug.so` 暴露 `update` 与 `get` 两个外部函数；只要把正确谱面的所有 hitpoint 传给 `update`，结束后调用 `get` 就能得到 flag。

## 解题过程

首先将apk解包，了解到题目是一个Unity3D，通过IL2CPP编译的游戏，然后打开 Il2CppDumper 对libil2cpp.so 和 global-meta.dat 进行处理。

对 DummyDll 里面的内容进行分析，可以拿到游戏中部分 MonoBehaviour 的符号表，将 Dump 出来的符号信息导入 IDA，然后两边对照着看，可以得知游戏在运行期间通过 GameManager 和

ScoreScene 引用了两个外部函数，一个是 update，参数为时间；另一个是 get，返回值是 IntPtr，应该是用来转换 C++ 字符串数据的。

通过查看 IDA 中 GameManagerNoteHit 和 GameManagerNoteMissed 以及 ScoreScene__Start 我们可以大致推断出游戏的逻辑，

游戏在运行时通过 NoteObject 的状态通知 GameManager 调用 NoteHit 函数或者 NoteMissed 函数，这两个函数会通过 NoteObject 的参数获取到当前的音乐时间，然后调用一个外部的 update 函数：

关键调用关系是：`NoteObject` 触发 `GameManager::NoteHit` 或 `GameManager::NoteMissed`，这两个函数从 `NoteObject` 取当前音乐时间，再把击打时间传给外部 `update` 函数。

在游戏结束进入分数结算时，ScoreScene__Start 会调用一个外部的 get 函数：

`ScoreScene::Start` 最后调用外部 `get` 函数取回结果字符串。

理清楚题目的基本流程之后就可以动手了，题目把每个note的击打时间通过update传递给libd3mug.so的update函数，游戏结束后通过get函数获取到一个字符串，那我们可以把资源中的谱面拆出来（资源中有三个谱面，题目只使用了Chrome VOX的那一个），然后直接调库update最后get出来的就是flag。

使用frida框架可以很轻松地做到这一点：

```javascript
let UpdateFunc = new NativeFunction(Module.findExportByName("libd3mug.so",
"update"), 'void', ['int'])
for (let i = 0; i < hitpoints.length; i++)
    UpdateFunc(hitpoints[i]);
let GetFunc = new NativeFunction(Module.findExportByName("libd3mug.so", "get"),
'pointer', []);
var result = GetFunc();
console.log(ptr(result));
```

如果要静态分析解题也是可行的，libd3mug.so中的算法是一个类似于feistel的东西，通过一个静态的种子初始化mt19937随机数生成器，然后先生成随机数判定是否要进入下一步解密，在解密中重新生成随机数作为key，然后选取一个偏移在数据中取出32字节，加密其中的16字节并将左右位置互换，将每个note的击打时间都录入update函数，即可解出正确答案。会比较费时费力，可以自行尝试。

## 方法总结

- 核心技巧：Unity IL2CPP 题先用 Il2CppDumper 恢复 `DummyDll` 和符号，再把 C# 侧 `MonoBehaviour` 调用与 native so 导出函数对应起来。
- 动态路线：资源中提取题目实际使用的谱面，把所有 note 击打时间按顺序传入 `libd3mug.so!update`，最后调用 `get` 取回结果。
- 静态路线：`libd3mug.so` 内部是近似 Feistel 的解密过程，使用固定种子初始化 MT19937，根据 note 时间驱动条件和 key 生成；能静态复现，但工作量明显更大。
