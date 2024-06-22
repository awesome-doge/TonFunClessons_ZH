# NFT 銷售與市場

## 簡介

在本課程中，我們將了解如何組織 NFT 的銷售。我們將從官方 [token examples](https://github.com/ton-blockchain/token-contract) 中獲取合約範例，我們感興趣的是：
  - nft-marketplace.fc - 市場合約
  - nft-sale.fc - 特定 NFT 的銷售合約

我們將以以下方式構建本課程，首先我們將高層次地了解智能合約如何運作，然後我們將逐步解讀代碼。我們不會逐字分析代碼，因此如果您對 Func 不熟悉，建議您先參考這些 [lessons](https://github.com/romanovichim/TonFunClessons_Eng)。

## 功能概述

市場智能合約執行一個功能，即初始化/部署銷售智能合約。因此，市場智能合約僅接收包含初始化銷售所需的所有數據的消息，並用該消息初始化銷售智能合約。

智能銷售合約執行三個功能：
- 合約內資金的累積
- 銷售
- 取消銷售

成功銷售或取消後，合約會被「燒毀」。

累積資金是通過接收具有 op == 1 的資金消息來進行的。

取消銷售是通過轉移當前所有者的所有權來實現的，並「燒毀」智能合約。

銷售過程中，我們通過消息向所有者發送 TONcoins，支付市場佣金和版稅，最後發送關於所有權變更的消息並燒毀合約。

現在讓我們看看合約代碼。

### 市場合約

市場智能合約的任務是部署/初始化銷售網絡中的合約。我們將使用已熟悉的 State Init 來完成此操作。智能合約將接收 State Init 消息（代碼和存儲的初始數據），從中提取哈希，從而形成銷售合約的地址，然後向該地址發送消息進行初始化。

讓我們通過代碼進行更詳細的分析。

#### 存儲

我們將市場智能合約所有者的地址存儲在 `c4` 寄存器中，我們需要它來檢查消息的來源，以便只有市場智能合約所有者的地址才能初始化銷售。

為了處理存儲，這個智能合約有兩個輔助函數 `load_data()` 和 `save_data()`，分別用來上傳和保存數據到存儲中。

```func
(slice) load_data() inline {
  var ds = get_data().begin_parse();
  return 
    (ds~load_msg_addr() ;; owner
     );
}

() save_data(slice owner_address) impure inline {
  set_data(begin_cell()
    .store_slice(owner_address)
    .end_cell());
}
```

#### 處理內部消息

讓我們繼續解析 `recv_internal()`。智能合約不會處理空消息，因此我們將使用 `slice_empty()` 檢查並在空消息的情況下使用 `return()` 結束智能合約的執行。

```func
() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) { ;; ignore empty messages
        return ();
    }
}
```

接下來，我們獲取標誌並檢查進入的消息是否為 bounced。如果這是一個 bounce，我們結束智能合約的工作：

```func
() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) { ;; ignore empty messages
        return ();
    }
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) {  ;; ignore all bounced messages
        return ();
    }
}
```
> 有關 bounce 的更多信息，請參見 [這裡第 78 頁](https://ton-blockchain.github.io/docs/tblkch.pdf)

根據市場邏輯，只有市場智能合約的所有者才能初始化銷售合約，因此我們從 `c4` 中獲取發件人的地址並檢查它們是否匹配：

```func
() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) { ;; ignore empty messages
        return ();
    }
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) {  ;; ignore all bounced messages
        return ();
    }
    slice sender_address = cs~load_msg_addr();

    var (owner_address) = load_data();
    throw_unless(401, equal_slices(sender_address, owner_address));

}
```

接下來，我們進行智能銷售合約的初始化，為此我們將獲取 op 並檢查其是否等於 1。
> 提醒一下，使用 op 是 TON 智能合約 [文檔](https://ton-blockchain.github.io/docs/#/howto/smart-contract-guidelines?id=smart-contract-guidelines) 中的建議。

```func
() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) { ;; ignore empty messages
        return ();
    }
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) {  ;; ignore all bounced messages
        return ();
    }
    slice sender_address = cs~load_msg_addr();

    var (owner_address) = load_data();
    throw_unless(401, equal_slices(sender_address, owner_address));
    int op = in_msg_body~load_uint(32);

    if (op == 1) { ;; deploy new auction

    }
}
```

##### Op == 1

我們繼續解析消息正文，獲取將發送到銷售合約的 TONcoin 數量，並獲取銷售合約的 StateInit 和部署的消息正文。

```func
if (op == 1) { ;; deploy new auction
  int amount = in_msg_body~load_coins();
  (cell state_init, cell body) = (cs~load_ref(), cs~load_ref());

}
```

使用 [cell_hash](https://ton-blockchain.github.io/docs/#/func/stdlib?id=cell_hash) 計算 StateInit 哈希並收集銷售合約的地址：

```func
if (op == 1) { ;; deploy new auction
  int amount = in_msg_body~load_coins();
  (cell state_init, cell body) = (cs~load_ref(), cs~load_ref());
  int state_init_hash = cell_hash(state_init);
  slice dest_address = begin_cell().store_int(0, 8).store_uint(state_init_hash, 256).end_cell().begin_parse();
}
```

剩下的就是發送消息，這樣當收到帶有 op == 1 的消息時，市場智能合約將初始化銷售合約。

```func
if (op == 1) { ;; deploy new auction
  int amount = in_msg_body~load_coins();
  (cell state_init, cell body) = (cs~load_ref(), cs~load_ref());
  int state_init_hash = cell_hash(state_init);
  slice dest_address = begin_cell().store_int(0, 8).store_uint(state_init_hash, 256).end_cell().begin_parse();

  var msg = begin_cell()
    .store_uint(0x18, 6)
    .store_uint(4, 3).store_slice(dest_address)
    .store_grams(amount)
    .store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)
    .store_ref(state_init)
    .store_ref(body);
  send_raw_message(msg.end_cell(), 1); ;; paying fees, revert on errors
}
```

市場智能合約的全部代碼如下。

```func
;; NFT marketplace smart contract

;; storage scheme
;; storage#_ owner_address:MsgAddress
;;           = Storage;

(slice) load_data() inline {
  var ds = get_data().begin_parse();
  return 
    (ds~load_msg_addr() ;; owner
     );
}

() save_data(slice owner_address) impure inline {
  set_data(begin_cell()
    .store_slice(owner_address)
    .end_cell());
}

() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
    if (in_msg_body.slice_empty?()) { ;; ignore empty messages
        return ();
    }
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) {  ;; ignore all bounced messages
        return ();
    }
    slice sender_address = cs~load_msg_addr();

    var (owner_address) = load_data();
    throw_unless(401, equal_slices(sender_address, owner_address));
    int op = in_msg_body~load_uint(32);

    if (op == 1) { ;; deploy new auction
      int amount = in_msg_body~load_coins();
      (cell state_init, cell body) = (cs~load_ref(), cs~load_ref());
      int state_init_hash = cell_hash(state_init);
      slice dest_address = begin_cell().store_int(0, 8).store_uint(state_init_hash, 256).end_cell().begin_parse();

      var msg = begin_cell()
        .store_uint(0x18, 6)
        .store_uint(4, 3).store_slice(dest_address)
        .store_grams(amount)
        .store_uint(4 + 2 + 1, 1 + 4 + 4 + 64 + 32 + 1 + 1 + 1)
        .store_ref(state_init)
        .store_ref(body);
      send_raw_message(msg.end_cell(), 1); ;; paying fees, revert on errors
    }
}

() recv_external(slice in_msg) impure {
}
```

### 銷售合約

#### 概述

讓我們看看 `recv_internal()` 中的 `op` 來了解這個智能合約「能做什麼」。
- `op` == 1 - 空 op，只是接收 Toncoin 合約（您可以使用此 op 在合約中累積加密貨幣以備後用）
- `op` == 2 - 購買 NFT - 對於這個 op，有一個輔助函數 buy()，它將發送消息以完成 NFT 購買
- `op` == 3 - 取消銷售

#### 存儲

首先我們將分析合約在 `c4` 寄存器中存儲了什麼（換句話說，存儲）。我們的智能合約有兩個輔助函數 `load_data()` 和 `save_data()`，分別用來上傳和保存數據到存儲中。

在存儲中：
- `slice marketplace_address` - 市場智能合約地址
- `slice nft_address` - 售出的 nft 地址
- `slice nft_owner_address` - nft 所有者地址
- `int full_price` - 價格
- `cell fees_cell` - 包含佣金信息的 cell，例如：市場佣金和版稅

用於處理存儲的函數代碼：

```func
(slice, slice, slice, int, cell) load_data() inline {
  var ds = get_data().begin_parse();
  return 
    (ds~load_msg_addr(), ;; marketplace_address 
      ds~load_msg_addr(), ;; nft_address
      ds~load_msg_addr(),  ;; nft_owner_address
      ds~load_coins(), ;; full_price
      ds~load_ref() ;; fees_cell
     );
}

() save_data(slice marketplace_address, slice nft_address, slice nft_owner_address, int full_price, cell fees_cell) impure inline {
  set_data(begin_cell()
    .store_slice(marketplace_address)
    .store_slice(nft_address)
    .store_slice(nft_owner_address)
    .store_coins(full_price)
    .store_ref(fees_cell)
    .end_cell());
}
```

#### 處理內部消息

和市場智能合約一樣，銷售智能合約也從卸載標誌開始，並檢查消息是否為 bounced。

```func
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) {  ;; ignore all bounced messages
        return ();
    }

}
```

接下來，我們上傳發送消息到智能合約的發件人地址，以及 `c4` 寄存器中的數據（存儲）。

```func
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) {  ;; ignore all bounced messages
        return ();
    }

    slice sender_address = cs~load_msg_addr();

    var (marketplace_address, nft_address, nft_owner_address, full_price, fees_cell) = load_data();
}
```

##### 未初始化的 NFT

在進行銷售和取消銷售「拍賣」的邏輯之前，我們需要處理未初始化的 NFT 的情況。要了解 NFT 是否已初始化，請檢查所有者的地址是否為零。然後，使用波浪號進行檢查（`~` 是 FUNC 中的位元取反運算符）。

```func
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    slice cs = in_msg_full.begin_parse();
    int flags = cs~load_uint(4);

    if (flags & 1) {  ;; ignore all bounced messages
        return ();
    }

    slice sender_address = cs~load_msg_addr();

    var (marketplace_address, nft_address, nft_owner_address, full_price, fees_cell) = load_data();

    var is_initialized = nft_owner_address.slice_bits() > 2; ;; not initialized if null address

    if (~ is_initialized) {


    }
}
```

我立即指出，在未初始化的 NFT 情況下，我們只會接收來自 NFT 地址的消息，並且只包含 op 表示所有權轉移的消息，因此在銷售合約中處理未初始化的 NFT 主要是確定所有者。但是讓我們按順序進行：

如果消息是從市場發送的，我們僅在合約中累積 Toncoins（例如，在合約部署時）。

```func
if (~ is_initialized) {

  if (equal_slices(sender_address, marketplace_address)) {
     return (); ;; just accept coins on deploy
  }

}
```

接下來，我們檢查消息是否來自 NFT 合約，並且檢查該消息的 `op` 是否等於 `ownership_assigned`，即這是一個所有權變更的消息。

```func
if (~ is_initialized) {

  if (equal_slices(sender_address, marketplace_address)) {
     return (); ;; just accept coins on deploy
  }

  throw_unless(500, equal_slices(sender_address, nft_address));
  int op = in_msg_body~load_uint(32);
  throw_unless(501, op == op::ownership_assigned());

}
```

只需獲取地址並保存有關所有權變更的信息。

```func
if (~ is_initialized) {

  if (equal_slices(sender_address, marketplace_address)) {
     return (); ;; just accept coins on deploy
  }

  throw_unless(500, equal_slices(sender_address, nft_address));
  int op = in_msg_body~load_uint(32);
  throw_unless(501, op == op::ownership_assigned());
  int query_id = in_msg_body~load_uint(64);
  slice prev_owner_address = in_msg_body~load_msg_addr();

  save_data(marketplace_address, nft_address, prev_owner_address, full_price, fees_cell);

  return ();

}
```

##### 空消息體

在這個銷售合約範例中，還有一種情況是進入合約的消息體為空，在這種情況下，合約只會嘗試通過調用輔助函數 `buy()` 進行購買。

```func
if (in_msg_body.slice_empty?()) {
    buy(my_balance, marketplace_address, nft_address, nft_owner_address, full_price, fees_cell, msg_value, sender_address, 0);
    return ();
}
```

處理空消息的情況後，我們獲取 `op` 和 `query_id`。我們將使用 `op` 構建邏輯，最後我們將添加一個錯誤調用，以應對接收到的「不明」消息：

```func
int op = in_msg_body~load_uint(32);
int query_id = in_msg_body~load_uint(64);

if (op == 1) { 
    ;; 累積合約中的 TONCoins
    return ();
}

if (op == 2) { 
     ;; 購買 NFT
    return ();
}

if (op == 3) { 
    ;; 取消銷售
    return ();
}

throw(0xffff);
```

##### 購買

購買過程中，會編譯一個單獨的輔助函數，我們會在 `recv_internal()` 中調用它。

```func
if (op == 2) { ;; buy
 
  buy(my_balance, marketplace_address, nft_address, nft_owner_address, full_price, fees_cell, msg_value, sender_address, query_id);

  return ();

}
```

在進行銷售之前，首先要檢查消息中是否有足夠的資金。為此，您需要檢查是否有足夠的資金來支付價格以及與發送消息相關的佣金。

我們將定義一個 `min_gas_amount()` 函數，該函數將儲存一個 1 TON 的值進行驗證，該函數定義為一個低級 TVM 原語，使用 `asm` 關鍵字。

```func
int min_gas_amount() asm "1000000000 PUSHINT"; ;; 1 TON
```

我們將進行檢查，同時立即上傳有關版稅的信息，為此有一個單獨的輔助函數：

```func
() buy(int my_balance, slice marketplace_address, slice nft_address, slice nft_owner_address, int full_price, cell fees_cell, int msg_value, slice sender_address, int query_id) impure {
  throw_unless(450, msg_value >= full_price + min_gas_amount());

  var (marketplace_fee, royalty_address, royalty_amount) = load_fees(fees_cell);

}

(int, slice, int) load_fees(cell fees_cell) inline {
  var ds = fees_cell.begin_parse();
  return 
    (ds~load_coins(), ;; marketplace_fee,
      ds~load_msg_addr(), ;; royalty_address 
      ds~load_coins() ;; royalty_amount
     );
}
```

讓我們繼續發送消息。我們將向當前的 NFT 所有者發送第一個消息，將 TONcoins 轉移給他。數量應等於：NFT 價格減去市場佣金和版稅，以及智能合約的剩餘餘額，例如，如果購買是通過多個消息進行的，並且合約中已經有 TONcoins。

```func
() buy(int my_balance, slice marketplace_address, slice nft_address, slice nft_owner_address, int full_price, cell fees_cell, int msg_value, slice sender_address, int query_id) impure {
  throw_unless(450, msg_value >= full_price + min_gas_amount());

  var (marketplace_fee, royalty_address, royalty_amount) = load_fees(fees_cell);

  var owner_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(nft_owner_address)
           .store_coins(full_price - marketplace_fee - royalty_amount + (my_balance - msg_value))
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(owner_msg.end_cell(), 1);

  }
```

然後我們發送版稅和市場佣金，這裡很簡單，相應的金額將發送到版稅和市場地址：

```func
() buy(int my_balance, slice marketplace_address, slice nft_address, slice nft_owner_address, int full_price, cell fees_cell, int msg_value, slice sender_address, int query_id) impure {
  throw_unless(450, msg_value >= full_price + min_gas_amount());

  var (marketplace_fee, royalty_address, royalty_amount) = load_fees(fees_cell);

  var owner_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(nft_owner_address)
           .store_coins(full_price - marketplace_fee - royalty_amount + (my_balance - msg_value))
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(owner_msg.end_cell(), 1);


  var royalty_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(royalty_address)
           .store_coins(royalty_amount)
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(royalty_msg.end_cell(), 1);


  var marketplace_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(marketplace_address)
           .store_coins(marketplace_fee)
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(marketplace_msg.end_cell(), 1);

}
```

剩下的就是發送最後一條消息，即關於 NFT 轉移合約的消息（帶有 `op::transfer()`）

```func
() buy(int my_balance, slice marketplace_address, slice nft_address, slice nft_owner_address, int full_price, cell fees_cell, int msg_value, slice sender_address, int query_id) impure {
  throw_unless(450, msg_value >= full_price + min_gas_amount());

  var (marketplace_fee, royalty_address, royalty_amount) = load_fees(fees_cell);

  var owner_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(nft_owner_address)
           .store_coins(full_price - marketplace_fee - royalty_amount + (my_balance - msg_value))
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(owner_msg.end_cell(), 1);


  var royalty_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(royalty_address)
           .store_coins(royalty_amount)
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(royalty_msg.end_cell(), 1);


  var marketplace_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(marketplace_address)
           .store_coins(marketplace_fee)
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(marketplace_msg.end_cell(), 1);

  var nft_msg = begin_cell()
           .store_uint(0x18, 6) 
           .store_slice(nft_address)
           .store_coins(0)
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
           .store_uint(op::transfer(), 32)
           .store_uint(query_id, 64)
           .store_slice(sender_address) ;; new_owner_address
           .store_slice(sender_address) ;; response_address
           .store_int(0, 1) ;; empty custom_payload
           .store_coins(0) ;; forward amount to new_owner_address
           .store_int(0, 1); ;; empty forward_payload


  send_raw_message(nft_msg.end_cell(), 128 + 32);
}
```

一切似乎都已完成，但值得一提的是我們發送最後一條消息時使用的模式。

###### 「燒毀合約」模式 == 128 + 32

發送關於 NFT 轉移的消息後，銷售智能合約不再相關，問題是如何「銷毀」它，或者換句話說，「燒毀」它。TON 有一個消息發送模式，它會銷毀當前的合約。

`mode' = mode + 32` 表示如果結果餘額為零，則應銷毀當前賬戶。([文檔鏈接](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message))

因此，在 `buy()` 函數的最後，我們發送關於所有權變更的消息並燒毀此銷售合約。

`buy()` 函數的最終代碼：

```func
() buy(int my_balance, slice marketplace_address, slice nft_address, slice nft_owner_address, int full_price, cell fees_cell, int msg_value, slice sender_address, int query_id) impure {
  throw_unless(450, msg_value >= full_price + min_gas_amount());

  var (marketplace_fee, royalty_address, royalty_amount) = load_fees(fees_cell);

  var owner_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(nft_owner_address)
           .store_coins(full_price - marketplace_fee - royalty_amount + (my_balance - msg_value))
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(owner_msg.end_cell(), 1);


  var royalty_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(royalty_address)
           .store_coins(royalty_amount)
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(royalty_msg.end_cell(), 1);


  var marketplace_msg = begin_cell()
           .store_uint(0x10, 6) ;; nobounce
           .store_slice(marketplace_address)
           .store_coins(marketplace_fee)
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1);

  send_raw_message(marketplace_msg.end_cell(), 1);

  var nft_msg = begin_cell()
           .store_uint(0x18, 6) 
           .store_slice(nft_address)
           .store_coins(0)
           .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
           .store_uint(op::transfer(), 32)
           .store_uint(query_id, 64)
           .store_slice(sender_address) ;; new_owner_address
           .store_slice(sender_address) ;; response_address
           .store_int(0, 1) ;; empty custom_payload
           .store_coins(0) ;; forward amount to new_owner_address
           .store_int(0, 1); ;; empty forward_payload


  send_raw_message(nft_msg.end_cell(), 128 + 32);
}
```

##### 取消銷售

取消只是一條將所有權從當前所有者轉移到當前所有者的消息，帶有 `mode == 128 + 32`，以便稍後銷毀合約。但當然，首先需要檢查一些條件。

首先檢查我們是否有足夠的 TONcoin 發送消息

```func
if (op == 3) { ;; cancel sale
     throw_unless(457, msg_value >= min_gas_amount());

    return ();
}
```

第二，檢查取消銷售的消息是否來自市場或 NFT 所有者。為此，我們使用位元或 `|`。

```func
if (op == 3) { ;; cancel sale
     throw_unless(457, msg_value >= min_gas_amount());
     throw_unless(458, equal_slices(sender_address, nft_owner_address) | equal_slices(sender_address, marketplace_address));

    return ();
}
```

最後，發送一條將所有權從所有者轉移到所有者的消息）

```func
if (op == 3) { ;; cancel sale
     throw_unless(457, msg_value >= min_gas_amount());
     throw_unless(458, equal_slices(sender_address, nft_owner_address) | equal_slices(sender_address, marketplace_address));

     var msg = begin_cell()
       .store_uint(0x10, 6) ;; nobounce
       .store_slice(nft_address)
       .store_coins(0)
       .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
       .store_uint(op::transfer(), 32)
       .store_uint(query_id, 64) 
       .store_slice(nft_owner_address) ;; new_owner_address
       .store_slice(nft_owner_address) ;; response_address;
       .store_int(0, 1) ;; empty custom_payload
       .store_coins(0) ;; forward amount to new_owner_address
       .store_int(0, 1); ;; empty forward_payload

    send_raw_message(msg.end_cell(), 128 + 32);

    return ();
}
```

## 結論

我在 [Telegram 頻道](https://t.me/ton_learn) 上發布類似的分析和教程，歡迎訂閱。