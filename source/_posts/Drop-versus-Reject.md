---
title: Drop versus Reject
date: 2024-11-02 00:21:00
categories:
  - Networking
tags:
  - Firewall
  - Translation
code_block_shrink: false
---

[![hackmd-github-sync-badge](https://hackmd.io/5mHJ-CziSKS7yMUDvWLnTw/badge)](https://hackmd.io/5mHJ-CziSKS7yMUDvWLnTw)


> 譯自 [Drop versus Reject](https://www.chiark.greenend.org.uk/~peterb/network/drop-vs-reject)
> [name=Peter Benie <peterb@chiark.greenend.org.uk>]

許多人提倡使用一種**幾乎封閉**的策略來設定封包過濾器 (packet filters)，將不知道是否安全的封包直接丟棄 (drop)。這種策略在不增加額外安全性的情況下，會導致使用者難以診斷相關的問題。

當一個封包抵達防火牆時，會對照一組規則執行檢查。這些規則可能會依賴現存的狀態 (例如匹配現有或相關的連線)，或者是無狀態的 (例如匹配目標主機的 80 通訊埠)。這些規則會決定防火牆將對封包採取以下哪一操作:

- 允許 (**ALLOW**，也稱為 **ACCEPT**)
  允許封包穿越防火牆。相當於防火牆不在的行為。
- 拒絕 (**REJECT**)
  禁止封包通過，並向來源主機發送一目的地無法到達 (destination-unreachable) 的 ICMP 回應 (除非該 ICMP 訊息在正常情況下不被允許，像是針對廣播地址的訊息)。
- 丟棄 (**DROP**，也稱為 **DENY** 或 **BLACKHOLE**)
  禁止封包通過，且不作任何回應。

再遇到不想要的封包時，可以選擇 REJECT 或 DROP。在分析這類選擇時，我們必須考慮它對合法和非法應用的正面以及負面特性。

REJECT 和 DROP 之間的最大的差別在於，REJECT 會回覆一個 ICMP 錯誤。這有什麼作用呢? 讓我們來看看相關標準文件 STD0003 (RFC1122) 中的摘錄:

```
         3.2.2.1  Destination Unreachable: RFC-792

                  A Destination Unreachable message that is received MUST be
            reported to the transport layer.  The transport layer SHOULD
            use the information appropriately; for example, see Sections
            4.1.3.3, 4.2.3.9, and 4.2.4 below.  A transport protocol
            that has its own mechanism for notifying the sender that a
            port is unreachable (e.g., TCP, which sends RST segments)
            MUST nevertheless accept an ICMP Port Unreachable for the
            same purpose.

         4.2.3.9  ICMP Messages

            TCP MUST act on an ICMP error message passed up from the IP
            layer, directing it to the connection that created the
            error.  The necessary demultiplexing information can be
            found in the IP header contained within the ICMP message.

            o    Destination Unreachable -- codes 2-4

                 These are hard error conditions, so TCP SHOULD abort
                 the connection.

```

# 合法的使用者

我們先來考慮一般使用者的情境:

REJECT 未知的封包，TCP 會中止連線，應用程式會在一次 RTT (round-trip time) 後知道連線失敗了。這讓嘗試建立連線的應用程式能夠馬上通知使用者。

DROP 封包，會導致 TCP 不斷重試連線，直到超過重傳 (retranmission) 的門檻。這應該至少需要 100 秒。

在 Linux 上的實驗顯示，在使用 REJECT 時，TCP 連線 0.01 秒內就會回覆應用程式錯誤，而 DROP 需要 189 秒。

# 不友善的使用者

現在讓我們來考慮具有敵意的一方 (hostile forces):

使用 DROP 而不是 REJECT 的一個常見原因是希望避免透露哪些通訊埠是開放的。然而，DROP 洩露的資訊量實際上跟 REJECT 是相同的。

使用 REJECT 時，掃描結果可以分為連線已建立 (connection established) 和連線被拒絕 (connection rejected) 兩類。

而使用 DROP 時，結果則可分為連線已建立 (connection established) 和連線已超時 (connection timed out)。

最簡易的 (trivial) 掃描工具會使用作業系統的 connect 系統呼叫，並會等到一個連線嘗試完成後再開始下一次嘗試。這種掃描工具在遇到封包被 DROP 時會大幅變慢。然而，如果攻擊者為每次嘗試的連線設定 5 秒的超時，有可能掃描一台機器所有保留的通訊埠 (1 到 1023) 只需 1.5 小時。掃描都是自動化的，攻擊者不會在意結果是否即時。

更高階的 (sophisticated) 掃描工具會直接發送封包，而不是去依賴作業系統的 TCP 實作。這種掃描工具快速、有效率，且無論被 REJECT 還是 DROP 都無所謂。

隱藏哪些通訊埠是啟用 (active) 的資訊需要讓伺服器無論是否提供特定服務，回應都必須相同。這在技術上非常困難，且在某些情況下可能比實際提供該服務更容易出錯！

實際上，或許可以透過接受來自未知通訊埠的連線，丟棄 (blackholing) 資料，然後在超時後關閉連線來逃過掃描。然而，若合法使用者的應用程式嘗試連線到這些通訊埠，這麼做會導致問題。

# 總結

| **情境**                                  | **REJECT**           | **DROP**                           |
| :---------------------------------------- | :------------------- | :--------------------------------- |
| 應用程式連線到不存在的服務                | 立即向使用者報告失敗 | 應用程式會暫停很長時間，然後才失敗 |
| 使用作業系統的 connect 進行簡易的網路掃描 | 掃描速度快           | 掃描速度尚可，只要設定好超時       |
| 使用專業掃描工具進行網路掃描 (例如 nmap)  | 掃描速度快           | 掃描速度快                         |

# 結論

DROP 對於有敵意的勢力來說沒有提供有效的阻礙，但卻可能大幅拖慢合法使用者的應用程式。所以 DROP 通常不應被使用。

# 補充

Sean Taylor 提出了有趣的反對觀點:

這是一個非常發人深省的討論。有一種情境下，DROP 具有顯著的優勢，那就是當你遭受 DoS (denial of service) 攻擊，且擁有一個高度不對稱的網路連線 (下載速度遠快於上傳速度)，像是 DSL 連線。

在這種情況下 DROP 會很有幫助，因為攻擊者可能無法壓垮你的下載速度，但 ICMP 回應可能會超過你的上傳速度，意味著你將無法遠程登入並管理正受到攻擊的網路，且合法流量被完全阻擋。

我相信有更好 (sophisticated) 的方法來處理 DoS 攻擊，但作為一個小型伺服器的兼職管理員，且上傳速度極低，DoS 攻擊是我目前面臨的最大問題。
