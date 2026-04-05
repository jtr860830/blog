---
title: 認識 inode
date: 2026-04-06 00:03:30
categories:
  - Linux
tags:
  - Linux
  - Operating System
---

[![hackmd-github-sync-badge](https://hackmd.io/TcoIW5oGS8KzDeW32bkkCQ/badge)](https://hackmd.io/TcoIW5oGS8KzDeW32bkkCQ)

# inode 是什麼

inode 是 Unix-like 作業系統的檔案系統核心資料結構，每個檔案或目錄在磁碟上都對應一個 inode。它儲存關於檔案的所有 metadata，**但不包含檔名**。主要包含 (但不限於) 以下資訊:

- 類型: 一般檔案 (**r**egular file)、目錄 (**d**irectory)、軟連結 (sym**l**ink)、裝置 (device)、socket...
- 權限與所有權: mode bits、uid、gid
- 大小: 位元組數
- 時間戳記: atime (最後存取)、mtime (最後修改)、ctime (inode 本身最後變更)
- 連結數 (nlink): 有多少目錄 entry 指向這個 inode，也就是 **硬連結 (hard link) 的數量**
- 區塊指標 (block pointers): 指向實際資料所在的磁碟區塊

## 為什麼檔名不在 inode 裡

因為目錄本身就是一張對應表，記錄檔案名稱跟 inode 編號的對應。這個設計帶來幾個重要的特性:
- 方便硬連結實作，因為同一個 inode 可以被多個目錄 entry 指向，連結數計數加一，不佔額外磁碟空間
- 讓檔案重新命名變的單純，因為改檔名只是改目錄裡的 entry，inode 完全不動
- 軟連結 (symbolic link, 符號連結) 與硬連結機制分離。軟連結是另一個檔案，有不一樣的 inode 編號，檔案內容是指向原路徑的字串

## inode 名字由來

關於 **i** 的真正含意，沒有留下明確的說法。最廣被接受的推測有幾個:

- index: **i** 代表 **index**，因為 inode 在早期 Unix 的磁碟中是以固定大小的陣列儲存，可以用編號直接取得
- information: 也有人認為是 **information node** 的縮寫，描述其儲存 metadata 的功能

# 觀察 inode 編號

```shell
# 查看檔案的 inode 編號
ls -i file.txt

# 查看 inode 詳細資訊
stat file.txt

# 查看檔案系統的 inode 使用量
df -i
```

> 值得注意的是，inode 用完也會造成磁碟滿了的現象——即使磁碟中尚有空間，只要 inode 耗盡，就無法再建立新檔案。這在有**大量小檔案**的場景是有可能發生的問題。

# Linux 更新機制跟 inode 的關係

假設有一個行程 (process) 正在執行，它載入了 `libssl.so.3` 這個函式庫。現在系統更新了這個函式庫。更新程式 (例如 `apt`) 做的事情不是直接覆蓋原本的檔案，而是:

1. 刪除舊的 `libssl.so.3` (解除檔名跟 inode 的連結，讓 inode 的連結數減一)
2. 建立新的 `libssl.so.3` (新檔案，分配一個全新的 inode 編號)

這樣檔案換新之後，行程還是能夠繼續用原始的 inode 編號持有舊的函式庫檔案，更新後新執行的行程也能夠拿到新的 inode 編號達到更新的效果。

這邊要特別提到 Linux 的刪除機制，只要還有資源持有這個 inode，資料就不會真正從磁碟被刪除。連結數歸零之後，如果還有行程開著這個檔案，作業系統核心會等到所有人都關閉它，才真正釋放磁碟空間。

所以舊行程繼續用舊的 inode 跑舊版的函式庫，完全不受影響；新啟動的行程則會載入新的函式庫，兩者互不干涉。