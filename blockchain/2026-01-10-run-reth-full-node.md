---
 layout: post
 category: blockchain
---

## 执行客户端

reth: https://reth.rs/run/ethereum#running-without-a-consensus-layer

```bash
reth node \
  --full \
  --chain mainnet \
  --datadir /root/code/node/eth/data \
  --http \
  --http.addr 0.0.0.0 \
  --http.port 7001 \
  --http.api "eth,net,web3,txpool,debug,trace" \
  --http.corsdomain "*" \
  --authrpc.jwtsecret /secrets/jwt.hex \
  --authrpc.addr 127.0.0.1 \
  --authrpc.port 8551 \
  --ws \
  --ws.addr 0.0.0.0 \
  --ws.port 7002 \
  --ws.api "eth,net,web3,txpool,debug,trace" \
  --ws.origins "*" \
  --ipcpath /root/code/node/eth/data/reth.ipc
```

## 共识客户端

lighthouse: https://lighthouse-book.sigmaprime.io/run_a_node.html

```bash
lighthouse bn \
  --network mainnet \
  --execution-endpoint http://localhost:8551 \
  --execution-jwt /secrets/jwt.hex \
  --checkpoint-sync-url https://mainnet.checkpoint.sigp.io \
  --purge-db \
  --disable-backfill-rate-limiting
```

用了上述参数，磁盘占用**不会随时间显著增大**，也就几十G

解释一下参数：

| 参数                                                       | 作用                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| `lighthouse bn`                                            | 启动 **Beacon Node**（共识层节点），不跑验证者客户端。       |
| `--network mainnet`                                        | 告诉 Lighthouse 连接 以太坊主网。                            |
| `--execution-endpoint http://localhost:8551`               | 指定 **执行层（reth/geth/…）的 Engine-API** 地址与端口，用于接收 payload 与 fork-choice 消息。 |
| `--execution-jwt /secrets/jwt.hex`                         | 与执行层通信所需的 **JWT 密钥文件**；两边必须完全一致，否则引擎调用会被拒（InvalidToken）。 |
| `--checkpoint-sync-url https://mainnet.checkpoint.sigp.io` | **轻量快速同步**：从 Sigma Prime 提供的公开检查点节点一次性拉取最新 finalized 状态，跳过从创世开始的漫长下载。 |
| `--purge-db`                                               | **首次启动时强制以“修剪模式”建库**，不保留历史状态，数据库体积保持最小（≈ 20–30 GB）。 |
| `--disable-backfill-rate-limiting`                         | 关闭回填空区块时的速率限制，让节点在后台更快地补齐旧区块，缩短同步时间。 |

## 输出

output: okay我们现在正在同步中

![image-20260110215905659](./assets/image-20260110215905659.png)

