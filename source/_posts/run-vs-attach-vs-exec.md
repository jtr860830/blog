---
title: run vs attach vs exec
date: 2025-11-12 04:00:00
categories:
  - Container
tags:
  - Cloud Native
  - Kubernetes
  - Linux
code_block_shrink: false
---

如果用過 docker、podman 之類的 container 管理工具，一定看過 `run` `attach` `exec` 這三個子命令，也大概知道它們可以**跟容器互動**，但實際上差別在哪?

先來看一下 docker 啟動 container 時會用到的幾個 component 以及順序:

```
docker-cli (docker client)
  ⭣
dockerd (docker daemon)
  ⭣
containerd (high-level runtime)
  ⭣
containerd-shim (runtime shim)
  ⭣
runc (low-level OCI runtime)
  ⭣
container (isolated and restricted Linux process)
```

如果是 podman 的話，因為是 **daemon-less** 所以會變成:

```
podman (daemonless container engine / CLI)
  ⭣
conmon (runtime shim)
  ⭣
runc / crun (low-level OCI runtime)
  ⭣
container (isolated and restricted Linux process)
```

# `run`

當我們使用下面這個指令去執行一個 Ubuntu container 時，整個體驗看起來很像是在啟動一個一般的前景行程 (foreground process): 這個指令會卡住 (block) 現在的終端機 session，container 應用程式的輸出 (包含 `stdout` 和 `stderr`) 會直接輸出在終端機上，且在終端機裡輸入的任何東西，會被 container 中的應用程式讀取 (`stdin`)。如果按下 `Ctrl+C`，那個中斷訊號 (`SIGINT`) 也會被送到 container 內的應用程式。

```shell!
docker run -it --rm ubuntu
```

但要是用 `ps`、`top` 或是 `htop` 觀察行程樹，可以發現這個 container 的父行程 (parent process) 既不是原本的 `docker run ...` 指令也不是 `dockerd`，而是 `containerd-shim-runc-v2`，也就是前面提到的 runtime shim

```shell
$ ps auxf
...
... /usr/bin/containerd
... /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
... sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
...  \_ sshd: root [priv]
...      \_ sshd: root@pts/0
...          \_ -bash
...              \_ docker run -it --rm ubuntu
... /usr/bin/containerd-shim-runc-v2 -namespace moby -id xxx -address /run/containerd/containerd.sock
...  \_ /bin/bash
...
```

甚至直接關閉執行 `docker run ...` 的終端機，這個 container 也不會被停止。

## 流程

`docker run` 是一個複合指令，它會在背後依序做幾件事: 拉 image、建立 container、把 container 跟網路連起來、啟動 container，最後再幫你把終端機 attach 上 container 的應用程式。實際執行細節會依照下的 flags 和 image 而略有不同，但流程大致相同。

從主機的角度來看，前景行程是 docker 這個 CLI，它跑在終端機 session 裡。當輸入 `docker run nginx:latest` 時，docker 並不是自己去建立容器，而是透過 UNIX socket 對 dockerd (docker daemon) 發送 HTTP REST API 請求。所以 CLI 只是客戶端，真正執行的是 dockerd。

dockerd 收到 run 的請求後，會再透過另一個 UNIX socket，用 gRPC API 呼叫 containerd 這個 daemon。containerd 則負責更低一層 (相較於 dockerd) 的 container 生命週期管理: 建立 container、準備 rootfs、設定 namespace 還有 cgroup 等等。

## runtime shim

當 container 要真的跑起來時，containerd 會為這個 container 啟動一個 runtime shim (Docker 是 containerd-shim，podman 則是 [conmon](https://github.com/containers/conmon))，它本身是一個背景行程。這個 shim 會去呼叫 OCI runtime (通常是 runc 或 crun)，依據 OCI spec 在主機上 `fork/exec` 出 container (隔離且被限制的行程)，並在裡面啟動像 nginx 這樣的應用程式。runc 完成後就直接退出，接下來由 runtime shim 接手，持續管理這個 container。

在這個過程中，被 reparent 到 PID 1 (init 行程，通常是 systemd) 的，是 shim，而不是 container 裡的應用程式本身:

- shim 會將自己轉為背景行程、reparent 到 PID 1，並將自己的 stdio stream 關掉
- shim 接管被封裝的應用程式的 stdio stream (`stdin`、`stdout`、`stderr`)

整個 pipeline 裡，`stdin`、`stdout`、`stderr` 並不是直接從 container 連接到終端機，而是一路被轉接:

```
terminal session <-> docker-cli <-> dockerd <-> containerd <-> runtime shim <-> container
```

同樣地，像 `Ctrl+C` 這種訊號也是由 CLI 經過 daemon 一層層轉發到容器裡的主行程。

傳統把程式 daemonize (丟到背景、跟啟動它的終端機分離) 時，典型做法是:

- 把它 reparent 到 PID 1 (init / systemd)
- 把它的 stdio stream 全部關閉

如果 container 也是這樣實作，就看不到它的輸出、更別說從終端機輸入資料。實際上，Docker 把這套行為用在 shim 身上，同時讓 shim 接管 container 的 stdio，於是才有現在的使用體驗: 看得到 log、可以送資料到 stdin，還能 attach 及 detach。

下面這個指令會在背景啟動一個 Ubuntu container:

```shell
docker run -it --rm -d --name ubuntu ubuntu
```

這行指令會很快結束，把控制權還給終端機。但我們可以用 attach 重新把終端機接回這個 container，讓它又看起來像前景行程:

```shell
docker attach ubuntu
```

背後之所以做得到，就是因為 shim 一直在背景維護 container 的 stdio 與生命週期。

這個背景執行的 shim 會從 container 的 `stdout`、`stderr` 讀資料，再把讀到的資料丟給對應的 log driver (預設是 `json-file`，或自行設定的 driver)。

> 預設情況下，shim 會關閉容器的 `stdin`。但如果 `docker run` 有加 `-i`，shim 就會把 `stdin` 保持打開，這樣之後用 `docker attach` 時，才能把鍵盤輸入的資訊送到 container 裡的 process。

# `attach`

大部分 production 的應用程式都是跑在背景的 container 裡，當需要**臨時連進去**看看它目前在輸出什麼，或想直接把鍵盤輸入送進去時，`attach` 就是那條接回去的線。

前面提到 runtime shim 會接管 container 的 stdio streams。具體一點來說，shim 扮演一個伺服器的角色:

- 在本機上提供一個 RPC 介面 (通常是 UNIX socket)
- 當有 client 連上這個 socket 時，shim 會開始把 container 的 `stdout`、`stderr` 內容往 socket 另一端串流回去
- 反過來也可以從這個 socket 讀資料，再轉送給 container 的 `stdin`

當你執行 `docker attach` 時，Docker CLI 就是扮演 client，透過 dockerd 一路接到 shim，於是你的終端機又重新連接到 container 的 stdio streams 上。

所以，如果一開始用 `docker run -d` 啟動一個背景 container，只要這個 container 還在、shim 還活著，都可以隨時用 `docker attach` 把終端機再接回去，讓它暫時又變得像前景行程一樣。

另外值得注意的是，`docker attach` 只能看到接回去後新產生的輸出，完整的歷史 log 則是由 shim 同步寫到對應的 log driver，可以用 `docker logs -f` 去查看。attach 比較像是把目前的 stdio 直接接到眼前，而 logs 則是讀取已經被保存下來的輸出紀錄。

# `exec`

`exec` 最常出現在排查和除錯的場景，例如已經有一個 container 在跑了，臨時想要:

- 進去開個 shell 看環境
- 裡面跑個 `curl`、`ping`、`ps`、`top`
- 看某個檔案、查某個設定

這時通常會下:

```shell
docker exec -it CONTAINER bash

docker exec -it CONTAINER curl 127.0.0.1:80
```

乍看之下，`exec` 跟 `attach` 有點像: 兩者都是對一個已存在的 container 進行操作，但關鍵差別在於

- `attach` 只是把終端機**重新接回**原本的 container 應用程式的 stdio 上，外加訊號轉發
- `exec` 則會**新開**一個隱藏的輔助 container，在目標 container 內，再跑一個新的行程

所以 `exec` 其實更接近 `run`，而不是 `attach`。只是它不是重新建立一整個新 container，而是在既有 container 的那一組 namespaces 跟 cgroup 內再塞一個行程進去。

這邊補充一下，OCI Runtime Spec 其實並沒有定義 `run` 或 `exec` 指令。runtime 只需要會 `create` 跟 `start` 就夠了。`exec` 只是高階一點的便利功能——底層還是透過**再 create 一個臨時的 container，然後 start** 去實作。

`docker exec` 做的事情，可以想成:

1. docker CLI 收到 `docker exec` 指令。
2. 一樣透過 HTTP / gRPC 把請求送到 dockerd / containerd。
3. containerd 透過 shim，要求 runtime 幫忙在目標 container 相同的 namespaces 跟 cgroups 裡，再起一個新 process
4. 這個新 process 有自己的 stdio streams，shim 再把它的 stdio 接到終端機上

# 小結

- `run`: 建立並啟動一個新的 container
- `attach`: 什麼都不新建，只是把終端機**重新接回**既有 container 的 stdio 上，並把輸入及訊號轉發進去
- `exec`: 在既有 container 的 namespaces 及 cgroups 裡再新開一個 process，並把終端機接到這個新 process 的 stdio 上，比較像是在 container 裡多跑一個指令

不管用的是 Docker、Podman，還是在 Kubernetes 上透過 `kubelet` 跟 `kubectl` 操作 container，它們的核心概念其實都大同小異:

- 一個負責接收 user 指令的前端 (CLI / API)
- 一個或多個負責管理 lifecycle 的 daemon / manager
- 一層 runtime shim 幫忙轉接 stdio、轉發 signal、維持生命週期
- 最底下那個依 OCI 規範啟動、實際在 kernel 上跑的 Linux process
