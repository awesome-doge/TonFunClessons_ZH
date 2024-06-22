# 第九課：Jettons - TON 可替代代幣

## 前言 - 為什麼需要代幣和標準

#### 什麼是代幣

代幣是一個網絡中某些數位資產的計算單位。重要的是，代幣通常並不意味著一種加密貨幣，而是一個分布在區塊鏈上的記錄，通過智能合約進行管理。智能合約包含代幣持有者賬戶上的餘額值，並提供將代幣從一個賬戶轉移到另一個賬戶的能力。

#### 什麼是可替代和不可替代代幣？

代幣的一種可能分類是基於其可替代性。

**可替代代幣（Fungible Tokens）** 是不獨特的資產，可以輕易地與另一種相同類型的資產交換。這種代幣的設計使每個代幣都與其他代幣等價。

**不可替代代幣（Non-fungible Tokens）** 是每個實例都獨特（特定）的資產，不能被另一個類似資產替代。不可替代代幣是一種數位物品證書，並可以通過某些機制轉讓該證書。

#### 為什麼需要代幣標準及其意義

為了使代幣能夠在其他應用程式中使用（從錢包到去中心化交易所），需要引入代幣的智能合約介面標準。

> 在這種情況下，介面是指不包含函數實現的函數宣告語法結構。

#### 標準的「批准」在哪裡進行

區塊鏈通常在 GitHub 或具有必要機制的平台上設有專門的頁面，您可以在這些頁面上提出有關標準的建議。

在 TON 中，這是 [GitHub repository](https://github.com/ton-blockchain/TIPs)。

> 重要提示：這些頁面不是論壇或自由討論區塊鏈的地方，因此如果您想在此存放庫中提出建議，請對您的帖子負責。

#### 代幣的風險

由於代幣實際上是智能合約，儘管它們很有效，但仍存在某些風險。例如，智能合約代碼中可能存在漏洞，或者智能合約本身可能以某種方式編寫，使得欺詐者能夠竊取代幣持有者的資金。因此，建議您仔細研究代幣的智能合約。

## Jetton 標準在 TON 中的意義

TON 中的可替代代幣標準是 Jetton，標準描述在 [這裡](https://github.com/ton-blockchain/TIPs/issues/74)。Jetton 標準的代幣應包括兩種類型的智能合約：
- 主合約
- 錢包合約

每個 Jetton 都有一個主要智能合約（我們稱之為主合約），用於鑄造新的 Jetton、統計總供應量並提供代幣的一般信息。

有關每個用戶擁有的代幣數量的信息存儲在稱為 Jetton 錢包的智能合約中。

標準文件中有一個很好的例子：

如果您發行一個供應量為 200 Jetton 且由 3 個人持有的 Jetton，那麼您需要部署 4 個合約：1 個 Jetton 主合約和 3 個 Jetton 錢包合約。

#### Jetton 合約的功能

根據標準，主合約需要實現兩個 Get 方法：

- `get_jetton_data()` - 返回以下數據：
  - `total_supply` -（整數）- 總發行 Jetton 代幣數量
  - `mintable` -（-1/0）- 標誌是否可以增加代幣數量
  - `admin_address` -（MsgAddressInt）- 管理 Jetton 的智能合約地址
  - `jetton_content` - cell - 根據 [代幣標準](https://github.com/ton-blockchain/TIPs/issues/64) 的數據
  - `jetton_wallet_code` - cell - 該代幣的錢包代碼

- `get_wallet_address(slice owner_address)` - 返回該擁有者地址的 Jetton 錢包地址。

根據標準，錢包合約必須實現：
- 內部消息處理器：
  - 代幣轉移
  - 燒毀代幣
- Get 方法 `get_wallet_data()` 返回：
  - `balance` -（uint256）錢包中的 Jetton 代幣數量。
  - `owner` -（MsgAddress）錢包擁有者地址；
  - `jetton` -（MsgAddress）Jetton 主合約地址；
  - `jetton_wallet_code` -（cell）此錢包的代碼；

> 可能會出現的問題是為什麼我們需要錢包代碼，錢包代碼允許您重現錢包智能合約的地址，其運作方式將在下文討論。

> 重要提示：標準還描述了許多有關手續費、限制等方面的細節，但為了避免課程變成一本書，我們不會過多詳述。

#### 工作流程

接下來，我們將討論 [Jetton 標準示例](https://github.com/ton-blockchain/token-contract) 的實現。當然，這並不是唯一可能的 Jetton 實現，但它將允許您在不進行抽象層級分析的情況下解析所有內容。

為了方便討論代碼，我們來談談 Jetton 的工作原理，即代幣的轉移、鑄幣等功能。

[示例](https://github.com/ton-blockchain/token-contract/tree/main/ft) 包含以下文件：
- 兩個主合約示例：`jetton-minter.fc`、`jetton-minter-ICO.fc`
- 錢包合約 `jetton-wallet.fc`
- 其他輔助文件。

接下來，我們將以 `jetton-minter.fc` 主合約和 `jetton-wallet.fc` 錢包合約為例，分析其功能。

##### 鑄幣

在 `jetton-minter.fc` 中的鑄幣過程如下，主合約的擁有者發送一個包含 `op::mint()` 的消息，其中消息正文包含應將 Jetton 標準代幣發送到的錢包信息。接下來，消息將信息發送到錢包智能合約。

##### 燒毀代幣

錢包擁有者發送一個包含 `op::burn()` 的消息，錢包智能合約根據消息減少代幣數量，並向主合約發送一個通知（`op::burn_notification()`），告知代幣供應量已減少。

##### 代幣轉移（Transfer - 發送/接收）

代幣轉移分為兩部分：
- `op::transfer()` 外部轉移
- `op::internal_transfer()` 內部轉移

外部轉移從錢包智能合約的擁有者發送 `op::transfer()` 消息開始，並將代幣發送到另一個錢包智能合約（當然，這也會減少您代幣的餘額）。

內部轉移在接收到 `op::internal_transfer()` 消息後，更改餘額並發送 `op::transfer_notification()` 消息 - 一條關於轉移的通知消息。

是的，當您將 Jetton 代幣發送到一個地址時，您可以請求與該地址相關聯的錢包在代幣到達時通知該地址。

## 分析代碼

在分析代碼之前，我要指出，整體的「機制」是重複的，因此越深入分析，分析的層次就越高。

我們將按以下順序解析 [repository](https://github.com/ton-blockchain/token-contract/tree/main/ft) 中的文件：

- jetton-minter.fc
- jetton-wallet.fc
- jetton-minter-ICO.fc

其餘文件（jetton-utils.fc、op-codes.fc、params.fc）將與前三個文件並行分析，因為它們是「服務性」的。

## jetton-minter.fc

主合約從兩個輔助函數開始，用於加載和卸載數據。

##### 從 c4 加載和卸載數據

現在讓我們來看兩個輔助函數，這些函數將從寄存器 c4 加載和卸載數據。「主合約儲存庫」將存儲：

- total_supply - 代幣的總供應量
- admin_address - 管理 Jetton 的智能合約地址
- content - 根據 [標準](https://github.com/ton-blockchain/TIPs/issues/64) 的 cell
- jetton_wallet_code - 此代幣的錢包代碼

為了從 c4 中「獲取」數據，我們需要兩個來自 FunC 標準庫的函數。即：`get_data` - 從 c4 寄存器中取出一個 cell。`begin_parse` - 將 cell 轉換為 slice。

```func
(int, slice, cell, cell) load_data() inline {
  slice ds = get_data().begin_parse();
  return (
	  ds~load_coins(), ;; total_supply
	  ds~load_msg_addr(), ;; admin_address
	  ds~load_ref(), ;; content
	  ds~load_ref()  ;; jetton_wallet_code
  );
}
```

使用我們已經熟悉的 `load_` 函數，我們從 slice 中卸載數據並返回。

為了保存數據，我們需要做三件事：
- 為未來的 cell 創建一個 Builder
- 將值寫入 Builder
- 從 Builder 創建 Cell（cell）
- 將生成的 cell 寫入寄存器

我們將再次使用 [FunC 標準庫函數](https://ton-blockchain.github.io/docs/#/func/stdlib) 來完成這些操作。

`begin_cell()` - 創建一個 Builder，為未來的 cell
`end_cell()` - 創建一個 Cell（cell）
`set_data()` - 將 cell 寫入寄存器 c4

```func
() save_data(int total_supply, slice admin_address, cell content, cell jetton_wallet_code) impure inline {
  set_data(begin_cell()
			.store_coins(total_supply)
			.store_slice(admin_address)
			.store_ref(content)
			.store_ref(jetton_wallet_code)
		   .end_cell()
		  );
}
```

使用我們已經熟悉的 `store_` 函數，我們將存儲數據。

##### 輔助鑄幣函數

在 [代碼](https://github.com/ton-blockchain/token-contract/blob/main/ft/jetton-minter.fc) 中接下來是一個鑄幣函數，用於鑄造代幣。

```func
() mint_tokens(slice to_address, cell jetton_wallet_code, int amount, cell master_msg) impure {

}
```

它接收參數：
- `slice to_address` - 代幣應發送到的地址
- `cell jetton_wallet_code` - 此代幣的錢包代碼
- `int amount` - 代幣數量
- `cell master_msg` - 來自主合約的消息

這裡出現了一個問題，我們有一個地址，但這不是代幣錢包的地址，那麼如何獲取代幣錢包智能合約的地址呢？

這裡有一個小技巧。如果我們研究 [如何編譯智能合約的文件](https://ton-blockchain.github.io/docs/#/howto/step-by-step?id=_3-compiling-a-new-smart-contract)，我們會看到以下內容：

代碼和數據將為新的智能合約組合成一個 StateInit 結構（在下一行），計算並輸出新智能合約的地址（等於該 StateInit 結構的哈希值），然後將其外部通訊給新的智能合約的適當目標地址。外部消息包含新智能合約的正確 StateInit，並帶有一個非平凡的有效負載（私下用私鑰簽名）。

對我們來說，這意味著我們可以從需要發送代幣的地址中獲取代幣智能合約的地址。如果我們談到代幣合約，我們可以使用地址和錢包代碼來組合錢包的 StateInit。

這是可能的，因為 [哈希函數](https://en.wikipedia.org/wiki/Hash_function) 是確定性的，這意味著對於不同的輸入會有不同的哈希值，同時對於相同的輸入數據，哈希函數將始終返回相同的哈希值。

為此，jetton-utils.fc 文件中有兩個函數 `calculate_jetton_wallet_state_init` 和 `calculate_jetton_wallet_address`。

```func
cell calculate_jetton_wallet_state_init(slice owner_address, slice jetton_master_address, cell jetton_wallet_code) inline {
  return begin_cell()
		  .store_uint(0, 2)
		  .store_dict(jetton_wallet_code)
		  .store_dict(pack_jetton_wallet_data(0, owner_address, jetton_master_address, jetton_wallet_code))
		  .store_uint(0, 1)
		 .end_cell();
}

slice calculate_jetton_wallet_address(cell state_init) inline {
  return begin_cell().store_uint(4, 3)
					 .store_int(workchain(), 8)
					 .store_uint(cell_hash(state_init), 256)
					 .end_cell()
					 .begin_parse();
}
```

`calculate_jetton_wallet_state_init` 函數根據 [代幣標準](https://github.com/ton-blockchain/TIPs/issues/64) 組合 StateInit，並設置為零餘額。

`calculate_jetton_wallet_address` 函數根據 [TL-B schema](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L99) 組合地址。

> `cell_hash()` 函數用於計算 cell 表示的哈希值。

因此，鑄幣函數現在看起來如下：

```func
() mint_tokens(slice to_address, cell jetton_wallet_code, int amount, cell master_msg) impure {
  cell state_init = calculate_jetton_wallet_state_init(to_address, my_address(), jetton_wallet_code);
  slice to_wallet_address = calculate_jetton_wallet_address(state_init);

}
```

接下來，您需要將消息發送到智能合約：

```func
var msg = begin_cell()
  .store_uint(0x18, 6)
  .store_slice(to_wallet_address)
  .store_coins(amount)
  .store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)
  .store_ref(state_init)
  .store_ref(master_msg);
```

關於發送消息的詳細說明，請參見 [這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/messages)，以及第三課。我們使用 `store_ref` 發送帶有錢包合約信息的消息。

只需發送消息即可，為此我們使用來自 [標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message) 的 `send_raw_message`。

我們已經組裝了 `msg` 變量，只需確定 `mode` 即可。每個模式在 [文件](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message) 中都有描述。讓我們看一個例子，以便更清楚。

假設智能合約的餘額為 100 coins，我們收到一個包含 60 coins 的內部消息並發送一個 10 coins 的消息，總費用為 3。

`mode = 0` - 餘額（100+60-10 = 150 coins），發送（10-3 = 7 coins）
`mode = 1` - 餘額（100+60-10-3 = 147 coins），發送（10 coins）
`mode = 64` - 餘額（100-10 = 90 coins），發送（60+10-3 = 67 coins）
`mode = 65` - 餘額（100-10-3=87 coins），發送（60+10 = 70 coins）
`mode = 128` - 餘額（0 coins），發送（100+60-3 = 157 coins）

在合約代碼中，我們有 `mode 1`，即 `mode' = mode + 1`，這意味著發送者希望單獨支付轉賬費用。

```func
send_raw_message(msg.end_cell(), 1); ;; pay transfer fees separately, revert on errors
```

`mint_tokens()` 函數的最終代碼：

```func
() mint_tokens(slice to_address, cell jetton_wallet_code, int amount, cell master_msg) impure {
  cell state_init = calculate_jetton_wallet_state_init(to_address, my_address(), jetton_wallet_code);
  slice to_wallet_address = calculate_jetton_wallet_address(state_init);
  var msg = begin_cell()
	.store_uint(0x18, 6)
	.store_slice(to_wallet_address)
	.store_coins(amount)
	.store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)
	.store_ref(state_init)
	.store_ref(master_msg);
  send_raw_message(msg.end_cell(), 1); ;; pay transfer fees separately, revert on errors
}
```

#### 分析 recv_internal()

讓我提醒您，TON 網絡中的智能合約有兩個保留方法可以訪問。

首先，`recv_external()` 這個函數在來自外部世界的請求到達合約時執行，即不是來自 TON，例如，當我們自己組成一個消息並通過 lite-client 發送時（關於安裝 [lite-client](https://ton.org/docs/#/compile?id=lite-client))）。

第二，`recv_internal()` 這個函數在 TON 內部執行，例如，當任何合約引用我們的合約時。

在我們的情況下，`recv_internal()` 將接收以下參數：

```func
() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {

}
```

>`impure` 是一個關鍵字，表明函數更改了智能合約數據。例如，我們必須指定 `impure` 限定符，如果該函數可以修改合約存儲、發送消息或在某些數據無效時引發異常且該函數旨在驗證這些數據。重要提示：如果未指定 impure，且函數調用的結果未使用，則 FunC 編譯器可能會刪除該函數調用。

現在讓我們來看看這個函數的代碼。開始檢查消息是否為空。

```func
if (in_msg_body.slice_empty?()) { ;; ignore empty messages
    return ();
}
```

接下來，我們開始解析（讀取）消息：

```func
slice cs = in_msg_full.begin_parse();
```

我們取出標誌並檢查消息是否被退回（這裡是指 bounced）。

```func
if (flags & 1) { ;; ignore all bounced messages
    return ();
}
```

我們在 `recv_internal()` 上獲取消息發送者的地址：

```func
slice sender_address = cs~load_msg_addr();
```

接下來是 `op` 和 `query_id`，您可以在合約指南或第五課中閱讀它們。簡而言之，`op` 是操作識別碼。

```func
int op = in_msg_body~load_uint(32);
int query_id = in_msg_body~load_uint(64);
```

接下來，我們將使用先前編寫的輔助函數 - `load_data()`。

```func
(int total_supply, slice admin_address, cell content, cell jetton_wallet_code) = load_data();
```

現在，使用條件運算符，我們圍繞 `op` 建立邏輯。為了方便起見，這些代碼存儲在一個單獨的文件 `op-codes.fc` 中。最後還有一個異常，即如果合約不執行與 `op` 相關的某些操作，將會引發異常。

> 重要提示：由於代幣必須符合標準，因此對於標準中描述的操作，您需要使用相應的代碼，例如對於 `burn_notification()`，這是 0x7bdd97de。

我們得到：

```func
() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) { ;; ignore empty messages
        return ();
    }
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) { ;; ignore all bounced messages
        return ();
    }
    slice sender_address = cs~load_msg_addr();

    int op = in_msg_body~load_uint(32);
    int query_id = in_msg_body~load_uint(64);

    (int total_supply, slice admin_address, cell content, cell jetton_wallet_code) = load_data();

    if (op == op::mint()) {
        ;; token minting
    }

    if (op == op::burn_notification()) {
        ;; processing a message from the wallet that the tokens have burned down
    }

    if (op == 3) { ;; change admin
        ;; change of "admin" or how else can you call the owner of the contract
    }

    if (op == 4) { ;; change content, delete this for immutable tokens
        ;; change of data in c4 register
    }

    throw(0xffff);
}
```

##### op::mint()

首先處理 `op::mint()`，如果合約管理員（擁有者）地址等於發送者地址，則引發異常：

```func
if (op == op::mint()) {
    throw_unless(73, equal_slices(sender_address, admin_address));
}
```

接下來，從消息正文中獲取應發送代幣的地址、用於「運輸」Jetton 標準代幣的 TON 數量以及主合約消息。

```func
if (op == op::mint()) {
    throw_unless(73, equal_slices(sender_address, admin_address));
    slice to_address = in_msg_body~load_msg_addr();
    int amount = in_msg_body~load_coins();
    cell master_msg = in_msg_body~load_ref();
}
```

我們從主合約消息中獲取代幣數量，略過 `op` 和 `query_id`。

```func
if (op == op::mint()) {
    throw_unless(73, equal_slices(sender_address, admin_address));
    slice to_address = in_msg_body~load_msg_addr();
    int amount = in_msg_body~load_coins();
    cell master_msg = in_msg_body~load_ref();
    slice master_msg_cs = master_msg.begin_parse();
    master_msg_cs~skip_bits(32 + 64); ;; op + query_id
    int jetton_amount = master_msg_cs~load_coins();
}
```

我們調用先前編寫的 `mint_tokens()` 函數，並使用 `save_data()` 輔助函數將數據保存到 `c4`。最後，我們將完成函數，`return` 運算符將幫助我們。

```func
if (op == op::mint()) {
    throw_unless(73, equal_slices(sender_address, admin_address));
    slice to_address = in_msg_body~load_msg_addr();
    int amount = in_msg_body~load_coins();
    cell master_msg = in_msg_body~load_ref();
    slice master_msg_cs = master_msg.begin_parse();
    master_msg_cs~skip_bits(32 + 64); ;; op + query_id
    int jetton_amount = master_msg_cs~load_coins();
    mint_tokens(to_address, jetton_wallet_code, amount, master_msg);
    save_data(total_supply + jetton_amount, admin_address, content, jetton_wallet_code);
    return ();
}
```

##### op::burn_notification()

首先處理 `op::burn_notification()`，從消息正文中獲取代幣數量和通知來自的地址。

```func
if (op == op::burn_notification()) {
    int jetton_amount = in_msg_body~load_coins();
    slice from_address = in_msg_body~load_msg_addr();
}
```

接下來，使用已知的技巧「重現」錢包地址（`calculate_user_jetton_wallet_address()` 函數），如果發送者地址（`sender_address`）不等於錢包地址，則引發異常。

```func
if (op == op::burn_notification()) {
    int jetton_amount = in_msg_body~load_coins();
    slice from_address = in_msg_body~load_msg_addr();
    throw_unless(74,
        equal_slices(calculate_user_jetton_wallet_address(from_address, my_address(), jetton_wallet_code), sender_address)
    );
}
```

現在將數據存儲在 `c4` 中，同時減少代幣總供應量，減少數量為燒毀的代幣數量。

```func
if (op == op::burn_notification()) {
    int jetton_amount = in_msg_body~load_coins();
    slice from_address = in_msg_body~load_msg_addr();
    throw_unless(74,
        equal_slices(calculate_user_jetton_wallet_address(from_address, my_address(), jetton_wallet_code), sender_address)
    );
    save_data(total_supply - jetton_amount, admin_address, content, jetton_wallet_code);
}
```

然後，我們獲取需要返回答覆的地址。

```func
if (op == op::burn_notification()) {
    int jetton_amount = in_msg_body~load_coins();
    slice from_address = in_msg_body~load_msg_addr();
    throw_unless(74,
        equal_slices(calculate_user_jetton_wallet_address(from_address, my_address(), jetton_wallet_code), sender_address)
    );
    save_data(total_supply - jetton_amount, admin_address, content, jetton_wallet_code);
    slice response_address = in_msg_body~load_msg_addr();
}
```

如果不是「null」（不是 `addr_none`），我們將發送關於多餘的消息（`op::excesses()`），當然，使用 `return` 運算符完成函數。

```func
if (op == op::burn_notification()) {
    int jetton_amount = in_msg_body~load_coins();
    slice from_address = in_msg_body~load_msg_addr();
    throw_unless(74,
        equal_slices(calculate_user_jetton_wallet_address(from_address, my_address(), jetton_wallet_code), sender_address)
    );
    save_data(total_supply - jetton_amount, admin_address, content, jetton_wallet_code);
    slice response_address = in_msg_body~load_msg_addr();
    if (response_address.preload_uint(2) != 0) {
      var msg = begin_cell()
        .store_uint(0x10, 6) ;; nobounce - int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool src:MsgAddress -> 011000
        .store_slice(response_address)
        .store_coins(0)
        .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
        .store_uint(op::excesses(), 32)
        .store_uint(query_id, 64);
      send_raw_message(msg.end_cell(), 2 + 64);
    }
    return ();
}
```

##### op 3 和 op 4

在主合約示例中，還顯示了兩個可選功能，即更改主合約管理員（admin）的所有權（`op == 3`）以及替換 `c4` 寄存器中的所有內容（`op == 4`）。

> 重要提示：在與任何合約互動之前，請務必研究其代碼，因為開發人員經常會留下漏洞，這些漏洞可以完全改變合約的邏輯。

在每個此類「控制數據合約」的 `op` 中，必須檢查發送者地址是否為合約管理員地址。然後只需保存數據。

```func
if (op == 3) { ;; change admin
    throw_unless(73, equal_slices(sender_address, admin_address));
    slice new_admin_address = in_msg_body~load_msg_addr();
    save_data(total_supply, new_admin_address, content, jetton_wallet_code);
    return ();
}

if (op == 4) { ;; change content, delete this for immutable tokens
    throw_unless(73, equal_slices(sender_address, admin_address));
    save_data(total_supply, admin_address, in_msg_body~load_ref(), jetton_wallet_code);
    return ();
}
```

#### Get 方法

因此，根據 Jetton 標準，主合約必須有兩個 Get 方法：
- `get_jetton_data()` - 返回有關 Jetton 標準代幣的數據
- `get_wallet_address()` - 根據地址返回錢包智能合約地址

##### get_jetton_data()

該函數從 `c4` 中獲取數據並返回：

- total_supply -（整數）- 總發行代幣數量
- mintable -（-1/0）- 標誌是否可以增加代幣數量
- admin_address -（MsgAddressInt）- 管理 Jetton 的智能合約地址
- jetton_content - cell - 根據 [代幣數據標準](https://github.com/ton-blockchain/TIPs/issues/64) 的數據
- jetton_wallet_code - cell - 此代幣的錢包代碼

代碼：

```func
(int, int, slice, cell, cell) get_jetton_data() method_id {
    (int total_supply, slice admin_address, cell content, cell jetton_wallet_code) = load_data();
    return (total_supply, -1, admin_address, content, jetton_wallet_code);
}
```

##### get_wallet_address()

使用輔助函數「重現」錢包智能合約地址。

代碼：

```func
slice get_wallet_address(slice owner_address) method_id {
    (int total_supply, slice admin_address, cell content, cell jetton_wallet_code) = load_data();
    return calculate_user_jetton_wallet_address(owner_address, my_address(), jetton_wallet_code);
}
```

## jetton-wallet.fc

此文件從兩個函數開始，我們將其定義為低層級 TVM 原語，使用 `asm` 關鍵字。

```func
int min_tons_for_storage() asm "10000000 PUSHINT"; ;; 0.01 TON
int gas_consumption() asm "10000000 PUSHINT"; ;; 0.01 TON
```

我們將需要這兩個函數來檢查 gas 限額和最小的 TON 數。

> PUSHINT 將一個整數推入堆疊

##### 從 c4 加載和卸載數據

現在讓我們來看兩個輔助函數，這些函數將從寄存器 c4 加載和卸載數據。在我們的「儲存」中，我們將存儲：

- int balance - 代幣餘額
- slice owner_address - 代幣擁有者地址
- slice jetton_master_address - 該代幣的主合約地址
- cell jetton_wallet_code - 此代幣的錢包代碼

為了從 c4 中「獲取」數據，我們需要兩個來自 FunC 標準庫的函數。即：`get_data` - 從 c4 寄存器中取出一個 cell。`begin_parse` - 將 cell 轉換為 slice。

```func
(int, slice, slice, cell) load_data() inline {
  slice ds = get_data().begin_parse();
  return (ds~load_coins(), ds~load_msg_addr(), ds~load_msg_addr(), ds~load_ref());
}
```

使用我們已經熟悉的 `load_` 函數，我們從 slice 中卸載數據並返回。

為了保存數據，使用 `set_data()` 將 cell 寫入寄存器 c4。

```func
() save_data (int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) impure inline {
  set_data(pack_jetton_wallet_data(balance, owner_address, jetton_master_address, jetton_wallet_code));
}
```

我們將使用 jetton-utils.fc 文件中的 `pack_jetton_wallet_data()` 輔助函數來組合數據 cell。

`pack_jetton_wallet_data()` 函數代碼：

```func
cell pack_jetton_wallet_data(int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) inline {
   return  begin_cell()
			.store_coins(balance)
			.store_slice(owner_address)
			.store_slice(jetton_master_address)
			.store_ref(jetton_wallet_code)
		   .end_cell();
}
```

`begin_cell()` - 創建一個 Builder 為未來的 cell
`store_` - 寫入值
`end_cell()` - 創建一個 Cell（cell）

##### 發送代幣的函數（外部轉移）

發送代幣的函數，根據 [標準](https://github.com/ton-blockchain/TIPs/issues/74) 檢查條件並發送相應的消息。

```func
() send_tokens (slice in_msg_body, slice sender_address, int msg_value, int fwd_fee) impure {
}
```

讓我們逐步解析函數代碼。函數代碼從讀取 `in_msg_body` 的數據開始：

```func
int query_id = in_msg_body~load_uint(64);
int jetton_amount = in_msg_body~load_coins();
slice to_owner_address = in_msg_body~load_msg_addr();
```

- query_id - 任意查詢編號
- jetton_amount - 代幣數量
- to_owner_address - 擁有者地址，需要用來重現智能合約地址

接下來調用來自 params.fc 文件的 `force_chain()` 函數。

```func
force_chain(to_owner_address);
```

`force_chain` 函數檢查地址是否在 workchain 編號 0（基礎工作鏈）中。關於地址和編號的更多信息，請參見 [這裡](https://github.com/ton-blockchain/ton/blob/master/doc/LiteClient-HOWTO) 的最開始。讓我們解析 params.fc 的代碼：

```func
int workchain() asm "0 PUSHINT";

() force_chain(slice addr) impure {
  (int wc, _) = parse_std_addr(addr);
  throw_unless(333, wc == workchain());
}
```

我們定義輔助函數 `workchain()` 為低層級 TVM 原語，使用 `asm` 關鍵字。整數 == 0，我們需要比較。

```func
int workchain() asm "0 PUSHINT";
```

為了從地址中提取我們需要的信息，使用 `parse_std_addr()`。`parse_std_addr()` 返回工作鏈和 256 位整數地址，來自 `MsgAddressInt`。

```func
() force_chain(slice addr) impure {
  (int wc, _) = parse_std_addr(addr);
  throw_unless(333, wc == workchain());
}
```

為了在工作鏈不相等時引發異常，我們將使用 `throw_unless()`。

回到我們的 `send_tokens()` 函數。

從 c4 中加載數據：

```func
(int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) = load_data();
```

然後立即從餘額中減去代幣數量（jetton_amount），並立即檢查（拋出異常），確保餘額未變為負數，並且地址匹配。

```func
balance -= jetton_amount;
throw_unless(705, equal_slices(owner_address, sender_address));
throw_unless(706, balance >= 0);
```

現在，使用我們已知的技巧，重現錢包地址：

```func
cell state_init = calculate_jetton_wallet_state_init(to_owner_address, jetton_master_address, jetton_wallet_code);
slice to_wallet_address = calculate_jetton_wallet_address(state_init);
```

接下來，從消息正文中獲取需要返回答覆的地址、自定義有效載荷（或許有人希望與代幣一起傳輸一些信息），以及將發送到目標地址的 nanoTons 數量。

```func
slice response_address = in_msg_body~load_msg_addr();
cell custom_payload = in_msg_body~load_dict();
int forward_ton_amount = in_msg_body~load_coins();
```

現在使用 `slice_bits` 函數，該函數返回 slice 中數據位的數量。讓我們檢查消息正文中是否只剩下另一個有效載荷，並同時將其卸載。

```func
throw_unless(708, slice_bits(in_msg_body) >= 1);
slice either_forward_payload = in_msg_body;
```

接下來，組合一個消息（提醒您，`to_wallet_address` 是錢包智能合約的地址）：

```func
var msg = begin_cell()
  .store_uint(0x18, 6)
  .store_slice(to_wallet_address)
  .store_coins(0)
  .store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)
  .store_ref(state_init);
```

消息正文單獨根據 [標準](https://github.com/ton-blockchain/TIPs/issues/74) 組合。即：

> 發送代幣與轉移有關，因此我們完全根據 transfer 組合消息正文

`op` - 取自 jetton-utils.fc 文件（根據標準，這是 `internal_transfer()`，即 0x178d4519）
`query_id` - 任意查詢編號。
`jetton amount` - 轉移的代幣數量。
`owner_address` - 新代幣擁有者地址。
`response_address` - 要返回答覆的地址，確認成功轉移並帶有剩餘的進入消息中的 Tone。
`forward_ton_amount` - 將發送到目標地址的 nanoTons 數量。
`forward_payload` - 要發送到目標地址的可選用戶數據。

消息正文代碼並將其添加到消息中：

```func
var msg_body = begin_cell()
  .store_uint(op::internal_transfer(), 32)
  .store_uint(query_id, 64)
  .store_coins(jetton_amount)
  .store_slice(owner_address)
  .store_slice(response_address)
  .store_coins(forward_ton_amount)
  .store_slice(either_forward_payload)
  .end_cell();
msg = msg.store_ref(msg_body);
```

> 細心的讀者可能會問 `custom_payload` 在哪裡，但這個字段是可選的。

看起來一切都已準備好發送，但標準中還剩下兩個重要條件，即：

- 處理操作、部署受益人錢包代幣和發送 `forward_ton_amount` 所需的 TON 不足。
- 處理請求後，接收方的錢包代幣必須至少發送 `in_msg_value - forward_amount - 2 * max_tx_gas_price` 到 `response_destination` 地址。

```func
int fwd_count = forward_ton_amount ? 2 : 1;
throw_unless(709, msg_value >
					 forward_ton_amount +
					 ;; 3 messages: wal1->wal2,  wal2->owner, wal2->response
					 ;; but last one is optional (it is ok if it fails)
					 fwd_count * fwd_fee +
					 (2 * gas_consumption() + min_tons_for_storage()));
					 ;; universal message send fee calculation may be activated here
					 ;; by using this instead of fwd_fee
					 ;; msg_fwd_fee(to_wallet, msg_body, state_init, 15)
```

> 這裡我不會詳細說明，因為評論和代幣標準中的描述給出了詳細的說明

只需發送消息並將數據保存到 `c4` 寄存器即可：

```func
send_raw_message(msg.end_cell(), 64); ;; revert on errors
save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
```

最終代碼：

```func
() send_tokens (slice in_msg_body, slice sender_address, int msg_value, int fwd_fee) impure {
  int query_id = in_msg_body~load_uint(64);
  int jetton_amount = in_msg_body~load_coins();
  slice to_owner_address = in_msg_body~load_msg_addr();
  force_chain(to_owner_address);
  (int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) = load_data();
  balance -= jetton_amount;

  throw_unless(705, equal_slices(owner_address, sender_address));
  throw_unless(706, balance >= 0);

  cell state_init = calculate_jetton_wallet_state_init(to_owner_address, jetton_master_address, jetton_wallet_code);
  slice to_wallet_address = calculate_jetton_wallet_address(state_init);
  slice response_address = in_msg_body~load_msg_addr();
  cell custom_payload = in_msg_body~load_dict();
  int forward_ton_amount = in_msg_body~load_coins();
  throw_unless(708, slice_bits(in_msg_body) >= 1);
  slice either_forward_payload = in_msg_body;
  var msg = begin_cell()
	.store_uint(0x18, 6)
	.store_slice(to_wallet_address)
	.store_coins(0)
	.store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)
	.store_ref(state_init);
  var msg_body = begin_cell()
	.store_uint(op::internal_transfer(), 32)
	.store_uint(query_id, 64)
	.store_coins(jetton_amount)
	.store_slice(owner_address)
	.store_slice(response_address)
	.store_coins(forward_ton_amount)
	.store_slice(either_forward_payload)
	.end_cell();

  msg = msg.store_ref(msg_body);
  int fwd_count = forward_ton_amount ? 2 : 1;
  throw_unless(709, msg_value >
					 forward_ton_amount +
					 ;; 3 messages: wal1->wal2,  wal2->owner, wal2->response
					 ;; but last one is optional (it is ok if it fails)
					 fwd_count * fwd_fee +
					 (2 * gas_consumption() + min_tons_for_storage()));
					 ;; universal message send fee calculation may be activated here
					 ;; by using this instead of fwd_fee
					 ;; msg_fwd_fee(to_wallet, msg_body, state_init, 15)

  send_raw_message(msg.end_cell(), 64); ;; revert on errors
  save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
}
```

##### 接收代幣的函數（內部轉移）

讓我們繼續接收代幣：

```func
() receive_tokens (slice in_msg_body, slice sender_address, int my_ton_balance, int fwd_fee, int msg_value) impure {

}
```

`receive_tokens()` 函數從卸載 c4 中的數據開始，然後我們從消息正文中獲取 `query_id` 和 `jetton_amount`：

```func
(int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) = load_data();
int query_id = in_msg_body~load_uint(64);
int jetton_amount = in_msg_body~load_coins();
```

由於錢包已經接收了代幣，我們需要將它們添加到餘額中：

```func
balance += jetton_amount;
```

我們繼續從 `in_msg_body` 中讀取數據，取兩個地址：收到代幣的地址和應返回答覆的地址。

```func
slice from_address = in_msg_body~load_msg_addr();
slice response_address = in_msg_body~load_msg_addr();
```

接下來，使用 [二元或運算符](https://en.wikipedia.org/wiki/Binary_operation) 我們同時檢查兩個地址的條件：

```func
throw_unless(707,
	equal_slices(jetton_master_address, sender_address)
	|
	equal_slices(calculate_user_jetton_wallet_address(from_address, jetton_master_address, jetton_wallet_code), sender_address)
);
```

我們也從消息正文中獲取 `forward_ton_amount` - 將發送到目標地址的 nanoTons 數量。

```func
int forward_ton_amount = in_msg_body~load_coins();
```

這時，我們定義的函數 `min_tons_for_storage()` 和 `gas_consumption()` 終於派上用場了，分別用於 gas 限額和最小的 TON 數量。

```func
int storage_fee = min_tons_for_storage() - min(ton_balance_before_msg, min_tons_for_storage());
msg_value -= (storage_fee + gas_consumption());
```

使用這些限額，我們獲取消息的值，這在稍後將用於確定是否需要發送多餘消息。

接下來，如果我們創建一個轉移通知消息：

```func
if(forward_ton_amount) {
  msg_value -= (forward_ton_amount + fwd_fee);
  slice either_forward_payload = in_msg_body;

  var msg_body = begin_cell()
	  .store_uint(op::transfer_notification(), 32)
	  .store_uint(query_id, 64)
	  .store_coins(jetton_amount)
	  .store_slice(from_address)
	  .store_slice(either_forward_payload)
	  .end_cell();

  var msg = begin_cell()
	.store_uint(0x10, 6) ;; we should not bounce here cause receiver can have uninitialized contract
	.store_slice(owner_address)
	.store_coins(forward_ton_amount)
	.store_uint(1, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	.store_ref(msg_body);

  send_raw_message(msg.end_cell(), 1);
}
```

> 重要的是，我們再次減少 `msg_value`，稍後我們將需要這一值來確定是否需要發送多餘消息。

現在是 `msg_value` 的時候了，從中我們持續減少各種費用（更多關於它們的信息，請參見 [這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/fees)）。

我們檢查地址不是 null 且 `msg_value` 還有剩餘，並發送多餘消息，帶有相應的多餘值。

```func
if ((response_address.preload_uint(2) != 0) & (msg_value > 0)) {
  var msg = begin_cell()
	.store_uint(0x10, 6) ;; nobounce - int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool src:MsgAddress -> 010000
	.store_slice(response_address)
	.store_coins(msg_value)
	.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	.store_uint(op::excesses(), 32)
	.store_uint(query_id, 64);
  send_raw_message(msg.end_cell(), 2);
}
```

當然，最後是數據保存。

```func
save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
```

最終代碼：

```func
() receive_tokens (slice in_msg_body, slice sender_address, int my_ton_balance, int fwd_fee, int msg_value) impure {
  ;; NOTE we can not allow fails in action phase since in that case there will be
  ;; no bounce. Thus check and throw in computation phase.
  (int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) = load_data();
  int query_id = in_msg_body~load_uint(64);
  int jetton_amount = in_msg_body~load_coins();
  balance += jetton_amount;
  slice from_address = in_msg_body~load_msg_addr();
  slice response_address = in_msg_body~load_msg_addr();
  throw_unless(707,
	  equal_slices(jetton_master_address, sender_address)
	  |
	  equal_slices(calculate_user_jetton_wallet_address(from_address, jetton_master_address, jetton_wallet_code), sender_address)
  );
  int forward_ton_amount = in_msg_body~load_coins();

  int ton_balance_before_msg = my_ton_balance - msg_value;
  int storage_fee = min_tons_for_storage() - min(ton_balance_before_msg, min_tons_for_storage());
  msg_value -= (storage_fee + gas_consumption());
  if(forward_ton_amount) {
	msg_value -= (forward_ton_amount + fwd_fee);
	slice either_forward_payload = in_msg_body;

	var msg_body = begin_cell()
		.store_uint(op::transfer_notification(), 32)
		.store_uint(query_id, 64)
		.store_coins(jetton_amount)
		.store_slice(from_address)
		.store_slice(either_forward_payload)
		.end_cell();

	var msg = begin_cell()
	  .store_uint(0x10, 6) ;; we should not bounce here cause receiver can have uninitialized contract
	  .store_slice(owner_address)
	  .store_coins(forward_ton_amount)
	  .store_uint(1, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	  .store_ref(msg_body);

	send_raw_message(msg.end_cell(), 1);
  }

  if ((response_address.preload_uint(2) != 0) & (msg_value > 0)) {
	var msg = begin_cell()
	  .store_uint(0x10, 6) ;; nobounce - int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool src:MsgAddress -> 010000
	  .store_slice(response_address)
	  .store_coins(msg_value)
	  .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	  .store_uint(op::excesses(), 32)
	  .store_uint(query_id, 64);
	send_raw_message(msg.end_cell(), 2);
  }

  save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
}
```

##### 燒毀代幣函數（內部轉移）

我們不會詳細分析燒毀函數，因為在閱讀了先前函數的分析之後，一切應該一目了然。

我只會指出工作邏輯 - 在選定的代幣數量被減少之後，一條消息將被發送到主合約，通知燒毀。

```func
() burn_tokens (slice in_msg_body, slice sender_address, int msg_value, int fwd_fee) impure {
  ;; NOTE we can not allow fails in action phase since in that case there will be
  ;; no bounce. Thus check and throw in computation phase.
  (int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) = load_data();
  int query_id = in_msg_body~load_uint(64);
  int jetton_amount = in_msg_body~load_coins();
  slice response_address = in_msg_body~load_msg_addr();
  ;; ignore custom payload
  ;; slice custom_payload = in_msg_body~load_dict();
  balance -= jetton_amount;
  throw_unless(705, equal_slices(owner_address, sender_address));
  throw_unless(706, balance >= 0);
  throw_unless(707, msg_value > fwd_fee + 2 * gas_consumption());

  var msg_body = begin_cell()
	  .store_uint(op::burn_notification(), 32)
	  .store_uint(query_id, 64)
	  .store_coins(jetton_amount)
	  .store_slice(owner_address)
	  .store_slice(response_address)
	  .end_cell();

  var msg = begin_cell()
	.store_uint(0x18, 6)
	.store_slice(jetton_master_address)
	.store_coins(0)
	.store_uint(1, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	.store_ref(msg_body);

  send_raw_message(msg.end_cell(), 64);

  save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
}
```

#### Bounce

在我們進入 `recv_internal()` 之前，還有一個函數需要編寫。在 `recv_internal()` 函數中，我們必須處理 bounced 消息。（有關 bounce 的更多信息，請參見 [這裡的第 78 頁](https://ton-blockchain.github.io/docs/tblkch.pdf)）。

在 bounce 時，我們必須執行以下操作：
- 將代幣返回餘額
- 如果 `op::internal_transfer()` 或 `op::burn_notification()` 引發異常

我們將消息正文傳遞給函數框架：

```func
() on_bounce (slice in_msg_body) impure {

}
```

從正文中取出 `op` 並引發異常，如果 `op::internal_transfer()` 或 `op::burn_notification()`

```func
() on_bounce (slice in_msg_body) impure {
  in_msg_body~skip_bits(32); ;; 0xFFFFFFFF
  (int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) = load_data();
  int op = in_msg_body~load_uint(32);
  throw_unless(709, (op == op::internal_transfer()) | (op == op::burn_notification()));
}
```

繼續從正文中讀取數據，我們將代幣返回餘額並將數據保存到 `c4` 寄存器。

```func
() on_bounce (slice in_msg_body) impure {
  in_msg_body~skip_bits(32); ;; 0xFFFFFFFF
  (int balance, slice owner_address, slice jetton_master_address, cell jetton_wallet_code) = load_data();
  int op = in_msg_body~load_uint(32);
  throw_unless(709, (op == op::internal_transfer()) | (op == op::burn_notification()));
  int query_id = in_msg_body~load_uint(64);
  int jetton_amount = in_msg_body~load_coins();
  balance += jetton_amount;
  save_data(balance, owner_address, jetton_master_address, jetton_wallet_code);
}
```

#### recv_internal()

為了使我們的錢包接收消息，我們將使用外部方法 `recv_internal()`

```func
() recv_internal()  {

}
```

##### 外部方法參數

根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文檔，當 TON 鏈中的某個賬戶發生事件時，會觸發一筆交易。

每筆交易由最多 5 個階段組成。更多信息請參見 [這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)。

我們感興趣的是 **計算階段（Compute phase）**。具體來說，當初始化時，「堆疊」上的內容。對於正常的消息觸發的交易，堆疊的初始狀態如下：

5 個元素：
- 智能合約餘額（以 nanoTons 計）
- 進入消息的餘額（以 nanoTons 計）
- 包含進入消息的 cell
- 進入消息正文，slice 類型
- 函數選擇器（對於 recv_internal 是 0）

結果我們得到以下代碼：

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body)  {

}
```

##### 從消息正文中獲取數據

所以在 `recv_internal()` 中的第一件事是檢查消息是否為空：

```func
if (in_msg_body.slice_empty?()) { ;; ignore empty messages
  return ();
}
```

接下來，我們獲取標誌並檢查進入消息是否為 bounced。如果是 bounce，我們將使用先前編寫的 `on_bounce()` 函數。

```func
slice cs = in_msg_full.begin_parse();
int flags = cs~load_uint(4);
if (flags & 1) {
  on_bounce(in_msg_body);
  return ();
}
```

之後，我們繼續獲取數據（評論揭示了這是什麼），包括 `op`。通過 `op`，我們將構建進一步的邏輯。

```func
slice sender_address = cs~load_msg_addr();
cs~load_msg_addr(); ;; skip dst
cs~load_coins(); ;; skip value
cs~skip_bits(1); ;; skip extracurrency collection
cs~load_coins(); ;; skip ihr_fee
int fwd_fee = cs~load_coins(); ;; we use message fwd_fee for estimation of forward_payload costs

int op = in_msg_body~load_uint(32);
```

使用條件語句，我們圍繞 `op` 構建邏輯並使用我們編寫的函數來實現內部邏輯。

```func
if (op == op::transfer()) { ;; outgoing transfer
  send_tokens(in_msg_body, sender_address, msg_value, fwd_fee);
  return ();
}

if (op == op::internal_transfer()) { ;; incoming transfer
  receive_tokens(in_msg_body, sender_address, my_balance, fwd_fee, msg_value);
  return ();
}

if (op == op::burn()) { ;; burn
  burn_tokens(in_msg_body, sender_address, msg_value, fwd_fee);
  return ();
}
```

最後有一個異常，即如果合約不根據 `op` 執行某些操作，將引發異常。最終的 `recv_internal()` 代碼：

```func
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
  if (in_msg_body.slice_empty?()) { ;; ignore empty messages
	return ();
  }

  slice cs = in_msg_full.begin_parse();
  int flags = cs~load_uint(4);
  if (flags & 1) {
	on_bounce(in_msg_body);
	return ();
  }
  slice sender_address = cs~load_msg_addr();
  cs~load_msg_addr(); ;; skip dst
  cs~load_coins(); ;; skip value
  cs~skip_bits(1); ;; skip extracurrency collection
  cs~load_coins(); ;; skip ihr_fee
  int fwd_fee = cs~load_coins(); ;; we use message fwd_fee for estimation of forward_payload costs

  int op = in_msg_body~load_uint(32);

  if (op == op::transfer()) { ;; outgoing transfer
	send_tokens(in_msg_body, sender_address, msg_value, fwd_fee);
	return ();
  }

  if (op == op::internal_transfer()) { ;; incoming transfer
	receive_tokens(in_msg_body, sender_address, my_balance, fwd_fee, msg_value);
	return ();
  }

  if (op == op::burn()) { ;; burn
	burn_tokens(in_msg_body, sender_address, msg_value, fwd_fee);
	return ();
  }

  throw(0xffff);
}
```

#### Get 方法

根據 [Jetton](https://github.com/ton-blockchain/TIPs/issues/74) 標準，錢包智能合約必須實現一個 Get 方法，返回：

- `balance` -（uint256）錢包中的代幣數量。
- `owner` -（MsgAddress）錢包擁有者地址；
- `jetton` -（MsgAddress）主合約地址；
- `jetton_wallet_code` -（cell）該錢包的代碼；

即從 `c4` 中卸載數據：

```func
(int, slice, slice, cell) get_wallet_data() method_id {
  return load_data();
}
```

## jetton-minter-ICO.fc

這個文件是主合約的一個變體，適用於您希望進行 ICO 的情況。

> ICO（Initial Coin Offering）- 代幣初次發行，一種通過向投資者出售固定數量的新加密貨幣/代幣單位來吸引投資的形式。

與 `jetton-minter.fc` 唯一顯著的區別是，您可以通過向合約發送 TON 為自己獲取代幣。

此外，`jetton-minter.fc` 中的可選 `op` 被移除了。

##### 理解 recv_internal() 中的 ICO 機制

進入消息的餘額（以 nanoTons 計）是 `msg_value`。從中我們將減去一小部分 nanoTons 作為鑄幣消息，並將剩餘的值按某種比例兌換為 Jetton 標準代幣。

檢查消息正文是否為空：

```func
if (in_msg_body.slice_empty?()) { ;; buy jettons for Toncoin
}
```

計算將兌換為 Jetton 標準代幣的 nanoTons 數量：

```func
int amount = 10000000; ;; for mint message
int buy_amount = msg_value - amount;
```

檢查結果不是負值，如果是負值則引發異常：

```func
throw_unless(76, buy_amount > 0);
```

設置比例：

```func
int jetton_amount = buy_amount; ;; rate 1 jetton = 1 toncoin; multiply to price here
```

接下來，為 `mint_tokens()` 函數組合消息：

```func
var master_msg = begin_cell()
	.store_uint(op::internal_transfer(), 32)
	.store_uint(0, 64) ;; query_id
	.store_coins(jetton_amount)
	.store_slice(my_address()) ;; from_address
	.store_slice(sender_address) ;; response_address
	.store_coins(0) ;; no forward_amount
	.store_uint(0, 1) ;; forward_payload in this slice, not separate cell
	.end_cell();
```

我們調用鑄幣函數：

```func
mint_tokens(sender_address, jetton_wallet_code, amount, master_msg);
```

並保存數據到 `c4` 寄存器，改變 Jetton 標準代幣的總供應量。最後完成 `recv_internal()` 函數的執行。

```func
save_data(total_supply + jetton_amount, admin_address, content, jetton_wallet_code);
return ();
```