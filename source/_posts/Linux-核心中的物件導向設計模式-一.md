---
title: Linux 核心中的物件導向設計模式 (一)
date: 2024-08-30 18:07:34
categories:
  - Operating System
tags:
  - C
  - OOP
  - Linux
  - Translation
code_block_shrink: false
---

[![hackmd-github-sync-badge](https://hackmd.io/wzt18MbQTEeNBMA9D-fo-w/badge)](https://hackmd.io/wzt18MbQTEeNBMA9D-fo-w)

> 譯自 [Object-oriented design patterns in the kernel, part 1](https://lwn.net/Articles/444910)
> [name=Neil Brown] 1 June 2011

儘管 Linux 核心大部分使用 C 語言撰寫，它廣泛運用了物件導向程式設計領域中的一些技巧。想要使用這些物件導向技巧的開發者在 C 語言中很少相關的支援或指導，因此只能透過自行探索。通常這樣是一把雙面刃。開發者擁有足夠的靈活性來做出很酷的事情，同時也擁有做出非常愚蠢的事情的靈活性，而乍看之下，往往不好判斷是哪種情況，或者更準確地說: 某個特定方法在這個光譜上的哪個位置。

與其依賴程式語言提供指導，軟體工程師應該參考既有的實作，來了解哪些方式效果良好，哪些則需要避免。解讀既有的實作並不總是向一般人認為的容易，但只要付出努力，是非常值得保存的。為了保留作者的努力，本文將帶來 Linux 核心設計模式系列文章的另一篇章，並嘗試透過範例，闡述在 Linux 核心中實現物件導向風格的設計模式。

雖然提供一個簡短的物件導向風格的介紹很吸引人，但我們將假設讀者已經具備物件、類別、方法、繼承和一些相關詞彙的基礎知識。對於尚未熟悉這些概念的人，可以在網路上找到許多相關資源。

在接下來的兩週，我們將探討兩個領域中的模式: 方法分派和資料繼承。儘管它們看似簡單，但能引導許多深入的探討。本篇文章將專注於方法分派。

# 方法分派 (Method Dispatch)

現今各種繼承風格以及其使用規則的多樣性似乎表明，對於**物件導向**的真正含義沒有統一的理解。這個詞有點像愛 (love): 每個人都認為自己知道它的意思，但當深入探討細節時，人們會發現彼此的想法可能非常不同。儘管**導向** (oriented) 的含義可能不太明確，但**物件** (Object) 的定義似乎是普遍認同的。它只是一個包含狀態與行為的抽象概念。物件就像是 Pascal 中的記錄 `record` 或 C 語言中的結構體 `struct`，但其中一些成員的名稱是指那些作用於物件其他欄位的函式。這些函式成員有時被稱為**方法** (methods)。

在 C 語言中實現物件的最明顯方式是宣告一個 `struct`，其中一些欄位是指向函式的指標 (pointer)，這些函式將指向該 `struct` 的指標作為它們的第一個參數。在物件 `bar` 中呼叫方法 `foo` 的約定方式就是: `bar->foo(bar, ...args);`。儘管這種模式在 Linux 核心中有被使用到，但它並不是主要的模式，因此我們將稍後再進行討論。

由於方法 (不同於狀態) 通常不會針對每個物件進行更改，更常見且僅稍微不那麼顯而易見的做法是將某一類物件的所有方法收集到一個獨立的 `struct` 中，這個結構體有時被稱為虛擬函式表 (virtual function table) 或 `vtable`。這樣物件只需擁有一個指向這個表的指標，而不是為每個方法設置一個單獨的指標，從而使用更少的記憶體。

這就引出了我們的第一個模式 —— 純 `vtable`，它是一個僅包含函式指標的結構體，其中每個函式的第一個參數都是指向其他結構體 (即物件類型) 的指標，而該結構體本身包含一個指向這個 `vtable` 的指標。在 Linux 核心中，這方面的一些簡單例子包括 `file_lock_operations` 結構體，它包含兩個函式指標，每個都接收一個指向 `struct file_lock` 的指標；以及 `seq_operations`，它包含四個函式指標，每個函式都提供操作給一個 `struct seq_file`。這兩個例子顯示了一個明顯的命名模式 —— 持有 `vtable` 的結構體命名為持有物件的結構體名稱 (可能是縮寫) 在其後加上 `_operations`。儘管這種模式很常見，但並非總被適用。在 2.6.39 版本左右，約有 30 個 `_operations` 結構體，以及超過 100 個 `_ops` 結構體，其中大部份都是某物件類型的 `vtable`。還有一些結構體，例如 `struct mdk_personality`，本質上也是 `vtable`，但名稱不太明確。

```c
/* ref: http://lxr.linux.no/#linux+v2.6.39/include/linux/fs.h#L1062 */
struct file_lock_operations {
    void (*fl_copy_lock)(struct file_lock *, struct file_lock *);
    void (*fl_release_private)(struct file_lock *);
};

/* ref: http://lxr.linux.no/#linux+v2.6.39/include/linux/seq_file.h#L29 */
struct seq_operations {
    void * (*start) (struct seq_file *m, loff_t *pos);
    void (*stop) (struct seq_file *m, void *v);
    void * (*next) (struct seq_file *m, void *v, loff_t *pos);
    int (*show) (struct seq_file *m, void *v);
};
```

在這將近 200 個 `vtable` 結構體中存在著大量的變化，因此有很多範圍可以尋找有趣的模式。特別是我們可以尋找與上述**純 vtable 模式**的常見變化，並確定這些變化如何增進我們對於 Linux 中物件使用的理解。

## NULL 函式指標 (NULL function pointers)

第一個觀察到在某些 `vtable` 中，一些函式指標是允許為 `NULL` 的。嘗試呼叫這樣的函式顯然是沒有用的，因此呼叫這些方法的程式碼通常會明確測試該指標是否為 `NULL`。這些 `NULL` 指標存在的理由有幾種。其中最容易理解的原因應該是**疊代式開發** (incremental development)。由於 `vtable` 結構體的初始化 (initialize) 方式，在結構體定義中新增函式指標會導致所有現有的表格宣告將該指標初始化為 `NULL`。因此，可以在任何實例 (instance) 支持該方法之前添加對新方法的呼叫者，並檢查是否為 `NULL`，然後執行預設行為。隨著疊代式開發的進行，那些需要該方法的 vtable 實例可以獲得非預設的方法。

最近的一個例子是 [commit `77af1b2641faf4`](http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=77af1b2641faf4)，它將 `set_voltage_time_sel()` 新增到 `struct regulator_ops` 中，該方法作用於 `struct regulator_dev`。隨後的 [commit `42ab616afe8844`](http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=42ab616afe8844) 為特定設備定義了此方法。這只是這一相當常見主題的最新例子。

另一個常見的原因是特定方法在特定情況下並沒有特別的意義，因此呼叫程式碼只需檢查是否為 NULL，並在發現時回傳適當的錯誤。在虛擬檔案系統 (VFS) 層中有一些這樣的例子。例如，`inode_operations` 中的 `create()` 函式僅在相關的 inode 是目錄 (directory) 時才有意義。所以非目錄的 `inode_operations` 結構體通常會將 `create()` 函式 (以及許多其他函式) 設為 `NULL`，而 `vfs_create()` 中的呼叫程式碼會檢查是否為 `NULL` 並返回 `-EACCES`。

而最後一個 vtable 有時包含 `NULL` 的原因是某些功能可能正在從一個介面 (interface) 過渡到另一個介面。這種方式的一個好例子是 `file_operations` 中的 `ioctl()` 操作。在 2.6.11 版中，新增了一個叫做 `unlocked_ioctl()` 的方法，該方法在沒持有 [Big Kernel Lock](https://kernelnewbies.org/BigKernelLock) 的情況下被呼叫。在 2.6.36 版中，當所有驅動程式和檔案系統都轉換為使用 `unlocked_ioctl()` 後，原始的 `ioctl()` 終於被移除。在此過渡期間，檔案系統通常只會定義其中一個方法，另一個則預設為 `NULL`。

一個較微妙的例子是 `file_operations` 中的 `read()` 和 `aio_read()`，以及相應的 `write()` 和 `aio_write()`。`aio_read()` 是為了支持非同步 IO 而引入的，如果提供了 `aio_read()`，則不再需要一般的同步 `read()` (它是透過 `do_sync_read()` 實現，因為 `do_sync_read()` 會呼叫 `aio_read()` 方法)。在這樣的情況下，就沒有打算移除 `read()`，它將保留在非同步 IO 無關的情況，像 procfs 和 sysfs 這樣的特殊檔案系統。因此檔案系統只需定義每對方法中的一個，但這不僅僅是過渡期，而是一種長期的狀態。

儘管使用 `NULL` 函式指標似乎有幾種不同的原因，但幾乎所有例子都可以總結為一個簡單的模式 —— 為該方法提供一個預設的實作。在疊代式開發的例子和無意義方法的情況下，這是相當直接的。例如，`inode->create()` 的預設行為只是回傳一個錯誤。在介面轉變的情況下，稍微不那麼明顯。`unlocked_ioctl()` 的預設行為是取得 kernel lock，然後呼叫 `ioctl()` 方法。`read()` 的預設行為正是 `do_sync_read()`，而一些檔案系統像是 ext3 實際上明確提供了這個預設函式，而不是使用 `NULL` 來表示預設行為。

考慮到這一點，稍微反思一下會發現: 如果真正的目標是提供一個預設行為，那或許最好的方式是明確地給出一個預設值，而不是使用 `NULL` 作為預設值並進行特殊處理這種迂迴的方法。

雖然 `NULL` 肯定是最容易作為預設值提供的選項 —— 因為 C 標準保證結構體中未初始化的成員會被設置為 `NULL` (在 `static` 或是 global 的 scope) —— 但設置一個更有意義的預設值也並不困難。非常感謝 LWN 的讀者 wahern 指出 C99 允許結構體中的欄位被多次初始化，並且只有最後一個值會生效，這使得設置預設值變得更加容易，像是以下的簡單範例:

```c
#define FOO_DEFAULTS  .bar = default_bar, .baz = default_baz
struct foo_operations my_foo = { FOO_DEFAULTS,
    .bar = my_bar,
};
```

這樣會宣告 `my_foo` 並具有 `baz` 預定義的預設值，而 `bar` 則具有本地自訂的值。因此只花費少量代價定義一些**預設函式**並在每個宣告中包含一個 `_DEFAULTS`，就可以輕鬆地在欄位首次建立時選擇其預設值，並在每次使用結構體時自動包含該預設值。

有意義的預設值不僅易於實作，還可以導致更有效率的實作。在那些函式指標實際為 `NULL` 的情況，進行測試和分支可能比間接的函式調用更快，但 `NULL` 的情況通常是例外 (exception) 而不是常規，為例外情況進行最佳化並不符合一般慣例。在函式指標不為 `NULL` 這樣更為常見的情況下，測試 `NULL` 只是浪費程式碼空間和執行時間。如果我們禁止 `NULL` 值，則可以使所有函式呼叫的位置變得更小更簡單。

一般來說，任何在呼叫方法之前由呼叫者 (caller) 執行的測試都可以視為[之前文章](https://lwn.net/Articles/336262/)中討論的 **mid-layer mistake** 的實例。這表明中間層 (mid-layer) 在對底層驅動程式的行為做出假設，而不是簡單地給予驅動程式自由去做出最合適的行為。這可能不總是一個昂貴的錯誤，但仍應盡量避免。不過在 Linux 核心中，vtable 中的指標有時可以為 `NULL`，通常但非總是為了過渡，並且在這些情況下，呼叫位置就應在進行之前檢查是否為 `NULL`。

細心的讀者可能注意到上述的邏輯中反對使用 `NULL` 指標作為預設值的論點存在一個漏洞。在預設情況是常見情況且效能至關重要的情況下，這種推理並不成立，此時使用 `NULL` 指標是合理的。Linux 核心提供了這樣一個天然的情況供我們研究。

VFS 用於快取檔案系統資訊的資料結構之一是 `dentry`。一個 `dentry` 代表檔案系統中的一個名稱，因此每個 `dentry` 都有一個親代，表示包含它的目錄，還有一個 `inode` 來代表該名稱所對應的檔案。`dentry` 與 `inode` 是分開的，因為單個檔案可以有多個名稱 (因此一個 `inode` 可以對應多個 `dentry`)。`dentry_operations` vtable 包含了多個行為，例如 `d_compare` 用於比較兩個名稱，`d_hash` 用於為名稱生成 hash，用來引導 `dentry` 在 hash table 的位置。大多數檔案系統不需要這種彈性，它們將名稱視為未經解釋的字串位元組 (strings of bytes)，因此預設的比較和 hash 函式是較常見的情況。少數檔案系統會定義這些函式來處理不區分大小寫的名稱，但這不是普遍的情況。

另外，檔案名稱搜尋是 Linux 中一個常見的操作，因此最佳化它是優先的。因此這兩個操作似乎是適合進行 `NULL` 測試並內聯 (inline) 預設操作的良好候選。但我們發現當這樣的最佳化是合理時，僅僅這樣還不夠。呼叫 `d_compare()` 和 `d_hash()` (以及其他 `dentry` 操作) 的程式碼並不直接測試這些函式是否為 `NULL`，它們要求在 `dentry` 中設置一些旗標 (flag bits，如 `DCACHE_OP_HASH` 和 `DCACHE_OP_COMPARE`)，以指示應使用常見的預設操作，還是應該呼叫該函式。由於這些旗標很可能已經在快取中，而 `dentry_operations` 結構體通常根本不需要，這種方法避免了在 hot path 中進行記憶體提取。

所以我們發現，在唯一使用 `NULL` 函式指標來表示預設值的情況下，實際上並未被使用；取而代之的是使用了一種不同且更有效的機制來指示需要呼叫預設方法。

## 函式指標以外的成員 (Members other than function pointers)

儘管核心中的大多數類似 vtable 的結構體只有包含函式指標，但仍有不少包含非函式指標的欄位。表面上看，這些欄位中有許多似乎相當隨意，進一步檢查後發現，其中一些可能是糟糕設計或程式碼腐敗 (bit-rot) 的結果，移除它們只會改進程式碼。

有一個**只有函式**的模式的例外情況不斷出現且提供真正的價值，因此值得探討。這種模式最普遍的形式出現在 `struct mdk_personality` 中，它為特定的軟體磁碟陣列 (software RAID) 層級提供一些操作。特別是這個結構體包含一個 `owner`、`name` 和 `list` 欄位。`owner` 是提供實現的模組。`name` 是一個簡單的識別碼: 有些 vtable 使用字串名稱，有些使用數字名稱，而且它經常被稱為不同的名字，如 `version`、`family`、`drvname` 或 `level`。但從概念上來說，它仍然是一個名稱。在這個例子中，有兩個名稱: 一個是字串名稱，另一個是數字 `level`。

`list` 雖然屬於相同的功能，但比較少見。`mdk_personality` 結構體中有一個 `struct list_head`，`struct ts_ops` 也是如此。`struct file_system_type` 則有一個指向下一個 `struct file_system_type` 的簡單指標。這裡的基本概念是，為了使任何特定的介面實現 (或類別 (class) 的最終定義）可用，必須以某種方式將其註冊，以便能夠找到它。此外，一旦找到它就必須確保擁有該實現的模組在使用期間不會被移除。

在 Linux 核心中，針對介面的註冊方式幾乎有多少介面需要註冊就有多少種註冊風格，因此在這裡找到一致性的模式會是一項困難的任務。然而將 vtable 視為特定介面實現的主要方法，並且包含一個用來指向提供該介面實現模組的 `owner` 指標是相當常見的做法。

所以我們發現的模式是: 作為物件方法分派的 vtable 使用的函式指標結構體通常應只包含函式指標，例外情況需要明確的理由。常見的例外包括允許模組指標以及其他可能的欄位，如名稱和列表指標。這些欄位用於支援特定介面的註冊協定。當沒有列表指標時，整個 vtable 很可能會被視為唯讀 (read-only)。在這種情況下，vtable 通常會被宣告為 `const struct`，因此甚至可以存儲在唯讀記憶體中。

## 結合不同物件的方法 (Combining Methods for different objects)

我們在 Linux 核心看到的**純 vtable 模式**的最後一種偏離常見情況出現於當函式的第一個參數並不總是相同的物件類型時。在一個純 vtable 中，該 vtable 由特定資料結構中的指標引用，每個函式的第一個參數就是該資料結構本身。那麼，偏離這種模式的原因是什麼呢? 事實證明有幾個原因，而其中一些比其他的更有趣。

最簡單且最無趣的解釋是: 沒有明顯原因的情況下，目標資料結構在參數清單的其他位置被列出。例如 `struct fb_ops` 中的所有函式都接收一個 `struct fb_info`。儘管在 18 個案例中，該結構體都是第一個參數，但在 5 個案例中，它是最後一個參數。這個選擇沒有明顯的錯誤，且不太可能讓開發者感到困惑。這只是對於像本文作者這樣的資料探勘人員來說，是一個需要過濾的無關模式。

在 `struct rfkill_ops` 中，我們看到這個模式的一個些微變化，其中有兩個函式接收一個 `struct rfkill`，但第三個函式 `set_block()` 則接收一個 `void *data`。進一步調查顯示這個不清楚的 `data` 正是儲存在 `rfkill->data` 中的內容，因此 `set_block()` 完全可以定義為接收一個 `struct rfkill`，並自行加上 `->data`。這種偏差非常不清楚，可能會讓開發者以及資料探勘人員感到困惑，應該避免。

接下來的變化可以在 `platform_suspend_ops`、`oprofile_operations`、`security_operations` 以及其他一些結構體中看到。這些函式接收各種奇特的參數，沒有明顯的模式。然而這些實際上是非常不同類型的 vtable 結構體，因為它們所屬的物件是單例 (singleton) 模式。只有一個活躍的平台、一個分析器、只有一個安全策略。因此這些操作所作用的物件是全域 (global) 狀態的一部分，因此不需要包含在任何函式的參數中。

在排除這兩種沒那麼有趣的模式後，我們還剩下的兩種模式確實能告訴我們一些關於 Linux 核心中物件使用的資訊。

`quota_format_ops` 和 `export_operations` 是兩個不同的儲存操作的結構體，它們作用於各種不同的資料結構。在每個情況下，表面上的主要物件 (例如 `struct super_block` 或 `struct dentry`) 已經有一個專屬的 vtable 結構 (如 `super_operations` 或 `dentry_operations`)，而這些新結構則新增一些操作。在每個情況下，這些新操作形成了一個匯聚的單位，提供了一組相關的功能 —— 無論是支持磁碟配額還是 NFS 匯出。它們並非都作用於相同的物件，只因所涉及的功能依賴於多種物件。

最適合描述這種情況的**物件導向程式設計術語**可能是 **mixin**。儘管適用性可能不夠完美 —— 取決於你對 mixin 的具體理解 —— 但引入一組功能而不使用嚴格的階層繼承的概念，與 `quota_format_ops` 和 `export_operations` 的目的非常接近。

一旦我們知道要關注這類 mixin，我們就能找到更多例子。我們要察覺的模式不是最初引導我們的那種 —— 包含操作的結構體作用在多種不同物件上，而是我們發現到這種結構體中的函式作用於已經有屬於自己的操作結構體的物件。當一個物件有大量相關操作時，這些操作自然的形成子集，將它們分為獨立的類似 vtable 的結構體是非常合理的。在網路的程式碼中有幾個例子，例如 `tcp_congestion_ops` 和 `inet_connection_sock_af_ops` 都主要作用於 `struct sock`，而 `struct sock` 本身已經有一小組專用的操作。

因此，mixin 模式 —— 至少定義為一組適用於一或多個物件但並非這些物件的主要操作的操作集合 —— 是一種在 Linux 核心中經常出現的模式，並且在允許更好的程式碼模組化方面顯得相當有價值。

解釋函式目標不一致的最後一種模式可能是最有趣的，尤其是它與物件導向程式設計風格的明顯應用形成的對比。這種模式大量存在於 `ata_port_operations`、`tty_operations`、`nfs_rpc_ops` 和 `atmdev_ops`，這些都是有用的例子。然而我們將主要關注檔案系統層的一些例子，特別是 `super_operations` 和 `inode_operations`。

檔案系統的實作中存在一個強烈的物件階層，其中檔案系統 (由 `super_block` 表示) 包含多個檔案 (`struct inode`)，這些檔案可能有多個名稱或連結 (`struct dentry`)。此外每個檔案可能會在分頁快取 (page cache) 中儲存資料 (`struct address_space`)，而分頁快取包含多個獨立的分頁 (`struct page`)。這裡有個概念: 所有這些不同的物件都屬於整個檔案系統。如果需要將資料從檔案載入到分頁，檔案系統知道如何執行，並且對每個檔案中的每個分頁來說，可能使用的機制都是相同的。在機制不總是相同的情況下，檔案系統也知道該如何處理。所以我們可以想到將這些物件上的每個操作都存儲在 `struct super_block` 中，因為它表示檔案系統，並且知道在每種情況下該做什麼。

這種極端做法不實用在實際的情況。雖然一般的檔案和目錄的儲存方式間可能有相似之處，但也有重要的區別，將這些區別編碼在不同的 vtable 中可能會有所幫助。有時小型符號連結 (symbolic links) 直接儲存在 `inode` 中，而較大的連結則像一般檔案的內容一樣儲存。對這兩種情況使用不同的 `readlink()` 操作可以使程式碼更加好讀。

儘管將每個操作都附加到一個集中的結構上這樣的極端做法並不理想，同樣地，另一個極端也並非理想。Linux 中的 `struct page` 根本沒有 vtable 指標 —— 部分原因是我們希望保持該結構體儘可能小，因為它的數量非常龐大。相反，`address_space_operations` 結構體包含了作用於分頁的操作。同樣地，`super_operations` 結構體包含了一些適用於 `inode` 的操作，而 `inode_operations` 則包含了一些適用於 `dentry` 的操作。

顯然可以將包含操作的結構體附加到目標物件的親代物件上 —— 前提是目標物件通常持有對親代物件的參考 —— 雖然這樣做是否是有益的並不總是那麼明顯。對於完全避免擁有 vtable 指標的 `struct page`，這樣做的好處是明顯的。但對於擁有自己 vtable 指標的 `struct inode`，將某些操作 (如 `destroy_inode()` 或 `write_inode()`)附加到 `super_block` 上的好處則不明顯。

由於在多個 vtable 結構中都可以存放任何給定的函式指標，實際的選擇在許多情況下只是歷史的偶然產物。確實，`inode_operations` 中 `struct dentry` 操作的大量出現，很大程度上是因為其中一些操作曾直接作用於 `inode`，但 VFS 的最後變更讓這些操作有所改變。例如在 2.1.78-pre1 版中，`link()`、`readlink()`、`followlink()` (以及一些現在已經廢棄的操作) 從使用 `struct inode` 改為使用 `struct dentry`。這為在 `inode_operations` 中包含 `dentry` 的操作奠定了基礎，因此當 `setattr` 和 `getattr` 在 2.3.48 版中被新增時，儘管它們主要作用於 `dentry`，但將它們包含在 `inode_operations` 中似乎是非常自然的。

或許我們可以通過完全擺脫 `dentry_operations` 來簡化事情。某些作用於 `dentry` 的操作已經位於 `inode_operations` 和 `super_operations` 中 —— 為什麼不將它們全部移到這裡呢? 雖然 `dentry` 的數量沒有 `struct page` 那麼多，但仍然相當龐大，移除 `d_op` 欄位可以節省該結構體所使用的 5% 記憶體 (在 x86-64)。

除了兩個例外，每個活躍的檔案系統都只有一個 `dentry_operations` 結構體在運行。一些檔案系統實現，像是 vfat，定義了兩個 —— 例如，一個進行區分大小寫匹配，另一個不區分大小寫匹配 —— 但每個 super-block 只有一個是活躍的。因此，`dentry_operations` 中的操作似乎可以移到 `super_operations`，或至少透過 `s_d_op` 來存取。而這兩個例外是 ceph 和 procfs。這些檔案系統在檔案系統的不同部分使用不同的 `d_revalidate()` 操作，而在 procfs 的情況，還使用不同的 `d_release()` 操作。這些必要的區別可以輕鬆地在每個 superblock 版本的這些操作中進行。而這些情況能證明那 5% 空間成本是合理的嗎? 可能不行。

## 直接嵌入的函式指標 (Directly embedded function pointers)

最後有必要回顧一下前面提到的另一種模式，即函式指標直接儲存在物件中，而不是在單獨的 vtable 結構中。這種模式可以在 `struct request_queue` (包含九個函式指標)、`struct efi` (包含十個函式指標)和 `struct sock` (包含六個函式指標) 中看到。

嵌入式指標的代價顯然是空間成本。當使用 vtable 時，通常只有一個 vtable 副本和多個物件副本，因此如果需要多個函式指標，vtable 可以節省空間。vtable 的代價是額外的記憶體引用，儘管在某些情況下快取可能會減少這種成本。vtable 也有彈性上的成本。當每個物件需要完全相同的操作集合時，vtable 是不錯的選擇，但如果需要為每個物件個別定制某些操作，那麼嵌入式函式指標可以提供這種彈性。這一點在 `struct pcmcia_socket` 中 `zoom_video` 的註解中得到了很好地闡述:

```c
/* Zoom video behaviour is so chip specific its not worth adding
   this to _ops */
```

因此，當物件數量不多、函式指標清單很小並且需要多個 mixin 時，嵌入式函式指標會被用來取代單獨的 vtable。

## 方法分派小結 (Method Dispatch Summary)

如果我們將在 Linux 中發現的所有模式元素結合起來，我們可以發現:

> 操作特定類型物件的方法指標通常會被蒐集在一個直接與該物件相關聯的 vtable 中，但它們也可以出現在以下情況中:
>
> - 在一個 mixin vtable 中，該 vtable 收集了相關的功能，這些功能可以獨立於物件的基本類型進行選擇
> - 在親代物件的 vtable 中，這樣做可以避免在多數物件中需要一個 vtable 指標。
> - 直接嵌入到物件中，當函式指標很少或它們需要為特定物件量身定制時。
>
> 這些 vtable 很少包含函式指標以外的東西，儘管為物件類別註冊所需的欄位是合適的。允許這些函式指標為 NULL 是一種常見的處理預設值的技巧但不一定理想。

所以在探索 Linux 核心程式碼時，我們發現即使它不是用物件導向程式語言撰寫的，它確實包含物件、類別 (以 vtable 表示)，甚至 mixin。它還包含了物件導向語言中通常沒有的概念，比如將物件方法委派給親代物件。

希望了解這些不同的模式以及在它們之間做出選擇的原因，能夠促進這些模式在 Linux 核心中更一致的應用，從而使新手更容易理解正在遵循的模式。在我們對物件導向模式的第二部分探討中，我們將研究在 Linux 核心中實現資料繼承的各種方式，並討論每種方法的優缺點，以了解每種方法最適合的情境。
