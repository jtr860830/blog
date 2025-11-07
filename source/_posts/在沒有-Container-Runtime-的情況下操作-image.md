---
title: 在沒有 Container Runtime 的情況下操作 image
date: 2025-11-07 13:00:00
categories:
  - Container
tags:
  - Cloud Native
  - Kubernetes
  - Linux
code_block_shrink: false
---

在日常的 container 管理工作中，對容器映像檔 (container image) 進行**重新命名 (打 tag)** 或**複製**、**同步 image** 是非常常見的需求。

傳統上，我們會使用 Docker CLI:

```shell!
docker pull nginx:latest
docker tag nginx:latest registry.example.com/nginx:latest
docker push registry.example.com/nginx:latest
```

但這需要 Docker Daemon、containerd 等 runtime。若在 CI/CD pipeline、air-gapped 環境、minimal base image 之類的沒有 runtime 的地方，這樣的方式就變得不可行了。

為了在這樣的環境中仍能靈活操作 image，我們可以使用三個比較輕量的工具直接操作 image 與 registry: [`skopeo`](https://github.com/containers/skopeo)、[`crane`](https://github.com/google/go-containerregistry/blob/main/cmd/crane/README.md)、[`regctl`](https://github.com/regclient/regclient)。

# 使用場景

## 重新打 tag

有時候不需要重新上傳整個 image，只想在 registry 裡多一個新 tag。

```shell!
# skopeo
skopeo copy docker://registry.example.com/nginx:latest docker://registry.example.com/nginx:stable

# crane
crane cp nginx:latest registry.example.com/nginx:v1.25.2

# regctl
regctl tag put registry.example.com/nginx:latest registry.example.com/nginx:stable
```

注意其實這邊三種指令的行為**並非等價**的。

`skopeo copy` 會把來源 image 的所有層 (layer) 與 manifest 完整下載，然後再上傳到目標 registry，即便是相同的 registry。

`crane cp` 會先取得來源 image 的 manifest，然後檢查目標 registry 是否已經存在相同的內容。如果 registry 有對應的 blobs，就不會重傳，只會重新上傳 manifest 建立新的 tag。

`regctl tag put` 不會上傳任何 image layer 或 manifest，而是直接使用 [OCI Distribution Specification 中定義的 API](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#pushing-manifests) 為 image 建立一個新的 tag。

## 列出 image tags

有時候需要快速查看一個 image 所有的 tag

```shell!
# skopeo 沒有指令能直接呼叫

# crane
crane ls nginx

# regctl
regctl tag ls nginx
```

## 複製 image

例如從 Docker Hub 複製一個公開 image 到組織內部 registry。

```shell!
# skopeo
skopeo copy docker://nginx:latest docker://registry.example.com/nginx:latest

# crane
crane copy nginx:latest registry.example.com/nginx:latest

# regctl
regctl image copy nginx:latest registry.example.com/nginx:latest
```

`skopeo` 會檢查並同步所有 image layer 與設定檔，就算是同一個 registry，它仍會驗證並重新上傳 image。

`crane` 先取得想上傳的 image manifest，如果目標 registry 已有對應 blobs，就不會重傳。像是同個 registry 內執行時，通常只會重新上傳 manifest 不會是整個 image。

`regctl` 完全依照 OCI 標準執行，讀取想上傳的 image manifest，然後檢查目標 registry 是否已有對應資料，並只上傳缺少的部分，最後建立新的 tag 指向 image。

三者都是預設上傳 multi-platform image 的，如果只想上傳特定架構的話，可以加上下面的參數:

```shell!
# skopeo
skopeo copy --override-arch amd64 docker://nginx:latest docker://registry.example.com/nginx:amd64

# crane
crane copy --platform linux/arm64 nginx:latest registry.example.com/nginx:arm64

# regctl
regctl image copy --platform linux/amd64 nginx:latest registry.example.com/nginx:amd64
```

# 小結

無論是 `skopeo`、`crane` 還是 `regctl`，三個工具都能在沒有 container runtime 的情況下操作 image 與 registry，但筆者會希望說能越符合 OCI 標準會越好，所以我會最推薦使用 `regctl`，每個指令都對應到 OCI Distribution Spec 的定義，沒有隱藏邏輯、沒有模糊行為，同時又提供足夠的靈活性來操作 manifest 與 tag。
