---
title: 設定 Nginx 為具容錯能力的附載平衡器
date: 2026-03-18 00:00:00
categories:
  - Linux
tags:
  - Linux
  - Nginx
---

[![hackmd-github-sync-badge](https://hackmd.io/TK1VFyO9QzmOXsmsGtxyAg/badge)](https://hackmd.io/TK1VFyO9QzmOXsmsGtxyAg)

本文將說明如何設定 Nginx，使其成為一個具備容錯能力 (Fault Tolerance) 的負載平衡器 (Load Balancer)。

# 範例設定檔

```nginx
events {}

http {
    upstream http_service {
        server http-server-1:80 max_fails=3 fail_timeout=2s;
        server http-server-2:80 max_fails=3 fail_timeout=2s;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://http_service;

            proxy_next_upstream error timeout http_502 http_503 http_504;
            proxy_next_upstream_tries 2;

            add_header X-Served-By $upstream_addr;
            add_header X-Upstream-Status $upstream_status;
        }
    }
}
```

除了基礎的 `proxy_pass` 與 `upstream` 設定外，實現容錯的重點在於 `proxy_next_upstream` 及其相關設定:

- `proxy_next_upstream`: 定義當後端伺服器回傳哪些狀況時，Nginx 應將請求轉發給下一個節點。在本範例中，若發生連線錯誤 (error)、timeout 或收到的 HTTP 狀態碼為 502/503/504，Nginx 會自動嘗試其他可用的伺服器。
- `max_fails` 與 `fail_timeout`: Nginx 的 Passive Health Check 機制。
    - `max_fails=3`: 在 `fail_timeout` 時間內，如果失敗次數達到 3 次，伺服器就會被標記為不可用。
    - `fail_timeout=2s`: 伺服器被標記為不可用的持續時間 (範例中為 2 秒後會再次嘗試)。
- `proxy_next_upstream_tries`: 限制一個請求最多能嘗試幾個後端節點，避免在所有節點都不可用時造成請求迴圈或過長的等待時間。

> [參考資料](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream)

# 實驗

使用 Docker Compose 搭配上方的設定檔做個小測試，`compose.yaml` 的內容如下:

```yaml
services:
  nginx-lb:
    image: nginx:alpine
    container_name: nginx-lb
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - http-server-1
      - http-server-2

  http-server-1:
    image: nginxdemos/hello:plain-text
    container_name: http-server-1

  http-server-2:
    image: nginxdemos/hello:plain-text
    container_name: http-server-2
```

## 測試步驟

1. 啟動環境: 在資料夾下執行 `docker compose up -d`
2. 觀察行為: 使用 `curl -I localhost:8080` 發送請求。可以觀察到請求在兩台伺服器間輪詢
3. 模擬故障: 手動停止其中一個 http-server 容器 (e.g. `docker stop http-server-1`)。
4. 驗證容錯: 再次發送 `curl` 請求。

能夠發現即便其中一台伺服器斷線，Nginx 仍能回傳正常結果。在剛斷線時，第一個轉發到故障節點的請求可能會有短暫的延遲 (取決於 Timeout 設定)，隨後 Nginx 會因 `proxy_next_upstream` 機制將請求轉給可用的節點。之後在 `fail_timeout` 期間內，Nginx 將不再嘗試已故障的節點。
