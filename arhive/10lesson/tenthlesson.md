# 課程 10 NFT 標準

在課程9中，我們討論了代幣的分類，包括不可替代代幣（NFT）和可替代代幣（Fungible Token），以及可替代代幣的標準。本課將討論不可替代代幣，並根據標準分析示例。

## TON 中的 NFT 標準

不可替代代幣是指每個實例都是獨特的（特定），無法用其他類似資產替代。NFT 是一種數位實體證書，具有通過某些機制轉移證書的能力。

[TOT 中的 NFT 標準](https://github.com/ton-blockchain/TIPs/issues/62) 描述了：
- 所有權的變更。
- 將項目合併到集合中的方法。
- 集合公共部分的去重方法。

> 去重 - 消除重複副本和重複數據的方法

就像 Jetton 一樣，NFT 標準也有一個主合約 - 集合合約和集合中單個 NFT 的智能合約。標準中有一個很好的例子：如果您發行包含 10,000 個項目的集合，則需要部署 10,001 個智能合約，一個集合合約和 10,000 個每個項目的合約。

> NFT 標準還解釋了為什麼選擇這種 NFT 實現方案，包含如此多的合約，每個項目都有其合理性和下一個邏輯。

在 TON 中有一些針對 [NFT 標準](https://github.com/ton-blockchain/TIPs/issues/62) 的擴展（截至 2022 年 7 月 29 日，其中一些處於草案狀態）：
- [NFTRoyalty](https://github.com/ton-blockchain/TIPs/issues/66) - 關於如何獲取版稅信息並在所有 NFT 市場和生態系統成員之間提供通用的版稅支付支持。
- [NFTBounceable](https://github.com/ton-blockchain/TIPs/issues/67) - 如果接收者駁回通知，則回滾 NFT 轉移的方法。（例如，如果 NFT 被發送到錯誤地址且接收者的智能合約不知道如何與 NFT 交互。）
- [NFTEditable](https://github.com/ton-blockchain/TIPs/issues/68) - 關於 NFT 批量更改。
- [NFTUpgradable](https://github.com/ton-blockchain/TIPs/issues/69) - 關於可擴展的 NFT。

#### 根據 NFT 標準的合約功能

標準描述了兩個關鍵的 NFT 智能合約：
- 集合智能合約
- 單個 NFT 的智能合約

> 在 [示例](https://github.com/ton-blockchain/token-contract/tree/main/nft) 中，還有一個實現銷售和某種市場的智能合約，但在本課中我們不會分析這些合約，我們將專注於 NFT 標準。

集合智能合約應實現：
- 部署（deploy）此集合的 NFT 元素的智能合約。（在我們將要分析的示例中，將有單個 NFT 部署和批量 NFT 部署）
- Get-method `get_collection_data()`，返回集合擁有者的地址、集合的內容以及當前集合中的 NFT 計數器
- Get-method `get_nft_address_by_index(int ​​index)`，根據此集合的 NFT 元素的數量返回此 NFT 元素的智能合約的地址（`MsgAddress`）
- Get-method `get_nft_content(int index, cell individual_content)`，返回集合中特定 NFT 的信息

單個 NFT 的智能合約應實現：
- Get-method `get_nft_data()`，返回此 NFT 的數據
- NFT 所有權轉移
- 內部方法 `get_static_data`，通過內部消息獲取特定 NFT 的數據。

> 重要：標準還描述了許多關於手續費、限制等方面的細節，但我們不會過多地討論這些細節，以免本課變成一本書。

#### NFT 標準的元數據

- `uri` - 可選參數，指向包含元數據的 JSON 文檔的鏈接。
- `name` - NFT 標識符字符串，即識別資產。
- `description` - 資產描述。
- `image` - 指向 MIME 類型圖像資源的 URI。
- `image_data` - 網頁佈局的圖像二進制表示，或離線佈局的 base64。

## 解析代碼

在解析代碼之前，我要注意到一般的「機制」是重複的，因此分析越深入，分析越高層次。

我們將按照以下順序解析 [repository](https://github.com/ton-blockchain/token-contract/tree/main/nft) 中的文件：

- nft-collection.fc
- nft-item.fc

## nft-collection.fc

集合合約以兩個輔助函數開始，用於加載和卸載數據。

##### 從 c4 加載和卸載數據

 「集合合約存儲」將存儲：
 
- `owner_address` - 集合擁有者的地址，如果沒有擁有者，則為零地址
- `next_item_index` - 當前部署的 NFT 項目的數量*。
- `content` - 按照 [token](https://github.com/ton-blockchain/TIPs/issues/64) 標準的格式存儲的集合內容。
- `nft_item_code` - 單個 NFT 的代碼，將用於「再現」智能合約的地址。
- `royalty_params` - 版稅參數

> * - 如果 `next_item_index` 的值為 -1，則這是個不一致的集合，這樣的集合必須提供自己的項目索引/枚舉方式。

讓我們編寫輔助函數 `load_data()` 和 `save_data()`，用於從寄存器 c4 卸載和加載數據。（我們不會詳細分析加載和卸載，因為類似的功能已在前幾課中多次分析過。）

##### 「再現」函數

在此智能合約中，我們需要再現擁有者地址處單個 NFT 的智能合約地址。為此，我們將使用與 Jetton 示例中相同的「技巧」。

讓我提醒你，如果我們研究如何編譯智能合約的[文檔](https://ton-blockchain.github.io/docs/#/howto/step-by-step?id=_3-compiling-a-new-smart-contract) 。

我們可以看到以下內容：

新智能合約的代碼和數據連接成一個 StateInit 結構（在接下來的行中），計算並輸出新智能合約的地址（等於該 StateInit 結構的哈希），然後生成一個外部消息，其目標地址等於新智能合約的地址。此外部消息包含新智能合約的正確 StateInit 以及非平凡的有效載荷（使用正確的私鑰簽名）。

對我們來說，這意味著我們可以使用 `item_index` 和單個 NFT 的智能合約代碼來獲取單個 NFT 的智能合約地址，我們將組裝 NFT 的 StateInit。

這是可能的，因為 [哈希函數](https://en.wikipedia.org/wiki/Hash_function) 是確定性的，這意味著對於不同的輸入會有不同的哈希，同時對於相同的輸入數據，哈希函數將始終返回一致的哈希。

為此，智能合約有 `calculate_nft_item_state_init()` 和 `calculate_nft_item_address()` 函數：

```func
cell calculate_nft_item_state_init(int item_index, cell nft_item_code) {
  cell data = begin_cell().store_uint(item_index, 64).store_slice(my_address()).end_cell();
  return begin_cell().store_uint(0, 2).store_dict(nft_item_code).store_dict(data).store_uint(0, 1).end_cell();
}

slice calculate_nft_item_address(int wc, cell state_init) {
  return begin_cell().store_uint(4, 3)
					 .store_int(wc, 8)
					 .store_uint(cell_hash(state_init), 256)
					 .end_cell()
					 .begin_parse();
}
```

`calculate_nft_item_state_init()` 函數根據給定的 `item_index` 組裝 StateInit。

`calculate_nft_item_address()` 函數根據 [TL-B schema](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L99) 組裝地址。

> 函數 `cell_hash()` 用於計算哈希 - 它計算 cell 表示的哈希。

##### 部署單個 NFT 的輔助函數

>*Deploy - 將單個 NFT 轉移到網路上的過程

要部署 NFT，我們需要將必要的 NFT 信息發送到智能合約地址，分別為：

- 再現單個 NFT 的智能合約地址
- 通過消息發送信息

智能合約地址：

```func
() deploy_nft_item(int item_index, cell nft_item_code, int amount, cell nft_content) impure {
  cell state_init = calculate_nft_item_state_init(item_index, nft_item_code);
  slice nft_address = calculate_nft_item_address(workchain(), state_init);

}
```

workchain() 是來自 `params.fc` 的輔助函數。它使用 `asm` 關鍵字定義為低級 TVM 原語。

```func
int workchain() asm "0 PUSHINT";
```

數字 0 是基本 workchain。

我們通過消息發送信息：

```func
() deploy_nft_item(int item_index, cell nft_item_code, int amount, cell nft_content) impure {
  cell state_init = calculate_nft_item_state_init(item_index, nft_item_code);
  slice nft_address = calculate_nft_item_address(workchain(), state_init);
  var msg = begin_cell()
			.store_uint(0x18, 6)
			.store_slice(nft_address)
			.store_coins(amount)
			.store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)
			.store_ref(state_init)
			.store_ref(nft_content);
  send_raw_message(msg.end_cell(), 1); ;; pay transfer fees separately, revert on errors
}
```

##### 發送版稅參數的輔助函數

這個輔助函數將在內部消息發送到 `recv_internal()` 的情況下發送靜態版稅數據。

從技術上講，這裡很簡單，我們發送帶有 `op` 代碼 `op::report_royalty_params()` 的消息：

```func
() send_royalty_params(slice to_address, int query_id, slice data) impure inline {
  var msg = begin_cell()
	.store_uint(0x10, 6) ;; nobounce - int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool src:MsgAddress -> 011000
	.store_slice(to_address)
	.store_coins(0)
	.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	.store_uint(op::report_royalty_params(), 32)
	.store_uint(query_id, 64)
	.store_slice(data);
  send_raw_message(msg.end_cell(), 64); ;; carry all the remaining value of the inbound message
}
```
#### recv_internal()

為了使我們的錢包接收消息，我們將使用外部方法 `recv_internal()`

```func
() recv_internal()  {
}
```

我們的集合智能合約的外部方法應實現：
- 發送版稅參數
- 部署單個 NFT
- 批量部署多個 NFT
- 更改擁有者
- 以及檢查工作邏輯的許多例外情況

##### 外部方法參數

根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文檔，當事件發生在 TON 鏈中的某個帳戶上時，它會觸發一個交易。

每個交易最多由 5 個階段組成。詳情參見 [這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)。

我們感興趣的是 **Compute phase**。具體來說，是初始化期間「堆棧」的狀態。對於由正常消息觸發的交易，堆棧的初始狀態如下：

5 個元素：
- 智能合約餘額（以納噸為單位）
- 傳入消息餘額（以納噸為單位）
- 帶有傳入消息的 Cell
- 傳入消息體，slice 類型
- 函數選擇器（對於 recv_internal 它是 0）

結果我們得到以下代碼：

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body)  {
}
```

##### 構建外部方法框架

所以我們在 `recv_internal()` 中做的第一件事是檢查消息是否為空：

```func
if (in_msg_body.slice_empty?()) { ;; ignore empty messages
    return ();
}
```

接下來，我們獲取標誌並檢查傳入的消息是否是被退回的。如果是退回的，我們完成該函數：

```func
slice cs = in_msg_full.begin_parse();
int flags = cs~load_uint(4);

if (flags & 1) { ;; ignore all bounced messages
    return ();
}
```

接下來，我們獲取發送者地址以及 `op` 和 `query_id`：

```func
slice sender_address = cs~load_msg_addr();
int op = in_msg_body~load_uint(32);
int query_id = in_msg_body~load_uint(64);
```

從寄存器 `c4` 卸載數據：

```func
var (owner_address, next_item_index, content, nft_item_code, royalty_params) = load_data();
```

使用先前描述的傳送版稅信息的函數，我們發送此信息：

```func
if (op == op::get_royalty_params()) {
    send_royalty_params(sender_address, query_id, royalty_params.begin_parse());
    return ();
}
```

接下來，將功能僅限於集合的擁有者（NFT 發行等），所以我們檢查地址並在不符合條件時拋出異常：

```func
throw_unless(401, equal_slices(sender_address, owner_address));
```

使用條件語句和 `op` 繼續創建智能合約的邏輯：

```func
if (op == 1) { ;; deploy new nft

}
if (op == 2) { ;; batch deploy of new nfts

}
if (op == 3) { ;; change owner

}
throw(0xffff);
```

最後的異常處理，即如果合約根據 `op` 不執行某些操作，將拋出異常。最終框架 `recv_internal()`：

```func
() recv_internal(cell in_msg_full, slice in_msg_body) impure {
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

	var (owner_address, next_item_index, content, nft_item_code, royalty_params) = load_data();

	if (op == op::get_royalty_params()) {
		send_royalty_params(sender_address, query_id, royalty_params.begin_parse());
		return ();
	}

	throw_unless(401, equal_slices(sender_address, owner_address));

	if (op == 1) { ;; deploy new nft

	}
	if (op == 2) { ;; batch deploy of new nfts

	}
	if (op == 3) { ;; change owner

	}
	throw(0xffff);
}
```

##### op == 1 部署 NFT

我們從消息體中獲取單個 NFT 的索引：

```func
if (op == 1) { ;; deploy new nft
  int item_index = in_msg_body~load_uint(64);

  return();
}
```

檢查索引是否不大於從 c4 卸載的下一個索引：

```func
if (op == 1) { ;; deploy new nft
  int item_index = in_msg_body~load_uint(64);
  throw_unless(402, item_index <= next_item_index);

  }
  return();
}
```

我們將 `is_last` 變量用於檢查，並將 `item_index` 的值更改為 `next_item_index`。
然後我們使用輔助函數來部署 NFT：

```func
if (op == 1) { ;; deploy new nft
  int item_index = in_msg_body~load_uint(64);
  throw_unless(402, item_index <= next_item_index);
  var is_last = item_index == next_item_index;
  deploy_nft_item(item_index, nft_item_code, in_msg_body~load_coins(), in_msg_body~load_ref());
}
```

現在保存數據到寄存器 `c4`，檢查 `is_last`，將 `next_item_index` 計數器加 1，並將數據保存到 `c4`。

```func
if (op == 1) { ;; deploy new nft
  int item_index = in_msg_body~load_uint(64);
  throw_unless(402, item_index <= next_item_index);
  var is_last = item_index == next_item_index;
  deploy_nft_item(item_index, nft_item_code, in_msg_body~load_coins(), in_msg_body~load_ref());
  if (is_last) {
    next_item_index += 1;
    save_data(owner_address, next_item_index, content, nft_item_code, royalty_params);
  }
  return();
}
```

最後，用 `return ()` 結束該函數。

##### op == 2 批量部署 NFT

批量部署僅僅是循環中的 NFT 部署，循環將遍歷字典，數據將從消息體中卸載（簡單來說，是字典的開始）。

> 我們在第七課詳細討論了字典（Hashmaps）的使用

重要的是要注意，在 TON 中「一次性」批量部署是有限制的。在 [TVM](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=tvm-is-stack-machine) 中，一次交易中的輸出操作數必須 `<=255`。

> 讓我提醒你，FunC 有三個 [循環](https://ton-blockchain.github.io/docs/#/func/statements?id=loops)：`repeat`，`until`，`while`

我們創建一個計數器 `counter`，將在循環中使用，並卸載 NFT 列表的鏈接。

```func
if (op == 2) { ;; batch deploy of new nfts
  int counter = 0;
  cell deploy_list = in_msg_body~load_ref();
}
```

接下來，我們必須使用 `udict::delete_get_min(cell dict, int key_len)` 函數 - 計算 `dict` 字典中的最小鍵 `k`，刪除它並返回（dict'，x，k，-1），其中 `dict '` 是修改後的 `dict` 和 x 是與 `k` 關聯的值。如果 `dict` 為空，則返回（dict，null，null，0）。最後一個值為 -1，這是標誌，如果該函數返回修改後的字典，則標誌為 -1，如果沒有，則為 0。我們將使用標誌作為循環條件

所以，我們標示一個循環，並使用 `udict::delete_get_min(cell dict, int key_len)` 來獲取部署的 NFT 值。

```func
if (op == 2) { ;; batch deploy of new nfts
  int counter = 0;
  cell deploy_list = in_msg_body~load_ref();
  do {
    var (item_index, item, f?) = deploy_list~udict::delete_get_min(64);
   
  } until ( ~ f?);
}
```

> ~ - 按位非運算符 ? - 條件運算符

檢查標誌（即有東西要處理），立即之後增加計數器 `counter`，我們之前定義的。我們這樣做是為了檢查批量部署期間的 NFT 單元數量是否超出了 TVM 的限制（如上所述）。

```func
if (op == 2) { ;; batch deploy of new nfts
  int counter = 0;
  cell deploy_list = in_msg_body~load_ref();
  do {
    var (item_index, item, f?) = deploy_list~udict::delete_get_min(64);
    if (f?) {
      counter += 1;
      if (counter >= 250) { ;; Limit due to limits of action list size
        throw(399);
      }
      
  } until ( ~ f?);
}
```

我們還檢查索引是否沒有混亂，即當前索引不大於下一個索引。然後部署 NFT。此外，我們將處理當前 NFT 的編號等於下一個編號的情況，將其加 1。

```func
if (op == 2) { ;; batch deploy of new nfts
  int counter = 0;
  cell deploy_list = in_msg_body~load_ref();
  do {
    var (item_index, item, f?) = deploy_list~udict::delete_get_min(64);
    if (f?) {
      counter += 1;
      if (counter >= 250) { ;; Limit due to limits of action list size
        throw(399);
      }

      throw_unless(403 + counter, item_index <= next_item_index);
      deploy_nft_item(item_index, nft_item_code, item~load_coins(), item~load_ref());
      if (item_index == next_item_index) {
        next_item_index += 1;
      }
    }
  } until ( ~ f?);
}
```

最後，保存數據並結束該函數。最終代碼 `op == 2`。

```func
if (op == 2) { ;; batch deploy of new nfts
  int counter = 0;
  cell deploy_list = in_msg_body~load_ref();
  do {
    var (item_index, item, f?) = deploy_list~udict::delete_get_min(64);
    if (f?) {
      counter += 1;
      if (counter >= 250) { ;; Limit due to limits of action list size
        throw(399);
      }

      throw_unless(403 + counter, item_index <= next_item_index);
      deploy_nft_item(item_index, nft_item_code, item~load_coins(), item~load_ref());
      if (item_index == next_item_index) {
        next_item_index += 1;
      }
    }
  } until ( ~ f?);
  save_data(owner_address, next_item_index, content, nft_item_code, royalty_params);
  return ();
}
```

##### op == 3 更改擁有者

集合智能合約示例提供了更改集合擁有者功能 - 更改地址。它的工作方式如下：
- 我們從消息體中獲取新擁有者的地址，使用 `load_msg_addr()`
- 保存數據到寄存器 `c4` 中的新擁有者

```func
if (op == 3) { ;; change owner
  slice new_owner = in_msg_body~load_msg_addr();
  save_data(new_owner, next_item_index, content, nft_item_code, royalty_params);
  return ();
}
```

#### Get 方法

在我們的示例中，有四個 Get 方法：
- get_collection_data() - 返回有關集合的信息（擁有者地址，按 [Token 標準](https://github.com/ton-blockchain/TIPs/issues/64) 格式存儲的集合元數據，以及 NFT 索引計數）
- get_nft_address_by_index(int ​​index) - 按索引再現 NFT 智能合約
- royalty_params() - 返回版稅參數
- get_nft_content(int index, cell individual_nft_content) - 返回集合中特定 NFT 的信息

> NFT 版稅是在二級市場上每次 NFT 轉手時的版稅

get_collection_data()、get_nft_address_by_index()、get_nft_content() 方法是 TON 中 NFT 標準的必需方法。

##### get_collection_data()

我們從 `c4` 寄存器中獲取擁有者地址、索引（集合中當前部署的 NFT 元素數量）和有關集合的信息，並簡單地返回這些數據。

```func
(int, cell, slice) get_collection_data() method_id {
  var (owner_address, next_item_index, content, _, _) = load_data();
  slice cs = content.begin_parse();
  return (next_item_index, cs~load_ref(), owner_address);
}
```

##### get_nft_address_by_index()

獲取此集合的 NFT 元素的序列號並返回此 NFT 元素的智能合約地址（MsgAddress）。根據 StateInit（已經分析過）再現智能合約的地址。

```func
slice get_nft_address_by_index(int index) method_id {
	var (_, _, _, nft_item_code, _) = load_data();
	cell state_init = calculate_nft_item_state_init(index, nft_item_code);
	return calculate_nft_item_address(workchain(), state_init);
}
```

##### royalty_params()

返回版稅參數。這個功能屬於 NFT 標準的擴展，具體是 [NFTRoyalty](https://github.com/ton-blockchain/TIPs/issues/66)。
`royalty_params()` 返回分子、分母和用於發送版稅的地址。版稅份額是分子/分母。例如，如果分子 = 11 且分母 = 1000，則版稅率為 11/1000 * 100% = 1.1%。分子必須小於分母。

```func
(int, int, slice) royalty_params() method_id {
	 var (_, _, _, _, royalty) = load_data();
	 slice rs = royalty.begin_parse();
	 return (rs~load_uint(16), rs~load_uint(16), rs~load_msg_addr());
}
```

##### get_nft_content()

獲取此集合的 NFT 元素的序列號和此 NFT 元素的個別內容，並返回符合 [TIP-64 標準](https://github.com/ton-blockchain/TIPs/issues/64) 的 NFT 元素的完整內容。

這裡需要注意的是如何返回內容：

```func
  return (begin_cell()
					  .store_uint(1, 8) ;; offchain tag
					  .store_slice(common_content)
					  .store_ref(individual_nft_content)
		  .end_cell());
```

`store_uint(1, 8)` - 類似標籤表示數據離線存儲，有關數據存儲標籤的更多信息可以在代幣標準中找到 - [Content representation](https://github.com/ton-blockchain/TIPs/issues/64)。

完整函數代碼：

```func
cell get_nft_content(int index, cell individual_nft_content) method_id {
  var (_, _, content, _, _) = load_data();
  slice cs = content.begin_parse();
  cs~load_ref();
  slice common_content = cs~load_ref().begin_parse();
  return (begin_cell()
					  .store_uint(1, 8) ;; offchain tag
					  .store_slice(common_content)
					  .store_ref(individual_nft_content)
		  .end_cell());
}
```

## nft-item.fc

單個 NFT 的智能合約從輔助函數開始，用於處理 `c4` 寄存器，讓我們看看單個 NFT 智能合約的「存儲」中會保存什麼。

- `index` - 這個特定 NFT 的索引
- `collection_address` - 此 NFT 所屬的集合智能合約地址。
- `owner_address` - 當前這個 NFT 的擁有者地址
- `content` - 如果 NFT 有集合，則內容是 NFT 個別內容的任何格式，如果 NFT 沒有集合，則內容是 TIP-64 格式的 NFT 內容。

> 可能會有疑問，如果沒有集合，應該在 `collection_address` 和 `index` 中傳遞什麼，在 `collection_address` 中傳遞 addr_none，在 `index` 中傳遞任意但固定的值。

#### 加載數據

這裡我們使用 `store_` 函數來實現：

```func
() store_data(int index, slice collection_address, slice owner_address, cell content) impure {
	set_data(
		begin_cell()
			.store_uint(index, 64)
			.store_slice(collection_address)
			.store_slice(owner_address)
			.store_ref(content)
			.end_cell()
	);
}
```

但卸載數據和 `c4` 將比以前更複雜。

#### 卸載數據

除了從 `c4` 中卸載數據外，我們還會根據 NFT 是否完全初始化並準備好交互傳遞值 0 和 -1。
我們將獲得的值如下：
- 首先卸載 `c4` 中的 index 和 collection_address
- 然後使用 `slice_bits()` 函數檢查剩餘的 `owner_address` 和 `cell content` 中的位數

```func
(int, int, slice, slice, cell) load_data() {
	slice ds = get_data().begin_parse();
	var (index, collection_address) = (ds~load_uint(64), ds~load_msg_addr());
	if (ds.slice_bits() > 0) {
	  return (-1, index, collection_address, ds~load_msg_addr(), ds~load_ref());
	} else {  
	  return (0, index, collection_address, null(), null()); ;; nft not initialized yet
	}
}
```

#### 發送消息的輔助函數 send_msg()

單個 NFT 的智能合約必須支持以下功能：
- NFT 所有權轉移
- 獲取靜態 NFT 數據

根據標準，這兩個功能都涉及發送消息，因此讓我們編寫一個輔助函數來發送消息，該函數將接收：

- `slice to_address` - 發送消息的地址
- `int amount` - 噸的數量
- `int op` - 用於標識操作的 `op` 代碼
- `int query_id` - 用於所有內部請求-響應消息的 query_id。[更多](https://ton-blockchain.github.io/docs/#/howto/smart-contract-guidelines?id=smart-contract-guidelines)
- `builder payload` - 我們希望與消息一起發送的一些有效載荷
- `int send_mode` - 消息發送模式，有關模式的更多詳細信息可以在第三課中找到

發送消息輔助函數的框架：

```func
() send_msg(slice to_address, int amount, int op, int query_id, builder payload, int send_mode) impure inline {

}
```

> 讓我提醒你 `inline` - 表示實際上將代碼插入到每個函數調用的位置。

我們組裝消息，檢查是否有 `builder payload`，當然使用給定的 `mode` 發送消息。

最終代碼：

```func
() send_msg(slice to_address, int amount, int op, int query_id, builder payload, int send_mode) impure inline {
  var msg = begin_cell()
	.store_uint(0x10, 6) ;; nobounce - int_msg_info$0 ihr_disabled:Bool bounce:Bool bounced:Bool src:MsgAddress -> 011000
	.store_slice(to_address)
	.store_coins(amount)
	.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	.store_uint(op, 32)
	.store_uint(query_id, 64);

  if (~ builder_null?(payload)) {
	msg = msg.store_builder(payload);
  }

  send_raw_message(msg.end_cell(), send_mode);
}
```

#### NFT transfer_ownership() 函數

為了實現 NFT 所有權轉移，關鍵事項是：
- 檢查來自 [標準](https://github.com/ton-blockchain/TIPs/issues/62) 的不同條件
- 向新擁有者發送所有權已分配的通知
- 將剩餘的 TON 發回，或者發送到指定的地址（這裡寫作「發回」是為了便於理解）
- 在合約中保存新擁有者

所以該函數將接收：

`int my_balance` - 智能合約的餘額（扣除傳入消息的成本後）（以納噸為單位）。根據 [Compute phase](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=initialization-of-tvm)
`int index` - 單個 NFT 的集合索引
`slice collection_address` - 集合智能合約地址
`slice owner_address` - 擁有者地址
`cell content` - 帶有 NFT 內容的 cell
`slice sender_address` - 更改所有權消息的發送者地址
`int query_id` - 用於所有內部請求-響應消息的 query_id。[更多](https://ton-blockchain.github.io/docs/#/howto/smart-contract-guidelines?id=smart-contract-guidelines)
`slice in_msg_body` - 將在 `recv_internal()` 中剩下的消息體，內部我們需要地址
`int fwd_fees` - 發送到 `recv_internal()` 的消息的交易費用，這裡將用於估計執行轉移操作所需的 TON 值

該函數開始檢查「所有權更改命令發送者」的地址是否等於擁有者地址，即只有當前擁有者可以更改。

```func
throw_unless(401, equal_slices(sender_address, owner_address));
```

現在我們需要解析 `params.fc` 中的 `force_chain()`。

```func
force_chain(to_owner_address);
```

`force_chain` 函數檢查地址是否在 0 號 workchain 中（基本 workchain）。有關地址和編號的更多信息，可以在 [這裡](https://github.com/ton-blockchain/ton/blob/master/doc/LiteClient-HOWTO) 的開始部分找到。讓我們分析 params.fc 中的代碼：

```func
int workchain() asm "0 PUSHINT";

() force_chain(slice addr) impure {
  (int wc, _) = parse_std_addr(addr);
  throw_unless(333, wc == workchain());
}
```

我們將 `workchain()` 輔助函數定義為使用 `asm` 關鍵字的低級 TVM 原語。整數 == 0 我們需要用於比較。

```func
int workchain() asm "0 PUSHINT";
```

要提取所需信息，使用 `parse_std_addr()`。`parse_std_addr()` - 從 `MsgAddressInt` 返回 workchain 和 256 位整數地址。

```func
() force_chain(slice addr) impure {
  (int wc, _) = parse_std_addr(addr);
  throw_unless(333, wc == workchain());
}
```

為了在 workchain 不相等時引發異常，我們將使用 `throw_unless()`。

讓我們回到 nft-item.fc 函數。我們獲取新擁有者的地址，使用 `force_chain()` 函數檢查 workchain，並獲取通知所有權更改的地址。

```func
slice new_owner_address = in_msg_body~load_msg_addr();
force_chain(new_owner_address);
slice response_destination = in_msg_body~load_msg_addr();
```

由於示例中不涉及自定義有效載荷，我們將跳過它，並從 `forward_amount` 體中獲取將發送給新擁有者的 nanoTons 金額。現在函數如下所示：

```func
() transfer_ownership(int my_balance, int index, slice collection_address, slice owner_address, cell content, slice sender_address, int query_id, slice in_msg_body, int fwd_fees) impure inline {
	throw_unless(401, equal_slices(sender_address, owner_address));

	slice new_owner_address = in_msg_body~load_msg_addr();
	force_chain(new_owner_address);
	slice response_destination = in_msg_body~load_msg_addr();
	in_msg_body~load_int(1); ;; this nft don't use custom_payload
	int forward_amount = in_msg_body~load_coins();
}
```

接下來計算發送通知所有權更改後剩餘的 Ton 值。我不會在這裡詳細說明，以免拖長課程，但為了便於理解後面的代碼，建議閱讀 [Transaction fees](https://ton-blockchain.github.io/docs/#/smart-contracts/fees)。還要注意，我們在計算中考慮到地址可以是 `addr_none`。

```func
int rest_amount = my_balance - min_tons_for_storage();
if (forward_amount) {
  rest_amount -= (forward_amount + fwd_fees);
}
int need_response = response_destination.preload_uint(2) != 0; ;; if NOT addr_none: 00
if (need_response) {
  rest_amount -= fwd_fees;
}
```

如果剩餘值小於零，則引發異常：

```func
throw_unless(402, rest_amount >= 0); ;; base nft spends fixed amount of gas, will not check for response
```

現在我們向新擁有者發送通知：

```func
if (forward_amount) {
  send_msg(new_owner_address, forward_amount, op::ownership_assigned(), query_id, begin_cell().store_slice(owner_address).store_slice(in_msg_body), 1);  ;; paying fees, revert on errors
}
```

在檢查 workchain 後，我們將向指定通知地址發送通知。

```func
if (need_response) {
  force_chain(response_destination);
  send_msg(response_destination, rest_amount, op::excesses(), query_id, null(), 1); ;; paying fees, revert on errors
}
```

當然，我們會將變更保存到 `c4` 寄存器。最終結果：

```func
() transfer_ownership(int my_balance, int index, slice collection_address, slice owner_address, cell content, slice sender_address, int query_id, slice in_msg_body, int fwd_fees) impure inline {
	throw_unless(401, equal_slices(sender_address, owner_address));

	slice new_owner_address = in_msg_body~load_msg_addr();
	force_chain(new_owner_address);
	slice response_destination = in_msg_body~load_msg_addr();
	in_msg_body~load_int(1); ;; this nft don't use custom_payload
	int forward_amount = in_msg_body~load_coins();

	int rest_amount = my_balance - min_tons_for_storage();
	if (forward_amount) {
	  rest_amount -= (forward_amount + fwd_fees);
	}
	int need_response = response_destination.preload_uint(2) != 0; ;; if NOT addr_none: 00
	if (need_response) {
	  rest_amount -= fwd_fees;
	}

	throw_unless(402, rest_amount >= 0); ;; base nft spends fixed amount of gas, will not check for response

	if (forward_amount) {
	  send_msg(new_owner_address, forward_amount, op::ownership_assigned(), query_id, begin_cell().store_slice(owner_address).store_slice(in_msg_body), 1);  ;; paying fees, revert on errors
	}
	if (need_response) {
	  force_chain(response_destination);
	  send_msg(response_destination, rest_amount, op::excesses(), query_id, null(), 1); ;; paying fees, revert on errors
	}

	store_data(index, collection_address, new_owner_address, content);
}
```

##### 外部方法參數

根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文檔，當事件發生在 TON 鏈中的某個帳戶上時，它會觸發一個交易。

每個交易最多由 5 個階段組成。詳情參見 [這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)。

我們感興趣的是 **Compute phase**。具體來說，是初始化期間「堆棧」的狀態。對於由正常消息觸發的交易，堆棧的初始狀態如下：

5 個元素：
- 智能合約餘額（以納噸為單位）
- 傳入消息餘額（以納噸為單位）
- 帶有傳入消息的 Cell
- 傳入消息體，slice 類型
- 函數選擇器（對於 recv_internal 它是 0）

結果我們得到以下代碼：

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body)  {
}
```

##### 從消息體中獲取數據

所以我們在 `recv_internal()` 中做的第一件事是檢查消息是否為空：

```func
if (in_msg_body.slice_empty?()) { ;; ignore empty messages
    return ();
}
```

接下來，我們開始解析（讀取）消息：

```func
slice cs = in_msg_full.begin_parse();
```

我們獲取標誌並檢查消息是否未被退回（這裡指的是退回消息）。

```func
int flags = cs~load_uint(4);
if (flags & 1) { ;; ignore all bounced messages
    return ();
}
```

現在跳過我們不需要的值，可以在 [這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/messages) 找到這些值。

```func
cs~load_msg_addr(); ;; skip dst
cs~load_coins(); ;; skip value
cs~skip_bits(1); ;; skip extracurrency collection
cs~load_coins(); ;; skip ihr_fee
```

我們還獲取 `fwd_fee`，稍後將用於計算操作後發回的 TON 值。

現在我們從 `c4` 寄存器中獲取數據，包括 `init`，根據 NFT 是否完全初始化並準備好交互傳遞值 0 和 -1。

如果 NFT 未準備好，檢查消息發送者是否為集合擁有者並初始化該 NFT

```func
(int init?, int index, slice collection_address, slice owner_address, cell content) = load_data();
if (~ init?) {
  throw_unless(405, equal_slices(collection_address, sender_address));
  store_data(index, collection_address, in_msg_body~load_msg_addr(), in_msg_body~load_ref());
  return ();
}
```

接下來，我們獲取 `op` 和 `query_id` 以使用條件運算符構建邏輯：

```func
int op = in_msg_body~load_uint(32);
int query_id = in_msg_body~load_uint(64);
```

第一個 `op` 是所有權轉移，技術上很簡單：調用我們之前聲明的 `transfer_ownership()` 函數並結束執行。

```func
if (op == op::transfer()) {
  transfer_ownership(my_balance, index, collection_address, owner_address, content, sender_address, query_id, in_msg_body, fwd_fee);
  return ();
}
```

第二個 `op` 是獲取靜態數據，因此我們只需發送一條帶有數據的消息：

```func
if (op == op::get_static_data()) {
  send_msg(sender_address, 0, op::report_static_data(), query_id, begin_cell().store_uint(index, 256).store_slice(collection_address), 64);  ;; carry all the remaining value of the inbound message
  return ();
}
```

最後的異常處理，即如果合約根據 `op` 不執行某些操作，將拋出異常。最終 `recv_internal()` 代碼：

```func
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
	if (in_msg_body.slice_empty?()) { ;; ignore empty messages
		return ();
	}

	slice cs = in_msg_full.begin_parse();
	int flags = cs~load_uint(4);

	if (flags & 1) { ;; ignore all bounced messages
		return ();
	}
	slice sender_address = cs~load_msg_addr();

	cs~load_msg_addr(); ;; skip dst
	cs~load_coins(); ;; skip value
	cs~skip_bits(1); ;; skip extracurrency collection
	cs~load_coins(); ;; skip ihr_fee
	int fwd_fee = cs~load_coins(); ;; we use message fwd_fee for estimation of forward_payload costs

	(int init?, int index, slice collection_address, slice owner_address, cell content) = load_data();
	if (~ init?) {
	  throw_unless(405, equal_slices(collection_address, sender_address));
	  store_data(index, collection_address, in_msg_body~load_msg_addr(), in_msg_body~load_ref());
	  return ();
	}

	int op = in_msg_body~load_uint(32);
	int query_id = in_msg_body~load_uint(64);

	if (op == op::transfer()) {
	  transfer_ownership(my_balance, index, collection_address, owner_address, content, sender_address, query_id, in_msg_body, fwd_fee);
	  return ();
	}
	if (op == op::get_static_data()) {
	  send_msg(sender_address, 0, op::report_static_data(), query_id, begin_cell().store_uint(index, 256).store_slice(collection_address), 64);  ;; carry all the remaining value of the inbound message
	  return ();
	}
	throw(0xffff);
}
```

#### Get 方法 get_nft_data()

根據 [標準](https://github.com/ton-blockchain/TIPs/issues/62) 的要求，單個 NFT 的智能合約必須具有一個必需的 Get 方法。

此方法僅返回有關此單個 NFT 的數據，即從 `c4` 中卸載數據：

```func
(int, int, slice, slice, cell) get_nft_data() method_id {
  (int init?, int index, slice collection_address, slice owner_address, cell content) = load_data();
  return (init?, index, collection_address, owner_address, content);
}
```

---
如果有任何需要調整或進一步說明的部分，請告訴我。