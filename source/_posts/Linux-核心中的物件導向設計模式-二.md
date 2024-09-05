---
title: Linux 核心中的物件導向設計模式 (二)
date: 2024-09-06 00:49:41
categories:
  - Operating System
tags:
  - C
  - OOP
  - Linux
  - Translation
code_block_shrink: false
---

[![hackmd-github-sync-badge](https://hackmd.io/R_teixpOTMmUOk-e3JMHjg/badge)](https://hackmd.io/R_teixpOTMmUOk-e3JMHjg)

> 譯自 [Object-oriented design patterns in the kernel, part 2](https://lwn.net/Articles/446317)
> [name=Neil Brown] 7 June 2011

在這份分析的第一部分中，我們探討了 Linux 核心中如何使用一般的 C 語言語法來實作物件導向程式設計中的多型 (polymorphic)。我們特別研究了方法分派，探討 vtable 的不同形式，以及在何種情況下會避免使用獨立的 vtable，而選擇將函式指標直接儲存在物件中。在這個結論部分，我們將探討物件導向程式設計的另一個重要觀點 —— 繼承 (inheritance)，尤其是資料繼承。

# 資料繼承 (Data inheritance)

繼承是物件導向程式設計的核心概念之一，但它有多種形式，包括原型 (prototype) 繼承、mixin 繼承、子類型 (subtype) 繼承、介面 (interface) 繼承等，其中一些形式是重疊的。在探討 Linux 核心時，我們關注的形式最類似於子類型繼承，即具體或**最終**類型從**虛擬**的親代 (parent) 類型繼承一些資料欄位。為了強調這裡繼承的是資料而非行為，我們將這種形式稱為**資料繼承**。

換句話說，某個介面的多種不同實作共享並分別擴充了一個通用的資料結構。它們可以被說明為繼承了這個資料結構。在 Linux 核心中對這種共享 (sharing) 和擴充 (extending) 有三種不同的實作方式，所有這些方式都可以通過探討 `struct inode` 結構體及其演變來觀察到，儘管這些方式在其他地方也被廣泛使用。

## 透過 union 擴充 (Extension through unions)

第一種方法，可能是最明顯但也是最不靈活的方法，是將一個 `union` 宣告為通用結構體的一個元素，並且為每種實作在此 `union` 中宣告一個包含該特定實作所需的額外欄位項目。這種方法在 Linux-0.97.2 (1992 年 8 月) 中[引入](http://git.kernel.org/?p=linux/kernel/git/history/history.git;a=commitdiff;h=eb79918f272fe119902db3028e0fbdc752f4942d#patch22)到 `struct inode` 中，當時 `struct inode` 被添加了以下內容:

```c
union {
    struct minix_inode_info minix_i;
    struct ext_inode_info ext_i;
    struct msdos_inode_info msdos_i;
} u;
```

在 0.97.5 版本之前，這些結構體都保持為空，直到 `i_data` 從 `struct inode` [移動](http://git.kernel.org/?p=linux/kernel/git/history/history.git;a=commitdiff;h=06d9f6ff137579551a2ee18661847915fe2bb812#patch32)到 `struct ext_inode_info`。多年來不同檔案系統增加了更多的 `inode_info` 欄位，並在 2.4.14.2 版本中增加 28 個不同的 `inode_info` 結構體時達到高峰，在當時[加入了 ext3](http://git.kernel.org/?p=linux/kernel/git/tglx/history.git;a=blobdiff;f=include/linux/fs.h;h=935c6e9bfee8d331db28832c54e4bd99d6563e97;hp=33f3bb92af4011eb33a0b99c483803787c436d88;hb=a8a2069f432c5597bdf9c83ab3045b9ef32ab5e3;hpb=5db5272c0a5cd37e5a697e4750fbc4ce6317b7dc)。

這種資料繼承的方法簡單直接，但也有些笨拙。這裡有兩個明顯的問題: 首先，每個新的檔案系統實作都需要在 `union u` 中新增一個額外的欄位。當有 3 個欄位時，這可能看起來不是個問題，但當有 28 個時，就遠超過醜陋了。要求每個檔案系統都要更新這個結構體，增加新增檔案系統的負擔是沒必要的。其次，每個分配的 `inode` 都會是相同大小且會足夠大以儲存任何檔案系統的資料。所以一個需要大量空間來儲存其 `inode_info` 結構體的檔案系統，會將這個空間成本附加到所有其他檔案系統上。

第一個問題並不是一個無法克服的障礙，正如我們等等看到那樣。而第二個問題則是真正的麻煩，這種設計醜到促使了變更。在 2.5 版本開發系列的早期開始這項變更；在 2.5.7 版本時，`union u` 中已經沒有任何 `inode_info` 結構體 (雖然這個 `union` 本身被保留到 2.6.19 版本)。

## 嵌入式結構體 (Embedded structures)

在 2.5 版本的早期，`inode` 發生的變化實際上是一種反轉。這種變化將從 `struct inode.u` 中[移除](http://git.kernel.org/?p=linux/kernel/git/tglx/history.git;a=blobdiff;f=include/linux/fs.h;h=6bda17aed79ac146466c42bd2355c246e4814d0e;hp=4e5de1286d87969509d812eb9c2f813ae61fd252;hb=463727d199b089c420e750d43f75ea9403a45e12;hpb=0713f0290054eb9769d588120712e3dccfb3ec34) `ext3_i`，並在 `struct ext3_inode_info` 中[新增](http://git.kernel.org/?p=linux/kernel/git/tglx/history.git;a=blobdiff;f=include/linux/ext3_fs_i.h;h=104aea4e0c1958fe684c5fe02f33d02484adaa30;hp=3c8d398a81039044765ad86adaf09d2018867fd1;hb=463727d199b089c420e750d43f75ea9403a45e12;hpb=0713f0290054eb9769d588120712e3dccfb3ec34)一個 `struct inode vfs_inode`。所以原本嵌入在通用資料結構體中的私有結構體，現在變成了通用資料結構體嵌入在私有結構體中。這巧妙地避免了使用 `union` 的兩個問題；現在每個檔案系統只需要分配記憶體來儲存自己的結構體，而不需要了解其他檔案系統可能需要什麼。當然，這種變化並非沒有開銷，它也帶來了一些需要解決的其他問題，但解決這些問題並不昂貴。

第一個困難是當通用檔案系統的程式碼 (VFS 層) 呼叫特定檔案系統時，它會傳遞一個指向通用資料結構體 `struct inode` 的指標。使用這個指標檔案系統需要找到一個指向其私有資料結構的指標。一個顯而易見的方法: 都將 `struct inode` 放在私有 `inode` 結構的最上層，並簡單地將一個指標強制轉換為另一個指標。儘管這樣可行，但它缺乏任何型別安全的樣子，並使在 `inode` 中安排欄位以獲得最佳化的效能變得更加困難 —— 而這正是一些核心開發者習慣做的事情。

解決方案是使用 [`list_entry()`](http://lxr.linux.no/#linux-bk+v2.5.2/include/linux/list.h#L145) 巨集 (macro) 來執行必要的指標運算，通過從 `struct inode` 的地址中減去它在私有資料結構中的偏移量，然後進行適當的轉換。這個巨集被稱為 `list_entry()`，只是因為 `list.h` 的實作是第一個使用這種資料結構嵌入模式的。`list_entry()` 巨集準確地完成了所需的工作，因此儘管名稱有些奇怪，還是使用了這個巨集。這種做法一直持續到 2.5.28 版本，當時[新增](http://git.kernel.org/?p=linux/kernel/git/tglx/history.git;a=commitdiff;h=ec4f214232cfb99913308c20b9a3381e5fe1f04f#patch15)了一個名為 `container_of()` 的巨集，它實作了與 `list_entry()` 相同的功能，但具有略好的型別安全和更有意義的名稱。使用 `container_of()`，簡化了從嵌入的資料結構映射到其所嵌入的結構體。

第二個困難在於檔案系統必須負責分配 `inode`，因為通用的程式碼不再能夠分配正確大小的空間，因為它沒有足夠的資訊。為了解決這個問題，只需在 `super_operations` 結構體中[新增](http://git.kernel.org/?p=linux/kernel/git/tglx/history.git;a=blobdiff;f=include/linux/fs.h;h=4e5de1286d87969509d812eb9c2f813ae61fd252;hp=a01f0c3b4d34df475a84532417c42d46eb0974ed;hb=468e6d17ff42e6f291a88c87681b2b5e34e9ab33;hpb=d694597ed5e1f6613d0933ee692333ab2542b603) `alloc_inode()` 和 `destroy_inode()` 方法並適當呼叫它們。

## void 指標 (void pointers)

如前所述，`union` 模式並不是新增檔案系統的無法逾越的障礙。這是因為 `union u` 還有一個不是 `inode_info` 結構體的欄位。在 Linux-1.0.5 中[新增](http://git.kernel.org/?p=linux/kernel/git/history/history.git;a=commitdiff;h=4aad5d636d7c5a543a82757d9be2e3f3e5c6724f#patch14)了一個名為 `generic_ip` 的通用指標欄位，但直到 1.3.7 才開始使用。任何沒有在 `struct inode` 本身中擁有結構體的檔案系統都可以定義並分配一個單獨的結構體，並通過 `u.generic_ip` 將其連接到 `inode`。這種方法解決了 `union` 的兩個問題，因為不需要對共享的宣告進行任何更改，每個檔案系統只使用其需要的空間。然而它再次引入了自身的新問題。

使用 `generic_ip` 時，每個檔案系統需要為每個 `inode` 進行兩次分配，而不是一次，這可能會導致更多的浪費，具體取決於分配時結構體大小四捨五入的方式；此外，還需要撰寫更多的錯誤處理的程式碼。還有一部分記憶體用於 `generic_ip` 指標，通常還有一個從私有結構體指回通用 `struct inode` 的指標。與 `union` 方式或嵌入方式相比，這兩者都會導致空間浪費。

更糟糕的是，從通用結構體存取私有結構體時，需要額外的記憶體解參考 (dereference)；這種解參考最好避免。檔案系統程式碼通常需要同時存取通用結構體和私有結構體。這可能需要大量額外的記憶體解參考，或需要在暫存器中儲存私有結構體的位址，這會增加暫存器的負擔。正是這些顧慮阻止 `struct inode` 廣泛使用 `generic_ip` 指標。儘管這個指標確實被使用過，但並未被主流的高效能檔案系統採用。

儘管這種模式存在問題，但它仍然被廣泛使用。`struct super_block` 具有一個 `s_fs_info` 指標，其用途與 `u.generic_ip` 相同 (後者在 `union u` 最終被移除後更名為 `i_private` —— 為何沒有徹底移除，這留給讀者自行探討)。這是將檔案系統私有資料儲存在 `super_block` 中的唯一方式。簡單搜尋 Linux 的檔案，會發現有相當多的欄位是名為 **private** 或類似名稱的 `void` 指標。這些欄位中有許多是通過使用指向私有擴充的指標來擴充資料型別的例子，其中大多數都可以轉換為使用嵌入式結構體的模式。

## inodes 之外 (Beyond inodes)

雖然 `inode` 是介紹這三種模式的有效載體，但它們並未充分展示這些模式的全部範圍，因此我們有必要進一步探討，看看還能學到什麼。

對核心中其他地方使用 `union` 的調查顯示: `union` 被廣泛使用，雖然情境與 `struct inode` 中的情況非常不同。`inode` 中缺少的特定方面是: 各種不同的模組 (不同的檔案系統) 都想以不同的方式擴充 `inode`。而在大多數使用 `union` 的地方，基礎型別的子型別數量是固定的，並且不太可能會新增。一個簡單的例子是 [`struct nfs_fattr`](http://lxr.linux.no/#linux+v2.6.39/include/linux/nfs_xdr.h#L34)，它儲存從 NFS 回應中解碼的檔案屬性資訊。這些屬性的細節在 NFSv2 和 NFSv3 之間略有不同，因此這個結構體有兩個子型別，差異通過 `union` 編碼。由於 NFSv4 與 NFSv3 使用相同的資訊，因此這不太可能再進一步擴充。

在 Linux 中，`union` 其他常見的一種使用模式是用來編碼傳遞的訊息，通常在核心與使用者空間 (user space) 之間傳遞。[`struct siginfo`](http://lxr.linux.no/#linux+v2.6.39/include/asm-generic/siginfo.h#L40) 用來在訊號傳遞時夾帶額外資訊。每種訊號型別都有不同的補充資訊型別，因此 `struct siginfo` 使用了一個 `union` 來編碼六種不同的子型別。[`union inputArgs`](http://lxr.linux.no/#linux+v2.6.39/include/linux/coda.h#L654) 似乎是目前最大的 `union`，具有 22 種不同的子型別。它被 coda 網路檔案系統 (network file system, NFS) 用來在核心模組與處理網路通訊的使用者空間常駐程式之間傳送請求。

目前不清楚這些例子是否應被視為與最初 `struct inode` 相同的模式。它們是否代表了基礎型別的不同子型別，還是只是具有內部變體的單一型別? Eiffel 物件導向程式語言完全不支持變體類型，除非通過子類型繼承，因此顯然有一派認為應將所有 `union` 的使用視為一種子型別的形式。許多其他語言如 C++，則同時提供繼承和 `union`，允許程式設計師做出選擇。因此答案並不明確。

對於我們的目的來說，無論我們如何稱呼它，只要知道何時使用哪種模式。核心中的例子清楚地表明，當所有變體由單一模組理解時，`union` 就是非常合適的變體結構機制，不管是否將其視為使用資料繼承。當不同的子型別由不同的模組或至少是彼此分開的程式碼片段管理時，則更傾向於使用其他機制。這種情況下使用 `union` 的做法幾乎已經消失，目前僅剩 [`struct cycx_device`](http://lxr.linux.no/#linux+v2.6.39/include/linux/cyclomx.h#L43) 作為過時模式的例子。

## void 指標的問題 (Problems with void pointers)

`void` 指標不太容易分類。可以說，`void` 指標在現代相當於 `goto` 敘述。它們可以非常有用，但也可能導致非常迂迴的設計。特別的問題在於: 當你看到一個 `void` 指標時，就像看到一個 `goto` 一樣，你並不知道它實際指向什麼。一個名為 `private` 的 `void` 指標甚至更糟糕 —— 就像一個 `goto destination` 指令 —— 如果不閱讀大量上下文，它幾乎沒有意義。

檢視 `void` 指標的所有不同用途，遠遠超出了本文的範疇。因此，我們將僅限於探討一種與資料繼承相關的新用法: 說明 `void` 指標不受約束的特性如何使其在資料繼承中的使用中難以識別。我們用來解釋這種用法的例子是 [`struct seq_file`](http://lxr.linux.no/#linux+v2.6.39/include/linux/seq_file.h#L16)，它由 `seq_file` 函式庫使用，用來讓函式庫生成簡單的文字檔案變得容易，像 `/proc` 中的一些檔案那樣。`seq_file` 中的 `seq` 部分表示該檔案包含一系列行，對應到核心中一系列的資訊項目，所以 `/proc/mounts` 是一個 `seq_file`，它走訪掛載表 (mount table) 並將每個掛載回報為一行。

當使用 [`seq_open()`](http://lxr.linux.no/#linux+v2.6.39/fs/seq_file.c#L30) 建立新的 `seq_file` 時，它會分配一個 `struct seq_file` 並將其賦值給正在開啟的 `struct file` 的 `private_data` 欄位。這是一個基於 `void` 指標的資料繼承的直接例子，其中 `struct file` 是基礎型別，而 `struct seq_file` 是對該類型的簡單擴充。這個結構體從不單獨存在，總是作為某個檔案的 `private_data`。`struct seq_file` 本身有一個 `private` 欄位，這是一個 `void` 指標，`seq_file` 的客戶端可以使用它來為檔案添加額外的狀態。例如，[`md_seq_open()`](http://lxr.linux.no/#linux+v2.6.39/drivers/md/md.c#L6496) 分配一個 `struct mdstat_info` 結構體並通過這個 `private` 欄位附加它，用來滿足 md 的內部需求。這又是一個遵循正所描述模式的簡單資料繼承。

然而 `struct seq_file` 的 `private` 欄位在 [`svc_pool_stats_open()`](http://lxr.linux.no/#linux+v2.6.39/net/sunrpc/svc_xprt.c#L1239) 中以一種微妙但重要的不同方式使用。在這種情況下，所需的額外資料只是一個指標。所以 `svc_pool_stats_open` 並沒有分配一個本地資料結構來參考該 `private` 欄位，而是直接將該指標存儲在 `private` 欄位中。這似乎是一個合理的最佳化 —— 為了儲存一個指標而進行分配是浪費資源 —— 但這正好突顯了先前提到的混淆原因: 當你看到一個 `void` 指標時，你並不真正知道它指向什麼或為什麼會這樣使用。

為了更清楚地說明這裡發生的事情，可以將 `void *private` 想像成每一種不同指標型別的一 `union`。如果需要儲存的值是一個指標，它可以按照 「`union` 用於資料繼承」的模式儲存在這個 `union` 中。如果該值不僅僅是一個指標，那麼它會按照「`void` 指標用於資料繼承」的模式儲存在分配的空間中。因此，當我們看到一個 `void` 指標正在使用時，可能並不容易判斷它是用來指向一個資料繼承的擴充結構，還是本身被用作資料繼承的擴充 (或是用於其他用途)。

從另一個角度來強調這個問題，研究 [`struct v4l2_subdev`](http://lxr.linux.no/#linux+v2.6.39/include/media/v4l2-subdev.h#L490) 是有意義的，它代表 video4linux 裝置中的子裝置，例如網路攝影機中的感測器或相機控制器。根據 (相當有幫助的) 文件，預期該結構體通常會嵌入在包含額外狀態的更大的結構中。然而該結構體仍然有兩個 `void` 指標，且名稱都表示它們是子型別私有的:

```c
/* pointer to private data */
void *dev_priv;
void *host_priv;
```

v4l 子裝置 (通常是感測器) 通常會由像是 I2C 裝置實作 (就像儲存檔案系統的區塊裝置可能由 ATA 或 SCSI 裝置實作一樣)。為了應對這種常見情況，`struct v4l2_subdev` 提供了一個 `void` 指標 `dev_priv`，這樣驅動程式本身就不需要在包含 `struct v4l2_subdev` 的更大結構體中定義一個更具體的指標。`host_priv` 的目的是指向一個**親代裝置**，例如從感測器獲取影片資料的控制器。在使用此欄位的三個驅動程式中，有[一個](http://lxr.linux.no/#linux+v2.6.39/drivers/media/video/omap3isp/isp.c#L1751)似乎遵循了這個意圖，而[另外](http://lxr.linux.no/#linux+v2.6.39/drivers/media/video/pxa_camera.c#L1276)[兩個](http://lxr.linux.no/#linux+v2.6.39/drivers/media/video/sh_mobile_ceu_camera.c#L904)則將其用來指向分配的擴充結構。因此，這兩個指標都預期是按照「`union` 用於資料繼承」的模式使用，其中 `void` 指標用作多種其他指標型別的 `union`，但它們並不總是按照這種方式使用。

目前尚不清楚: 定義此 `void` 指標以備不時之需是否真的是一個有價值的服務，因為裝置驅動程式完全可以在擴充結構中輕鬆定義自己具備型別安全的指標。顯然一個看似**私有**的 `void` 指標可能被用於各種性質上完全不同的用途，而且正如我們在兩個不同情境中所見，它們的使用方式可能與預期不完全相符。

簡而言之，辨別「透過 `void` 指標進行資料繼承」的模式並不容易。需要對程式碼進行相當深入的檢查，才能確定 `void` 指標的具體目的和使用方式。

## 轉移到 struct page (A diversion into struct page)

在結束對 `union` 和 `void` 指標的討論之前，看看 [`struct page`](http://lxr.linux.no/#linux+v2.6.39/include/linux/mm_types.h#L34) 可能會很有趣。這個結構使用了這兩種模式，儘管它們因歷史包袱而有些隱晦。這個例子特別具有啟發性，因為它是 `struct` 嵌入不可行的一個案例。

Linux 中的記憶體被分割為分頁 (page)，這些分頁被用於各種不同的用途。有些分頁屬於**分頁快取** (page cache)，用來儲存檔案的內容；有些是**匿名分頁** (anonymous pages)，保存應用程式使用的資料；有些被用作 `slabs`，並被分割成小部分並回應 `kmalloc()` 的請求；還有一些分頁只是多頁分配的一部分，或者在自由列表 (free list) 中等待使用。這些不同的使用場景都可以被視為 `page` 這個通用型別的子型別，大多數情況下需要在 `struct page` 中新增一些專用欄位，例如分頁快取中的 `struct address_space` 指標和索引，或者作為 `slab` 使用時的 `struct kmem_cache` 和 `freelist` 指標。

每種分頁都由相同的 `struct page` 去描述，因此如果分頁的實際型別發生變化 —— 依據記憶體不同用途的需求隨時間必然產生的變化 —— 那麼在該結構體的生命週期內，`struct page` 的型別也必須改變。儘管許多型別系統的設計假設物件的型別是不可變的，但我們發現核心中有一個非常現實的需求: 型別的可變性 (mutability)。`union` 和 `void` 指標都允許型別的變化，正如先前所提到的，`struct page` 則同時使用了這兩者。

在子型別的第一層只有少數幾個不同的如上所指的子型別；這些子型別都為核心內記憶體管理的程式碼所知，因此使用 `union` 是理想的選擇。不幸的是 `struct page` 具有三個 `union`，並且某些子型別的欄位分散在這三個 `union` 中，這稍微隱藏了其實際結構。

當分頁主要作為分頁快取被使用時，它所屬的特定 `address_space` 可能需要進一步擴充資料結構。為此，結構體中有一個 `private` 欄位可以使用。然而，這個欄位並不是 `void` 指標，而是 `unsigned long`。核心中的許多地方假設 `unsigned long` 和 `void *` 具有相同大小，這裡就是其中之一。大多數使用這個欄位的程式碼實際上在這裡儲存一個指標，並且需要來回轉換型別。buffer_head 函式庫提供了 [`attach_page_buffers`](http://lxr.linux.no/#linux+v2.6.39/include/linux/buffer_head.h#L239) 和 [`page_buffers`](http://lxr.linux.no/#linux+v2.6.39/include/linux/buffer_head.h#L132) 這些巨集來設定和取得這個欄位。

所以儘管 `struct page` 不是最優雅的例子，但它是一個具啟發性的例子，說明在某些情況 `union` 和 `void` 指標是實作資料繼承的唯一選擇。

## 結構體嵌入的細節 (The details of structure embedding)

當可以使用結構體嵌入且可能的子型別無法預先確定時，這似乎逐漸成為更好的選擇。為了要充分理解它，我們需要再次深入探索 `inode` 之外，並對比資料繼承與結構體嵌入的其他用途。

結構體嵌入有三個基本用途 —— 也是將一個結構體包含在另一個結構體中的三個原因。有時候，這並沒有特別的含義。資料項目被收集到結構體中，並在結構體內包含其他結構體，僅僅是為了強調這些不同項目之間的緊密關係。在這種情況下，很少會取得嵌入結構體的地址，也從不會使用 `container_of()` 將其映射回包含該結構體的外部結構體。

第二種用途就是我們已經討論過的資料繼承嵌入。第三種用途與其類似，但有重要的區別。第三種用途的典型例子是 `struct list_head` 以及其他在建立抽象資料型別時作為[嵌入式錨點](https://lwn.net/Articles/336255/) (embedded anchor) 使用的結構體。

像 `struct list_head` 這樣作為嵌入式錨點的使用可以被視為一種繼承風格，因為包含它的結構體通過繼承自 `struct list_head` 成為串列的一員。然而這並不是嚴格的子型別，因為單個物件可以嵌入多個 `struct list_head` —— 例如，`struct inode` 就有六個 (如果算上類似的 `hlist_node`)。所以將這類嵌入視為 **mixin** 風格的繼承應該更為恰當。`struct list_head` 提供了一項服務 —— 成為串列的一部分，這可以多次混入其他物件中。

區分用於資料繼承的結構嵌入與其他兩種用途的一個關鍵點是: 在最內層結構體中存在參考計數 (reference counter)。這一現象與 Linux 核心依賴參考計數來管理物件的生命週期有直接關聯。所以這種特徵在使用垃圾回收 (garbage collection) 等方式來管理生命週期的系統中可能不會出現。

在 Linux 中，每個獨立存在的物件都會有一個參考計數，有時是簡單的 `atomic_t` 或甚至是 `int`，但通常是更明確的 [`struct kref`](https://lwn.net/Articles/336224/)。當一個物件通過多層繼承建立時，參考計數可能會被埋的很深。例如，[`struct usb_device`](http://lxr.linux.no/#linux+v2.6.39/include/linux/usb.h#L426) 嵌入了 [`struct device`](http://lxr.linux.no/#linux+v2.6.39/include/linux/device.h#L404)，而 `struct device` 又嵌入了 [`struct kobject`](http://lxr.linux.no/#linux+v2.6.39/include/linux/kobject.h#L60)，而 `struct kobject` 則包含了一個 [`struct kref`](http://lxr.linux.no/#linux+v2.6.39/include/linux/kref.h#L20)。所以`usb_device` (它也可能進一步嵌入到特定裝置的結構中) 確實有一個參考計數，但這個計數被嵌入在多層結構中。這與 `list_head` 和類似結構體形成了鮮明對比。這些結構體沒有參考計數，不獨立存在，只是為其他資料結構提供服務。

儘管這樣似乎顯而易見，但仍有必要記住: 一個物件不能有兩個參考計數 —— 至少不能有兩個用於管理其生命週期的參考計數 (像 `struct super_block` 中的 `s_active` 和 `s_count` 這樣用於計算不同事物的計數器是沒問題的)。這意味著**資料繼承**風格的多重繼承是不可行的。唯一可行的多重繼承形式是上面提到的 `list_head` 所使用的 **mixin** 風格。

這也意味著: 在設計資料結構時，必須考慮**生命週期的議題**，並且需要決定這個資料結構是否應該有自己的參考計數，還是應依賴其他物件來管理生命週期。也就是說: 這個結構體是獨立存在的物件，還是僅為其他物件提供服務。這些問題並不新鮮，同樣適用於 `void` 指標繼承。然而 `void` 指標的不同之處在於: 之後如果想改變主意，將擴充結構體變成一個完全獨立的物件相對容易。結構體嵌入需要在一開始就要清楚地思考問題並做出正確的決定 —— 而這種自律是值得鼓勵的。

資料繼承的結構嵌入另一個關鍵特徵是分配和初始化結構體實例的規則，這一點已經有所提及。當使用 `union` 或 `void` 指標繼承時，主結構通常由通用程式碼 (中間層) 分配和初始化，然後呼叫特定的 `open()` 或 `create()` 函式來選擇性地分配和初始化任何擴充物件。相比之下，當使用結構體嵌入時，結構體需要由最底層的裝置驅動程式分配，然後初始化自己的欄位，並呼叫通用程式碼來初始化通用的欄位。

延續上面提到的 `struct inode` 範例，它在 `super_block` 中有一個 `alloc_inode()` 方法來請求分配資源，我們發現初始化是通過 [`inode_init_once()`](http://lxr.linux.no/#linux+v2.6.39/fs/inode.c#L342) 和 [`inode_init_always()`](http://lxr.linux.no/#linux+v2.6.39/fs/inode.c#L185) 完成的。第一個函式用於當我們不確定這段記憶體之前的用途時，而第二個函式則是在我們知道這段記憶體之前是用來存放其他 `inode` 時。我們在 [`kobject_init()`](http://lxr.linux.no/#linux+v2.6.39/lib/kobject.c#L270)、[`kref_init()`](http://lxr.linux.no/#linux+v2.6.39/lib/kref.c#L22) 和 [`device_initialize()`](http://lxr.linux.no/#linux+v2.6.39/drivers/base/core.c#L587) 中也看到了這種分配與初始化分離的模式。

所以除了明顯的結構體嵌入之外，**透過結構體嵌入進行資料繼承**的模式還可以透過以下特徵來辨認: 在最內層結構體中存在參考計數，結構體分配的責任交由該結構體的最終使用者，並且提供用於初始化已分配記憶體的結構體的初始化函式。

# 結論 (Conclusion)

在探索 Linux 核心中的方法分派 (上週) 和資料繼承 (本週) 時我們發現: 儘管某些模式看起來佔主導地位，但它們並非是通用的。幾乎所有的資料繼承都可以通過結構體嵌入來實作，但在一些特定情況下，`union` 也能提供實際價值。同樣，雖然單純的 vtable 很常見，但 mixin vtable 也非常重要，將方法委派給相關物件的能力也具有很大的價值。

我們還發現了一些有在使用但不太推薦的模式。使用 `void` 指標進行繼承雖然在一開始看起來簡單，但長期來看會導致浪費，容易導致混淆，並且幾乎總能被嵌入式繼承所取代。同樣，使用 `NULL` 指標來表示預設行為也是一個不太好的選擇 —— 當預設行為重要時，有更好的方式來實作它。

或許最有價值的一課是: Linux 核心不僅是一個實用的程式，還是一份值得研究的文件。這樣的研究可以找到針對實際問題的優雅且實用的解決方案，也能發現一些不那麼優雅的解決方案。有志的學生可以透過研究前者來提升自己的思維能力，並透過研究後者來改進核心本身。考慮到這一點，以下練習可能對某些人感興趣。

> - 由於 `inode` 現在使用結構體嵌入來實作繼承，因此 `void` 指標應該不是必需的。研究從 `struct inode` 中移除 `i_private` 的後果及其可行性。
> - 重新整理 `struct page` 中的三個 `union`，將其合併為一個 `union`，使不同子型別的列舉 (enumeration) 更加明確。
> - 如文中所述，`struct seq_file` 可以通過 `void` 指標以及有限形式的 `union` 資料繼承進行擴充。解釋 `seq_open_private()` 如何讓該結構體也通過嵌入式結構體的資料繼承進行擴充，並舉一個核心中從 `void` 指標轉換為嵌入結構的使用案例。如果這樣的改進有意義，考慮提交補丁 (patch)。比較這種嵌入式結構體繼承的實作方式與 `inode` 所使用的機制的異同。
> - 儘管子型別在核心中被廣泛使用，但一個物件包含一些不是所有使用者都感興趣的欄位的情況並不罕見。這可能表明可以進行更細粒度的子型別歸納。檔案描述子 (file descriptor) 可以代表非常多完全不同的東西，因此 `struct file` 可能是進一步進行子型別歸納的候選。
>   找出可以作為通用 `struct file` 的最小欄位集合，並探討將其嵌入到不同結構體中以實現一般檔案、插座 (socket) 檔案、事件檔案和其他檔案類型的影響。研究提議中的 `inode` 的 [`open()`](https://lkml.org/lkml/2011/5/17/182) 方法更廣泛地使用方式，可能會有所幫助。
> - 找出一種物件導向語言，其物件模型能夠滿足這兩篇文章中所指出的 Linux 核心的所有需求。
