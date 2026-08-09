# Level 729 易画行

## 题目简述

题目要求在 Sepolia 测试网上完成一次 NFT 转账：将指定 ERC-721 合约中的 `tokenId = 1` 转给目标地址。附件给出了基于 `ethers.js` 的交互框架；真正需要补全的是合约地址、接收地址以及转账调用。

- NFT 合约：`0x0c5ABBB0743a3Ac77C2c301eD63810F3353c59F8`
- 接收地址：`0x74520Ad628600F7Cc9613345aee7afC0E06EFd84`
- Token ID：`1`
- 网络：Sepolia

这道题的关键不只是发起交易，还要沿链上记录找到 NFT 的铸造交易，解码其中的 `tokenURI`，再从 IPFS 元数据中读取 flag。

## 解题过程

先补全附件中的转账逻辑。连接 Sepolia RPC 后，用自己的测试网账户签名，并调用 ERC-721 的 `safeTransferFrom`：

```javascript
const { ethers } = require("ethers");
const config = require("./config.json");

const provider = new ethers.JsonRpcProvider("https://sepolia.drpc.org");
const wallet = new ethers.Wallet(config.privateKey, provider);

const nftAddress = "0x0c5ABBB0743a3Ac77C2c301eD63810F3353c59F8";
const recipient = "0x74520Ad628600F7Cc9613345aee7afC0E06EFd84";
const tokenId = 1n;

const abi = [
  "function safeTransferFrom(address from, address to, uint256 tokenId)",
  "function ownerOf(uint256 tokenId) view returns (address)",
  "function tokenURI(uint256 tokenId) view returns (string)"
];

async function main() {
  const nft = new ethers.Contract(nftAddress, abi, wallet);
  const owner = await nft.ownerOf(tokenId);
  if (owner.toLowerCase() !== wallet.address.toLowerCase()) {
    throw new Error(`当前账户不是 token ${tokenId} 的持有者：${owner}`);
  }

  const tx = await nft.safeTransferFrom(wallet.address, recipient, tokenId);
  console.log(`transaction: ${tx.hash}`);
  await tx.wait();
  console.log("transfer confirmed");
}

main().catch(console.error);
```

私钥只应保存在本地配置或环境变量中，不能提交到仓库。账户还需要少量 Sepolia ETH 支付 gas。

完成转账后，在 [Sepolia Etherscan 的接收地址页面](https://sepolia.etherscan.io/address/0x74520Ad628600F7Cc9613345aee7afC0E06EFd84#nfttransfers) 查看 NFT 转入记录。顺着同一个 `tokenId` 的历史交易向前追溯，可以定位到铸造交易：

```text
0xb8f2cad0956513e6d466b5bec77cdfcaafb45871454fc7463b63865893458f62
```

在 [该交易的 Input Data](https://sepolia.etherscan.io/tx/0xb8f2cad0956513e6d466b5bec77cdfcaafb45871454fc7463b63865893458f62) 中选择解码视图，可以看到铸造函数的动态字符串参数。这里记录的 NFT 元数据 URI 为：

```text
ipfs://QmUusCYT8GTNgbDk5WAHZsHmHSxqcxuHov94inyFcpPqM6
```

`ipfs://` URI 可通过任意可信 IPFS 网关读取，例如：

```text
https://ipfs.io/ipfs/QmUusCYT8GTNgbDk5WAHZsHmHSxqcxuHov94inyFcpPqM6
```

返回的元数据为：

```json
{
  "name": "Vidar NFT",
  "description": "flag{Tr4d1ng_on_t3st_n3t}",
  "image": "ipfs://QmfRnBpi97gKowcZH932yeJPvtWDvkj1kcakRaV4GVMvWm",
  "attributes": [
    {
      "trait_type": "Season",
      "value": "Autumn"
    },
    {
      "trait_type": "Year",
      "value": "2024"
    }
  ]
}
```

因此 flag 为：

```text
flag{Tr4d1ng_on_t3st_n3t}
```

## 方法总结

本题是一条标准的 ERC-721 链上取证与交互链：确认合约、持有者和 `tokenId`，调用 `safeTransferFrom` 完成测试网转账，再从 NFT Transfer 历史追溯铸造交易，解码交易输入得到 `tokenURI`，最后读取 IPFS 元数据。浏览器展示的 NFT 图片不是 flag 本身，决定性信息位于元数据的 `description` 字段。链上交易和 IPFS 内容都可由哈希直接复核，因此应优先记录合约地址、交易哈希与 CID，而不是只保留浏览器截图。
