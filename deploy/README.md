# 多出口部署

一个 HE 隧道分配一个 /48，里面有 65536 个 /64。每个容器绑定其中一个 /64，
监听独立端口；oc2api 用 `__rate_limit_switch_no_direct__` 在 429 时轮转到下一个出口，
避免单个 /64 被目标站限速/封禁。

## 步骤

```bash
cp .env.example .env      # 填 HE_SERVER、PROXY_USER/PASS、HE_PREFIX_1..4
docker compose up -d
```

消防要求：

- Oracle/云安全组放行来自 `HE_SERVER` 的**协议 41**（6in4 没有端口）
- 代理端口（默认 37321-37324）只给 oc2api 用，别对公网放行

每个出口自检：

```bash
curl -x socks5h://$PROXY_USER:$PROXY_PASS@127.0.0.1:37321 https://api64.ipify.org
# 应为 HE_PREFIX_1 下的地址
```

## oc2api 侧

`socks5_proxies` 里加 4 条，`active_socks5` 设为 `__rate_limit_switch_no_direct__`：

```json
"socks5_proxies": [
  {"name": "HE出口1", "addr": "172.18.0.1:37321", "username": "...", "password": "..."}
]
```

`addr` 用 oc2api 容器所在 bridge 网络的网关地址（`docker network inspect` 可查），
因为出口容器是 host 网络。
