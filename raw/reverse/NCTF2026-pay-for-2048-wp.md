# Pay For 2048

## 题目简述

Electron 逆向题。解包 `resources/app.asar` 后，从 `license-service.js` 与 `wasm-bridge.js` 找到 JS 调 WASM 的 key 校验和 flag 解密接口，分析 WASM 逻辑生成有效 license 并解锁。

## 解题过程

Electron逆向，先把resource的app.asar解压找到 license-service.js wasm-bridge.js 通过这个接口js层调用wasm层的key校验和flag解密，对wasm分析即可。

### Exploit

```javascript
const { LicenseService } = require("./src/license-service");
const { WasmBridge } = require("./src/wasm-bridge");
const EXPECTED_DIGEST = 0xFC97CA2F;
function normalizeKey(key) {
    return key.split("-").slice(1).join("");
}
function customDigest(input) {
    const bytes = Buffer.from(input, "ascii");
    let acc = 0x1357;
    for (let index = 0; index < bytes.length; index += 1) {
        const lane = index % 2 === 0 ? 0x45D9 : 0x9E37;
        const mixed = (((bytes[index] + lane) >>> 0) * (index + 11)) >>> 0;
        acc = (((acc << 3) | (acc >>> 29)) >>> 0) ^ mixed;
        acc = (acc + (((bytes[index] << (index % 5)) >>> 0) ^ 0xA5A55A5A)) >>>
0;
    }
    return acc >>> 0;
}
function toLeet(text) {
    return text
        .replace(/A/g, "4")
        .replace(/B/g, "8")
        .replace(/E/g, "3")
        .replace(/G/g, "6")
        .replace(/I/g, "1")
        .replace(/O/g, "0")
        .replace(/S/g, "5")
        .replace(/T/g, "7")
        .replace(/Z/g, "2");
}
function deriveKeyFromWasmAnalysis() {
    const inferredWords = ["RUST", "WASM", "2048"];
    const candidate =
`NCTF-${toLeet(inferredWords[0])}-${toLeet(inferredWords[1])}-${inferredWords[2
]}`;
    const normalized = normalizeKey(candidate);
    if (customDigest(normalized) !== EXPECTED_DIGEST) {
        throw new Error(`derived key does not satisfy wasm digest:
${candidate}`);
    }
    return candidate;
}
async function main() {
    const wasmBridge = new WasmBridge();
    const licenseService = new LicenseService(wasmBridge);
    await licenseService.init();
    const key = deriveKeyFromWasmAnalysis();
    console.log("derived key:", key);
    const verifyContext = {
        score: 512,
        steps: 42,
        maxTile: 256,
        board: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 2, 4, 16, 256],
        sessionToken: ""
    };
    const verifyResult = await licenseService.verifyKey(key, verifyContext);
    console.log("verifyResult:", JSON.stringify(verifyResult, null, 2));
    if (!verifyResult.ok) {
        process.exitCode = 1;
        return;
    }
    const unlockContext = {
        score: 4096,
        steps: 128,
        maxTile: 2048,
        board: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 2, 4, 16, 2048],
        sessionToken: verifyResult.data.sessionToken
    };
    const unlockResult = await licenseService.unlockFlag(key, unlockContext);
    console.log("unlockResult:", JSON.stringify(unlockResult, null, 2));
    if (!unlockResult.ok) {
        process.exitCode = 1;
    }
}
main().catch((error) => {
    console.error(error);
    process.exitCode = 1;
});
```

## 方法总结

- 核心技巧：Electron asar 解包 + JS/WASM 授权校验逆向。
- 识别信号：Electron 游戏把 license 校验和 flag 解密拆在 JS 与 WASM 间时，要从 bridge 接口定位输入输出约束。
- 复用要点：先用 Node 直接调用应用内服务代码复现 verify/unlock，再补齐 WASM 侧 key 派生。
