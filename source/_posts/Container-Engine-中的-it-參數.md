---
title: Container Engine 中的 -it 參數
date: 2025-11-11 01:00:00
categories:
  - Container
tags:
  - Cloud Native
  - Kubernetes
  - Linux
code_block_shrink: false
---

[![hackmd-github-sync-badge](https://hackmd.io/-aRwrzC5TMqZxhh8HRQ_lA/badge)](https://hackmd.io/-aRwrzC5TMqZxhh8HRQ_lA)

在使用像是 Docker、Podman 或是一些 CLI 工具直接操作 containerd 這類的 runtime 時，常常會使用到 `-it` 這**兩個**參數，但很多人其實只知道加上去就能夠跟容器互動，卻不太清楚這兩個旗標 (flag) 各自代表什麼。

在解釋 `-it` 之前，我們先從其中的 `-t` 講起: 它跟一個古老但很重要的東西有關——TTY。

# 什麼是 TTY

TTY 原本是 **T**ele**ty**pewriter (電傳打字機) 的縮寫，在還沒有圖形介面也還沒有終端機視窗的年代，工程師是用一台長得像打字機的機器，透過線路連線到主機上進行操作。
後來這種可以**打字送指令、看主機回應**的互動方式，就被抽象成今天我們說的終端機 (terminal/console)，而 TTY 這個名字也就一直沿用下來。

在現代的 Linux 裡，你看到的這些東西，其實都可以被視為一種終端機或終端機的模擬器，像是:

- 桌面環境裡開的 Terminal 視窗
- `Ctrl + Alt + F1~F6` 那種純文字登入畫面 (virtual console)
- 透過 SSH 登進遠端主機看到的那個 shell

它們共同的特點是：

- 能夠接收你的鍵盤輸入
- 用**一般人看得懂**的方式顯示輸出 (像是換行、顏色...)

以使用 C 語言為例，就會使用下面這個函數去檢查一個行程 (process) 的 `stdin`、`stdout`、`stderr` 是否有連接到一個終端機或是虛擬終端機，像 bash、zsh、python REPL 這種互動式的程式就會透過類似的方式知道是否要啟用互動模式、顏色、或是一些使用者體驗更好的輸出方式

```c!
#include <unistd.h>

int isatty(int fd); // 回傳 1 代表傳入的檔案描述子 (fd) 有連接到 TTY，0 則否
```

# `-t` 是什麼

查詢一下 `docker-run` 的 man-page

```
-t, --tty=true|false
Allocate a pseudo-TTY. The default is false.

When set to true Docker can allocate a pseudo-tty and attach to the standard input of any container. This can be used, for example, to run a throwaway interactive shell. The default is false.

The -t option is incompatible with a redirection of the docker client standard input.
```

可以知道就是是否要為這個 container 開啟一個虛擬終端機，然後將 `stdin`、`stdout`、`stderr` 都綁定到這個虛擬終端機上，就能夠讓封裝在 container 內的程式去判斷是否在一個終端機的環境內，決定要用什麼樣的方式進行輸出。使用以下兩個指令來看一下差異:

```shell!
# 有 -t：像平常開一個終端機一樣
docker run -it --rm ubuntu bash

# 沒有 -t：bash 會覺得自己在無法互動的環境
docker run -i --rm ubuntu bash
```

![image](https://hackmd.io/_uploads/SJHano1l-e.png)

![image](https://hackmd.io/_uploads/S1SCnjye-g.png)

在裡面都使用了 `ls` 然後 `exit`，可以看出輸出方式的不同，沒有 `-t` 時可以發現:

- 沒有 prompt (PS1)
- 沒有好看的輸出
- Backspace、方向鍵、Ctrl+C 之類的行為可能跟預期有差

雖然 `-t` 能夠讓輸出的效果更好、更有互動性，但在 CI / 自動化腳本的場景下，會建議不要使用。因為有 TTY 會改變輸出格式 (顏色、互動性...)，某些程式還會啟用像是 progress bar 之類的輸出，使輸出沒有一致性，反而不便進行 log 的分析。

# 那 `-i` 呢

一樣看一下 man-page

```
-i, --interactive=true|false
Keep STDIN open even if not attached. The default is false.

When set to true, keep stdin open even if not attached.
```

意思是把使用者的鍵盤輸入 (`stdin`) 接到封裝在容器裡的程式，並且保持開啟。也來觀察一下有沒有這個選項的差別:

```shell!
docker run -it --rm ubuntu bash

docker run -t --rm ubuntu bash
```

![image](https://hackmd.io/_uploads/SJHano1l-e.png)

![image](https://hackmd.io/_uploads/Skjf-3yxWl.png)

可以看出要是沒有加這個選項，bash 是完全讀不到我的輸入的，所以自然沒有輸出任何內容，會直接卡在這個畫面

# 為什麼通常 `-it` 都會一起使用

先註明這是兩個選項 (`-i` + `-t`) 的簡寫，不是有一個叫做 `-it` 的選項，下方的指令都是等價的

```shell!
# docker
docker run -it ubuntu bash
docker run -i -t ubuntu bash
docker run -ti ubuntu bash

# kubectl
kubectl exec -it my-pod -- bash
kubectl exec -i -t my-pod -- bash
```

而在大部分的情況下，想要的效果都會是

- 我要可以打字進去 (`-i`)
- 我要它像平常開 terminal 一樣好用 (`-t`)

所以這兩個選項通常都會一起出現。

# 小結

`-it` 只是兩個非常單純的選項組合:

- `-i`: 把標準輸入保持打開，讓你可以把鍵盤輸入交由容器裡的程式讀取
- `-t`: 幫容器配一個 pseudo-TTY，讓裡面的程式知道**自己正跑在一個終端機裡**，進而啟用顏色等互動功能
