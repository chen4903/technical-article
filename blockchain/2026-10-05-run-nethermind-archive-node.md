---
layout: post
category: blockchain
---

## 背景

以前跑以太坊存档节点，动辄 20~30T（geth hash 模式甚至更多），reth 存档快照也要 2.2T 左右（见 [run-reth-full-node](./2026-01-10-run-reth-full-node.md)）。

Nethermind 2.0.0 引入了 **Flat DB**，并把存档历史直接建在 flat state 之上：

- 完整主网存档：执行层 DB 约 **2.1T**，节点总占用约 **2.3T**（共识层另算，几十 G）
- 官方建议预留 **3T**（留给增长）
- 不需要单独的存档数据库，也没有新协议，就是普通同步

> 数据来自 2026-09-01 的实测（区块高度约 25.88M），会随链增长。**2T 的盘装不下完整存档**，至少要 3T，别踩坑。

文档：https://docs.nethermind.io/

## 存档模式选择

Nethermind 2.0 的历史保留有几种形态：

| 配置 | 保留历史 | 执行层 DB | 建议预留 |
| ---- | -------- | --------- | -------- |
| 完整存档 | 从创世开始的全部状态 | ~2.1 T | 3 T |
| 窗口存档 | 约 2 个月状态 + 约 1 年区块 | ~0.9 T | 1.5 T |
| 按地址存档 | 6 个最活跃合约的完整历史 + 256 块 | ~1.8 T | 2.5 T |

对应参数：

| 参数 | 作用 |
| ---- | ---- |
| `FlatDb.HistoryRetention=Rolling` + `FlatDb.HistoryRetentionBlocks` | 只保留最近 N 个块的历史状态 |
| `FlatDb.HistoryRetention=SinceBlock` + `FlatDb.HistoryRetentionSinceBlock` | 保留某个高度之后的全部历史 |
| `FlatDb.HistorySliceAddresses` | 仅对指定合约保留深度历史 |

本文跑**完整存档**，直接用 `mainnet_archive` 配置即可。

## 注意事项

- 已有的 Patricia（旧版）数据库升级到 2.0 后**仍然是 Patricia**，不会变小。想要 2.1T 的新布局，必须**重新同步**，或设置 `FlatDb.ImportFromPruningTrieState=true` 原地导入。
- 全新的 datadir 默认就是 Flat DB。
- 历史开启是 forward-only，开启点之前的历史不会回填，所以一开始就要用存档配置。

## 执行客户端

安装：参考 https://docs.nethermind.io/get-started/installing-nethermind/ ，docker、apt、二进制都行，下面给出二进制和 Docker 两种启动方式。

先生成 JWT（执行层和共识层共用，沿用 reth 那篇的路径）：

```bash
mkdir -p /secrets
openssl rand -hex 32 | tr -d '\n' > /secrets/jwt.hex
```

启动：

```bash
nethermind \
  --config mainnet_archive \
  --data-dir /root/code/node/eth/nethermind \
  --FlatDb.Enabled true \
  --JsonRpc.Enabled true \
  --JsonRpc.Host 0.0.0.0 \
  --JsonRpc.Port 7001 \
  --JsonRpc.EnabledModules "[eth,net,web3,txpool,debug,trace]" \
  --JsonRpc.CorsOrigins "*" \
  --JsonRpc.JwtSecretFile /secrets/jwt.hex \
  --JsonRpc.EngineHost 127.0.0.1 \
  --JsonRpc.EnginePort 8551
```

**Docker 方式**（镜像入口就是 `nethermind`，参数不变，路径改成容器内路径）：

```bash
docker run -d --name nethermind \
  --restart unless-stopped \
  --network host \
  -v /root/code/node/eth/nethermind:/data \
  -v /secrets:/secrets:ro \
  nethermind/nethermind:2.1.0 \
  --config mainnet_archive \
  --data-dir /data \
  --FlatDb.Enabled true \
  --JsonRpc.Enabled true \
  --JsonRpc.Host 0.0.0.0 \
  --JsonRpc.Port 7001 \
  --JsonRpc.EnabledModules "[eth,net,web3,txpool,debug,trace,rpc]" \
  --JsonRpc.CorsOrigins "*" \
  --JsonRpc.JwtSecretFile /secrets/jwt.hex \
  --JsonRpc.EngineHost 127.0.0.1 \
  --JsonRpc.EnginePort 8551

# docker logs -f nethermind 查看日志
```

- `--network host`：容器直接用宿主机网络，`127.0.0.1:8551` 和 P2P 端口 30303 都不用映射。
- 不用 host 网络时，`EngineHost` 要改成 `0.0.0.0`，并用 `-p` 映射端口。
- 二进制安装（不用 Docker）：从 [releases](https://github.com/NethermindEth/nethermind/releases) 下载，tag 没有 `v` 前缀，例如 `2.1.0` 的 `nethermind-2.1.0-b3e7e84c-linux-x64.zip`，解压后直接运行 `./nethermind`。

参数解释：

| 参数 | 作用 |
| ---- | ---- |
| `--config mainnet_archive` | 主网**存档**预设配置；换成 `mainnet` 就是普通全节点 |
| `--data-dir` | 数据目录，务必放在安装目录之外，避免升级时丢数据 |
| `--FlatDb.Enabled true` | 显式使用 Flat DB（2.0 新节点默认开启，写上更保险）。**不要在已有 flat 数据目录上设为 false，否则会丢弃 flat 状态并全量重同步** |
| `--JsonRpc.Port 7001` | HTTP RPC 端口，与 reth 那篇保持一致 |
| `--JsonRpc.EnabledModules` | 开启的 RPC 命名空间，存档节点通常要 `debug` / `trace` |
| `--JsonRpc.JwtSecretFile` | Engine API 的 JWT 密钥，必须与共识层一致 |
| `--JsonRpc.EngineHost/EnginePort` | Engine API 监听地址，仅本机访问 |

和 reth 的差异：

- Nethermind 的 **WebSocket 默认和 HTTP 共用同一个端口**（`Init.WebSocketsEnabled` 默认 true），所以没有单独的 `ws` 端口，`ws://host:7001` 即可。
- Nethermind 的 RPC 参数是 `--Section.Option` 的风格，不是 reth 的 `--http.*`。
- 公网暴露 `debug` / `trace` 有风险，生产环境请用防火墙或反向代理限制来源。

## 共识客户端

沿用 lighthouse，参数与 reth 那篇完全一致，因为 Engine API 是标准接口，换执行层无需改动：

```bash
lighthouse bn \
  --network mainnet \
  --execution-endpoint http://localhost:8551 \
  --execution-jwt /secrets/jwt.hex \
  --checkpoint-sync-url https://mainnet-checkpoint-sync.stakely.io \
  --disable-backfill-rate-limiting \
  --http \
  --http-address 0.0.0.0 \
  --http-port 5052
```

共识层磁盘占用不会显著增长，也就几十 G。

## 同步方式

**方式一：直接从网络同步（默认）**

存档同步是最重的模式，Nethermind 会从创世块重放所有区块。官方说明主网需要**数天**，取决于磁盘和网络。磁盘强烈建议 NVMe，IOPS 越高越快。

日志里重点看这几类：

| 日志 | 含义 |
| ---- | ---- |
| `Downloaded x/y` | 已下载但未处理的区块数 |
| `Processed ...` | EVM 已处理到的高度，附带 `MGas/s`、`tps`、`blk/s` |
| `Waiting for peers...` | 还没找到可同步的节点，稍等 |

追上链头后，`Processed` 日志大约每 12 秒出现一次（每个新区块一条）。

**方式二：使用快照**

Nethermind 2.0 支持**流式下载快照**（不用先把压缩包完整存到本地），相关参数：

```bash
  --Snapshot.Enabled true \
  --Snapshot.DownloadUrl <snapshot-url> \
  --Snapshot.Checksum <sha256> \
  --Snapshot.StripComponents 1
```

注意：

- 需要预留**至少 1.5 倍快照大小**的空闲磁盘，否则导入会直接失败。
- 快照来源需要自行确认；`StripComponents` 要与压缩包内的目录层级一致。
- 和 reth 那篇一样：下载速度受 Cloudflare 等限流影响时，别开太多线程，慢慢下载更稳。

## 验证

同步完成后，查询一个很早的历史状态，验证存档能力，比如查询某地址在区块 1,000,000 的余额：

```bash
curl -s -X POST http://127.0.0.1:7001 \
  -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_getBalance","params":["0xde0B295669a9FD93d5F28D9Ec85E40f4cb697BAe","0xF4240"]}'
```

能返回结果而不是 `missing trie node` / `not retained`，就说明历史状态可查。

查看磁盘占用：

```bash
du -sh /root/code/node/eth/nethermind/*
```

预期执行层约 2.1T 级别。

## 结果

同步成功后，日志应出现类似下面的内容（按自己的实际输出截图替换）：

```
Processed  25880000 ...  MGas/s ... tps ... blk/s ...
```

## 小结

| 项目 | reth 全节点 | Nethermind 存档 |
| ---- | ----------- | --------------- |
| 磁盘 | ~1T | ~2.3T（预留 3T） |
| 历史状态 | 裁剪 | 完整 |
| WS 端口 | 独立 | 与 HTTP 共用 |
| 共识层 | lighthouse | lighthouse（不变） |
