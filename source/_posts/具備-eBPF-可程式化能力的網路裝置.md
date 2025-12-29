---
title: 具備 eBPF 可程式化能力的網路裝置
date: 2025-12-28 00:00:00
categories:
  - Linux
tags:
  - eBPF
  - Translation
---

[![hackmd-github-sync-badge](https://hackmd.io/B9ThJ4FmRTOXrIvgJdsDiQ/badge)](https://hackmd.io/B9ThJ4FmRTOXrIvgJdsDiQ)

> 譯自 [The BPF-programmable network device](https://lwn.net/Articles/949960/)
> [name=Jonathan Corbet]

在 Linux 中，容器與虛擬機器是透過虛擬網路裝置與外界進行通訊的。這種設計能夠使用 network stack 的完整功能，但同時也需負擔全部的效能開銷。通常，這類網路流量的路由只需要相對簡單的處理邏輯；而隨著可程式化 eBPF 的網路裝置在 kernel 6.7 版中被合併，使在在某些情境下有機會能夠避開昂貴的網路處理流程。

在目前的系統中，當 guest (容器或虛擬機器) 透過網路傳送資料時，資料會先進入 guest 內部的 network stack，並在那裡被封裝成封包，並經由虛擬網路介面送出。在 host 端，這個封包會再次被接收，並同樣由 network stack 處理。如果封包的目的地在 host 之外，它接著會被路由到一張 (實體) 網路介面上重新傳送。最終，guest 的資料成功送達外面，但是它已經先後通過了**兩層** network stack。

這個新裝置叫做 `netkit`，目標在於消除一部分這樣的效能開銷。某種程度上來說，它就是一種典型的虛擬網路裝置: 封包在一端送出後，只需經過 host 系統的記憶體，就會在另一端被接收。真正的差異在於**傳輸的運作方式**。每一個網路介面驅動程式都會提供一個 [`net_device_ops` 結構體](https://elixir.bootlin.com/linux/v6.6/source/include/linux/netdevice.h#L1057)，其中包含大量的函式指標——在 kernel 6.6 版中，最多可達 90 個。其中之一便是 `ndo_start_xmit()`:

```c
netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb, struct net_device *dev);
```

這個函式的職責是: 透過指定裝置 `dev`，初始化 `skb` 中封包資料的傳送。在一般的虛擬裝置中，這個函式會透過呼叫像是 [`netif_rx()`](https://elixir.bootlin.com/linux/v6.6/source/net/core/dev.c#L5108) 這類的函式，將封包立即**接收**回對端的 network stack 中 (host 的 RX path)。但 `netkit` 裝置的行為則有些許不同。

當這個虛擬網路介面被設定完成後，可以在介面的每一個端點載入一個或多個 eBPF 程式。由於 `netkit` 的 eBPF 程式可能會影響 host 端流量的路由，因此只有 host 端被允許為 host 或 guest 載入這些程式。`netkit` 所提供的 `ndo_start_xmit()` callback，不再只是單純地將封包送回 network stack，而是會依序呼叫所有已附加的 BPF 程式，並將封包傳遞給每一個程式處理。這些 eBPF 程式可以修改封包內容 (例如變更目的地裝置)，並且必須回傳一個值說明接下來應該如何處理封包: 

- `NETKIT_NEXT`: 繼續呼叫序列中的下一個 eBPF 程式 (如果有的話)。如果已經沒有其他程式需要執行，此回傳值會被視為 `NETKIT_PASS`。
- `NETKIT_PASS`: 立即將封包送入接收端的 network stack 處理，且不再呼叫任何其他 eBPF 程式。
- `NETKIT_DROP`: 立即丟棄該封包。
- `NETKIT_REDIRECT`: 立即將封包重新導向至另一個網路裝置，並將其排入傳送佇列且不需再經過 host 端的 network stack。

每一個介面都可以設定一個預設政策 (default policy，可以是 `NETKIT_PASS` 或 `NETKIT_DROP`)，當沒有載入任何 eBPF 程式進行判斷時，就會使用預設的政策。多數情況下，合適的預設政策應該是直接丟棄封包，以確保在介面尚未完整設定、能夠正確處理流量之前，不會有任何封包從 guest 端洩漏。

如果能夠在越早的階段就做出是否丟棄封包的決策，就能帶來顯著的效能提升。不需要的網路流量通常數量龐大，所以處理這些流量所花費的時間越少越好。不過，正如這個 [changelog](https://git.kernel.org/linus/35dfaad7188c) 所述，最大的效能收益，也許是來自於能夠在不重新進入 network stack 的情況下，直接重新導向封包的能力: 

> 舉例來說，如果 eBPF 程式判斷此 `skb` 必須被送出節點外，就能夠直接將封包重新導向至實體網路裝置，而不需要經過 per-CPU backlog queue。這樣就能夠幫助這類流量的處理從 softirq 轉移至 process context，進而帶來更好的排程決策與整體效能表現。

根據 2023 年 Linux Storage、Filesystem、Memory-Management 與 BPF Summit 上的一場演講所提供的[投影片](http://vger.kernel.org/bpfconf2023_material/tcx_meta_netdev_borkmann.pdf)，guest 系統透過 `netkit` 裝置 (當時叫做 `meta`)，TCP 資料傳輸速率能夠達到跟直接在 host 上執行幾乎相同的水準。換言之，在 guest 中執行所帶來的效能損耗已被完全消除。

考量到這項設計對部分使用者可能帶來的效能提升，由 Daniel Borkmann 提出，以及 Nikolay Aleksandrov 貢獻的這一系列 patch 能夠迅速被合併，也就不足為奇了。這段 patch 最早在 9 月 26 日[張貼](https://lwn.net/ml/bpf/20230926055913.9859-1-daniel@iogearbox.net)至 BPF mailing list，歷經四次修訂後，於一個月後進入 kernel 6.7 版的 merge window。這項功能並非適用於所有使用者，但對於在容器或虛擬機器中部署網路密集型應用程式的使用者而言，具有相當的吸引力。